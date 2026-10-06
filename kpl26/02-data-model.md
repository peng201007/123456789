# 2. 数据库表结构设计

PostgreSQL 16 · 字符集 UTF8 · 时区统一 UTC 存储

## 2.0 全局约定

| 约定 | 说明 |
|---|---|
| 主键 | `BIGSERIAL`（内部）或 `UUID v7`（对外暴露）。对外 ID 用 UUID v7 避免枚举攻击且保持时间有序 |
| 时间戳 | 全部 `TIMESTAMPTZ`，默认 `now()`；`created_at` / `updated_at` 必备 |
| 软删除 | `deleted_at TIMESTAMPTZ NULL`，业务查询一律带 `deleted_at IS NULL` |
| 命名 | 表名小写复数下划线；索引 `idx_<表>_<列>`；外键 `fk_<表>_<列>` |
| 枚举 | 用 PostgreSQL `ENUM` 类型或 `CHECK` 约束，避免自由字符串 |
| 隐私 | 不存明文 IP、不存明文手机号；IP 存 HMAC-SHA256 哈希；另存 **/24 网络前缀明文**用于向用户展示"大致登录地"（见下方说明） |

**隐私最小化落地**：
- `users` 表不存真实姓名、生日、地址——除非业务强必需且用户明示同意
- IP 仅存哈希值（用于风控与去重），不可逆推完整地址
- 向用户展示登录来源时，使用**独立的 /24 前缀列**（如 `203.0.113.0/24`），不返回完整 IP

> **关于 /24 前缀的取舍**：`GET /auth/sessions` 需要告诉用户"有个陌生设备从哪登录"，若只给哈希值用户无法判断。因此单独存储 /24 前缀。必须清楚：**/24 不构成匿名化**（可定位到具体 ISP 与大致城区），它是"可向本人展示的准标识符"。对应措施：该列仅本人与持有 `user:read_pii` 的管理员可见；用户注销时随记录一并清除；不用于任何跨用户关联分析。

---

## 2.1 Users — 用户表

```sql
CREATE TYPE user_role AS ENUM ('USER', 'MODERATOR', 'ADMIN', 'SUPER_ADMIN');
CREATE TYPE user_status AS ENUM ('ACTIVE', 'SUSPENDED', 'BANNED', 'PENDING_VERIFICATION', 'DELETED');

CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),  -- UUID v7
    email               CITEXT NOT NULL,
    email_verified_at   TIMESTAMPTZ,

    -- 凭据：绝不存明文，仅存 Argon2id 编码串
    password_hash       TEXT NOT NULL,          -- $argon2id$v=19$m=...,t=...,p=...$salt$hash
    password_changed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    password_algo       TEXT NOT NULL DEFAULT 'argon2id',

    -- 角色与状态
    role                user_role   NOT NULL DEFAULT 'USER',
    status              user_status NOT NULL DEFAULT 'PENDING_VERIFICATION',

    -- 管理方特权：必须完成双因子才允许激活
    privileged_activated_at TIMESTAMPTZ,
    mfa_enabled         BOOLEAN NOT NULL DEFAULT FALSE,

    -- 配额
    upload_quota_bytes_daily  BIGINT NOT NULL DEFAULT 2147483648,   -- 2 GiB
    upload_quota_count_daily  INTEGER NOT NULL DEFAULT 5,
    max_file_size_bytes       BIGINT NOT NULL DEFAULT 2147483648,

    -- 风控
    failed_login_count  SMALLINT NOT NULL DEFAULT 0,
    locked_until        TIMESTAMPTZ,
    last_login_at       TIMESTAMPTZ,
    last_login_ip_hash  BYTEA,                  -- HMAC 哈希，非明文

    -- 同意记录（GDPR 要求可证明）
    terms_accepted_at   TIMESTAMPTZ,
    privacy_policy_version TEXT,

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at          TIMESTAMPTZ,

    CONSTRAINT chk_email_format CHECK (email ~* '^[^@\s]+@[^@\s]+\.[^@\s]+$'),
    CONSTRAINT chk_quota_positive CHECK (upload_quota_bytes_daily >= 0 AND upload_quota_count_daily >= 0)
);

-- 邮箱唯一：仅对未删除用户生效，允许已注销邮箱重新注册
CREATE UNIQUE INDEX idx_users_email_active
    ON users (email) WHERE deleted_at IS NULL;

-- ⚠️ 不要为 id 再建唯一索引：id 已是主键，主键自带唯一索引。
-- 登录查询走 email（见 03 节 3.2.2），由上方 idx_users_email_active 服务。

-- 账户锁定扫描（定时任务：解锁到期账户）
CREATE INDEX idx_users_locked_until
    ON users (locked_until) WHERE locked_until IS NOT NULL;

-- 管理后台按角色+状态筛选
CREATE INDEX idx_users_role_status
    ON users (role, status) WHERE deleted_at IS NULL;

-- 风控：按注册时间分析批量注册行为
CREATE INDEX idx_users_created_at ON users (created_at DESC);
```

