# 4. API 接口设计规范

## 4.0 通用约定

| 项 | 规范 |
|---|---|
| Base URL | `https://api.example.com/api/v1` |
| 版本策略 | 路径版本化；`v1` 至少维护 24 个月，废弃前 6 个月发 `Deprecation` 头 |
| 认证 | `Authorization: Bearer <accessToken>`；Refresh Token 走 HttpOnly Cookie |
| 内容类型 | 请求/响应均 `application/json; charset=utf-8`（上传走对象存储，不经此通道） |
| 时间格式 | ISO 8601 UTC，如 `2026-01-15T08:30:00Z` |
| ID 格式 | UUID v7 字符串 |
| 分页 | 游标分页（cursor-based），避免深分页性能塌陷 |
| 幂等 | 所有 POST 创建类接口支持 `Idempotency-Key` 请求头 |
| 追踪 | 响应必含 `X-Request-Id`，与日志 `traceId` 一致 |

### 4.0.1 统一响应包络

**成功响应**：
```json
{
  "success": true,
  "data": { },
  "meta": { "requestId": "01HQX...", "timestamp": "2026-01-15T08:30:00Z" }
}
```

**分页响应**：
```json
{
  "success": true,
  "data": [ ],
  "meta": {
    "requestId": "01HQX...",
    "pagination": {
      "nextCursor": "eyJpZCI6IjAxSFFYLi4uIn0",
      "hasMore": true,
      "limit": 20
    }
  }
}
```

> **为何用游标而非 offset**：`OFFSET 100000` 在百万行表上需扫描并丢弃 10 万行。游标分页用 `WHERE (created_at, id) < (?, ?)` 直接走索引。首页请求不带游标，返回最新 N 条。

**错误响应**：
```json
{
  "success": false,
  "error": {
    "code": "DAILY_COUNT_QUOTA_EXCEEDED",
    "message": "Daily upload count quota exceeded.",
    "detail": { "limit": 5, "used": 5, "resetAt": "2026-01-16T00:00:00Z" },
    "requestId": "01HQX..."
  }
}
```

**错误码规范**：`<DOMAIN>_<REASON>`，全大写蛇形。`message` 面向开发者（英文），前端按 `code` 查本地化文案——**服务端不返回用户可见的本地化文本**，便于前端统一管理与多语言。

### 4.0.2 HTTP 状态码映射

| 码 | 语义 | 典型 code |
|---|---|---|
| 200 | 成功 | — |
| 201 | 已创建 | — |
| 202 | 已接受（异步处理中） | `PRECHECK_IN_PROGRESS` |
| 204 | 成功无内容 | — |
| 400 | 参数错误 | `INVALID_PARAMETER` |
| 401 | 未认证/凭据无效 | `INVALID_CREDENTIALS` |
| 403 | 已认证但无权限 | `AUDIT_NOT_PERMITTED` |
| 404 | 资源不存在 | `NOT_FOUND` |
| 409 | 状态冲突 | `ALREADY_CLAIMED_OR_RESOLVED` |
| 413 | 负载过大 | `FILE_TOO_LARGE` |
| 415 | 类型不支持 | `FORMAT_MISMATCH` |
| 422 | 语义校验失败 | `CATEGORY_DEPTH_EXCEEDED` |
| 429 | 限流 | `TOO_MANY_REQUESTS` |
| 500 | 服务端错误 | `INTERNAL_ERROR`（不外泄堆栈） |
| 503 | 依赖不可用 | `AUDIT_ENGINE_UNAVAILABLE` |

---

## 4.1 认证与账户 `/auth`

### POST `/auth/register`

注册普通用户。**默认角色为 USER，不接受客户端传入 role 字段**。

**请求**：
```json
{
  "email": "user@example.com",
  "password": "correct-horse-battery-staple",
  "termsAccepted": true,
  "privacyPolicyVersion": "2026-01-01"
}
```

| 字段 | 类型 | 必填 | 校验 |
|---|---|---|---|
| email | string | 是 | RFC 5322，最长 254 |
| password | string | 是 | 最短 12 字符；不在泄露密码库中（k-anonymity 查询） |
| termsAccepted | boolean | 是 | 必须为 true |
| privacyPolicyVersion | string | 是 | 必须匹配当前生效版本 |

**响应 201**：
```json
{
  "success": true,
  "data": {
    "userId": "018d3f2a-...",
    "email": "user@example.com",
    "role": "USER",
    "status": "PENDING_VERIFICATION",
    "emailVerificationSent": true
  }
}
```

**错误**：
- `409 EMAIL_ALREADY_REGISTERED`
- `422 PASSWORD_TOO_WEAK` — `detail.requirements`
- `429 REGISTRATION_RATE_LIMITED` — 同 IP 每小时 5 次

---

### POST `/auth/login`

**请求**：
```json
{
  "email": "admin@example.com",
  "password": "...",
  "totpCode": "123456",
  "captchaToken": "..."
}
```

