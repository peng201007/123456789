# 8. 部署与安全配置清单

## 8.1 Nginx 配置

### 8.1.1 全局配置

```nginx
# /etc/nginx/nginx.conf
user  nginx;
worker_processes  auto;                    # 匹配 CPU 核数
worker_rlimit_nofile 65535;                # 每个 worker 可打开的文件描述符上限

events {
    worker_connections  10240;
    multi_accept        on;
    use                 epoll;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    # ── 日志：包含请求耗时与上游耗时，便于定位瓶颈 ──
    log_format main escape=json
        '{"time":"$time_iso8601",'
        '"remote_addr":"$remote_addr",'
        '"request_id":"$request_id",'
        '"method":"$request_method",'
        '"uri":"$uri",'
        '"status":$status,'
        '"body_bytes":$body_bytes_sent,'
        '"request_time":$request_time,'
        '"upstream_time":"$upstream_response_time",'
        '"user_agent":"$http_user_agent",'
        '"referer":"$http_referer"}';

    access_log /var/log/nginx/access.log main buffer=64k flush=5s;
    error_log  /var/log/nginx/error.log warn;

    # ── 性能 ──
    sendfile        on;
    tcp_nopush      on;          # 配合 sendfile，减少小包
    tcp_nodelay     on;          # 低延迟优先
    keepalive_timeout  65s;      # 稍长，复用连接
    keepalive_requests 1000;
    types_hash_max_size 2048;
    server_tokens   off;         # 隐藏 Nginx 版本号

    # ── 客户端请求体：JSON API 限 1MB；上传走对象存储，不经此处 ──
    client_max_body_size    1m;
    client_body_buffer_size 128k;
    client_header_timeout   30s;
    client_body_timeout     30s;
    send_timeout            60s;

    # ── 大文件相关：即使不代理字节流，也影响元数据请求 ──
    large_client_header_buffers 4 16k;

    # ── 压缩：仅对文本类生效，不压视频/图片（已压缩） ──
    gzip              on;
    gzip_vary         on;
    gzip_proxied      any;
    gzip_comp_level   5;         # 6 以上收益递减，CPU 成本上升
    gzip_min_length   256;
    gzip_types
        application/javascript
        application/json
        application/xml
        application/manifest+json
        image/svg+xml
        text/css
        text/javascript
        text/plain
        text/xml;

    # brotli（若编译了模块）
    # brotli on;
    # brotli_comp_level 5;
    # brotli_types text/css application/javascript application/json image/svg+xml;

    # ── 限流区定义 ──
    # 登录：按 IP，突发容忍小
    limit_req_zone $binary_remote_addr zone=login_ip:10m  rate=20r/m;
    # 通用 API：按 IP
    limit_req_zone $binary_remote_addr zone=api_ip:10m    rate=300r/m;
    # 认证用户维度（由应用侧令牌识别，Nginx 用 header 传递）
    limit_req_zone $http_x_user_id     zone=api_user:10m  rate=600r/m;
    # 上传凭证申请
    limit_req_zone $http_x_user_id     zone=upload:10m    rate=10r/h;
    # 连接数限制：防单 IP 建立海量连接
    limit_conn_zone $binary_remote_addr zone=conn_ip:10m;

    # ── 限流被触发时的状态码与日志级别 ──
    limit_req_status  429;
    limit_conn_status 429;
    limit_req_log_level warn;

    # ── 安全响应头 ──
    add_header X-Content-Type-Options    "nosniff"        always;
    add_header X-Frame-Options           "DENY"           always;
    add_header Referrer-Policy           "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy        "camera=(), microphone=(), geolocation=(), payment=()" always;
    add_header Cross-Origin-Opener-Policy "same-origin"   always;

    include /etc/nginx/conf.d/*.conf;
}
```

### 8.1.2 主站配置（TLS 1.3 + HSTS）