### 索引策略说明

| 索引 | 服务的查询 | 为何必要 |
|---|---|---|
| `idx_users_email_active` | 登录、注册查重 | 唯一索引同时保证约束与查询性能；条件唯一允许注销后复用邮箱 |
| `idx_users_locked_until` | 每 5 分钟扫描解锁 | 部分索引只含被锁账户，索引极小 |
| `idx_users_role_status` | 管理后台筛选管理员 | 管理员数量少，部分索引避免为普通用户建条目 |
| `idx_users_created_at` | 反作弊分析 | 降序索引直接支持"最近注册"查询 |

> **不建索引的字段**：`password_hash` 永不用于查询，只按主键取回。

---

## 2.2 Permissions — 权限与角色表（RBAC）

采用**标准 RBAC**：角色 → 权限多对多，用户 → 角色多对多。权限字符串用 `资源:动作` 格式，便于代码中做声明式校验。

```sql
CREATE TABLE permissions (
    id          SERIAL PRIMARY KEY,
    code        TEXT NOT NULL,          -- 'content:upload' | 'content:audit' | 'user:ban'
    resource    TEXT NOT NULL,          -- 'content'
    action      TEXT NOT NULL,          -- 'upload'
    description TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT uq_permissions_code UNIQUE (code)
);

CREATE INDEX idx_permissions_resource ON permissions (resource);

CREATE TABLE roles (
    id          SERIAL PRIMARY KEY,
    code        TEXT NOT NULL,          -- 'USER' | 'MODERATOR' | 'ADMIN'
    name        TEXT NOT NULL,
    is_system   BOOLEAN NOT NULL DEFAULT FALSE,  -- 系统角色不可删除
    priority    SMALLINT NOT NULL DEFAULT 0,     -- 数值大者权限高
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT uq_roles_code UNIQUE (code)
);

CREATE TABLE role_permissions (
    role_id       INTEGER NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id INTEGER NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    granted_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by    UUID REFERENCES users(id),
    PRIMARY KEY (role_id, permission_id)
);

-- 反查：某权限被哪些角色持有
CREATE INDEX idx_role_permissions_permission ON role_permissions (permission_id);

CREATE TABLE user_roles (
    user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id     INTEGER NOT NULL REFERENCES roles(id) ON DELETE RESTRICT,
    granted_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by  UUID REFERENCES users(id),
    -- 特权角色的授予必须有到期时间，强制定期复核
    expires_at  TIMESTAMPTZ,
    PRIMARY KEY (user_id, role_id)
);

CREATE INDEX idx_user_roles_role ON user_roles (role_id);
CREATE INDEX idx_user_roles_expires ON user_roles (expires_at) WHERE expires_at IS NOT NULL;
```

### 权限种子数据

```sql
INSERT INTO roles (code, name, is_system, priority) VALUES
    ('USER',        '普通用户',   TRUE, 10),
    ('MODERATOR',   '审核员',     TRUE, 50),
    ('ADMIN',       '管理员',     TRUE, 80),
    ('SUPER_ADMIN', '超级管理员', TRUE, 100);

INSERT INTO permissions (code, resource, action, description) VALUES
    ('content:upload',        'content', 'upload',     '上传内容'),
    ('content:upload_unlimited','content','upload_unlimited','不受配额限制上传'),
    ('content:upload_batch',  'content', 'upload_batch','批量上传'),
    ('content:audit',         'content', 'audit',      '人工复审'),
    ('content:delete_any',    'content', 'delete_any', '删除任意内容'),
    ('user:ban',              'user',    'ban',        '封禁账户'),
    ('user:read_pii',         'user',    'read_pii',   '查看用户隐私数据'),
    ('system:config_audit',   'system',  'config_audit','调整审核阈值'),
    ('audit:read_logs',       'audit',   'read_logs',  '查阅审核日志');
```

**角色-权限映射（关键项）**：