**响应 200（普通用户）**：
```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJFZERTQSIs...",
    "expiresIn": 900,
    "tokenType": "Bearer",
    "user": {
      "id": "018d3f2a-...",
      "email": "user@example.com",
      "role": "USER",
      "permissions": ["content:upload"],
      "mfaEnabled": false
    }
  }
}
```
同时 `Set-Cookie: refresh_token=...; HttpOnly; Secure; SameSite=Strict; Path=/api/v1/auth`

**响应 200（特权用户缺第二因子）**：
```json
{
  "success": true,
  "data": {
    "status": "MFA_REQUIRED",
    "mfaToken": "eyJ...",
    "availableMethods": ["WEBAUTHN", "TOTP"]
  }
}
```
> `mfaToken` 权限域仅为 `MFA_PENDING`，不能访问任何业务接口。

**响应 200（特权用户未绑定 MFA）**：
```json
{
  "success": true,
  "data": {
    "status": "MFA_ENROLLMENT_REQUIRED",
    "enrollToken": "eyJ...",
    "enrollmentUrl": "/settings/security/mfa"
  }
}
```

**错误**：
- `401 INVALID_CREDENTIALS` — 统一错误，不区分"用户不存在/密码错误/账户锁定"
- `403 ACCOUNT_UNAVAILABLE`

---

### POST `/auth/mfa/enroll`

绑定第二因子。需 `enrollToken` 或已认证会话。

**请求（TOTP）**：
```json
{ "method": "TOTP", "label": "Work laptop" }
```

**响应 200**：
```json
{
  "success": true,
  "data": {
    "method": "TOTP",
    "secret": "JBSWY3DPEHPK3PXP",
    "otpauthUrl": "otpauth://totp/MediaPlatform:admin@example.com?secret=...&issuer=MediaPlatform",
    "recoveryCodes": ["a1b2-c3d4-e5f6", "..."]
  }
}
```
> `secret` 与 `recoveryCodes` **仅在本次响应中返回一次**；服务端只存加密后的 secret 与哈希后的恢复码。客户端必须提示用户立即保存。

**请求（WebAuthn 注册）**：
```json
{
  "method": "WEBAUTHN",
  "credential": {
    "id": "base64url...",
    "rawId": "base64url...",
    "response": { "attestationObject": "...", "clientDataJSON": "..." },
    "type": "public-key"
  },
  "label": "YubiKey 5C"
}
```

---

### POST `/auth/mfa/verify`

**请求**：`{ "mfaToken": "eyJ...", "code": "123456", "method": "TOTP" }`

**响应 200**：同 `/auth/login` 完整会话响应，且 `user.mfaEnabled = true`、`privilegedActivatedAt` 已写入。

**错误**：
- `401 INVALID_CREDENTIALS` — 第二因子错误
- `429 MFA_ATTEMPTS_EXCEEDED` — 5 次失败，锁定 30 分钟

> **三种第二因子的接口形态**：
>
> | 因子 | 端点 | 说明 |
> |---|---|---|
> | 动态令牌（TOTP） | `POST /auth/mfa/verify` `{ method: 'TOTP', code }` | 降级方案，6 位数字 |
> | **硬件密钥**（USB/NFC 安全密钥） | `POST /auth/mfa/webauthn/verify` `{ method: 'WEBAUTHN', credential, authenticatorAttachment: 'cross-platform' }` | 抗钓鱼，优先级最高 |
> | **生物特征**（指纹 / 面容 / Windows Hello） | `POST /auth/mfa/webauthn/verify` `{ method: 'WEBAUTHN', credential, authenticatorAttachment: 'platform', userVerification: 'required' }` | 与硬件密钥共用 WebAuthn 端点，靠 `authenticatorAttachment` 与 `userVerification` 区分 |
>
> 后两者返回 `{ accessToken, user }` 或错误 `401 ADMIN_SIGNATURE_INVALID`。服务端在 `mfa_credentials.cred_type` 中记录 `'WEBAUTHN'`，并在 `credential` 元数据中保存 `attachment` 与 `uv` 标志，以便审计时区分"用的是硬件密钥还是生物特征"。
>
> **优先级建议**：前端应优先引导管理员绑定 WebAuthn（硬件密钥或平台生物特征），TOTP 仅作为未绑定 WebAuthn 时的降级通道——TOTP 易受实时钓鱼代理攻击。

---

### POST `/auth/refresh`

使用 HttpOnly Cookie 中的 Refresh Token 换取新 Access Token。**每次刷新轮换 Refresh Token**。

**响应 200**：
```json
{
  "success": true,
  "data": { "accessToken": "eyJ...", "expiresIn": 900 }
}
```

**错误**：
- `401 TOKEN_REVOKED` — 检测到重放，全部会话已吊销
- `401 TOKEN_EXPIRED`

---

### POST `/auth/logout` · POST `/auth/logout-all`

吊销当前会话 / 全部会话。响应 `204`。

### GET `/auth/sessions`