```nginx
# /etc/nginx/conf.d/api.conf

# ── HTTP：仅用于跳转与 ACME 验证 ──
server {
    listen 80;
    listen [::]:80;
    server_name api.example.com;

    # Let's Encrypt 验证路径必须放行
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
        allow all;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

# ── HTTPS 主服务 ──
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name api.example.com;

    # ── 证书 ──
    ssl_certificate     /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;   # 权限 600，属主 nginx
    ssl_trusted_certificate /etc/nginx/certs/chain.pem;

    # ── 强制 TLS 1.3（提示词要求） ──
    # 兼容性说明：TLS 1.3 在主流浏览器已全面支持。
    # 若需兼容旧客户端，可放开 1.2 并配合强密码套件，但优先满足 1.3 要求。
    ssl_protocols TLSv1.3;

    # TLS 1.3 的密码套件由 OpenSSL 默认提供且均安全，
    # 显式指定以明确策略（顺序即优先级）
    ssl_conf_command Ciphersuites TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256;

    # 若必须临时放开 TLS 1.2，使用以下配置（不满足"强制 1.3"要求，仅作兼容过渡）
    # ssl_protocols TLSv1.2 TLSv1.3;
    # ssl_ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305;
    # ssl_prefer_server_ciphers off;

    ssl_prefer_server_ciphers off;      # TLS 1.3 下该指令无实际影响，保留以明确意图
    ssl_session_cache   shared:SSL:50m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;            # 关闭会话票据，提升前向保密性

    # OCSP Stapling：减少客户端验证延迟
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 1.1.1.1 8.8.8.8 valid=300s;
    resolver_timeout 5s;

    # ── HSTS：1 年，含子域，预加载 ──
    # ⚠️ 部署前务必确认所有子域均已支持 HTTPS，否则子域将无法访问
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

    # ── CSP：内容安全策略 ──
    # ⚠️ 与前端主题脚本的冲突处理（重要）：
    #   07 节要求主题初始化脚本必须**内联且同步**执行（否则会闪烁）。
    #   但 'strict-dynamic' 会让浏览器**忽略** 'self' 与 https:，
    #   只执行带 nonce 或 hash 的脚本 —— 那样内联主题脚本会被拦掉，
    #   浅色主题会闪一下再变深色。
    #
    #   因此这里采用「按 hash 放行」方案：把主题脚本的内容做 SHA-256，
    #   以 'sha256-...' 形式列出。这样既不需要 nonce（无需改 HTML 生成逻辑），
    #   也不破坏 strict-dynamic 的语义。
    #   hash 随脚本内容变化，构建流程需自动计算并注入（见下方说明）。
    #
    # 若构建流程无法自动注入 hash，则退化为：保留 'unsafe-inline' 但不加
    # 'strict-dynamic'，安全性略降但可正常工作。

    add_header Content-Security-Policy "
        default-src 'self';
        script-src 'self' 'strict-dynamic' 'sha256-THEME_SCRIPT_HASH_PLACEHOLDER' https:;
        style-src 'self' 'unsafe-inline';
        img-src 'self' data: https://cdn.example.com;
        media-src 'self' https://cdn.example.com blob:;
        font-src 'self';
        connect-src 'self' https://api.example.com wss://api.example.com;
        frame-ancestors 'none';
        base-uri 'self';
        form-action 'self';
        object-src 'none';
        upgrade-insecure-requests;
    " always;

    # 若前端使用 theme-color 等 meta，CSP 不阻碍
    # 若需放宽（如第三方统计），逐项添加域名，禁止使用 * 通配

    # ── 请求 ID：贯穿前后端日志 ──
    add_header X-Request-Id $request_id always;

    # ── 连接数限制 ──
    limit_conn conn_ip 50;

    # ── 通用代理头 ──
    proxy_http_version 1.1;
    proxy_set_header Host              $host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Request-Id      $request_id;
    proxy_set_header Connection        "";

    # ══════════════════════════════════════════════════════
    #  认证接口：严格限流 + 短超时
    # ══════════════════════════════════════════════════════
    location ~ ^/api/v1/auth/(login|register|refresh|mfa) {
        limit_req zone=login_ip burst=5 nodelay;

        proxy_pass http://auth_backend;

        proxy_connect_timeout 3s;
        proxy_send_timeout    10s;
        proxy_read_timeout    10s;

        # 认证请求不缓存
        proxy_buffering off;
        proxy_no_cache 1;

        # 禁止搜索引擎索引认证端点
        add_header X-Robots-Tag "noindex, nofollow" always;
    }

    # ══════════════════════════════════════════════════════
    #  上传凭证申请：低频接口
    # ══════════════════════════════════════════════════════
    location = /api/v1/contents/upload-intent {
        limit_req zone=upload burst=3 nodelay;

        proxy_pass http://api_backend;

        # 该接口含配额校验，需查 PG + Redis，超时略放宽
        proxy_connect_timeout 3s;
        proxy_send_timeout    15s;
        proxy_read_timeout    15s;
    }

    # ══════════════════════════════════════════════════════
    #  下载签名 URL 签发：含频次校验
    # ══════════════════════════════════════════════════════
    location ~ ^/api/v1/contents/[^/]+/download {
        limit_req zone=api_user burst=5 nodelay;

        proxy_pass http://api_backend;
        proxy_read_timeout 15s;
    }

    # ══════════════════════════════════════════════════════
    #  管理后台：需要更长超时（批量操作、报表）
    # ══════════════════════════════════════════════════════
    location /api/v1/admin/ {
        limit_req zone=api_user burst=50 nodelay;

        proxy_pass http://api_backend;

        proxy_connect_timeout 3s;
        proxy_send_timeout    30s;
        proxy_read_timeout    30s;

        # 管理操作禁止被任何中间层缓存
        proxy_buffering off;
        add_header Cache-Control "no-store" always;
    }

    # ══════════════════════════════════════════════════════
    #  内部接口：仅允许集群内网访问
    # ══════════════════════════════════════════════════════
    location /internal/ {
        # 多层防护：网络层 + 应用层
        allow 10.0.0.0/8;
        allow 172.16.0.0/12;
        allow 192.168.0.0/16;
        deny all;

        proxy_pass http://api_backend;
    }

    # ══════════════════════════════════════════════════════
    #  通用 API：兜底规则（必须最后，正则优先级低于精确匹配）
    # ══════════════════════════════════════════════════════
    location /api/ {
        limit_req zone=api_ip burst=100 nodelay;

        proxy_pass http://api_backend;

        proxy_connect_timeout 3s;
        proxy_send_timeout    30s;
        proxy_read_timeout    30s;

        # 流式响应（SSE 用于审核队列推送）
        proxy_buffering off;
        proxy_cache off;
    }

    # ══════════════════════════════════════════════════════
    #  WebSocket：弹幕、审核队列实时推送
    # ══════════════════════════════════════════════════════
    location /ws/ {
        proxy_pass http://api_backend;

        proxy_http_version 1.1;
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection "upgrade";

        # 长连接需更长超时，否则空闲被断开
        proxy_read_timeout  3600s;
        proxy_send_timeout  3600s;

        # WebSocket 不缓冲
        proxy_buffering off;
    }

    # ══════════════════════════════════════════════════════
    #  健康检查：不记日志，不限流
    # ══════════════════════════════════════════════════════
    location = /health {
        access_log off;
        return 200 "ok\n";
        add_header Content-Type text/plain;
    }

    # ══════════════════════════════════════════════════════
    #  静态资源：前端 SPA
    # ══════════════════════════════════════════════════════
    location /assets/ {
        root /var/www/web;
        # 带内容哈希的文件名 → 永久缓存
        expires 1y;
        add_header Cache-Control "public, immutable" always;
        access_log off;
    }

    location / {
        root /var/www/web;
        try_files $uri $uri/ /index.html;

        # index.html 绝不缓存（否则用户拿到旧版引用已删除的资源）
        location = /index.html {
            add_header Cache-Control "no-cache, must-revalidate" always;
        }
    }
}
```