| 角色 | 持有权限 |
|---|---|
| USER | `content:upload` |
| MODERATOR | USER + `content:audit`, `audit:read_logs` |
| ADMIN | MODERATOR + `content:upload_unlimited`, `content:upload_batch`, `user:ban`, `content:delete_any` |
| SUPER_ADMIN | ADMIN + `system:config_audit`, `user:read_pii` |

**权限判定**：应用启动时把 `role_permissions` 全量载入 Redis（key: `rbac:role:{roleCode}`），令牌校验时一次性取出用户所有权限集合，避免每次请求 join 数据库。变更权限时发布 Redis Pub/Sub 让所有实例刷新。

---

## 2.3 Categories — 多级分类表

采用 **物化路径 + 邻接表混合** 模型：邻接表保证写入简单，`path` 物化路径保证子树查询高效。

```sql
CREATE TABLE categories (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_id    UUID REFERENCES categories(id) ON DELETE RESTRICT,
    slug         TEXT NOT NULL,          -- URL 友好标识，如 'documentary'
    name         TEXT NOT NULL,
    description  TEXT,

    -- 物化路径：'/uuid1/uuid2/uuid3/'，含自身
    path         TEXT NOT NULL,
    depth        SMALLINT NOT NULL DEFAULT 0,

    -- 排序与展示
    sort_order   INTEGER NOT NULL DEFAULT 0,
    is_visible   BOOLEAN NOT NULL DEFAULT TRUE,

    -- 内容类型约束：一个分类可限定只允许某类内容
    allowed_types TEXT[] NOT NULL DEFAULT ARRAY['VIDEO','AUDIO','TEXT'],

    content_count INTEGER NOT NULL DEFAULT 0,   -- 冗余计数，异步更新

    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at   TIMESTAMPTZ,

    CONSTRAINT chk_no_self_parent CHECK (parent_id IS NULL OR parent_id <> id),
    CONSTRAINT chk_depth_limit CHECK (depth <= 6)   -- 最多 6 级嵌套
);

-- 同级下 slug 唯一
CREATE UNIQUE INDEX idx_categories_slug_sibling
    ON categories (COALESCE(parent_id, '00000000-0000-0000-0000-000000000000'::uuid), slug)
    WHERE deleted_at IS NULL;

-- 子树查询主路径：路径前缀匹配
CREATE INDEX idx_categories_path ON categories (path text_pattern_ops) WHERE deleted_at IS NULL;

-- 邻接查询：列出直接子节点
CREATE INDEX idx_categories_parent ON categories (parent_id, sort_order) WHERE deleted_at IS NULL;

-- 内容类型过滤（GIN 支持数组包含查询）
CREATE INDEX idx_categories_allowed_types ON categories USING GIN (allowed_types);
```

**关键查询示例**：

```sql
-- 查询某分类的全部子孙（利用 path 前缀，O(log n)，无需递归 CTE）
SELECT * FROM categories
WHERE path LIKE (SELECT path FROM categories WHERE id = $1) || '%'
  AND deleted_at IS NULL;

-- 查询完整面包屑（按 path 拆分子查询）
SELECT * FROM categories
WHERE id = ANY(string_to_array($1, '/')::uuid[])
ORDER BY depth;
```

> **为何不用纯邻接表**：递归 CTE 在深层级下性能不可控。物化路径让子树查询退化为一次索引范围扫描。
> **代价**：移动分类需批量更新子孙的 `path`。分类移动是低频操作，用事务批量 `UPDATE ... WHERE path LIKE ...` 即可接受。

**Tag 表**（分类是树，标签是网，两者互补）：

```sql
CREATE TABLE tags (
    id          BIGSERIAL PRIMARY KEY,
    name        TEXT NOT NULL,
    slug        TEXT NOT NULL,
    usage_count INTEGER NOT NULL DEFAULT 0,
    is_official BOOLEAN NOT NULL DEFAULT FALSE,   -- 官方标签不可被普通用户删除
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT uq_tags_slug UNIQUE (slug)
);

CREATE INDEX idx_tags_name_trgm ON tags USING GIN (name gin_trgm_ops);  -- 模糊搜索
CREATE INDEX idx_tags_usage ON tags (usage_count DESC);

CREATE TABLE content_tags (
    content_id UUID   NOT NULL REFERENCES contents(id) ON DELETE CASCADE,
    tag_id     BIGINT NOT NULL REFERENCES tags(id)     ON DELETE CASCADE,
    PRIMARY KEY (content_id, tag_id)
);
CREATE INDEX idx_content_tags_tag ON content_tags (tag_id);
```