列出活跃会话，供用户自查异常登录。

**响应 200**：
```json
{
  "success": true,
  "data": [
    {
      "id": "018d3f2b-...",
      "deviceLabel": "Chrome on Windows",
      "ipPrefix": "203.0.113.0/24",
      "current": true,
      "mfaVerified": true,
      "createdAt": "2026-01-15T08:00:00Z",
      "lastSeenAt": "2026-01-15T09:00:00Z"
    }
  ]
}
```
> `ipPrefix` 仅返回 /24 网段，不返回完整 IP——隐私最小化。

### DELETE `/auth/sessions/{sessionId}`

远程吊销指定会话（如发现陌生设备）。响应 `204`。

---

## 4.2 内容与上传 `/contents`

### POST `/contents/upload-intent`

申请上传凭证。**不传文件本体**，仅声明元信息。

**请求**：
```json
{
  "filename": "documentary-4k.mp4",
  "sizeBytes": 2147483648,
  "contentType": "VIDEO",
  "sha256": "e3b0c44298fc1c149afbf4c8996fb924...",
  "categoryId": "018d3f2c-...",
  "title": "Deep Sea Documentary"
}
```

| 字段 | 类型 | 必填 | 校验 |
|---|---|---|---|
| filename | string | 是 | 最长 255；扩展名白名单 |
| sizeBytes | integer | 是 | ≤ 用户上限 |
| contentType | enum | 是 | `VIDEO` / `AUDIO` / `TEXT` |
| sha256 | string | 否 | 64 位 hex；用于去重 |
| categoryId | uuid | 否 | 必须存在且允许该类型 |
| title | string | 否 | 最长 200 |

**响应 201**：
```json
{
  "success": true,
  "data": {
    "contentId": "018d3f2d-...",
    "uploadUrl": "https://storage.example.com/uploads/...?X-Amz-Signature=...",
    "uploadMethod": "PUT",
    "expiresAt": "2026-01-15T08:45:00Z",
    "callbackUrl": "/api/v1/contents/018d3f2d-.../upload-complete",
    "maxSizeBytes": 2147483648,
    "requiredHeaders": { "Content-Type": "video/mp4" },
    "quota": { "dailyBytesLimit": 2147483648, "dailyBytesUsed": 0, "dailyCountLimit": 5, "dailyCountUsed": 1 }
  }
}
```

**错误**：
- `429 DAILY_COUNT_QUOTA_EXCEEDED` / `429 DAILY_BYTES_QUOTA_EXCEEDED`
- `413 FILE_TOO_LARGE`
- `429 UPLOAD_RATE_LIMITED`
- `415 UNSUPPORTED_MEDIA_TYPE`
- `422 CATEGORY_TYPE_NOT_ALLOWED`

> **特权用户差异**：持有 `content:upload_unlimited` 的用户，`quota` 字段返回 `null`，`maxSizeBytes` 返回特权上限。持有 `content:upload_batch` 可在一个请求中提交数组形式的多条意图（最多 50 条）。

---

### POST `/contents/{contentId}/upload-complete`

客户端直传完成后回调。服务端独立校验对象真实性。

**请求**：`{ "etag": "\"a1b2c3...\"", "actualSizeBytes": 2147483648 }`

**响应 200**：
```json
{
  "success": true,
  "data": {
    "contentId": "018d3f2d-...",
    "status": "UPLOADED",
    "transcodeQueued": true,
    "estimatedReadyAt": "2026-01-15T08:52:00Z"
  }
}
```

**幂等**：重复调用返回 `200` 且 `data.idempotent = true`，不重复投递任务。

**错误**：
- `400 OBJECT_NOT_FOUND`
- `400 SIZE_MISMATCH`
- `415 FORMAT_MISMATCH` — 魔数嗅探与声明不符
- `400 MEDIA_INVALID` — 探测失败或超时长

---

### GET `/contents`

内容列表。支持多维筛选与游标分页。

**查询参数**：

| 参数 | 类型 | 说明 |
|---|---|---|
| cursor | string | 游标（base64） |
| limit | integer | 1–100，默认 20 |
| contentType | enum | `VIDEO` / `AUDIO` / `TEXT` |
| categoryId | uuid | 含子分类 |
| tag | string | 标签 slug，可重复传 |
| q | string | 全文检索关键词 |
| sort | enum | `recent` / `popular` / `trending` |
| uploaderId | uuid | 按上传者筛选 |

**响应 200**：
```json
{
  "success": true,
  "data": [
    {
      "id": "018d3f2d-...",
      "title": "Deep Sea Documentary",
      "contentType": "VIDEO",
      "status": "PUBLISHED",
      "summary": "An exploration of...",
      "thumbnailUrl": "https://cdn.example.com/thumb/...?signature=...",
      "durationSeconds": 5420,
      "category": { "id": "018d3f2c-...", "name": "Documentary", "slug": "documentary" },
      "tags": ["ocean", "nature"],
      "stats": { "views": 12400, "likes": 320, "comments": 45, "downloads": 88 },
      "uploader": { "id": "018d3f2a-...", "displayName": "ocean_films" },
      "publishedAt": "2026-01-14T12:00:00Z"
    }
  ],
  "meta": { "pagination": { "nextCursor": "eyJ...", "hasMore": true, "limit": 20 } }
}
```