### 8.1.3 上游与超时说明

```nginx
# /etc/nginx/conf.d/upstream.conf
upstream api_backend {
    least_conn;                             # 按连接数负载，适合长连接场景
    server 10.0.1.11:3000 max_fails=3 fail_timeout=15s;
    server 10.0.1.12:3000 max_fails=3 fail_timeout=15s;
    server 10.0.1.13:3000 max_fails=3 fail_timeout=15s;
    keepalive 64;                           # 复用上游连接，降低握手开销
    keepalive_requests 1000;
    keepalive_timeout 60s;
}

upstream auth_backend {
    least_conn;
    server 10.0.1.11:3000 max_fails=2 fail_timeout=10s;   # 认证失败更快摘除
    server 10.0.1.12:3000 max_fails=2 fail_timeout=10s;
    keepalive 32;
}
```

### 8.1.4 超时参数的设计依据

| 参数 | 值 | 依据 |
|---|---|---|
| `proxy_connect_timeout` | 3s | 内网连接建立应在毫秒级；3 秒说明实例已故障，快速失败并重试其他实例 |
| `proxy_read_timeout`（普通 API） | 30s | 数据库慢查询 + 外部 AI 审核调用可能耗时；超过 30 秒用户已放弃 |
| `proxy_read_timeout`（审核后台） | 30s | 批量操作与报表查询需要更长时间 |
| `proxy_read_timeout`（WebSocket） | 3600s | 长连接；心跳由应用层维护（30 秒间隔） |
| `client_body_timeout` | 30s | 大文件不走此处；此为 JSON 请求体上传时间 |
| `send_timeout` | 60s | 向客户端写响应的间隔上限 |

> **重要说明：大文件上传不经 Nginx。** 客户端直传对象存储（presigned URL），字节流完全绕过本层。因此 `client_max_body_size` 保持 1MB 即可——这既降低了 Nginx 的内存与带宽压力，也避免了"为了传大文件而放宽 Nginx 限制"带来的 DoS 面。若因架构限制必须经 Nginx 中转，需设置 `client_max_body_size 5g`、`client_body_temp_path` 到独立磁盘、`proxy_request_buffering off`（流式转发），并单独限流。

### 8.1.5 CDN 侧配置要点

```nginx
# CDN 回源配置（对象存储或专用回源服务器）

# ── 签名校验：边缘验签，未通过不回源 ──
# 由边缘函数（Cloudflare Workers / Lambda@Edge / 阿里云 EdgeScript）执行
# 伪配置形态：
#   if (!verifyHmac(request.uri, request.query.signature,
#                   request.query.expires, request.query.sid)) {
#       return 403;
#   }

# ── 分片缓存策略 ──
location ~* \.(ts|m4s|mp4|m4a|flac)$ {
    # HLS/DASH 分片不可变，长缓存
    expires 30d;
    add_header Cache-Control "public, max-age=2592000, immutable" always;
    add_header Accept-Ranges bytes always;

    # 防盗链第一层
    valid_referers none blocked server_names *.example.com;
    if ($invalid_referer) { return 403; }
}

location ~* \.m3u8$ {
    # 清单文件会随转码更新，短缓存
    expires 5s;
    add_header Cache-Control "public, max-age=5" always;
}

location ~* \.mpd$ {
    expires 5s;
    add_header Cache-Control "public, max-age=5" always;
}

# ── CORS：仅允许本站域名 ──
location /media/ {
    add_header Access-Control-Allow-Origin  "https://www.example.com" always;
    add_header Access-Control-Allow-Methods "GET, HEAD, OPTIONS" always;
    add_header Access-Control-Allow-Headers "Range" always;
    add_header Access-Control-Expose-Headers "Content-Length, Content-Range" always;

    # 允许 Range 请求（视频拖动必需）
    add_header Accept-Ranges bytes always;

    if ($request_method = OPTIONS) {
        return 204;
    }
}
```