---

## 2.4 Contents — 内容主表

涵盖视频、音频、文本三类，用 `content_type` 区分，类型特有字段放 MongoDB 扩展。

```sql
CREATE TYPE content_type   AS ENUM ('VIDEO', 'AUDIO', 'TEXT');
-- 状态枚举说明：
--   AUTO_PUBLISHED 是"已发布"的一种，与 PUBLISHED 并列而非其前驱。
--   两者对用户的可见性完全相同（可播放、可下载、出现在列表），
--   区分它们的唯一目的是记录"由系统自动放行"这一事实，用于误杀率回溯。
--   ⚠️ 因此所有"是否对外可见"的判断必须写成 status IN ('PUBLISHED','AUTO_PUBLISHED')，
--   绝不能只写 status = 'PUBLISHED'（见 02 索引与 04 接口中的相应处理）。
CREATE TYPE content_status AS ENUM (
    'DRAFT', 'UPLOADED', 'TRANSCODING', 'TRANSCODE_FAILED',
    'PRECHECK_DONE', 'PENDING_MANUAL', 'AUTO_PUBLISHED',
    'PUBLISHED', 'REJECTED', 'BLOCKED', 'APPEALING', 'DELETED'
);
CREATE TYPE risk_level AS ENUM ('NONE', 'LOW', 'MEDIUM', 'HIGH', 'CRITICAL');

CREATE TABLE contents (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    uploader_id      UUID NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    category_id      UUID REFERENCES categories(id) ON DELETE SET NULL,
    content_type     content_type   NOT NULL,
    status           content_status NOT NULL DEFAULT 'DRAFT',

    title            TEXT NOT NULL,
    description      TEXT,
    -- 摘要用于列表页，避免拉全文
    summary          TEXT,

    -- 存储定位（对象存储 key，不存完整 URL）
    source_object_key    TEXT,
    source_size_bytes    BIGINT,
    source_sha256        BYTEA,              -- 去重与完整性校验
    source_format        TEXT,               -- 'mp4' | 'flac' | 'pdf'

    -- 转码产物（HLS/DASH 清单 key）
    hls_master_key       TEXT,
    dash_manifest_key    TEXT,
    thumbnail_key        TEXT,
    sprite_key           TEXT,
    duration_seconds     INTEGER,
    width                INTEGER,
    height               INTEGER,

    -- 审核相关
    risk_score       SMALLINT NOT NULL DEFAULT 0,   -- 0-100
    risk_level       risk_level NOT NULL DEFAULT 'NONE',
    precheck_engine_version TEXT,
    precheck_detail  JSONB,          -- 各引擎原始输出，便于申诉复核
    auto_published   BOOLEAN NOT NULL DEFAULT FALSE,
    published_at     TIMESTAMPTZ,
    rejected_reason  TEXT,
    reviewed_by      UUID REFERENCES users(id),
    reviewed_at      TIMESTAMPTZ,

    -- 统计（冗余，异步更新）
    view_count       BIGINT NOT NULL DEFAULT 0,
    download_count   BIGINT NOT NULL DEFAULT 0,
    like_count       BIGINT NOT NULL DEFAULT 0,
    comment_count    INTEGER NOT NULL DEFAULT 0,

    -- 可见性
    visibility       TEXT NOT NULL DEFAULT 'PUBLIC',  -- PUBLIC | UNLISTED | PRIVATE

    created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at       TIMESTAMPTZ,

    CONSTRAINT chk_risk_score CHECK (risk_score BETWEEN 0 AND 100),
    CONSTRAINT chk_source_size CHECK (source_size_bytes IS NULL OR source_size_bytes > 0)
);
```

### Contents 索引策略（本表是热点，索引需精挑）