> **注意**：列表接口**不返回** `precheckDetail`、`riskScore`、`reviewedBy`——审核内部字段仅对持有 `content:audit` 的用户通过管理端接口可见。

> **状态过滤的实现约束**：列表、详情、播放、下载四个入口的"是否可见"判定**必须统一为 `status IN ('PUBLISHED','AUTO_PUBLISHED')`**。`AUTO_PUBLISHED` 是系统自动放行的已发布终态，与人工放行的 `PUBLISHED` 对用户完全等价。任何写成 `status = 'PUBLISHED'` 的查询都会导致自动放行的内容凭空消失——这是本设计中最容易在实现期踩中的陷阱，建议在公共查询构造器中封装为 `wherePubliclyVisible()` 而非各处手写。

---

### GET `/contents/{contentId}`

**响应 200（已发布）**：在列表字段基础上增加：
```json
{
  "description": "Full description...",
  "renditions": [
    { "profile": "1080p", "width": 1920, "height": 1080, "bitrateKbps": 5000 },
    { "profile": "720p", "width": 1280, "height": 720, "bitrateKbps": 2800 }
  ],
  "subtitles": [{ "lang": "zh-CN", "label": "中文" }],
  "downloadable": true,
  "visibility": "PUBLIC"
}
```

**响应 403（未通过审核）**：
```json
{
  "success": false,
  "error": {
    "code": "CONTENT_BLOCKED",
    "message": "This content was blocked by automated review.",
    "detail": {
      "contentId": "018d3f2d-...",
      "status": "BLOCKED",
      "appealable": true,
      "appealUrl": "/api/v1/appeals",
      "appealDeadline": "2026-02-14T08:30:00Z"
    }
  }
}
```
> 仅上传者本人可见此详情；其他用户得到 `404 NOT_FOUND`（避免泄露被拦截内容的存在）。

---

### PATCH `/contents/{contentId}`

更新元信息（标题、描述、分类、标签、可见性）。仅上传者或持有 `content:delete_any` 者可调用。

**请求**：
```json
{ "title": "New Title", "description": "...", "categoryId": "018d3f2c-...", "tags": ["ocean"], "visibility": "UNLISTED" }
```

**响应 200**：返回更新后对象。

> **序列化必须按调用者权限分级**：`PATCH` 的响应与 `GET /contents/{id}` 共用同一序列化器，因此**必须依据调用者是否持有 `content:audit` 决定字段集**。上传者调用时返回公开字段集（不含 `riskScore`、`reviewedBy`、`precheckDetail`）；审核员调用时返回完整字段集。这一点容易被忽略——若实现时统一用管理端序列化器，会把审核内部字段泄露给普通用户。

> **审核约束**：修改标题或描述会触发重新预审（文本内容扫描），若命中高风险则状态回退至 `PENDING_MANUAL`。响应中通过 `data.recheckTriggered = true` 告知前端。

---

### DELETE `/contents/{contentId}`

软删除。响应 `204`。异步清理转码产物与索引。

---

### GET `/contents/{contentId}/playback`

获取播放地址（含短期签名）。

**响应 200**：
```json
{
  "success": true,
  "data": {
    "manifestUrl": "https://cdn.example.com/hls/018d3f2d/master.m3u8?signature=...&expires=1737014400&sid=...",
    "dashManifestUrl": "https://cdn.example.com/dash/018d3f2d/manifest.mpd?signature=...",
    "renditions": [
      { "profile": "1080p", "width": 1920, "height": 1080, "bitrateKbps": 5000 },
      { "profile": "480p", "width": 854, "height": 480, "bitrateKbps": 1200 }
    ],
    "subtitles": [{ "lang": "zh-CN", "url": "https://cdn.example.com/subs/zh.vtt?signature=..." }],
    "resumePositionMs": 120000,
    "sessionId": "play_018d3f2e-...",
    "expiresAt": "2026-01-15T09:30:00Z",
    "danmakuToken": "eyJ..."
  }
}
```

**错误**：
- `403 PLAYBACK_NOT_PERMITTED`
- `429 TOO_MANY_CONCURRENT_STREAMS` — 同账户并发流超限

---

### GET `/contents/{contentId}/download`

**查询参数**：`assetKind` = `SOURCE` | `AUDIO` | `DOCUMENT`

**响应 200**：
```json
{
  "success": true,
  "data": {
    "url": "https://cdn.example.com/download/...?signature=...&expires=...&sid=...",
    "expiresAt": "2026-01-15T08:35:00Z",
    "maxDownloads": 3,
    "maxBytes": 2147483648,
    "filename": "Deep-Sea-Documentary.mp4",
    "sizeBytes": 2147483648,
    "notice": "This URL is bound to your session and expires shortly."
  }
}
```