---

## 8.2 防火墙规则建议

### 8.2.1 网络分层与安全组

```
┌─────────────────────────────────────────────────────────────┐
│  公网 (Internet)                                             │
└────────────────────────┬────────────────────────────────────┘
                         │ 仅 443 (HTTPS) + 80 (跳转/ACME)
┌────────────────────────▼────────────────────────────────────┐
│  DMZ 层：负载均衡 / WAF / CDN 回源                            │
│  安全组：入站 443, 80 from 0.0.0.0/0                          │
└────────────────────────┬────────────────────────────────────┘
                         │ 仅 443 from DMZ 安全组
┌────────────────────────▼────────────────────────────────────┐
│  应用层：Nginx / NestJS 集群                                  │
│  安全组：入站 443 from DMZ-SG；22 from 跳板机-SG              │
│  出站：5432/6379/9092 to 数据层-SG；443 to 对象存储/AI 服务    │
└────────────────────────┬────────────────────────────────────┘
                         │ 仅数据库端口 from 应用安全组
┌────────────────────────▼────────────────────────────────────┐
│  数据层：PostgreSQL / Redis / MongoDB / Kafka                 │
│  安全组：入站 5432/6379/27017/9092 from 应用层-SG             │
│  ❌ 绝不开放公网入站                                          │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  管理层：KMS / Vault / 监控 / 日志                            │
│  安全组：入站 from 应用层-SG 与 运维跳板机-SG                 │
│  ❌ 绝不开放公网入站                                          │
└─────────────────────────────────────────────────────────────┘
```

### 8.2.2 主机防火墙规则（iptables / nftables）

```bash
#!/usr/bin/env bash
# 应用节点基础防火墙规则（nftables 语法）
# ⚠️ 变量从环境或配置管理注入，不硬编码网段外的敏感值

set -euo pipefail

ADMIN_CIDR="${ADMIN_CIDR:?must be set}"        # 运维跳板机网段
APP_SG_CIDR="${APP_SG_CIDR:?must be set}"
DATA_SG_CIDR="${DATA_SG_CIDR:?must be set}"

nft flush ruleset

nft add table inet filter

# ── 输入链 ──
nft add chain inet filter input '{ type filter hook input priority 0; policy drop; }'

# 已建立连接与本地回环放行
nft add rule inet filter input ct state established,related accept
nft add rule inet filter input iif lo accept

# ICMP：仅允许必要类型，防 ICMP 洪水
nft add rule inet filter input ip protocol icmp icmp type { echo-request, destination-unreachable, time-exceeded } accept
nft add rule inet filter input ip6 nexthdr ipv6-icmp accept

# SSH：仅跳板机可达，且限速防爆破
nft add rule inet filter input ip saddr "$ADMIN_CIDR" tcp dport 22 \
    ct state new limit rate 6/minute accept

# HTTP/HTTPS：来自负载均衡
nft add rule inet filter input ip saddr "$APP_SG_CIDR" tcp dport { 80, 443 } accept

# 应用端口：仅同安全组内互访（服务间调用）
nft add rule inet filter input ip saddr "$APP_SG_CIDR" tcp dport 3000 accept

# 数据层端口：仅应用安全组（数据节点上的规则）
nft add rule inet filter input ip saddr "$APP_SG_CIDR" tcp dport { 5432, 6379, 27017, 9092 } accept

# 监控端口：仅内网
nft add rule inet filter input ip saddr "$APP_SG_CIDR" tcp dport 9100 accept

# 记录并丢弃其余所有入站
nft add rule inet filter input limit rate 5/minute log prefix "NFT-DROP-IN: " level warn
nft add rule inet filter input drop

# ── 转发链：默认丢弃（非路由器） ──
nft add chain inet filter forward '{ type filter hook forward priority 0; policy drop; }'

# ── 输出链：限制出站，防数据外泄 ──
nft add chain inet filter output '{ type filter hook output priority 0; policy drop; }'
nft add rule inet filter output ct state established,related accept
nft add rule inet filter output oif lo accept
nft add rule inet filter output tcp dport { 53, 123 } accept              # DNS, NTP
nft add rule inet filter output ip daddr "$DATA_SG_CIDR" tcp dport { 5432, 6379, 27017, 9092 } accept
nft add rule inet filter output tcp dport 443 accept                       # HTTPS 出站（对象存储、AI 服务、KMS）
nft add rule inet filter output limit rate 5/minute log prefix "NFT-DROP-OUT: " level warn
nft add rule inet filter output drop
```