```sql
-- ① 待审核队列：审核员最核心的查询
--    部分索引只含待审状态，索引体积远小于全表
CREATE INDEX idx_contents_pending_queue
    ON contents (risk_score DESC, created_at ASC)
    WHERE status = 'PENDING_MANUAL' AND deleted_at IS NULL;

-- ② 已发布内容按分类浏览（列表页主路径）
--    必须同时包含 AUTO_PUBLISHED —— 否则自动放行的内容在列表页彻底不可见
CREATE INDEX idx_contents_published_category
    ON contents (category_id, published_at DESC)
    WHERE status IN ('PUBLISHED', 'AUTO_PUBLISHED')
      AND deleted_at IS NULL AND visibility = 'PUBLIC';

-- ③ 用户主页：某用户全部内容
CREATE INDEX idx_contents_uploader
    ON contents (uploader_id, created_at DESC)
    WHERE deleted_at IS NULL;

-- ④ 去重：同用户重复上传相同文件
CREATE UNIQUE INDEX idx_contents_sha256_uploader
    ON contents (uploader_id, source_sha256)
    WHERE source_sha256 IS NOT NULL AND deleted_at IS NULL;

-- ⑤ 申诉/风控：按上传者与风险等级
CREATE INDEX idx_contents_uploader_risk
    ON contents (uploader_id, risk_level)
    WHERE risk_level IN ('HIGH', 'CRITICAL') AND deleted_at IS NULL;

-- ⑥ 审计报表：按发布时间统计（BRIN 适合时序大表）
CREATE INDEX idx_contents_published_brin
    ON contents USING BRIN (published_at);

-- ⑦ 全文检索（标题 + 描述）
CREATE INDEX idx_contents_fts
    ON contents USING GIN (to_tsvector('simple', title || ' ' || COALESCE(description, '')));

-- ⑧ JSONB 预审详情检索（如按命中关键词反查）
CREATE INDEX idx_contents_precheck_gin
    ON contents USING GIN (precheck_detail jsonb_path_ops);
```

| 索引 | 类型 | 理由 |
|---|---|---|
| `idx_contents_pending_queue` | B-tree 部分 | 审核队列只关心 `PENDING_MANUAL`，通常占全表 <1%，索引可常驻内存；`risk_score DESC` 让高风险优先被处理 |
| `idx_contents_published_category` | B-tree 部分 | 列表页最高频查询；过滤 `visibility='PUBLIC'` 减少索引膨胀 |
| `idx_contents_sha256_uploader` | B-tree 唯一部分 | 数据库层直接拦重复上传，不依赖应用判断 |
| `idx_contents_published_brin` | BRIN | 时序数据物理有序，BRIN 体积是 B-tree 的千分之一 |
| `idx_contents_fts` | GIN | 全文检索；`simple` 配置对中英混合更可控（中文另配 zhparser） |
| `idx_contents_precheck_gin` | GIN | `jsonb_path_ops` 比默认 `jsonb_ops` 索引更小更快，代价是不支持 `?` 键存在查询 |

> **大表分区**：当 `contents` 超过 5000 万行，按 `published_at` 做 RANGE 分区（月分区），历史分区可迁移至冷存储。

---

## 2.5 AuditLogs — 审核日志表（仅追加）

**不可变性是核心要求**。此表只允许 INSERT，禁止 UPDATE/DELETE（用数据库权限 + 触发器双重保证）。

```sql
CREATE TABLE audit_logs (
    id            BIGSERIAL PRIMARY KEY,
    content_id    UUID REFERENCES contents(id) ON DELETE RESTRICT,

    -- 操作主体
    actor_type    TEXT NOT NULL,        -- 'USER' | 'MODERATOR' | 'SYSTEM' | 'AI_ENGINE'
    actor_id      UUID REFERENCES users(id),
    actor_ip_hash BYTEA,                -- 哈希 IP，非明文
    actor_role    TEXT,

    action        TEXT NOT NULL,        -- 'PRECHECK_PASS' | 'MANUAL_APPROVE' | 'MANUAL_REJECT' | 'USER_BAN' | ...
    from_status   content_status,
    to_status     content_status,

    -- 决策依据
    risk_score    SMALLINT,
    risk_level    risk_level,
    reason        TEXT,
    rule_hits     JSONB,               -- 命中规则明细，便于申诉复核

    -- 引擎元信息（用于模型效果回溯）
    engine_name    TEXT,
    engine_version TEXT,
    model_confidence NUMERIC(5,4),

    -- 防篡改链：每条记录含前一条的哈希
    prev_hash     BYTEA,
    record_hash   BYTEA NOT NULL,

    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### AuditLogs 索引策略

```sql
-- ① 内容追溯：查某内容的完整审核历史（申诉复核主路径）
CREATE INDEX idx_audit_logs_content
    ON audit_logs (content_id, created_at DESC);

-- ② 审核员工作量与绩效统计
CREATE INDEX idx_audit_logs_actor
    ON audit_logs (actor_id, created_at DESC)
    WHERE actor_id IS NOT NULL;