> **签名方案说明（重要）**：返回的 URL 走**应用侧 CDN 边缘验签**通道，签名中绑定了 `sid`（播放/下载会话）与 `uid`。URL 被转发给他人后，对方无此会话，边缘验签通过但会话校验失败 → 403。
>
> 若部署时不具备 CDN 边缘计算能力，退化为**直连对象存储的 presigned URL**。此时必须明确：**S3/OSS 的 presigned URL 无法绑定用户身份**，在有效期内任何人都可使用。对应的补偿控制是：TTL 缩短到 ≤ 5 分钟、单 grant 限定 `maxBytes`、以及严格频次限制。采用此退化方案时，第 8 节"签名 URL 盗用"测试用例需相应调整为"验证 TTL 与配额生效"，而非"验证转发无效"。

**错误**：
- `429 DOWNLOAD_RATE_LIMITED` — 每小时每内容 5 次
- `429 DOWNLOAD_BANDWIDTH_QUOTA_EXCEEDED` — 当日累计下载字节超限（默认 10 GiB）
- `403 DOWNLOAD_NOT_PERMITTED`

---

### POST `/contents/{contentId}/progress`

上报播放进度（续播用）。

**请求**：`{ "positionMs": 120000, "durationMs": 5420000 }` → 响应 `204`

---

## 4.3 审核管理 `/admin/audit`（需 `content:audit`）

### GET `/admin/audit/queue`

**查询参数**：`cursor`、`limit`、`riskLevel`（`MEDIUM`/`HIGH`）、`contentType`、`sort`（`risk_desc` / `oldest_first`）

**响应 200**：
```json
{
  "success": true,
  "data": [
    {
      "contentId": "018d3f2d-...",
      "contentType": "VIDEO",
      "title": "Deep Sea Documentary",
      "riskScore": 62,
      "riskLevel": "MEDIUM",
      "precheckSummary": {
        "engines": ["nsfw", "asr_keyword"],
        "hitCount": 3,
        "topFindings": [
          { "engine": "asr_keyword", "ruleId": "RULE-1042", "severity": "WARN", "timeMs": 48200 },
          { "engine": "nsfw", "score": 0.61, "frameTimeMs": 120500 }
        ]
      },
      "uploader": { "id": "018d3f2a-...", "displayName": "ocean_films", "priorRejections": 0, "accountAgeDays": 180 },
      "thumbnailUrl": "https://cdn.example.com/thumb/...?signature=...",
      "previewUrl": "https://cdn.example.com/preview/...?signature=...",
      "queuedAt": "2026-01-15T08:35:00Z",
      "waitingMinutes": 12,
      "claimedBy": null
    }
  ],
  "meta": { "pagination": { "nextCursor": "eyJ...", "hasMore": true, "limit": 20 },
            "queueStats": { "pending": 142, "avgWaitMinutes": 18, "oldestWaitMinutes": 95 } }
}
```

> `previewUrl` 为低码率预览（480p、加水印），审核员查看受限内容不需下载源文件。

---

### POST `/admin/audit/{contentId}/claim`

乐观锁领取任务，防止多人重复处理。

**响应 200**：`{ "contentId": "...", "claimedBy": "018d3f2b-...", "claimExpiresAt": "..." }`

**错误**：`409 ALREADY_CLAIMED_OR_RESOLVED`

---

### POST `/admin/audit/{contentId}/decision`

**请求**：
```json
{
  "decision": "REJECT",
  "reason": "Contains unlicensed copyrighted material at 08:05.",
  "reasonCategory": "COPYRIGHT",
  "banUploader": false,
  "ruleFeedback": [
    { "ruleId": "RULE-1042", "verdict": "FALSE_POSITIVE" }
  ]
}
```

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| decision | enum | 是 | `APPROVE` / `REJECT` |
| reason | string | REJECT 必填 | 最长 1000，会展示给用户 |
| reasonCategory | enum | 否 | `COPYRIGHT` / `VIOLENCE` / `SEXUAL` / `SPAM` / `MISINFORMATION` / `OTHER` |
| banUploader | boolean | 否 | 需 `user:ban` 权限 |
| ruleFeedback | array | 否 | 审核员对 AI 规则判定的反馈，回流训练 |

**响应 200**：
```json
{
  "success": true,
  "data": {
    "contentId": "018d3f2d-...",
    "status": "REJECTED",
    "reviewedBy": "018d3f2b-...",
    "reviewedAt": "2026-01-15T08:50:00Z",
    "auditLogId": 8821043,
    "uploaderNotified": true
  }
}
```

**错误**：`403 AUDIT_NOT_PERMITTED` / `409 ALREADY_CLAIMED_OR_RESOLVED` / `403 BAN_NOT_PERMITTED`

---

### GET `/admin/audit/logs`

查询审核日志（需 `audit:read_logs`）。