**出站限制的价值**：多数入侵的最终目标是**数据外泄**。默认允许所有出站，攻击者拿到 shell 后可随意把数据库导出到任意地址。限制出站端口与目标网段后，外泄只能走 443 且受域名白名单约束，显著提高攻击成本。

### 8.2.3 主机加固清单

| 项 | 配置 | 说明 |
|---|---|---|
| SSH 密码登录 | 禁用（`PasswordAuthentication no`） | 仅密钥认证 |
| SSH root 登录 | 禁用（`PermitRootLogin no`） | 用普通用户 + sudo |
| SSH 端口 | 非 22（可选） | 仅减少日志噪音，非实质安全措施 |
| 会话超时 | `ClientAliveInterval 300` | 5 分钟无活动断开 |
| 自动安全更新 | `unattended-upgrades` 启用 | 关键补丁自动安装 |
| 内核参数 | `net.ipv4.tcp_syncookies=1` | 防 SYN 洪水 |
| 内核参数 | `net.ipv4.conf.all.rp_filter=1` | 防 IP 欺骗 |
| 内核参数 | `kernel.dmesg_restrict=1` | 限制内核日志读取 |
| 内核参数 | `fs.protected_hardlinks=1` `fs.protected_symlinks=1` | 防链接攻击 |
| 核心转储 | `fs.suid_dumpable=0` | 防内存转储泄露密钥 |
| 文件权限 | 证书私钥 600，属主 nginx | 防止其他进程读取 |
| SELinux/AppArmor | 启用 enforcing 模式 | 限制进程越权 |
| 容器运行时 | 非 root 用户，只读根文件系统 | 降低容器逃逸影响 |
| 容器能力 | 丢弃所有 capabilities，按需添加 | 遵循最小权限 |

### 8.2.4 关键内核参数

```bash
# /etc/sysctl.d/99-media-platform.conf

# ── 网络安全 ──
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1
net.ipv4.conf.all.log_martians = 1

# ── 连接与性能（高并发场景） ──
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.ip_local_port_range = 10000 65535
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 15
net.core.netdev_max_backlog = 16384

# ── 大文件/长连接场景 ──
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216

# ── 文件描述符 ──
fs.file-max = 2097152
fs.nr_open = 2097152

# ── 安全加固 ──
kernel.dmesg_restrict = 1
kernel.kptr_restrict = 2
kernel.yama.ptrace_scope = 1
fs.protected_hardlinks = 1
fs.protected_symlinks = 1
fs.suid_dumpable = 0
kernel.randomize_va_space = 2
```

---

## 8.3 密钥与凭据管理

### 8.3.1 凭据分层与存放位置

| 凭据类型 | 存放位置 | 轮换周期 | 绝不允许 |
|---|---|---|---|
| 数据库密码 | Vault / KMS Secrets | 90 天 | 代码、镜像、环境变量文件入库 |
| Redis 密码 | Vault | 90 天 | 同上 |
| JWT 签名私钥 | KMS（非对称，私钥不出 HSM） | 90 天 | 导出到文件系统 |
| TLS 私钥 | 文件系统 600 + 可选 HSM | 90 天（ACME 自动） | 提交到仓库 |
| 对象存储 AK/SK | Vault + 实例角色（优先用 RAM Role 免密钥） | 180 天 | 前端代码、客户端 |
| CDN 签名密钥 | KMS + 边缘配置 | 90 天 | 应用代码 |
| CAPTCHA Secret | Vault | 180 天 | 前端（仅 Site Key 可公开） |
| MFA TOTP 种子 | KMS 信封加密后存 PG | 不轮换（用户绑定） | 明文存储 |
| 内部服务 mTLS 证书 | Vault PKI 自动签发 | 30 天 | 手工管理 |

### 8.3.2 应用启动时的凭据注入

```typescript
// 启动时从 Vault 拉取，注入内存；不写磁盘、不读 .env 文件
// ⚠️ 示例中所有机密均为变量引用，无任何硬编码值

async function bootstrap() {
  // 优先使用实例角色 / Workload Identity 认证，避免静态凭据
  const vaultToken = await obtainVaultTokenFromInstanceIdentity();

  const secrets = await vaultClient.read(`secret/data/media-platform/${process.env.APP_ENV}`, {
    token: vaultToken,
  });

  // 注入到运行时配置对象（内存中），不设置 process.env（会被子进程继承）
  const config = {
    database: {
      host: secrets.data.DB_HOST,
      password: secrets.data.DB_PASSWORD,
    },
    redis: { password: secrets.data.REDIS_PASSWORD },
    kms: { masterKeyId: secrets.data.KMS_MASTER_KEY_ID },
    captcha: { secret: secrets.data.CAPTCHA_SECRET },
  };

  // 启动时校验必填机密存在，缺失立即失败（而非运行到一半才发现）
  assertRequiredSecrets(config);

  await NestFactory.create(AppModule, { logger });
}
```

**为何不写 `process.env`**：环境变量对子进程可见，且可能被 `/proc/<pid>/environ` 读取（取决于权限）。机密放在应用内存的配置对象中，只有显式传入的服务能访问。