-- ③ 时序报表（按月分区后每分区一个 BRIN）
CREATE INDEX idx_audit_logs_created_brin
    ON audit_logs USING BRIN (created_at);

-- ④ 按动作类型审计（如统计误杀率：MANUAL_REJECT 后被 APPEAL_APPROVE）
CREATE INDEX idx_audit_logs_action
    ON audit_logs (action, created_at DESC);

-- ⑤ 规则命中分析：找出误杀率高的规则
CREATE INDEX idx_audit_logs_rule_hits
    ON audit_logs USING GIN (rule_hits jsonb_path_ops);
```

### 按月分区

**注意：分区不能通过"再建一次表"实现。** 若需要分区，上面的 `CREATE TABLE audit_logs` 必须**从一开始**就带 `PARTITION BY`，且有两个强制约束：

1. **主键必须包含分区键**。PostgreSQL 要求分区表的唯一约束涵盖分区列，因此 `id BIGSERIAL PRIMARY KEY` 在分区表上非法，须改为复合主键 `(id, created_at)`。
2. **分区表不能有跨分区的唯一索引**，除非索引包含分区键。因此 `id` 不再是全局唯一——若业务需要全局唯一的日志 ID，改用 `(created_at, id)` 组合，或单独维护一个序列。

分区版本的表定义（**与上文非分区版本二选一，不可同时执行**）：

```sql
CREATE TABLE audit_logs (
    id            BIGSERIAL    NOT NULL,
    content_id    UUID         REFERENCES contents(id) ON DELETE RESTRICT,
    actor_type    TEXT         NOT NULL,
    actor_id      UUID         REFERENCES users(id),
    actor_ip_hash BYTEA,
    actor_role    TEXT,
    action        TEXT         NOT NULL,
    from_status   content_status,
    to_status     content_status,
    risk_score    SMALLINT,
    risk_level    risk_level,
    reason        TEXT,
    rule_hits     JSONB,
    engine_name    TEXT,
    engine_version TEXT,
    model_confidence NUMERIC(5,4),
    prev_hash     BYTEA,
    record_hash   BYTEA        NOT NULL,
    created_at    TIMESTAMPTZ  NOT NULL DEFAULT now(),

    -- 分区表主键必须包含分区键 created_at
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

CREATE TABLE audit_logs_2026_01 PARTITION OF audit_logs
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE audit_logs_2026_02 PARTITION OF audit_logs
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- 由定时任务自动创建未来 3 个月分区，并归档 24 个月前的分区

-- 分区表上的索引必须逐分区创建（PostgreSQL 11+ 支持在父表建索引并自动传播）
CREATE INDEX idx_audit_logs_content ON audit_logs (content_id, created_at DESC);
```

> **工程建议**：若初期日志量不大（月增 < 500 万行），**直接使用非分区版本**，避免复合主键带来的应用侧复杂度（`id` 不再全局唯一）。待单表超过 1 亿行或查询开始变慢时，再通过 `pg_partman` 迁移到分区。架构文档同时给出两版是为了说明迁移路径，**部署时只能启用其中一个**。

### 防篡改实现

```sql
-- 触发器：阻止 UPDATE / DELETE
CREATE OR REPLACE FUNCTION audit_logs_immutable()
RETURNS TRIGGER AS $$
BEGIN
    RAISE EXCEPTION 'audit_logs is append-only; % is forbidden', TG_OP;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_audit_logs_no_update
    BEFORE UPDATE OR DELETE ON audit_logs
    FOR EACH ROW EXECUTE FUNCTION audit_logs_immutable();
```

`record_hash = SHA256(prev_hash || canonical_json(本行业务字段))`。定期校验链完整性；若不匹配说明记录被外部篡改，立即告警。生产环境建议同步一份到 WORM 存储或独立 SIEM。

---

## 2.6 辅助表

### 2.6.1 上传配额追踪

```sql
CREATE TABLE upload_quota_usage (
    user_id       UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    quota_date    DATE NOT NULL,
    used_bytes    BIGINT  NOT NULL DEFAULT 0,
    used_count    INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (user_id, quota_date)
);

-- 清理历史配额记录
CREATE INDEX idx_quota_usage_date ON upload_quota_usage (quota_date);
```

> 配额判定需高频原子自增，实际走 Redis `INCRBY`（key: `quota:{userId}:{date}`），PG 表仅作持久化兜底，每小时由 Worker 落盘。Redis 与 PG 不一致时**以 PG 为准**，防止 Redis 故障被刷配额。

### 2.6.2 审核申诉

```sql
CREATE TYPE appeal_status AS ENUM ('PENDING', 'UNDER_REVIEW', 'APPROVED', 'REJECTED', 'WITHDRAWN');

CREATE TABLE appeals (
    id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content_id     UUID NOT NULL REFERENCES contents(id) ON DELETE CASCADE,
    appellant_id   UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    status         appeal_status NOT NULL DEFAULT 'PENDING',
    reason         TEXT NOT NULL,
    evidence_keys  TEXT[],            -- 用户提交的补充材料对象存储 key
    reviewer_id    UUID REFERENCES users(id),
    review_note    TEXT,
    resolved_at    TIMESTAMPTZ,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- 同一内容同时只能有一个"未结"申诉。
-- 用部分唯一索引实现，而非 UNIQUE(content_id, status) 表约束——
-- 后者会连带禁止同一内容的第二条 APPROVED 或 REJECTED 记录，
-- 从而堵死"申诉被驳回后、在允许的条件下再次申诉"的产品流程。
-- 此处仅约束 PENDING / UNDER_REVIEW 两种进行中状态。
CREATE UNIQUE INDEX idx_appeals_open_unique
    ON appeals (content_id)
    WHERE status IN ('PENDING', 'UNDER_REVIEW');

-- 申诉队列：按创建时间 FIFO，但高风险内容优先
CREATE INDEX idx_appeals_queue
    ON appeals (status, created_at ASC)
    WHERE status IN ('PENDING', 'UNDER_REVIEW');

CREATE INDEX idx_appeals_appellant ON appeals (appellant_id, created_at DESC);
```

### 2.6.3 会话与设备（安全审计）

```sql
CREATE TABLE sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    refresh_token_hash BYTEA NOT NULL,      -- 仅存哈希，库泄露也无法复用
    device_label    TEXT,
    user_agent_hash BYTEA,
    ip_hash         BYTEA,
    mfa_verified    BOOLEAN NOT NULL DEFAULT FALSE,
    expires_at      TIMESTAMPTZ NOT NULL,
    revoked_at      TIMESTAMPTZ,
    -- 轮换时间戳：非 NULL 表示该令牌已被轮换过。
    -- 若已轮换的令牌再次出现，即判定为重放攻击（见 03 节 3.6），
    -- 触发该用户全部会话吊销。此列是重放检测的必要条件。
    rotated_at      TIMESTAMPTZ,
    -- 轮换后指向新会话，便于审计时还原令牌链
    rotated_to      UUID REFERENCES sessions(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sessions_user ON sessions (user_id, created_at DESC) WHERE revoked_at IS NULL;
CREATE UNIQUE INDEX idx_sessions_token ON sessions (refresh_token_hash);
CREATE INDEX idx_sessions_expiry ON sessions (expires_at) WHERE revoked_at IS NULL;

CREATE TABLE mfa_credentials (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id       UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    cred_type     TEXT NOT NULL,          -- 'WEBAUTHN' | 'TOTP' | 'RECOVERY_CODE'
    -- WebAuthn 公钥可明文存；TOTP 密钥必须用 KMS 数据密钥加密后存
    public_key    BYTEA,
    credential_id BYTEA,
    encrypted_secret BYTEA,               -- 信封加密密文
    secret_kms_key_id TEXT,               -- KMS 中的数据密钥引用，非密钥本身
    sign_count    BIGINT NOT NULL DEFAULT 0,
    label         TEXT,
    last_used_at  TIMESTAMPTZ,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT uq_mfa_credential UNIQUE (user_id, cred_type, credential_id)
);

CREATE INDEX idx_mfa_user ON mfa_credentials (user_id);

-- 恢复码只存哈希，一次性使用
CREATE TABLE mfa_recovery_codes (
    id         BIGSERIAL PRIMARY KEY,
    user_id    UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    code_hash  BYTEA NOT NULL,
    used_at    TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_recovery_user_unused ON mfa_recovery_codes (user_id) WHERE used_at IS NULL;
```

### 2.6.4 下载授权与防盗链

```sql
CREATE TABLE download_grants (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content_id    UUID NOT NULL REFERENCES contents(id) ON DELETE CASCADE,
    user_id       UUID REFERENCES users(id) ON DELETE SET NULL,   -- 匿名下载为 NULL
    asset_kind    TEXT NOT NULL,          -- 'SOURCE' | 'AUDIO' | 'DOCUMENT'
    max_downloads INTEGER NOT NULL DEFAULT 3,
    used_downloads INTEGER NOT NULL DEFAULT 0,
    max_bytes     BIGINT,
    used_bytes    BIGINT NOT NULL DEFAULT 0,
    expires_at    TIMESTAMPTZ NOT NULL,
    revoked_at    TIMESTAMPTZ,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_download_grants_content ON download_grants (content_id, created_at DESC);
CREATE INDEX idx_download_grants_expiry ON download_grants (expires_at) WHERE revoked_at IS NULL;
CREATE INDEX idx_download_grants_user ON download_grants (user_id) WHERE user_id IS NOT NULL;
```

---

## 2.7 MongoDB 集合设计（非结构化部分）

### `danmaku` — 弹幕

```javascript
{
  _id: ObjectId(),
  contentId: "uuid",            // 引用 PG contents.id
  userId: "uuid",
  videoTimeMs: 12345,           // 出现时间点
  text: "弹幕内容",
  color: "#FFFFFF",
  fontSize: 25,
  position: "scroll",           // scroll | top | bottom
  mode: 1,
  status: "visible",            // visible | hidden | deleted
  deletedAt: null,              // 软删时写入时间戳，TTL 索引据此在 30 天后物理清理
  likeCount: 0,
  createdAt: ISODate()
}
```

索引：
```javascript
db.danmaku.createIndex({ contentId: 1, videoTimeMs: 1 }, { name: "idx_content_time" });
db.danmaku.createIndex({ contentId: 1, status: 1, createdAt: -1 });
// TTL 清理：软删除的弹幕保留 30 天后物理移除。
// 注意 TTL 索引的字段必须真实存在于文档中，因此软删时必须显式写入 deletedAt
// （仅把 status 改为 'deleted' 不会触发清理）。
db.danmaku.createIndex({ deletedAt: 1 }, { expireAfterSeconds: 2592000, sparse: true });
```

### `content_metadata` — 类型特有的扩展元数据

```javascript
{
  _id: ObjectId(),
  contentId: "uuid",
  contentType: "VIDEO",
  // 视频特有
  codecs: { video: "h264", audio: "aac" },
  renditions: [
    { profile: "1080p", bitrateKbps: 5000, width: 1920, height: 1080, key: "hls/1080p/index.m3u8" },
    { profile: "720p",  bitrateKbps: 2800, width: 1280, height: 720,  key: "hls/720p/index.m3u8" }
  ],
  // 音频特有
  audioMeta: { format: "flac", sampleRate: 96000, bitDepth: 24, channels: 2, lossless: true },
  // 文本特有
  textMeta: { wordCount: 12000, language: "zh-CN", chapters: [{ title: "序章", anchor: "ch-1" }] },
  // 多语言字幕
  subtitles: [{ lang: "zh-CN", vttKey: "subs/zh.vtt" }],
  updatedAt: ISODate()
}
```

索引：`db.content_metadata.createIndex({ contentId: 1 }, { unique: true })`

### `play_events` — 播放埋点（高写入量，定期归档）

```javascript
{
  _id: ObjectId(),
  contentId: "uuid",
  sessionIdHash: "sha256",      // 会话哈希，不含用户身份
  positionMs: 60000,
  quality: "720p",
  eventType: "heartbeat",       // start | heartbeat | pause | ended | seek
  createdAt: ISODate()
}
```

索引：`db.play_events.createIndex({ contentId: 1, createdAt: -1 })`，TTL 90 天自动清理（隐私最小化：埋点不留存超期）。

---

## 2.8 数据完整性与约束总览

| 约束 | 实现层 | 说明 |
|---|---|---|
| 邮箱唯一 | 部分唯一索引 | 注销后可复用 |
| 密码不可逆 | 应用层 + `password_algo` 字段 | 兼容量化升级算法 |
| 审核日志不可改 | 触发器 + 数据库账号权限 | 应用账号无 UPDATE/DELETE 权限 |
| 状态流转合法 | 应用层状态机 + CHECK | 非法跃迁在服务层拒绝 |
| 分类无环 | 应用层校验 path 前缀 | 移动时禁止将节点移入自身子树 |
| 配额不超发 | Redis 原子自增 + PG 兜底 | 双写以 PG 为准 |
| 内容去重 | 部分唯一索引 (uploader_id, sha256) | 数据库层拦截 |