**查询参数**：`contentId`、`actorId`、`action`、`from`、`to`、`cursor`、`limit`

**响应 200**：
```json
{
  "success": true,
  "data": [
    {
      "id": 8821043,
      "contentId": "018d3f2d-...",
      "actorType": "MODERATOR",
      "actor": { "id": "018d3f2b-...", "displayName": "mod_alice" },
      "action": "MANUAL_REJECT",
      "fromStatus": "PENDING_MANUAL",
      "toStatus": "REJECTED",
      "riskScore": 62,
      "reason": "Contains unlicensed copyrighted material at 08:05.",
      "engineName": "multi-modal-precheck",
      "engineVersion": "2026.01.3",
      "createdAt": "2026-01-15T08:50:00Z",
      "integrityVerified": true
    }
  ],
  "meta": { "pagination": { "nextCursor": "eyJ...", "hasMore": false, "limit": 20 } }
}
```

> `integrityVerified` 表示该记录的哈希链校验通过。定期批量校验，异常时置 `false` 并告警。

---

### GET `/admin/audit/metrics`

审核效果看板数据。

**响应 200**：
```json
{
  "success": true,
  "data": {
    "period": { "from": "2026-01-08", "to": "2026-01-15" },
    "throughput": { "submitted": 12480, "autoPublished": 10240, "manualReviewed": 2010, "autoBlocked": 230 },
    "manualReview": { "approved": 1580, "rejected": 430, "avgDecisionMinutes": 14.2 },
    "appeals": { "submitted": 96, "approved": 41, "rejected": 55 },
    "falsePositiveRate": 0.0426,
    "falsePositiveRateTarget": 0.05,
    "ruleLevelStats": [
      { "ruleId": "RULE-1042", "hits": 340, "appeals": 38, "falsePositiveRate": 0.082, "alert": true }
    ],
    "alerting": [
      { "code": "RULE_FALSE_POSITIVE_SPIKE", "ruleId": "RULE-1042", "message": "FPR 8.2% exceeds 5% threshold." }
    ]
  }
}
```
> `falsePositiveRate`（误杀率）是本平台最重要的质量指标，图表化后直接驱动阈值调优。

---

## 4.4 申诉 `/appeals`

### POST `/appeals`

**请求**：
```json
{
  "contentId": "018d3f2d-...",
  "reason": "The flagged segment is a licensed educational clip with written permission from the copyright holder.",
  "evidenceFiles": [
    { "filename": "permission-letter.pdf", "sizeBytes": 240128, "sha256": "..." }
  ]
}
```

**响应 201**：
```json
{
  "success": true,
  "data": {
    "appealId": "018d3f2f-...",
    "contentId": "018d3f2d-...",
    "status": "PENDING",
    "contentStatus": "APPEALING",
    "estimatedReviewHours": 24,
    "submittedAt": "2026-01-15T09:00:00Z"
  }
}
```

**错误**：
- `409 APPEAL_ALREADY_OPEN`
- `409 APPEAL_REASON_DUPLICATE` — 与历史申诉理由实质相同；`detail.hint` 提示补充新证据
- `400 CONTENT_NOT_APPEALABLE`
- `410 APPEAL_WINDOW_EXPIRED` — 超过 30 天申诉窗口；`detail` 含 `supportUrl` 引导人工客服
- `403 FORBIDDEN` — 非本人内容
- `429 APPEAL_RATE_LIMITED` — 每用户每日 3 次（仅成功提交计数）

---

### GET `/appeals` · GET `/appeals/{appealId}`

用户查自己的申诉 / 审核员查详情。

**响应 200**：
```json
{
  "success": true,
  "data": {
    "id": "018d3f2f-...",
    "contentId": "018d3f2d-...",
    "status": "APPROVED",
    "reason": "...",
    "reviewer": { "id": "018d3f2b-...", "displayName": "mod_bob" },
    "reviewNote": "License verified with rights holder.",
    "resolvedAt": "2026-01-15T18:00:00Z",
    "contentRestored": true,
    "timeline": [
      { "at": "2026-01-15T09:00:00Z", "event": "SUBMITTED" },
      { "at": "2026-01-15T10:00:00Z", "event": "UNDER_REVIEW" },
      { "at": "2026-01-15T18:00:00Z", "event": "APPROVED" }
    ]
  }
}
```

---

### POST `/appeals/{appealId}/withdraw`

用户主动撤回尚未审结的申诉。

**响应 200**：
```json
{
  "success": true,
  "data": {
    "appealId": "018d3f2f-...",
    "status": "WITHDRAWN",
    "contentStatus": "BLOCKED",
    "withdrawnAt": "2026-01-15T12:00:00Z"
  }
}
```

> 撤回后内容回到 `BLOCKED` 或 `REJECTED`（取决于申诉前的状态），申诉入口重新开放，用户可在申诉窗口内以新理由再次提交。
>
> **配额规则**：撤回**不退还**已消耗的每日申诉额度（额度在申诉成功落库时即计入），但**失败的提交不消耗额度**——配额自增发生在所有前置校验通过之后，因此 `409`/`400`/`410` 被拒的请求不会白扣次数。