### 8.3.3 密钥轮换流程

```
① 生成新密钥（KMS 或 Vault PKI）
   ↓
② 双密钥并存期
   ├─ 新签发操作 → 使用新密钥
   └─ 验证操作 → 同时接受新旧密钥
   ↓
③ 并存期结束（覆盖最长令牌/签名有效期）
   ├─ JWT：24 小时（覆盖 Access Token 15 分钟 + 缓冲）
   ├─ CDN 签名：10 分钟（覆盖签名 URL 5 分钟）
   └─ 数据库密码：需重启应用（用滚动更新平滑过渡）
   ↓
④ 吊销旧密钥，确认无验证失败告警
   ↓
⑤ 记录到审计日志：轮换时间、操作人、影响范围
```

**自动化要求**：轮换应有定时任务或 Vault 的动态凭据机制自动完成，避免依赖人工记忆。

### 8.3.4 开发者本地环境

```bash
# .env.example —— 提交到仓库，只含占位符，无真实值
DATABASE_URL=postgresql://user:CHANGE_ME@localhost:5432/media_dev
REDIS_URL=redis://:CHANGE_ME@localhost:6379
JWT_SIGNING_KEY_ID=CHANGE_ME
KMS_MASTER_KEY_ID=CHANGE_ME
CAPTCHA_SECRET=CHANGE_ME
```

```gitignore
# .gitignore —— 强制排除
.env
.env.*
!.env.example
*.pem
*.key
*.p12
*.jks
secrets/
credentials.json
```

**强制措施**：CI 中运行密钥扫描（gitleaks / trufflehog），发现疑似密钥立即阻塞合并。同时配置 Git pre-commit hook，在提交前拦截。

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks
```

---

## 8.4 漏洞扫描与安全测试计划

### 8.4.1 扫描矩阵

| 类型 | 频率 | 工具建议 | 覆盖范围 | 阻塞条件 |
|---|---|---|---|---|
| **依赖漏洞扫描** | 每次提交 + 每日 | `npm audit`, Dependabot, Trivy | 应用依赖、容器镜像 | high/critical 阻塞合并 |
| **容器镜像扫描** | 每次构建 | Trivy, Grype | 基础镜像 + 应用层 | critical 阻塞发布 |
| **SAST 静态分析** | 每次提交 | Semgrep, CodeQL, ESLint security 插件 | 业务代码 | high 阻塞合并 |
| **密钥扫描** | 每次提交 | gitleaks, trufflehog | 全仓库 + 提交历史 | 任何命中阻塞 |
| **IaC 扫描** | 每次变更 | Checkov, tfsec, kube-score | Terraform / K8s 清单 | high 阻塞 |
| **DAST 动态扫描** | 每周 | OWASP ZAP | 预发布环境全站 | high 进修复排期 |
| **渗透测试** | 每季度 / 重大版本前 | 内部团队 + 第三方 | 认证、上传、审核、下载链路 | critical 立即修复 |
| **API 模糊测试** | 每月 | schemathesis（基于 OpenAPI） | 全部 API 端点 | 崩溃/500 阻塞 |
| **配置合规扫描** | 每周 | OpenSCAP, Lynis | 主机基线 | 高危项 7 天内修复 |
| **云配置审计** | 每日 | Prowler, ScoutSuite | 云账号权限、存储桶策略 | 公开桶立即修复 |

### 8.4.2 CI 集成配置

```yaml
# .github/workflows/security.yml
name: Security Checks

on:
  pull_request:
  schedule:
    - cron: '0 2 * * *'      # 每日凌晨全量扫描

jobs:
  secret-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }      # 全历史，检出历史泄露
      - uses: gitleaks/gitleaks-action@v2
        env:
          GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}   # 来自 CI 机密存储

  dependency-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - name: Audit dependencies
        run: npm audit --audit-level=high
      - name: Scan filesystem for vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
          ignore-unfixed: true

  sast:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Semgrep
        uses: returntocorp/semgrep-action@v1
        with:
          config: >-
            p/security-audit
            p/owasp-top-ten
            p/typescript
            p/nodejs
      - name: CodeQL
        uses: github/codeql-action/analyze@v3
        with:
          languages: javascript-typescript
          queries: security-extended

  container-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t app:${{ github.sha }} .
      - uses: aquasecurity/trivy-action@master
        with:
          image-ref: app:${{ github.sha }}
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
          format: 'sarif'
          output: 'trivy-results.sarif'
      - uses: github/codeql-action/upload-sarif@v3
        with: { sarif_file: 'trivy-results.sarif' }

  iac-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: bridgecrewio/checkov-action@master
        with:
          directory: ./infra
          framework: terraform,kubernetes
          soft_fail: false