**错误**：
- `403 FORBIDDEN` — 非本人申诉
- `409 APPEAL_ALREADY_RESOLVED` — 申诉已通过或已驳回，无法撤回

---

### POST `/admin/appeals/{appealId}/resolve`（需 `content:audit`）

**请求**：`{ "decision": "APPROVE", "note": "License verified." }`

**响应 200**：返回更新后的申诉对象，含 `contentRestored: true` 与新的 `contentStatus`。

**错误**：`403 CONFLICT_OF_INTEREST` — 原审核员不得复核自己驳回的申诉。

---

## 4.5 分类与标签 `/categories` `/tags`

### GET `/categories`

返回分类树。默认返回完整树（分类总量有限，一次返回避免多次往返）。

**查询参数**：`depth`（最大深度）、`parentId`、`contentType`、`flat`（true 返回扁平列表）

**响应 200**：
```json
{
  "success": true,
  "data": [
    {
      "id": "018d3f2c-...",
      "slug": "documentary",
      "name": "Documentary",
      "depth": 0,
      "allowedTypes": ["VIDEO", "AUDIO"],
      "contentCount": 1240,
      "sortOrder": 10,
      "children": [
        { "id": "018d3f30-...", "slug": "nature", "name": "Nature", "depth": 1, "contentCount": 320, "children": [] }
      ]
    }
  ]
}
```

---

### POST `/admin/categories`（需 `system:config_audit`）

**请求**：`{ "parentId": "018d3f2c-...", "slug": "nature", "name": "Nature", "allowedTypes": ["VIDEO"], "sortOrder": 20 }`

**响应 201**：返回创建的分类对象。

**错误**：
- `409 CATEGORY_SLUG_CONFLICT`
- `422 CATEGORY_DEPTH_EXCEEDED` — 超过 6 级
- `422 CATEGORY_CYCLE_DETECTED` — 移动时目标在其子树内

---

### PATCH `/admin/categories/{categoryId}` · DELETE `/admin/categories/{categoryId}`

更新（含移动，会批量更新子孙 `path`）/ 软删除（有子分类或内容时返回 `409 CATEGORY_NOT_EMPTY`）。

---

### GET `/tags`

**查询参数**：`q`（模糊搜索）、`limit`、`sort`（`usage` / `name`）

**响应 200**：`{ "data": [{ "id": 1042, "slug": "ocean", "name": "Ocean", "usageCount": 8420, "isOfficial": true }] }`

---

## 4.6 用户与配额 `/me` `/admin/users`

### GET `/me`

**响应 200**：
```json
{
  "success": true,
  "data": {
    "id": "018d3f2a-...",
    "email": "user@example.com",
    "displayName": "ocean_films",
    "role": "USER",
    "status": "ACTIVE",
    "permissions": ["content:upload"],
    "mfaEnabled": false,
    "privilegedActivatedAt": null,
    "quota": {
      "dailyBytesLimit": 2147483648,
      "dailyBytesUsed": 536870912,
      "dailyCountLimit": 5,
      "dailyCountUsed": 2,
      "resetAt": "2026-01-16T00:00:00Z"
    },
    "createdAt": "2025-07-01T00:00:00Z",
    "termsAcceptedAt": "2025-07-01T00:00:00Z"
  }
}
```

### PATCH `/me`

更新 `displayName`、`locale`、`readingPreferences`（字号、行距、主题——阅读器个性化）。**不允许修改 `role`、`status`、`quota`**。

### GET `/me/data-export` · DELETE `/me`

GDPR 合规接口：
- `GET /me/data-export` → `202`，异步生成数据包，完成后邮件发送限时下载链接（24 小时有效）
- `DELETE /me` → `202`，触发 30 天冷静期，期间可撤销；到期后执行不可逆匿名化（保留审核日志的法律留痕，但解除与身份的关联）

### GET `/admin/users`（需 `user:read_pii`）

**查询参数**：`role`、`status`、`q`（邮箱/ID）、`cursor`、`limit`

**响应 200**：返回用户列表；**含 `email` 等 PII 字段，访问会写审计日志**。

### POST `/admin/users/{userId}/ban`（需 `user:ban`）

**请求**：`{ "reason": "Repeated copyright violations.", "duration": "PERMANENT", "notifyUser": true }`

**响应 200**：`{ "userId": "...", "status": "BANNED", "bannedAt": "...", "allSessionsRevoked": true, "contentHidden": 12 }`
> 封禁同时吊销全部会话并隐藏该用户已发布内容。

### POST `/admin/users/{userId}/roles`（需 `system:config_audit`）

**请求**：`{ "roleCode": "MODERATOR", "expiresAt": "2026-07-15T00:00:00Z" }`