```

### 8.4.3 针对本平台的重点测试用例

| 测试项 | 具体验证 | 期望结果 |
|---|---|---|
| 账户枚举 | 用不存在邮箱与存在邮箱登录 | 响应码、响应体、响应时间无可区分差异 |
| 时序攻击 | 对密码正确/错误/用户不存在做 1000 次计时统计 | 时间分布无显著差异（p > 0.05） |
| 暴力破解 | 单 IP 快速尝试 100 次 | 20 次后 429；账户 10 次后锁定 |
| 配额绕过 | 并发发起 20 个上传意图（配额 5） | 恰好 5 个成功，无超发 |
| 越权访问 | 用户 A 访问用户 B 的私密内容/下载 URL | 404 或 403 |
| 签名 URL 盗用 | 在 A 会话签发的 URL 用 B 身份请求 | 403（会话绑定校验生效） |
| 签名 URL 过期 | 过期后 1 秒请求 | 403 |
| 恶意文件上传 | 上传伪装成 mp4 的 HTML/SVG/可执行文件 | 415 FORMAT_MISMATCH，对象被删除 |
| 富文本 XSS | 上传含 `<script>`、`onerror`、`javascript:` 的内容 | 净化后渲染，无脚本执行 |
| 存储型 XSS | 通过下载接口获取 HTML 文件 | `Content-Disposition: attachment`，不在浏览器渲染 |
| SQL 注入 | 在全部查询参数注入 payload | 无异常、无数据泄露（参数化查询） |
| SSRF | 在上传/申诉中提供内网 URL | 拒绝，不发起请求 |
| 权限提升 | 普通用户调用 `/admin/*` 接口 | 403 |
| MFA 绕过 | 仅凭密码访问需 MFA 的管理接口 | 403 MFA_REQUIRED |
| 令牌重放 | 用已轮换的 Refresh Token | 401 + 全部会话吊销 + 告警 |
| 审核日志篡改 | 尝试 UPDATE audit_logs | 触发器异常 |
| 审核并发 | 两个审核员同时提交同一内容决策 | 一个 200，一个 409 |
| 冲突回避 | 原审核员复核自己的驳回申诉 | 403 CONFLICT_OF_INTEREST |
| 弹幕注入 | 弹幕含 HTML/脚本 | 作为纯文本渲染 |
| 限流绕过 | 用 X-Forwarded-For 伪造 IP | 取真实客户端 IP（trusted proxy 配置正确） |
| 大文件 DoS | 上传声明 100GB 的文件 | 413，未开始传输 |
| 慢速攻击 | Slowloris 慢速请求 | Nginx 超时断开 |

### 8.4.4 安全指标与 SLA

| 指标 | 目标 | 度量方式 |
|---|---|---|
| Critical 漏洞修复时间 | ≤ 24 小时 | 扫描发现到修复上线 |
| High 漏洞修复时间 | ≤ 7 天 | 同上 |
| Medium 漏洞修复时间 | ≤ 30 天 | 同上 |
| 依赖漏洞存量（high+） | 0 | 持续扫描 |
| 密钥硬编码 | 0 | gitleaks 全历史扫描 |
| 误杀率（审核） | < 5% | 申诉通过数 / 拦截总数 |
| 认证异常告警响应 | ≤ 15 分钟 | 告警到人工确认 |
| 渗透测试 critical 项 | 0 遗留 | 季度报告 |

### 8.4.5 监控与告警规则

```yaml
# alerting-rules.yml（Prometheus 告警规则形态）
groups:
  - name: security
    rules:
      - alert: HighAuthFailureRate
        expr: |
          sum(rate(auth_login_failure_total[5m])) by (instance)
          / sum(rate(auth_login_total[5m])) by (instance) > 0.3
        for: 5m
        labels: { severity: warning }
        annotations:
          summary: "登录失败率超过 30%"
          runbook: "检查是否为撞库攻击；考虑临时收紧 IP 限流阈值"

      - alert: AccountLockoutSpike
        expr: increase(account_locked_total[10m]) > 50
        for: 2m
        labels: { severity: critical }
        annotations:
          summary: "账户锁定激增，疑似定向攻击"

      - alert: RefreshTokenReplay
        expr: increase(refresh_token_replay_detected_total[5m]) > 0
        labels: { severity: critical }
        annotations:
          summary: "检测到 Refresh Token 重放，会话已吊销"
          runbook: "检查泄露来源；通知受影响用户；审查该用户近期操作"

      - alert: AuditLogIntegrityFailure
        expr: audit_log_hash_chain_invalid > 0
        labels: { severity: critical }
        annotations:
          summary: "审核日志哈希链校验失败，日志可能被篡改"
          runbook: "立即冻结写入；比对 SIEM 副本；启动安全事件响应"

      - alert: FalsePositiveRateHigh
        expr: audit_false_positive_rate > 0.05
        for: 30m
        labels: { severity: warning }
        annotations:
          summary: "审核误杀率超过 5%，考虑回滚阈值"
          runbook: "查看 rule_level_stats 定位高误杀规则；评估灰度回滚"

      - alert: UnusualDownloadVolume
        expr: |
          sum(rate(download_url_issued_total[10m])) by (user_id) > 20
        for: 5m
        labels: { severity: warning }
        annotations:
          summary: "单用户下载频率异常，疑似爬取"
          runbook: "临时限流该用户；检查是否为合法批量下载需求"

      - alert: QuotaBypassSuspected
        expr: upload_quota_exceeded_total > 0 and upload_succeeded_total > upload_quota_limit
        labels: { severity: critical }
        annotations:
          summary: "配额校验可能存在绕过"

      - alert: InternalEndpointExposed
        expr: increase(nginx_requests_total{path=~"/internal/.*", status!="403"}[5m]) > 0
        labels: { severity: critical }
        annotations:
          summary: "内部接口被非内网来源访问"
```

---

## 8.5 部署检查清单

### 8.5.1 上线前必检项

**传输与边界**
- [ ] 全站强制 HTTPS，HTTP 301 跳转
- [ ] TLS 1.3 已启用，旧协议已禁用（`ssl_protocols TLSv1.3`）
- [ ] HSTS 头已配置且 `includeSubDomains` 的子域全部支持 HTTPS
- [ ] 证书有效期 > 30 天，自动续期已配置并验证
- [ ] OCSP Stapling 生效
- [ ] 安全响应头齐全（CSP、X-Frame-Options、X-Content-Type-Options、Referrer-Policy）
- [ ] CSP 无 `unsafe-eval`；`unsafe-inline` 仅限 style（如需）

**认证与授权**
- [ ] 密码使用 Argon2id，参数 ≥ 64MiB / t=3
- [ ] 全部注册用户默认 `USER` 角色，无任何默认特权
- [ ] 特权账户强制 MFA，未激活前 RBAC 降级
- [ ] JWT 签名密钥在 KMS，应用不持有私钥
- [ ] Refresh Token 仅存哈希，轮换与重放检测已实现
- [ ] Access Token 仅存内存，不放 localStorage
- [ ] 登录接口限流、账户锁定、CAPTCHA 三层防护已启用
- [ ] 错误响应不区分"用户不存在/密码错误"

**数据与密钥**
- [ ] 无任何硬编码密钥（gitleaks 全历史扫描通过）
- [ ] 所有机密从 Vault / KMS 注入
- [ ] TOTP 种子经信封加密存储
- [ ] 数据库连接使用 TLS
- [ ] 数据库账号权限最小化（应用账号无 DDL、audit_logs 无 UPDATE/DELETE）
- [ ] 备份加密且已验证可恢复

**上传与内容安全**
- [ ] 上传直传对象存储，不经应用中转
- [ ] presigned URL 限定大小、类型、有效期
- [ ] 服务端魔数嗅探 + ffprobe 校验已启用
- [ ] 上传对象默认 `quarantine=true`，转码前 CDN 不可访问
- [ ] 未审核内容对非上传者返回 404（不泄露存在性）
- [ ] AI 预审引擎不可用时降级为全部转人工（非放行）
- [ ] 审核日志 append-only 触发器生效
- [ ] 申诉通道可用，误杀率监控与告警已配置

**防盗链与下载**
- [ ] 签名 URL 有效期 ≤ 5 分钟
- [ ] 签名绑定会话（sid），顶号后立即失效
- [ ] CDN 边缘验签生效，未签名请求 403 且不回源
- [ ] 下载频次与次数配额已启用
- [ ] 并发播放流数限制已启用
- [ ] 下载强制 `Content-Disposition: attachment`

**基础设施**
- [ ] 数据库、Redis、MQ 均无公网入站
- [ ] 内部接口限制内网来源
- [ ] 出站流量受限（白名单端口与网段）
- [ ] SSH 密钥认证，禁用密码与 root 登录
- [ ] 容器非 root 运行，只读根文件系统
- [ ] 内核安全参数已应用
- [ ] 日志脱敏（无密码、令牌、完整 IP、完整邮箱）

**可观测性**
- [ ] 审计日志接入 SIEM，保留 ≥ 24 个月
- [ ] 安全告警规则已配置并测试触发
- [ ] 应用日志含 requestId，可与 Nginx 日志关联
- [ ] 关键指标（登录失败率、误杀率、队列积压）已接入看板

**合规**
- [ ] 隐私政策与服务条款已上线，版本号记录在用户表
- [ ] 数据导出（GDPR 第 15 条）与删除（第 17 条）接口可用
- [ ] 仅采集必要数据，无过度收集
- [ ] 数据保留期限已定义并自动执行（埋点 90 天、日志 24 个月）
- [ ] 用户同意记录可证明（时间戳 + 政策版本）

### 8.5.2 灾难恢复演练

| 场景 | 演练频率 | 目标 |
|---|---|---|
| 数据库主库故障切换 | 每季度 | RTO < 2 min，RPO < 5 s |
| 对象存储区域故障 | 每半年 | CDN 切备区，播放不中断 |
| Redis 全量故障 | 每季度 | 会话可重建，限流降级但服务可用 |
| 密钥泄露模拟 | 每半年 | 90 分钟内完成全量轮换 |
| 误杀率突增 | 每季度 | 30 分钟内定位并回滚阈值 |
| 备份恢复 | 每月 | 从备份完整恢复至隔离环境并验证数据 |