**响应 200**：`{ "userId": "...", "roles": ["USER", "MODERATOR"], "requiresMfaEnrollment": true }`
> 授予特权角色后，该用户下次登录必须完成 MFA 绑定才能激活特权。

---

## 4.7 弹幕与评论 `/contents/{id}/danmaku` `/comments`

### GET `/contents/{contentId}/danmaku`

**查询参数**：`fromMs`、`toMs`（时间区间，支持分段加载）、`limit`

**响应 200**：
```json
{
  "success": true,
  "data": [
    { "id": "665f...", "videoTimeMs": 48200, "text": "Amazing footage", "color": "#FFFFFF", "position": "scroll", "likeCount": 12, "isMine": false }
  ],
  "meta": { "total": 8420, "returned": 200, "truncated": true }
}
```
> 弹幕量大时按时间窗分段返回；`truncated` 提示前端该区间弹幕已被采样。

### POST `/contents/{contentId}/danmaku`

**请求**：`{ "text": "...", "videoTimeMs": 48200, "color": "#FFFFFF", "position": "scroll" }`

**响应 201**。**弹幕文本同样经过敏感词与 AI 过滤**，命中则返回 `422 DANMAKU_CONTENT_REJECTED`（不进入审核队列，直接拒绝并提示）。

### POST `/comments` · GET `/contents/{contentId}/comments` · DELETE `/comments/{id}`

评论 CRUD。支持两级嵌套（`parentId`）。删除自己评论或由持有 `content:delete_any` 者删除任意评论。

---

## 4.8 系统配置 `/admin/config`（需 `system:config_audit`）

### GET `/admin/config/audit-policy`

**响应 200**：
```json
{
  "success": true,
  "data": {
    "global": {
      "lowThreshold": 30,
      "highThreshold": 70,
      "autoPublishEnabled": true,
      "rolloutPercent": 100
    },
    "overrides": [
      { "categoryId": "018d3f2c-...", "categoryName": "News & Politics", "lowThreshold": 15, "highThreshold": 55 },
      { "categoryId": "018d3f30-...", "categoryName": "Education", "lowThreshold": 45, "highThreshold": 80 }
    ],
    "hardBlockRules": ["RULE-0001", "RULE-0002"],
    "updatedAt": "2026-01-10T00:00:00Z",
    "updatedBy": "018d3f2b-..."
  }
}
```

### PATCH `/admin/config/audit-policy`

**请求**：`{ "global": { "lowThreshold": 25 }, "overrides": [{ "categoryId": "...", "lowThreshold": 15 }], "changeReason": "Adjusting for Q1 false-positive spike." }`

**响应 200**：返回新配置。

> **强制要求**：`changeReason` 必填。每次阈值变更写入 `audit_logs`（`action = 'CONFIG_CHANGED'`），记录变更前后值，便于回溯"某天误杀率突然上升是因为谁改了阈值"。

---

## 4.9 内部接口（服务间，不经公网）

| 端点 | 说明 |
|---|---|
| `POST /internal/precheck/callback` | AI 引擎回调，HMAC 签名校验 |
| `POST /internal/transcode/callback` | 转码完成回调 |
| `GET /internal/health` | 存活探针 |
| `GET /internal/ready` | 就绪探针（检查 PG/Redis/MQ 连通性） |
| `GET /internal/metrics` | Prometheus 指标 |

**保护措施**：
- 独立端口，仅集群内网可达（NetworkPolicy 限制）
- mTLS 双向认证
- 请求体 HMAC 签名 + 时间戳（防重放，窗口 5 分钟）

---

## 4.10 接口安全清单

| 接口类别 | 限流策略 | 额外防护 |
|---|---|---|
| `/auth/login` | 20 次/IP/分钟 + 10 次/账户 | CAPTCHA、账户锁定、统一错误码 |
| `/auth/register` | 5 次/IP/小时 | 邮箱验证、泄露密码库检查 |
| `/auth/refresh` | 60 次/会话/小时 | 轮换 + 重放检测 |
| `/contents/upload-intent` | 10 次/用户/小时 | 配额、大小、类型三重校验 |
| `/contents/*/download` | 5 次/用户/内容/小时 | 签名 URL + 次数配额 |
| `/contents/*/playback` | 30 次/用户/小时 | 并发流限制 |
| `/admin/*` | 300 次/用户/分钟 | 强制 MFA + 全量审计日志 |
| `/appeals` | 3 次/用户/日 | 防止申诉通道被滥用 |
| `/internal/*` | 不限（内网） | mTLS + HMAC + IP 白名单 |

**所有接口通用防护**：
1. 请求体大小限制（JSON 接口 1 MB，内部接口 10 MB）
2. JSON Schema 严格校验，**拒绝未声明字段**（防 mass assignment 提权）
3. 输出字段白名单序列化（`sanitize()`），杜绝 `passwordHash` 等内部字段外泄
4. CORS 白名单域名 + 不允许 `credentials` 跨域
5. 危险操作（封禁、删除、改配置）强制二次确认参数 `confirm: true`
