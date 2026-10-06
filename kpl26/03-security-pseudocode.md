# 3. 核心安全逻辑伪代码

> **约束遵守声明**：本文所有代码示例中**不存在任何硬编码密钥或密码**。所有机密通过 KMS / Vault / SecretManager 在运行时注入，代码中仅出现**引用**（如 `CONFIG.KMS_MASTER_KEY_ID`、`SecretManager.get('CAPTCHA_SECRET_KEY')`）或占位符（如 `CHANGE_ME`）。私钥材料一律标记为不可导出，应用侧无从持有。

## 3.1 密码哈希：Argon2id

### 3.1.1 参数选择

| 参数 | 值 | 依据 |
|---|---|---|
| 算法 | Argon2id | 兼顾侧信道抗性与 GPU 抗性（OWASP 首选） |
| `memoryCost` | 65536 KiB (64 MiB) | OWASP 推荐下限，显著提高 GPU 破解成本 |
| `timeCost` | 3 | 与内存参数配合，目标单次哈希 100–500ms |
| `parallelism` | 4 | 匹配典型服务端 vCPU |
| `hashLength` | 32 bytes | 256 bit 输出 |
| `salt` | 16 bytes 随机 | 每条记录独立，绝不复用 |

**参数升级策略**：`password_algo` 字段记录算法标识。用户成功登录时，若发现哈希参数低于当前标准，则用新参数重新哈希并更新——密码明文仅在登录那一刻存在于内存，这是唯一的透明升级时机。

```typescript
// 声明式配置；数值是公开参数，不是机密
const ARGON2_PARAMS = {
  type: argon2.argon2id,
  memoryCost: 65536,
  timeCost: 3,
  parallelism: 4,
  hashLength: 32,
} as const;

// 生成密码哈希
async function hashPassword(plaintext: string): Promise<string> {
  // salt 由 argon2 库内部用 CSPRNG 生成，不需要手工拼接
  return argon2.hash(plaintext, ARGON2_PARAMS);
}

// 校验密码；同时返回是否需要升级参数
async function verifyPassword(
  storedHash: string,
  plaintext: string
): Promise<{ valid: boolean; needsRehash: boolean }> {
  // 先做格式校验，避免对畸形输入抛异常泄露信息
  if (!storedHash.startsWith('$argon2id$')) {
    // 旧算法（如 bcrypt/argon2i）走兼容路径，见 3.1.2
    return verifyLegacy(storedHash, plaintext);
  }

  let valid = false;
  try {
    valid = await argon2.verify(storedHash, plaintext);
  } catch {
    // 任何解析异常都视为校验失败，绝不向上抛出细节
    valid = false;
  }

  return {
    valid,
    // 仅在验证成功时才判断重哈希，避免为无效登录做额外工作
    needsRehash: valid && argon2.needsRehash(storedHash, ARGON2_PARAMS),
  };
}
```

### 3.1.2 算法迁移（bcrypt → Argon2id）

提示词允许 bcrypt (cost ≥ 12) 作为备选。若存量系统使用 bcrypt，采用**登录时透明迁移**，不强制用户改密：

```typescript
async function verifyLegacy(storedHash: string, plaintext: string) {
  if (storedHash.startsWith('$2b$') || storedHash.startsWith('$2a$')) {
    // 从哈希串中解析 cost，低于 12 则拒绝并要求重置密码
    const cost = parseInt(storedHash.split('$')[2], 10);
    if (cost < 12) {
      return { valid: false, needsRehash: false, mustReset: true };
    }
    const valid = await bcrypt.compare(plaintext, storedHash);
    // 验证通过 → 用 Argon2id 重新哈希，落库并标记 algo 变更
    return { valid, needsRehash: valid, migrateTo: 'argon2id' };
  }
  return { valid: false, needsRehash: false };
}
```

---

## 3.2 管理员登录验证流程（含双因子与加密比对）

### 3.2.1 设计要点

1. **统一错误响应**：无论"用户不存在"还是"密码错误"，返回同样的错误码与耗时，防止账户枚举与时序攻击。
2. **恒定时间比对**：对非密码类的机密比对（恢复码、API Key）必须用 `timingSafeEqual`，不能用 `===`。
3. **特权激活是独立步骤**：默认注册为 `USER`。即使数据库中被改为 `ADMIN`，未完成双因子的账户 `privileged_activated_at` 为 NULL，RBAC 判定时降级为普通用户。
4. **IP 限流 + 账户锁定 + CAPTCHA** 三层防护并行，不是二选一。

### 3.2.2 主流程伪代码

```
FUNCTION adminLogin(request):
    INPUT: email, password, totpCode?, captchaToken?, clientIp

    # ── 第 0 层：IP 维度限流（先于任何数据库查询，防扫描） ────────────
    ipHash = HMAC_SHA256(key = SECRET_RATE_LIMIT_PEPPER, msg = clientIp)
    # 注意：pepper 来自环境变量，用于防止哈希被彩虹表反推
    ipAttempts = Redis.INCR("login:ip:{ipHash}:" + currentMinute())
    Redis.EXPIRE("login:ip:{ipHash}:" + currentMinute(), 60)

    IF ipAttempts > CONFIG.AUTH_MAX_ATTEMPTS_PER_IP_PER_MINUTE (default 20):
        AuditLog.record(event = "LOGIN_RATE_LIMITED_IP",
                        actorType = "SYSTEM", ipHash = ipHash, severity = "WARN")
        RETURN error(429, "TOO_MANY_REQUESTS", retryAfter = 60)
        # 恒定响应时间：即使拒绝也 sleep 到统一阈值
        SLEEP_UNTIL(UNIFORM_RESPONSE_TIME)

    # ── 第 1 层：CAPTCHA（在连续失败后触发，避免正常用户被打扰） ───────
    recentFailures = Redis.GET("login:fail:{ipHash}") OR 0
    IF recentFailures >= CONFIG.CAPTCHA_TRIGGER_THRESHOLD (default 3):
        IF NOT verifyCaptcha(captchaToken, clientIp):
            RETURN error(400, "CAPTCHA_REQUIRED")

    # ── 第 2 层：取用户（含恒定时间填充） ──────────────────────────────
    user = DB.users.findOne(email = normalize(email), deletedAt IS NULL)

    IF user IS NULL:
        # 关键：执行一次等价的 Argon2 计算，消除"用户不存在"的时序差异
        AWAIT argon2.verify(DUMMY_HASH_CONSTANT, password)
        recordFailedAttempt(ipHash, email)
        RETURN error(401, "INVALID_CREDENTIALS")

    # ── 第 3 层：账户锁定检查 ────────────────────────────────────────
    IF user.lockedUntil IS NOT NULL AND user.lockedUntil > now():
        AuditLog.record(event = "LOGIN_LOCKED_ACCOUNT",
                        actorType = "USER", actorId = user.id, ipHash = ipHash)
        # 同样返回统一错误，不暴露"账户已锁定"这一信息给攻击者
        # 但不锁定真实用户——通过邮件通知本人
        RETURN error(401, "INVALID_CREDENTIALS")

    IF user.status IN ('BANNED', 'DELETED'):
        RETURN error(403, "ACCOUNT_UNAVAILABLE")

    # ── 第 4 层：密码验证（Argon2id 加密比对） ────────────────────────
    # argon2.verify 内部为恒定时间比较，不会因前缀匹配长度泄露信息
    verifyResult = AWAIT verifyPassword(user.passwordHash, password)

    IF NOT verifyResult.valid:
        newFailCount = Redis.INCR("login:fail:{ipHash}")
        Redis.EXPIRE("login:fail:{ipHash}", 3600)

        # 账户维度计数（与 IP 维度独立，防分布式撞库）
        user.failedLoginCount = user.failedLoginCount + 1
        IF user.failedLoginCount >= CONFIG.ACCOUNT_LOCK_THRESHOLD (default 10):
            # 指数退避锁定：第 1 次锁 5min，第 2 次 15min，第 3 次 60min
            lockDuration = backoffDuration(user.failedLoginCount)
            user.lockedUntil = now() + lockDuration
            AuditLog.record(event = "ACCOUNT_LOCKED", actorType = "USER",
                            actorId = user.id,
                            detail = { lockDuration: toString(lockDuration) })
            Notify.send(user.email, template = "ACCOUNT_LOCKED_ALERT")

        DB.users.update(user, { failedLoginCount, lockedUntil })
        RETURN error(401, "INVALID_CREDENTIALS")

    # ── 第 5 层：密码正确 → 重置失败计数 + 参数升级 ────────────────────
    IF verifyResult.needsRehash:
        newHash = AWAIT hashPassword(password)
        DB.users.update(user, {
            passwordHash: newHash,
            passwordAlgo: 'argon2id',
            passwordChangedAt: now()   # 注意：参数升级不强制其他会话登出
        })

    DB.users.update(user, { failedLoginCount: 0, lockedUntil: NULL })
    Redis.DEL("login:fail:{ipHash}")

    # ── 第 6 层：特权判定 → 强制双因子 ────────────────────────────────
    permissions = RBAC.resolvePermissions(user.id)
    IS_PRIVILEGED = permissions CONTAINS ANY OF
        ['content:audit', 'user:ban', 'system:config_audit', 'content:delete_any']

    IF IS_PRIVILEGED:
        # 特权账户必须完成第二因子，否则不放行
        IF NOT user.mfaEnabled:
            # 首次登录的管理员必须立即绑定 MFA，签发受限令牌仅用于绑定
            tempToken = issueToken(user, scope = 'MFA_ENROLLMENT_ONLY', ttl = 10min)
            AuditLog.record(event = "PRIVILEGED_LOGIN_MFA_REQUIRED",
                            actorType = "USER", actorId = user.id)
            RETURN { status: "MFA_ENROLLMENT_REQUIRED", enrollToken: tempToken }

        IF totpCode IS NULL:
            # 密码已通过但缺第二因子：签发半成品令牌，不授予任何权限
            mfaToken = issueToken(user, scope = 'MFA_PENDING', ttl = 5min)
            RETURN { status: "MFA_REQUIRED", mfaToken }

        # 校验第二因子：优先 WebAuthn（抗钓鱼），降级 TOTP
        mfaResult = AWAIT verifySecondFactor(user, totpCode)
        IF NOT mfaResult.valid:
            # 第二因子失败同样计数，防止绕过密码后无限爆破 TOTP
            Redis.INCR("mfa:fail:{user.id}")
            IF Redis.GET("mfa:fail:{user.id}") > CONFIG.MFA_MAX_FAILURES (default 5):
                user.lockedUntil = now() + 30min
                Notify.send(user.email, "MFA_REPEATED_FAILURE_ALERT")
            AuditLog.record(event = "MFA_FAILED", actorType = "USER", actorId = user.id)
            RETURN error(401, "INVALID_CREDENTIALS")

        # 双因子全部通过 → 激活特权身份
        IF user.privilegedActivatedAt IS NULL:
            DB.users.update(user, { privilegedActivatedAt: now() })
            AuditLog.record(event = "PRIVILEGE_ACTIVATED", actorType = "USER",
                            actorId = user.id,
                            detail = { method: mfaResult.method })

    # ── 第 7 层：签发会话 ────────────────────────────────────────────
    session = createSession(user, {
        mfaVerified: IS_PRIVILEGED,
        ipHash:       ipHash,
        userAgentHash: HMAC_SHA256(pepper, request.userAgent)
    })

    accessToken  = signJWT(
        payload = {
            sub: user.id,
            role: user.role,
            sid: session.id,
            mfa: IS_PRIVILEGED,
            # 权限快照，避免每请求查库；变更时通过 Pub/Sub 主动吊销
            perms: permissions
        },
        expiresIn = '15m',
        algorithm = 'EdDSA'      # Ed25519，密钥由 KMS 提供
    )

    refreshToken = generateOpaqueToken()   # 随机 256bit，非 JWT
    DB.sessions.update(session, {
        # 只存哈希：数据库泄露也无法复用令牌
        refreshTokenHash: SHA256(refreshToken),
        expiresAt: now() + 30d
    })

    AuditLog.record(event = "LOGIN_SUCCESS", actorType = "USER",
                    actorId = user.id, ipHash = ipHash,
                    detail = { mfaVerified: IS_PRIVILEGED })

    # Refresh Token 走 HttpOnly Cookie，Access Token 走响应体（前端内存持有）
    RETURN {
        accessToken,
        refreshToken: setHttpOnlyCookie(refreshToken, {
            httpOnly: TRUE, secure: TRUE, sameSite: 'Strict', path: '/api/v1/auth'
        }),
        user: sanitize(user)   # 绝不返回 passwordHash / lockedUntil 等内部字段
    }
```

### 3.2.3 配套工具函数

```
# 恒定时间字符串比对（用于恢复码、API Key、签名比对）
FUNCTION secureCompare(a: bytes, b: bytes) -> bool:
    IF len(a) != len(b):
        # 先做一次等价比较再返回，避免长度差异造成时序泄露
        timingSafeEqual(a, a)
        RETURN FALSE
    RETURN crypto.timingSafeEqual(a, b)

FUNCTION backoffDuration(failCount) -> Duration:
    # 10-14 次 → 5min；15-19 → 15min；20+ → 60min
    tier = FLOOR((failCount - 10) / 5)
    RETURN MIN(5min * POW(3, tier), 60min)

FUNCTION verifyCaptcha(token, ip) -> bool:
    # 验证码服务密钥从机密管理器读取，不落代码
    secret = SecretManager.get('CAPTCHA_SECRET_KEY')
    RETURN captchaProvider.verify(token, ip, secret)

# ── 脱敏：审核日志与通知中出现的用户内容片段一律经此处理 ──────────
# 用途有二：
#   ① 防止审核系统自身成为泄露源（日志被拖库即泄露用户隐私内容）
#   ② 驳回通知中只给用户足够定位问题的线索，不给完整原文
FUNCTION redact(text, options = {}) -> string:
    IF text IS NULL OR text IS EMPTY:
        RETURN ''

    keepHead = options.keepHead ?? 2      # 保留首部字符数
    keepTail = options.keepTail ?? 2      # 保留尾部字符数

    # 短文本一律全遮蔽——保留 2+2 字符几乎等于没遮蔽
    IF length(text) <= keepHead + keepTail + 4:
        RETURN '***'

    head = text[0 : keepHead]
    tail = text[length(text) - keepTail : ]
    maskedCount = length(text) - keepHead - keepTail

    RETURN head + '***(' + maskedCount + ' chars)***' + tail

# PII 类命中的专用脱敏：连首尾都不保留，仅给出类型与长度
# 理由：证件号、银行卡号的前后几位组合仍可能缩小检索范围
FUNCTION redactPII(value, piiType) -> string:
    IF value IS NULL:
        RETURN '[' + piiType + ':unknown]'
    RETURN '[' + piiType + ':' + length(value) + 'chars]'

# 校验字符串是否为脱敏形态（形如 [CN_ID_CARD:18chars] 或 ab***(12 chars)***yz）
FUNCTION isRedactedForm(s) -> bool:
    RETURN s MATCHES /^\[[A-Z_]+:(\d+chars|unknown)\]$/ OR s CONTAINS '***'

# 命中详情在写库与下发前统一走此网关，作为纵深防御的第二道：
# 各引擎在构造 findings 时就应已自行脱敏（第一道），
# 此处兜底是为了防止未来新增引擎时遗漏。
FUNCTION sanitizeFindingsForLog(findings) -> list:
    FOR f IN findings:
        IF f.engine == 'pii_detection':
            # PII 的明文从不进入 findings（rawValue 恒为 NULL），
            # 因此这里只做一致性校验：确保 redactedSample 已是脱敏形态。
            IF f.redactedSample IS NULL OR NOT isRedactedForm(f.redactedSample):
                f.redactedSample = redactPII(NULL, f.piiType)
            DELETE f.rawValue                    # 双保险：即便被误填也一并清除
        ELSE IF f.snippet IS NOT NULL:
            f.snippet = redact(f.snippet, { keepHead: 2, keepTail: 2 })
    RETURN findings
```

### 3.2.4 防暴力破解的三层协同

| 层 | 维度 | 阈值（可配） | 目的 |
|---|---|---|---|
| IP 限流 | 单 IP / 分钟 | 20 次 | 阻止单点暴力破解与扫描 |
| 账户锁定 | 单账户累计失败 | 10 次 → 指数退避 | 阻止针对特定账户的慢速爆破 |
| CAPTCHA | 单 IP 累计失败 | 3 次后触发 | 提高自动化成本，同时不打扰正常用户 |
| MFA 失败限制 | 单账户 MFA 失败 | 5 次 → 锁 30min | 防止拿到密码后爆破 6 位 TOTP |

> **为何 IP 限流阈值（20）远高于账户阈值（10）**：共享 NAT 出口（办公室、校园网）下大量用户同 IP，阈值过严会误伤。账户维度才是精准锁定。

---

## 3.3 密钥管理（非对称加密 + KMS）

### 3.3.1 密钥分层

```
┌─────────────────────────────────────────────────────────┐
│ 第 0 层：根密钥 (Root Key)                                │
│   位置：HSM / 云 KMS，永不出硬件边界                       │
│   用途：加密数据密钥，不可导出                             │
└───────────────────────┬─────────────────────────────────┘
                        │ 加密
┌───────────────────────▼─────────────────────────────────┐
│ 第 1 层：数据密钥 (DEK)                                   │
│   位置：以密文形式存数据库；明文仅在内存中短暂存在          │
│   用途：加密 TOTP 种子、备份文件、用户敏感字段             │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│ 第 2 层：签名密钥                                         │
│   · JWT 签名：Ed25519（见 3.3.4 的算法选型说明）           │
│   · 管理私钥：RSA-4096 **或** ECC P-384，用于管理操作签名  │
│   · Signed URL：HMAC-SHA256，密钥在 KMS 并定期轮换         │
└─────────────────────────────────────────────────────────┘
```

### 3.3.1.1 管理私钥的算法选型（RSA-4096 / ECC）

**需求对照说明**：本项目要求"采用非对称加密（RSA-4096 或 ECC）保护管理私钥"。这里需要区分两处非对称密钥，它们的安全目标不同，因此算法选择也不同：

| 用途 | 算法选择 | 理由 |
|---|---|---|
| **管理操作签名**（审批放行、封禁、阈值变更、密钥轮换授权） | **ECC P-384（首选）或 RSA-4096（备选）** | 直接对应需求。P-384 提供约 192-bit 安全强度，与 RSA-4096 等价，但密钥仅 48 字节、签名 96 字节，签发与验签快一个数量级 |
| **JWT 签名** | **Ed25519** | JWT 每次请求都要验签，性能敏感。Ed25519 是与 P-384 同族的现代曲线（Edwards 形式），安全性等价（约 128-bit 强度，满足 JWT 15 分钟有效期的威胁模型），且无 padding oracle、无随机数依赖 |

**兼容性说明**：若合规要求**必须是 RSA-4096**（例如与既有的 HSM 或 CA 体系对接），JWT 与管理签名可统一改为 RSASSA-PSS with SHA-256 (RSA-4096)。代价是每次验签约 0.2–1ms（P-384 约 0.05ms），在 QPS 上万时需评估 CPU 开销。**不推荐 RSASSA-PKCS1-v1_5**（即 `RS256`），因其不具备 PSS 的可证明安全性。

```
# ── 管理私钥的生成与使用（密钥材料始终留在 KMS/HSM 内） ──
FUNCTION provisionAdminSigningKey():
    # 关键：GenerateKey 在 KMS 内部完成，私钥**永不导出**
    # 这是"私钥不得明文存储于数据库或代码中"的强制实现——
    # 应用侧根本没有私钥可用，因此不存在泄露的可能。
    keyHandle = KMS.createKey({
        purpose:        'ASYMMETRIC_SIGN',
        algorithm:      'ECDSA_SHA_384',         # 或 'RSA_4096_PSS_SHA_256'
        keyUsage:       ['SIGN', 'VERIFY'],
        origin:         'HSM',                   # 强制在硬件内生成，禁止导入
        exportable:     FALSE,                   # 不可导出，杜绝备份泄露
        deletionPolicy: 'RETAIN'                 # 防止误删导致历史签名无法验证
    })

    # 仅公钥可导出，用于分发验签能力
    publicKey = KMS.getPublicKey(keyHandle.id)
    publishToJWKS(publicKey, keyId = keyHandle.id)

    AuditLog.record(event = 'ADMIN_SIGNING_KEY_PROVISIONED',
                    actorType = 'SYSTEM',
                    detail = { keyId: keyHandle.id,
                               algorithm: 'ECDSA_SHA_384',
                               origin: 'HSM' })

    RETURN keyHandle.id


# ── 管理操作签名：审批、封禁、阈值变更等高风险动作 ──
FUNCTION signAdminAction(adminId, payload):
    # 管理员的第二步验证：用其硬件密钥（WebAuthn/FIDO2）对上一步的
    # 服务端挑战做签名。这里的签名密钥由用户的认证器持有，服务端只验签。
    challenge = issueChallenge(adminId, ttl = 60s)

    # 客户端用认证器的私钥签名；私钥存在于安全芯片中，从不出芯片
    assertion = await webauthnClient.getAssertion(challenge, {
        userVerification: 'required',      # 要求生物识别或 PIN，实现"生物特征"因子
        timeout: 60000
    })

    IF NOT verifyWebAuthnAssertion(assertion, challenge, adminId):
        AuditLog.record(event = 'ADMIN_ACTION_SIGNATURE_INVALID',
                        actorType = 'USER', actorId = adminId, severity = 'CRITICAL')
        RAISE error(401, 'ADMIN_SIGNATURE_INVALID')

    # 服务端再用 KMS 中的管理私钥对"操作内容 + 管理员身份 + 时间戳"签名，
    # 形成不可否认的操作凭证，写入审计日志
    attestation = KMS.sign(
        keyId     = CONFIG.ADMIN_SIGNING_KEY_ID,     # 引用，非密钥本身
        algorithm = 'ECDSA_SHA_384',
        message   = canonicalJSON({
            adminId, payload,
            challengeId: challenge.id,
            webauthnCredentialId: assertion.credentialId,
            signedAt: now()
        })
    )

    RETURN { attestation, signatureAlgorithm: 'ECDSA_SHA_384' }
```

**"私钥不得明文存储"的三重保证**：

1. **应用侧无私钥**。私钥由 KMS/HSM 生成并保管，标记 `exportable: FALSE`。应用只持有 `keyId` 这一引用字符串，即便应用被完全攻破，攻击者也拿不到密钥材料。
2. **代码中仅出现引用**。所有调用都是 `KMS.sign(keyId = CONFIG.ADMIN_SIGNING_KEY_ID, ...)`，仓库中不存在任何 PEM/DER 密钥块。这由 CI 的 gitleaks 与自定义规则（检测 `BEGIN ... PRIVATE KEY`）双重拦截。
3. **数据库无密钥列**。`mfa_credentials` 表中仅有 `public_key`、`credential_id` 与 `secret_kms_key_id`（KMS 的数据密钥引用），没有任何字段存储可用于签名的私钥。

> **关于"硬件密钥/生物特征/动态令牌"三种第二因子的映射**：
> - **硬件密钥** → WebAuthn with `authenticatorAttachment: 'cross-platform'`（如 YubiKey）
> - **生物特征** → WebAuthn with `userVerification: 'required'` + 平台认证器（Touch ID / Windows Hello）
> - **动态令牌** → TOTP（RFC 6238），作为前两者的降级方案
>
> 优先级：硬件密钥 > 生物特征 > TOTP。TOTP 易受实时钓鱼攻击，因此仅在用户未绑定 WebAuthn 时允许。

### 3.3.2 信封加密（Envelope Encryption）伪代码

```
FUNCTION encryptSensitiveField(plaintext: bytes, context: string) -> EncryptedBlob:
    # 向 KMS 请求数据密钥；KMS 返回明文 DEK + 加密后的 DEK
    # 明文 DEK 只在本次调用内存中存在，用完立即清零
    dataKey = KMS.generateDataKey(keyId = CONFIG.KMS_MASTER_KEY_ID, keySpec = 'AES_256')

    TRY:
        nonce = CSPRNG.randomBytes(12)
        ciphertext = AES_256_GCM.encrypt(
            key = dataKey.plaintext,
            nonce = nonce,
            plaintext = plaintext,
            aad = context          # 附加认证数据：绑定字段用途，防止密文挪用到其他字段
        )
        RETURN {
            ciphertext:  ciphertext,
            nonce:       nonce,
            wrappedKey:  dataKey.ciphertextBlob,   # 加密后的 DEK，可安全入库
            kmsKeyId:    CONFIG.KMS_MASTER_KEY_ID,
            algorithm:   'AES-256-GCM',
            context:     context
        }
    FINALLY:
        # 关键：无论成功失败都清零内存中的明文密钥
        secureZero(dataKey.plaintext)

FUNCTION decryptSensitiveField(blob: EncryptedBlob) -> bytes:
    # 只有 KMS 能解出 DEK；应用侧即使数据库全量泄露也无法解密
    dataKeyPlain = KMS.decrypt(blob.wrappedKey, keyId = blob.kmsKeyId)
    TRY:
        RETURN AES_256_GCM.decrypt(
            key = dataKeyPlain,
            nonce = blob.nonce,
            ciphertext = blob.ciphertext,
            aad = blob.context
        )
    FINALLY:
        secureZero(dataKeyPlain)
```

> **AAD 的作用**：`context` 参数绑定密文用途。若攻击者把 `totp_secret` 的密文复制到 `recovery_code` 字段，解密会因 AAD 不匹配而失败。这防止了密文重排攻击。

### 3.3.3 JWT 签名与轮换

```
# 密钥集：支持多版本共存以实现无停机轮换
# JWKS 端点公开公钥，私钥永不出 KMS
FUNCTION signJWT(payload, expiresIn):
    keyId = Config.SIGNING_KEY_ACTIVE_ID       # 从配置中心读取，非硬编码
    # 签名操作在 KMS 内部完成，私钥不进入应用内存
    RETURN KMS.sign(
        keyId = keyId,
        algorithm = 'EdDSA',
        message = base64url(header) + '.' + base64url(payload)
    )

FUNCTION verifyJWT(token):
    header = decodeHeader(token)
    # 从 JWKS 缓存取公钥；缓存 1 小时，轮换后新旧公钥并存 24 小时
    publicKey = JWKSCache.get(header.kid)
    IF publicKey IS NULL:
        RETURN INVALID
    # 强制校验算法，拒绝 'none' 与算法混淆攻击
    IF header.alg NOT IN ['EdDSA']:
        RETURN INVALID
    RETURN KMS.verify(publicKey, token)
```

**轮换计划**：签名密钥每 90 天轮换一次；轮换时新密钥立即生效，旧密钥保留验签能力 24 小时（覆盖最长 Access Token 有效期）。

---

## 3.4 上传文件预处理与审核状态流转

### 3.4.1 上传意图与凭证签发

```
FUNCTION createUploadIntent(user, request):
    INPUT: filename, sizeBytes, contentType, sha256?, categoryId

    # ── 步骤 1：权限与配额校验 ──────────────────────────────────────
    IS_PRIVILEGED = RBAC.hasPermission(user, 'content:upload_unlimited')

    IF NOT RBAC.hasPermission(user, 'content:upload'):
        RETURN error(403, "UPLOAD_NOT_PERMITTED")

    # 配额检查走 Redis 原子计数（PG 兜底），避免并发超发
    IF NOT IS_PRIVILEGED:
        quotaKeyBytes = "quota:bytes:{user.id}:" + today()
        quotaKeyCount = "quota:count:{user.id}:" + today()

        currentBytes = Redis.GET(quotaKeyBytes) OR 0
        currentCount = Redis.GET(quotaKeyCount) OR 0

        IF currentCount + 1 > user.uploadQuotaCountDaily:
            RETURN error(429, "DAILY_COUNT_QUOTA_EXCEEDED",
                         detail = { limit: user.uploadQuotaCountDaily, resetAt: tomorrow() })

        IF currentBytes + sizeBytes > user.uploadQuotaBytesDaily:
            RETURN error(429, "DAILY_BYTES_QUOTA_EXCEEDED",
                         detail = { limit: user.uploadQuotaBytesDaily, used: currentBytes })

    # 单文件大小上限
    maxSize = IS_PRIVILEGED ? CONFIG.MAX_FILE_SIZE_PRIVILEGED : user.maxFileSizeBytes
    IF sizeBytes > maxSize:
        RETURN error(413, "FILE_TOO_LARGE", detail = { maxSize })

    # ── 步骤 2：频率限制（防脚本化批量灌水） ────────────────────────
    rateKey = "upload:rate:{user.id}"
    IF Redis.INCR(rateKey) > CONFIG.UPLOAD_RATE_PER_HOUR (default 10):
        RETURN error(429, "UPLOAD_RATE_LIMITED")
    Redis.EXPIRE(rateKey, 3600)

    # ── 步骤 3：客户端预声明 MIME 类型校验（服务端不信任） ───────────
    declaredType = detectContentType(filename, contentType)
    IF declaredType IS NULL:
        RETURN error(415, "UNSUPPORTED_MEDIA_TYPE")

    # ── 步骤 4：创建内容记录（DRAFT 状态） ─────────────────────────
    contentId = UUID_V7()
    objectKey = buildObjectKey(user.id, contentId, filename)
    # 分区前缀：{userId前2位}/{contentId}/{filename}，避免单前缀热点

    DB.contents.insert({
        id: contentId, uploaderId: user.id, contentType: declaredType,
        status: 'DRAFT', title: basename(filename), sourceSizeBytes: sizeBytes,
        sourceSha256: sha256 ? hexToBytes(sha256) : NULL,
        categoryId: categoryId
    })

    # ── 步骤 5：签发 presigned URL（限定条件，防越权覆盖） ───────────
    presignedUrl = S3.presignPutObject({
        key: objectKey,
        expiresIn: 900,                       # 15 分钟
        # 把大小与类型写进签名条件：客户端即便篡改请求也会被对象存储拒绝
        conditions: {
            contentLengthRange: [1, maxSize],
            contentType: CONTENT_TYPE_WHITELIST[declaredType]
        },
        # 强制临时隔离标签，转码前不可被 CDN 公开访问
        tagging: 'quarantine=true'
    })

    AuditLog.record(event = "UPLOAD_INTENT_CREATED", actorType = "USER",
                    contentId = contentId, actorId = user.id)

    RETURN {
        contentId,
        uploadUrl: presignedUrl,
        expiresAt: now() + 900,
        # 客户端需在完成后回调此端点
        callbackUrl: '/api/v1/contents/{contentId}/upload-complete',
        maxSizeBytes: maxSize
    }
```

### 3.4.2 上传完成回调与校验

```
FUNCTION onUploadComplete(user, contentId):
    content = DB.contents.findOne(id = contentId, deletedAt IS NULL)

    # 越权检查：只能操作自己的内容（管理员除外）
    IF content.uploaderId != user.id AND NOT RBAC.hasPermission(user, 'content:delete_any'):
        RETURN error(403, "FORBIDDEN")

    # 状态前置条件：防止重复回调造成重复转码
    IF content.status != 'DRAFT':
        # 幂等：重复回调直接返回当前状态，不报错
        RETURN { status: content.status, idempotent: true }

    # ── 服务端独立校验对象真实性（不信任客户端声明） ────────────────
    head = S3.headObject(content.sourceObjectKey)

    IF head IS NULL:
        RETURN error(400, "OBJECT_NOT_FOUND")

    IF head.contentLength != content.sourceSizeBytes:
        DB.contents.update(content, { status: 'REJECTED', rejectedReason: 'SIZE_MISMATCH' })
        AuditLog.record(event = "UPLOAD_VALIDATION_FAILED", actorType = "USER",
                        contentId = contentId, actorId = user.id,
                        reason = 'SIZE_MISMATCH')
        RETURN error(400, "SIZE_MISMATCH")

    # ── 服务端魔数嗅探：不信任扩展名与 Content-Type ─────────────────
    magicBytes = S3.readRange(content.sourceObjectKey, offset = 0, length = 512)
    actualType = sniffMagicBytes(magicBytes)

    IF actualType NOT IN ALLOWED_FORMATS[content.contentType]:
        # 典型攻击：把 .exe 改名为 .mp4，或上传 SVG/HTML 做存储型 XSS
        S3.deleteObject(content.sourceObjectKey)
        DB.contents.update(content, { status: 'REJECTED', rejectedReason: 'FORMAT_MISMATCH' })
        AuditLog.record(event = "UPLOAD_FORMAT_REJECTED", actorType = "USER",
                        contentId = contentId, actorId = user.id,
                        detail = { declared: content.contentType, actual: actualType },
                        severity = 'WARN')
        RETURN error(415, "FORMAT_MISMATCH")

    # ── 容器/元数据深度检查（防畸形文件触发解码器漏洞） ─────────────
    probeResult = AWAIT probeMedia(content.sourceObjectKey)   # ffprobe，沙箱内执行
    IF probeResult.hasErrors OR probeResult.durationSeconds > CONFIG.MAX_DURATION_SECONDS:
        DB.contents.update(content, { status: 'REJECTED', rejectedReason: 'CORRUPT_OR_TOO_LONG' })
        RETURN error(400, "MEDIA_INVALID")

    # ── 通过 → 更新配额并投递异步任务 ──────────────────────────────
    Redis.INCRBY("quota:bytes:{user.id}:" + today(), head.contentLength)
    Redis.INCR("quota:count:{user.id}:" + today())

    DB.contents.update(content, {
        status: 'UPLOADED',
        durationSeconds: probeResult.duration,
        width: probeResult.width, height: probeResult.height,
        sourceFormat: probeResult.format
    })

    # 幂等键：contentId。重复投递由消费者侧 SETNX 去重
    MQ.publish('content.uploaded', {
        contentId, objectKey, contentType: content.contentType,
        uploaderId: user.id, traceId: currentTraceId()
    })

    AuditLog.record(event = "UPLOAD_ACCEPTED", actorType = "USER",
                    contentId = contentId, actorId = user.id)

    RETURN { status: 'UPLOADED', contentId }
```

### 3.4.3 AI 预审流水线

```
FUNCTION runPrecheck(message):
    contentId = message.contentId

    # ── 幂等锁：同一内容同一引擎版本只处理一次 ─────────────────────
    lockKey = "precheck:lock:{contentId}:" + CURRENT_ENGINE_VERSION
    IF NOT Redis.SETNX(lockKey, ttl = 3600):
        RETURN   # 已处理或正在处理

    content = DB.contents.findOne(id = contentId)
    IF content.status NOT IN ['UPLOADED', 'TRANSCODING']:
        RETURN   # 状态已变更（如用户取消），放弃

    findings = []

    # 收集本轮所有可扫描的文本，供后续关键词与 PII 检测共用。
    # ⚠️ 必须在此处初始化：各引擎的局部变量（frames / transcript / ocrText）
    #    作用域仅限各自的 IF 块，跨块引用会取到 undefined。
    scannableTexts = []      # [{ source: 'ASR'|'OCR'|'BODY', text, timeMs? }]
    frames = []              # 抽帧结果，供引擎 1 与引擎 3 共用，避免重复抽帧

    # ── 引擎 1：视频抽帧 + 图像识别 ─────────────────────────────────
    IF content.contentType == 'VIDEO':
        frames = extractKeyframes(content.sourceObjectKey,
                                  intervalSeconds = CONFIG.FRAME_INTERVAL,
                                  maxFrames = CONFIG.MAX_FRAMES)
        FOR frame IN frames:
            result = AWAIT nsfwClassifier.classify(frame)
            IF result.score > CONFIG.NSFW_FRAME_THRESHOLD:
                findings.append({ engine: 'nsfw', frameTimeMs: frame.timeMs,
                                  score: result.score, labels: result.labels })

    # ── 引擎 2：语音转文字 + 关键词/语义过滤 ────────────────────────
    IF content.contentType IN ['VIDEO', 'AUDIO']:
        transcript = AWAIT asrService.transcribe(content.sourceObjectKey, language = 'auto')
        FOR segment IN transcript.segments:
            scannableTexts.append({ source: 'ASR', text: segment.text,
                                    timeMs: segment.startMs })
            FOR rule IN loadKeywordRules():          # 规则缓存在 Redis，热更新
                IF rule.match(segment.text):
                    # 区分规则等级：硬拦截 vs 加权计分
                    IF rule.severity == 'BLOCK':
                        findings.append({ engine: 'asr_keyword', severity: 'BLOCK',
                                          ruleId: rule.id, snippet: redact(segment.text),
                                          timeMs: segment.startMs })
                    ELSE:
                        findings.append({ engine: 'asr_keyword', severity: 'WARN',
                                          ruleId: rule.id, weight: rule.weight })

    # ── 引擎 3：OCR 抽帧文字（识别画面内违规文字） ──────────────────
    # 复用引擎 1 已抽取的 frames，不重复解码（抽帧是 I/O 与 CPU 密集操作）
    IF content.contentType == 'VIDEO':
        FOR frame IN frames:
            ocrText = AWAIT ocrService.recognize(frame)
            IF ocrText IS EMPTY: CONTINUE
            scannableTexts.append({ source: 'OCR', text: ocrText, timeMs: frame.timeMs })
            FOR rule IN loadKeywordRules():
                IF rule.match(ocrText):
                    findings.append({ engine: 'ocr', ruleId: rule.id,
                                      severity: rule.severity, timeMs: frame.timeMs })

    # ── 引擎 4：文本内容直接扫描（阅读器类内容） ────────────────────
    IF content.contentType == 'TEXT':
        text = S3.readText(content.sourceObjectKey, maxBytes = CONFIG.MAX_TEXT_SCAN_BYTES)
        scannableTexts.append({ source: 'BODY', text })
        findings.extend(scanText(text))

    # ── 引擎 5：敏感信息检测（PII / 证件号 / 银行卡 / 联系方式） ─────
    # 与"违规内容过滤"目标不同：这里保护的是**上传者自己**——
    # 用户常在视频、字幕、文档正文中无意暴露身份证号、手机号、银行卡号。
    # 命中不必然拦截，默认标记为"需脱敏提示"，由策略决定是否要求用户处理。
    FOR target IN scannableTexts:
        piiHits = detectPII(target.text, {
            patterns: ['CN_ID_CARD', 'PHONE_CN', 'BANK_CARD', 'EMAIL',
                       'PASSPORT', 'CN_SOCIAL_CREDIT'],
            # 上下文校验：单纯 11 位数字不一定是手机号，
            # 需结合前后文关键词（如"电话""联系方式"）降低误报
            requireContext: TRUE
        })
        FOR hit IN piiHits:
            findings.append({
                engine: 'pii_detection',
                severity: 'WARN',              # 默认不硬拦截
                piiType: hit.type,
                # ⚠️ 关键：日志与返回值中**绝不记录命中的明文**，
                #    否则审核系统本身变成了 PII 泄露源。
                #    PII 用专用脱敏（redactPII），连首尾字符都不保留——
                #    证件号、卡号的"首 2 位 + 尾 2 位"组合仍可大幅缩小检索范围。
                redactedSample: redactPII(hit.value, hit.type),
                # 明文仅在本函数内存中短暂存在，绝不写入任何持久化介质。
                # 显式声明此处不保留 rawValue，避免下游误以为可以从 findings 取到原文。
                rawValue: NULL,
                location: hit.location,
                weight: CONFIG.PII_RULE_WEIGHT (default 15)
            })

    # 若命中 PII，额外给上传者一条可操作的提示（而非等审核员处理）
    IF findings CONTAINS engine == 'pii_detection':
        Notify.send(content.uploaderId, 'PII_DETECTED_IN_UPLOAD',
                    detail = {
                        contentId,
                        types: DISTINCT(f.piiType FOR f IN findings
                                        WHERE f.engine == 'pii_detection'),
                        suggestion: '建议在上传前对个人信息区域做模糊或打码处理。',
                        # 明确告知这不影响发布，避免用户误以为违规
                        blocksPublication: FALSE
                    })

    # ── 聚合为风险分 ────────────────────────────────────────────────
    riskScore = 0
    FOR f IN findings:
        IF f.severity == 'BLOCK':
            # 硬规则直接顶到最高分，但保留原始命中项供申诉复核
            riskScore = MAX(riskScore, CONFIG.HARD_BLOCK_SCORE (default 95))
        ELSE:
            riskScore = riskScore + (f.weight * f.score)

    # 多引擎交叉命中加权（同时命中 NSFW 与关键词比单项更可疑）
    distinctEngines = COUNT_DISTINCT(f.engine FOR f IN findings WHERE f.severity == 'BLOCK')
    IF distinctEngines >= 2:
        riskScore = MIN(100, riskScore + CONFIG.CROSS_ENGINE_BONUS (default 10))

    riskScore = CLAMP(riskScore, 0, 100)
    riskLevel = classifyRisk(riskScore)

    # 保存原始命中明细——这是申诉复核与模型调优的数据基础
    # ⚠️ riskScore / precheckDetail 等"产物字段"直接 UPDATE 是允许的，
    #    因为它们不是状态；但 status 必须走 applyTransition 统一入口，
    #    否则会绕过审计日志，破坏"状态变更必有日志"的不变式（见 3.4.5）。
    # ⚠️ findings 在落库前必须过脱敏网关：命中片段可能包含用户隐私内容，
    #    审核日志本身也是会被访问的数据，不能成为新的泄露面。
    safeFindings = sanitizeFindingsForLog(findings)

    DB.contents.update(contentId = content.id, {
        riskScore, riskLevel,
        precheckEngineVersion: CURRENT_ENGINE_VERSION,
        precheckDetail: { findings: safeFindings,
                          engineVersion: CURRENT_ENGINE_VERSION,
                          precheckedAt: now() }
    })

    # 状态推进走统一入口（内部重新读取当前状态，容错转码侧已先推进的情况）
    transitionStatus(contentId = content.id, to = 'PRECHECK_DONE',
                     actor = 'AI_ENGINE',
                     reason = 'precheck_completed',
                     detail = { engineName: 'multi-modal-precheck',
                                engineVersion: CURRENT_ENGINE_VERSION,
                                riskScore, riskLevel, findings: safeFindings })

    MQ.publish('audit.precheck.completed', { contentId, riskScore, riskLevel })
```

### 3.4.4 策略判定与状态流转（防误杀核心）

```
FUNCTION decideByPolicy(message):
    contentId = message.contentId
    content = DB.contents.findOne(id = contentId)

    # ── 读取可配阈值（支持按分类差异化 + 灰度） ────────────────────
    policy = ConfigCenter.getAuditPolicy(
        categoryId = content.categoryId,
        contentType = content.contentType
    )
    # policy = {
    #   lowThreshold:  30,    # < 30 直接发布
    #   highThreshold: 70,    # >= 70 自动拦截
    #   autoPublishEnabled: true,
    #   rolloutPercent: 100   # 新模型灰度比例
    # }

    # 灰度判定：非灰度流量走上一稳定版本的策略
    IF hashToPercent(contentId) > policy.rolloutPercent:
        policy = ConfigCenter.getAuditPolicyStable(content.categoryId)

    # ── 三档决策 ────────────────────────────────────────────────────
    IF content.riskScore < policy.lowThreshold AND policy.autoPublishEnabled:
        # 低风险 → 直接发布（AUTO_PUBLISHED 即为对外可见终态）
        transitionStatus(contentId, to = 'AUTO_PUBLISHED', actor = 'SYSTEM',
            reason = 'risk_below_low_threshold',
            detail = { riskScore: content.riskScore, threshold: policy.lowThreshold })
        publishContent(contentId)   # 仅做缓存/索引/CDN 预热，不再改 status

    ELSE IF content.riskScore < policy.highThreshold:
        # 中风险 → 人工复核队列（宁可多一次人工，也不错杀）
        transitionStatus(contentId, to = 'PENDING_MANUAL', actor = 'SYSTEM',
            reason = 'risk_in_manual_review_band',
            detail = { riskScore: content.riskScore, band: [policy.lowThreshold, policy.highThreshold] })
        # 按风险分排序入队，高风险优先被处理
        MQ.publish('audit.manual.required', {
            contentId, riskScore: content.riskScore, priority: content.riskScore
        })
        Notify.send(content.uploaderId, 'CONTENT_UNDER_REVIEW',
                    detail = { contentId, estimatedWait: estimateQueueWait() })

    ELSE:
        # 高风险 → 自动拦截
        transitionStatus(contentId, to = 'BLOCKED', actor = 'AI_ENGINE',
            reason = 'risk_above_high_threshold',
            detail = { riskScore: content.riskScore })
        # 隔离对象：打上 quarantine 标签，CDN 拒绝服务
        S3.putObjectTagging(content.sourceObjectKey, { quarantine: 'true', blocked: 'true' })
        # 必须告知用户并开放申诉——这是防误杀的最后一道防线
        Notify.send(content.uploaderId, 'CONTENT_BLOCKED_WITH_APPEAL',
                    detail = {
                        contentId,
                        reasonSummary: summarizeFindings(content.precheckDetail),  # 不暴露完整规则，防规避
                        # 与 checkAppealDeadline 共用同一配置，避免两处数值漂移
                        appealDeadline: now() + CONFIG.APPEAL_WINDOW (default 30d),
                        appealUrl: '/appeals/new?contentId={contentId}'
                    })
```

### 3.4.5 状态流转的统一入口（保证状态与日志原子）

#### 状态跃迁白名单（先定义，供下方函数使用）

```
# ── 合法的状态跃迁白名单 ──────────────────────────────────────────
# 说明：本表是**代码层的权威约束**，比 01 节的示意图更细。
# 示意图为了可读性省略了若干运维与异常分支（如转码彻底失败后的归档、
# 人工复审中直接判定为严重违规而跳过 REJECTED 直接 BLOCKED）。
# 两者冲突时以本表为准，并需同步更新 01 节的图。
STATE_MACHINE = {
    'DRAFT':            ['UPLOADED', 'DELETED'],
    'UPLOADED':         ['TRANSCODING', 'PRECHECK_DONE', 'DELETED'],
    'TRANSCODING':      ['PRECHECK_DONE', 'TRANSCODE_FAILED'],
    # 转码与预审并行：两侧都可能先到，故互为主备后继
    'TRANSCODE_FAILED': ['REJECTED', 'DELETED'],
    'PRECHECK_DONE':    ['AUTO_PUBLISHED', 'PENDING_MANUAL', 'BLOCKED'],
    'AUTO_PUBLISHED':   ['BLOCKED', 'DELETED'],        # 后续被举报并复核为违规
    # BLOCKED 是"人工复审中直接判定严重违规"的路径，示意图未画出
    'PENDING_MANUAL':   ['PUBLISHED', 'REJECTED', 'BLOCKED', 'DELETED'],
    # BLOCKED ← APPEALING：申诉因一致性修复被撤销时的回退目标（见 auditAppealHealth）
    #          ← PUBLISHED：发布后被下架
    'BLOCKED':          ['APPEALING', 'DELETED'],
    # APPEALING → BLOCKED：申诉记录丢失/撤回导致回退，用于自动收敛悬挂状态
    #           → PUBLISHED / REJECTED：申诉的两种正常结果
    'APPEALING':        ['PUBLISHED', 'REJECTED', 'BLOCKED'],
    'PUBLISHED':        ['BLOCKED', 'DELETED'],        # 发布后被下架
    # REJECTED → APPEALING：允许就新理由/新证据再次申诉（06 节产品流程）。
    #            受"同一内容同时仅一个未结申诉"与每日 3 次总量共同约束。
    'REJECTED':         ['APPEALING', 'DELETED'],
    'DELETED':          [],
}

# 对"是否对外可见"的统一定义，供 02 索引与 04 接口共用
FUNCTION isPubliclyVisible(status) -> bool:
    RETURN status IN ('PUBLISHED', 'AUTO_PUBLISHED')

# ── 状态 → 审计动作名的映射（契约函数，必须与 04 节示例严格一致） ──────
# ⚠️ checkAppealDeadline 依赖本表产出的 'MANUAL_REJECT' / 'AUTO_BLOCK' /
#    'MANUAL_BLOCK' 三个动作名来定位申诉窗口起点。若此处改名而不同步
#    checkAppealDeadline，所有合法申诉都会被误判为 CONTENT_NOT_APPEALABLE。
ACTION_FOR_TRANSITION = {
    'UPLOADED':          'UPLOAD_ACCEPTED',
    'PRECHECK_DONE':     'PRECHECK_COMPLETED',
    'AUTO_PUBLISHED':    'AUTO_PUBLISH',
    'PENDING_MANUAL':    'QUEUED_FOR_MANUAL',
    'PUBLISHED':         'MANUAL_APPROVE',
    'REJECTED':          'MANUAL_REJECT',
    'BLOCKED':           'AUTO_BLOCK',        # 由 decideByPolicy 触发时
    'APPEALING':         'APPEAL_SUBMITTED',
    'TRANSCODE_FAILED':  'TRANSCODE_FAILED',
    'DELETED':           'CONTENT_DELETED',
}

FUNCTION actionForTransition(to, actor) -> string:
    # 需要区分 actor 的两种状态：
    #   BLOCKED  —— 人工判定严重违规 vs AI 自动拦截，混淆会污染误杀率统计
    #   REJECTED —— 人工驳回 vs 系统归档（如转码彻底失败后归档为 REJECTED）
    IF to == 'BLOCKED':
        RETURN actor == 'MODERATOR' ? 'MANUAL_BLOCK' : 'AUTO_BLOCK'
    IF to == 'REJECTED':
        RETURN actor == 'MODERATOR' ? 'MANUAL_REJECT' : 'AUTO_REJECT'
    RETURN ACTION_FOR_TRANSITION[to] ?? ('STATUS_' + to)

# ── 状态 → 附加字段（契约函数） ────────────────────────────────────────
FUNCTION toStatusExtraFields(to, actor, actorId, reason) -> object:
    SWITCH to:
        CASE 'PUBLISHED', 'AUTO_PUBLISHED':
            # autoPublished 标记用于区分"系统放行"与"人工放行"，
            # 是 01 节所述误杀率回溯的数据基础
            RETURN { publishedAt: now(), rejectedReason: NULL,
                     autoPublished: (to == 'AUTO_PUBLISHED') }
        CASE 'REJECTED', 'BLOCKED':
            # rejectedReason 只在进入这两种状态时写入。
            # 注意：离开这些状态时**不清除**——它记录的是历史事实，
            # 因此不可用作"当前是否被驳回"的判断依据（见 withdrawAppeal）。
            RETURN { rejectedReason: reason }
        CASE 'DELETED':
            RETURN { deletedAt: now() }
        DEFAULT:
            RETURN {}

# ── 申诉理由的规范化（用于"实质相同"判重） ───────────────────────────
# 目的：拦住"改两个字重发"的滥用，同时避免把确实不同的理由误判为重复。
# 刻意采用保守策略——只做归一化后判等，不做语义相似度，原因：
#   ① 语义判相似必然产生误判，把用户合理的补充说明挡在门外，
#      而这恰恰与"申诉是防误杀的最后一道防线"的定位冲突；
#   ② 真正需要拦的是复制粘贴，归一化足以覆盖。
FUNCTION normalizeReason(text) -> string:
    IF text IS NULL:
        RETURN ''
    s = text
    s = TRIM(s)
    s = LOWER_CASE(s)                       # 统一大小写
    s = NORMALIZE_UNICODE(s, 'NFKC')        # 全角→半角，兼容字符归一
    s = REMOVE(s, ALL_WHITESPACE)           # 去除所有空白（含全角空格）
    s = REMOVE(s, PUNCTUATION)              # 去除标点，避免"。"与"."的差异
    RETURN s

# ── 审计记录的通用入口 ────────────────────────────────────────────────
# ⚠️ 本函数是 audit_logs 的唯一写入通道（除 transitionStatus /
#    withdrawAppeal 内部因需与业务更新同事务而直接 insert）。
#    它**必须**自行计算 prev_hash / record_hash —— record_hash 在 schema 中
#    为 NOT NULL（见 02 节），缺省会导致整条插入失败。
#    调用方只关心"发生了什么"，不应重复关心哈希链的维护。
#
# 入参约定：接受一个具名参数对象，而非固定位置参数。原因是有二十余处调用
# 各自携带不同上下文（登录风控带 ipHash、账号锁定带 lockDuration、
# 上传带 contentId），用一个宽签名去覆盖必然导致错位赋值。
FUNCTION AuditLog.record({ event, actorType, contentId?, actorId?,
                            fromStatus?, toStatus?, riskScore?, riskLevel?,
                            ruleHits?, reason?, engineName?, engineVersion?,
                            ipHash?, detail?, severity? }) -> void:
    # actorType 是 schema 中的 NOT NULL 列。若调用方未显式提供，
    # 依据已知上下文推断，绝不写入空值导致整条记录插入失败。
    effectiveActorType = actorType ?? inferActorType({ actorId, contentId, event })
    IF effectiveActorType IS NULL:
        # 宁可记为 SYSTEM 也不要丢日志——审计缺失比归类不精确更严重
        effectiveActorType = 'SYSTEM'

    DB.transaction(async tx => {
        prevLog  = tx.auditLogs.findLatest(contentId = contentId)
        prevHash = prevLog ? prevLog.recordHash : ZERO_HASH

        record = {
            contentId,
            actorType: effectiveActorType,
            actorId:       actorId       ?? NULL,
            actorIpHash:   ipHash        ?? NULL,   # 对应 schema 的 actor_ip_hash
            action:        event,                   # 入参名 event → schema 列 action
            fromStatus:    fromStatus    ?? NULL,
            toStatus:      toStatus      ?? NULL,
            riskScore:     riskScore     ?? NULL,
            riskLevel:     riskLevel     ?? NULL,
            ruleHits:      ruleHits      ?? NULL,
            reason:        reason        ?? detail?.reason ?? NULL,
            engineName:    engineName    ?? detail?.engineName    ?? NULL,
            engineVersion: engineVersion ?? detail?.engineVersion ?? NULL
        }

        tx.auditLogs.insert({
            ...record,
            prevHash,
            recordHash: SHA256(concat(prevHash, canonicalJSON(record)))
        })
    })

    IF severity IN ['WARN', 'ERROR', 'CRITICAL']:
        Metrics.increment('audit.event_by_severity', tags = { event, severity })
    IF severity == 'CRITICAL':
        Alerting.fire('AUDIT_CRITICAL_EVENT', detail = { event, contentId, actorId })

# 依据上下文推断 actor 类型，仅在调用方未显式提供时使用
FUNCTION inferActorType({ actorId, contentId, event }) -> string | NULL:
    IF actorId IS NOT NULL:
        RETURN 'USER'                       # 有具体操作者即视为用户行为
    IF contentId IS NOT NULL AND event CONTAINS 'PRECHECK':
        RETURN 'AI_ENGINE'
    IF event CONTAINS 'SYSTEM' OR event CONTAINS 'RECONCIL':
        RETURN 'SYSTEM'
    RETURN 'SYSTEM'                         # 兜底

# ── 哈希链完整性校验（离线任务，每日执行） ────────────────────────────
# 这是"日志不可篡改"承诺的实际兑现方式：链式哈希只有在被验证时才有意义。
# ⚠️ 校验的规范化规则必须与写入侧**完全一致**：写入时哈希的是
#    "展开 prevHash/recordHash 之前"的 record 对象（见上方 insert 与
#    transitionStatus 内联写入）。因此此处必须剔除同样的两个字段，
#    且键顺序由 canonicalJSON 统一决定——不得额外剔除 id/createdAt，
#    否则所有记录都会校验失败并误报 CRITICAL 级链断裂。
FUNCTION withoutHashFields(row) -> object:
    copy = SHALLOW_COPY(row)
    DELETE copy.prevHash
    DELETE copy.recordHash
    RETURN copy


FUNCTION verifyAuditChain(fromDate, toDate) -> Report:
    rows = DB.auditLogs.findWhere(createdAt BETWEEN [fromDate, toDate],
                                  orderBy = 'createdAt ASC, id ASC')
    prevHash = ZERO_HASH
    broken = []

    FOR row IN rows:
        expected = SHA256(concat(prevHash, canonicalJSON(withoutHashFields(row))))
        IF NOT secureCompare(row.recordHash, expected) OR
           NOT secureCompare(row.prevHash ?? ZERO_HASH, prevHash):
            broken.append({ id: row.id, contentId: row.contentId,
                            createdAt: row.createdAt })
            # ⚠️ 一旦断裂，后续所有记录的 prev_hash 都不可能再匹配当前
            #    期望值。若继续用 row.recordHash 推进游标，会把"首条被篡改"
            #    放大成"其后每条都损坏"，误导定位。
            #    因此断裂后重置游标为 ZERO_HASH 并标记链已断——真正需要
            #    人看的是第一条断裂位置。
            prevHash = ZERO_HASH
        ELSE:
            prevHash = row.recordHash

    IF COUNT(broken) > 0:
        # 链断裂是最高级别的安全事件——可能意味着日志被外部篡改
        Alerting.fire('AUDIT_CHAIN_BROKEN',
                      detail = { brokenCount: COUNT(broken),
                                 firstBrokenId: broken[0].id,
                                 firstBrokenAt: broken[0].createdAt,
                                 range: [fromDate, toDate] },
                      severity = 'CRITICAL')

    RETURN { checked: COUNT(rows), broken, verifiedAt: now() }
```

> **注意 `AUTO_PUBLISHED` 与 `PUBLISHED` 都出现在"已发布"集合中**，且 `APPEALING → PUBLISHED` 是合法跃迁。这两点必须与 02 节的索引谓词、04 节的可见性判断保持一致，否则会出现"自动放行的内容搜不到"或"申诉通过后内容仍不出现"的缺陷。

#### 流转函数

```
FUNCTION transitionStatus(contentId, to, actor, actorId?, reason, detail?):
    # ⚠️ 关键：以 contentId 而非调用方的对象快照作为输入。
    # 若接受 content 对象并从中读 from，在"申诉中 → 已通过"这类
    # 二次流转场景下，调用方持有的是流转前的旧快照（如 BLOCKED），
    # 条件更新 WHERE status = 'BLOCKED' 将匹配不到实际已是 APPEALING 的行，
    # 导致流转静默失败或误判为并发冲突。
    # 正确做法：在事务内重新锁定并读取当前状态。

    # ── 事务内：锁定 + 合法性校验 + 条件更新 + 写日志 ────────────────
    # 注意：事务回调内的 RETURN 只结束回调本身，**不会**跳过事务外的副作用。
    # 因此这里把结果提升到外层变量，由外层统一决定是否执行副作用。
    outcome = DB.transaction(async tx => {
        # 行级锁：FOR UPDATE 阻止并发流转同一内容
        row = tx.contents.selectForUpdate(id = contentId, deletedAt IS NULL)
        IF row IS NULL:
            RAISE NotFound()

        from = row.status       # ← 以数据库当前值为准，非调用方快照

        # 幂等：目标状态与当前一致 → 不改状态、不写日志、不产生副作用
        IF from == to:
            RETURN { idempotent: TRUE, from, to }

        # 合法性校验：白名单状态机
        IF NOT STATE_MACHINE.isLegalTransition(from, to):
            # ⚠️ 非法跃迁的审计记录必须写在**独立事务**里。
            #    若用当前的 tx 写入，紧随其后的 RAISE 会回滚整个事务，
            #    这条安全事件就永远不会落库——恰恰是最需要留痕的攻击尝试。
            #    因此在抛错前先退出当前事务，用独立连接写入。
            RAISE IllegalStateTransition(from, to)   # 回滚业务事务

        # ── 以下为事务成功路径 ──

        # 条件更新：WHERE 同时带 id 与 from，构成双重保险
        affected = tx.contents.updateWhere(
            where = { id: contentId, status: from },
            set   = { status: to, updatedAt: now(),
                      ...toStatusExtraFields(to, actor, actorId, reason) }
        )

        IF affected == 0:
            # 已持有行锁的情况下理论上不会发生；保留以防未来去掉锁
            RAISE ConcurrentModification()

        # 计算防篡改链哈希
        prevLog = tx.auditLogs.findLatest(contentId = contentId)
        prevHash = prevLog ? prevLog.recordHash : ZERO_HASH

        record = {
            contentId, actorType: actor, actorId,
            # ⚠️ 必须传 actor：BLOCKED 的动作名依赖它区分
            #    "人工判定严重违规"(MANUAL_BLOCK) 与 "AI 自动拦截"(AUTO_BLOCK)。
            #    漏传会让所有拦截都被记为 AUTO_BLOCK，直接污染误杀率统计。
            action: actionForTransition(to, actor), fromStatus: from, toStatus: to,
            riskScore: row.riskScore, riskLevel: row.riskLevel,
            reason, ruleHits: detail,
            engineName: detail.engineName, engineVersion: detail.engineVersion
        }

        tx.auditLogs.insert({
            ...record,
            prevHash,
            recordHash: SHA256(concat(prevHash, canonicalJSON(record)))
        })

        RETURN { idempotent: FALSE, from, to }
    })

    # ── 幂等路径：状态已一致，直接返回，不广播、不刷索引、不失效缓存 ──
    # （重复消费 MQ 消息时走这条路径，避免对下游产生重复信号）
    IF outcome.idempotent:
        RETURN { idempotent: TRUE, status: to }

    # ── 事务外：副作用（失败可重试，不影响状态一致性） ──────────────
    MQ.publish('content.status.changed', { contentId,
                                           from: outcome.from, to: outcome.to })
    SearchIndex.upsert(contentId, { status: to })      # 索引同步
    InvalidCache(contentId)                             # 缓存失效

    RETURN { idempotent: FALSE, status: to }
```

非法跃迁的审计记录用独立事务写入（**必须在业务事务回滚之后**）。捕获者是 `applyTransition`（见下方定义）：

```
# 由 transitionStatus 的调用方捕获，或在此函数内用 TRY/CATCH 包裹以上事务块
FUNCTION onIllegalTransitionAttempt(contentId, from, to, actor, actorId, reason):
    # 独立事务：即便业务事务已回滚，这条安全事件也必须持久化
    DB.transaction(async auditTx => {
        prevLog  = auditTx.auditLogs.findLatest(contentId = contentId)
        prevHash = prevLog ? prevLog.recordHash : ZERO_HASH

        record = {
            contentId, actorType: actor, actorId,
            action: 'ILLEGAL_TRANSITION_ATTEMPT',
            fromStatus: from, toStatus: to,
            reason: reason
        }

        auditTx.auditLogs.insert({
            ...record,
            prevHash,
            recordHash: SHA256(concat(prevHash, canonicalJSON(record)))
        })
    })

    # 非法跃迁通常意味着代码缺陷或攻击尝试，需要告警而非静默记录
    Metrics.increment('audit.illegal_transition_attempt', tags = {
        from, to, actorType: actor
    })
    Alerting.fire('ILLEGAL_STATE_TRANSITION',
                  detail = { contentId, from, to, actor, actorId, reason })
```

> **为什么非法跃迁必须用独立事务**：审计日志的价值恰恰在于记录"不该发生的事"。如果这条记录和业务事务同生共死，那么每一次非法跃迁都会连带回滚掉它自己的审计记录——系统会显得"从未发生过任何非法跃迁"，而这正是攻击者希望看到的。

**调用方必须真正调用它**。`transitionStatus` 内部无法自行记录非法跃迁（它的日志写入与业务更新同事务），因此约定：**所有调用点统一通过一个薄包装调用，而非直接调用 `transitionStatus`**。

```
# 唯一的对外入口：包装 transitionStatus，负责捕获并持久化非法跃迁
FUNCTION applyTransition(contentId, to, actor, actorId?, reason?, detail?):
    TRY:
        RETURN transitionStatus(contentId, to, actor, actorId, reason, detail)
    CATCH IllegalStateTransition AS e:
        # 业务事务此时已回滚；用独立事务写入安全事件。
        # 依赖约定：IllegalStateTransition 异常必须携带 from/to 字段
        # （由 transitionStatus 在 raise 时构造，见下方 raise 处的约定）。
        onIllegalTransitionAttempt(contentId, e.from, e.to,
                                   actor, actorId, reason)
        RAISE e        # 继续向上抛：调用方仍需感知失败，不能静默吞掉
```

**约定**：本文其余各处出现的 `transitionStatus(...)` 调用，在实现时一律应写作 `applyTransition(...)`；`transitionStatus` 本身只应由 `applyTransition` 调用，不直接对外暴露。**01 节所述的"单一入口"即指 `applyTransition`。** 本文为了突出状态机本身的逻辑，在业务伪代码中保留了 `transitionStatus` 的简写形式。

**异常必须携带状态的约定**：`transitionStatus` 在抛出时须显式构造异常对象，而非依赖调用方快照：

```
RAISE IllegalStateTransition({ from: from, to: to,
                               contentId: contentId,
                               attemptedBy: { actor, actorId } })
```

> 状态白名单与 `isPubliclyVisible()` 定义见本节开头的「状态跃迁白名单」。

### 3.4.6 人工复审与申诉

```
FUNCTION manualReview(moderator, contentId, decision, note, banUploader?):
    # 权限校验：必须在事务外先做，且用权限而非角色判断
    IF NOT RBAC.hasPermission(moderator, 'content:audit'):
        RETURN error(403, "AUDIT_NOT_PERMITTED")

    content = DB.contents.findOne(id = contentId)

    # ⚠️ 先快照"领取前"的关键字段：领取操作会立刻写入 reviewedBy，
    #    若领取后再读 content.reviewedBy，条件恒为 false，反馈闭环永不触发。
    #    因此必须在 updateWhere 之前取值。
    priorReview = {
        reviewedBy: content.reviewedBy,
        riskScore:  content.riskScore,
        riskLevel:  content.riskLevel,
        findings:   content.precheckDetail?.findings ?? []
    }

    # ── 乐观锁领取：防止两个审核员同时处理同一内容 ──────────────────
    claimed = DB.contents.updateWhere(
        where = { id: contentId, status: 'PENDING_MANUAL',
                  reviewedBy: NULL, deletedAt: NULL },
        set   = { reviewedBy: moderator.id, reviewedAt: now() }
    )
    IF claimed == 0:
        RETURN error(409, "ALREADY_CLAIMED_OR_RESOLVED")

    IF decision == 'APPROVE':
        transitionStatus(contentId, to = 'PUBLISHED', actor = 'MODERATOR',
                         actorId = moderator.id, reason = note)
        publishContent(contentId)

        # ── 反馈闭环：人工推翻 AI 判定 → 记录为潜在误杀样本 ─────────
        # 触发条件：内容此前从未被人审过（priorReview.reviewedBy 为空）
        # 且 AI 给出了中高风险评分——即"AI 认为有问题，但人认为没问题"，
        # 这正是误杀的信号，是调整阈值的核心训练数据。
        IF priorReview.reviewedBy IS NULL
           AND priorReview.riskLevel IN ('MEDIUM', 'HIGH', 'CRITICAL'):
            MLFeedback.record({
                contentId, aiRiskScore: priorReview.riskScore,
                humanDecision: 'APPROVE',
                hitRules: priorReview.findings,
                label: 'FALSE_POSITIVE_CANDIDATE'
            })

    ELSE IF decision == 'REJECT':
        transitionStatus(contentId, to = 'REJECTED', actor = 'MODERATOR',
                         actorId = moderator.id, reason = note)

        # 驳回必须给用户明确理由，且理由要可申诉
        Notify.send(content.uploaderId, 'CONTENT_REJECTED',
                    detail = { contentId, reason: note,
                               appealUrl: '/appeals/new?contentId={contentId}' })

        IF banUploader:
            IF NOT RBAC.hasPermission(moderator, 'user:ban'):
                RETURN error(403, "BAN_NOT_PERMITTED")
            banUser(content.uploaderId, byUser = moderator.id, reason = note)

    RETURN { status: content.status, reviewedBy: moderator.id }


# ── 超时未决策的自动释放 ─────────────────────────────────────────────
# 对应 06 节"领取后 15 分钟未决策 → 自动释放回队列"。
# 没有这个任务，审核员关闭页面后该内容会永久停留在"已被领取"状态：
# reviewedBy 非空使其他审核员无法领取，而 PENDING_MANUAL 自身不会离开。
FUNCTION releaseStaleClaims():
    cutoff = now() - CONFIG.REVIEW_CLAIM_TIMEOUT (default 15min)

    staleRows = DB.contents.findWhere(
        status: 'PENDING_MANUAL',
        reviewedBy IS NOT NULL,
        reviewedAt < cutoff,
        deletedAt IS NULL
    )

    FOR row IN staleRows:
        released = DB.contents.updateWhere(
            where = { id: row.id, status: 'PENDING_MANUAL',
                      reviewedBy: row.reviewedBy },   # 乐观锁：确认仍是同一领取者
            set   = { reviewedBy: NULL, reviewedAt: NULL }
        )

        IF released == 0:
            CONTINUE        # 已被他人处理或已决策，跳过

        # 注意：状态本身不变（仍是 PENDING_MANUAL），只是解除占用，
        # 因此不需要走 applyTransition，也不产生状态流转日志。
        # 但仍需写审计记录，以便发现"反复领取却不处理"的行为。
        AuditLog.record(event = 'REVIEW_CLAIM_RELEASED',
                        contentId = row.id,
                        actorType = 'SYSTEM',
                        detail = { previousReviewer: row.reviewedBy,
                                   claimedAt: row.reviewedAt,
                                   heldMinutes: minutesBetween(row.reviewedAt, now()) })

        Metrics.increment('audit.claim_released_stale',
                          tags = { reviewerId: row.reviewedBy })


# 行为监控：同一审核员频繁超时未决策时告警，而非静默重试
FUNCTION monitorReviewerClaimHygiene():
    FOR reviewerId IN recentReviewers(window = '7d'):
        released = Metrics.get('audit.claim_released_stale', tags = { reviewerId })
        total    = Metrics.get('audit.claim_acquired',     tags = { reviewerId })

        IF total > 20 AND released / total > 0.3:
            Alerting.fire('REVIEWER_CLAIM_ABANDONMENT_HIGH',
                          detail = { reviewerId, releasedCount: released,
                                     totalClaims: total })


# ── 申诉时效的落地 ───────────────────────────────────────────────────
# 06 节向用户展示了"申诉截止时间 = 拦截时间 + 30 天"，但仅靠前端倒计时
# 是**不可执行**的承诺：用户可以越过前端直接调用接口。因此必须有服务端
# 时效校验，否则 BLOCKED 内容将永久可申诉。
#
# ⚠️ 本函数返回错误对象或 NULL，**不**用 RETURN error(...) 的简写形式。
#    原因是它被 submitAppeal 以函数调用方式使用；若声明为 void 并
#    RETURN error(...)，错误值会被丢弃，调用方照常继续 —— 校验形同虚设。
FUNCTION checkAppealDeadline(contentId) -> Error | NULL:
    content = DB.contents.findOne(id = contentId)

    # 申诉起始时间取"进入 BLOCKED/REJECTED 的那一刻"，以审计日志为准。
    # 不能用 contents.updatedAt——内容可能因无关字段（如浏览量）被更新。
    # ⚠️ 本列表必须与 actionForTransition 的产出保持一致；
    #    新增会进入 BLOCKED/REJECTED 的 actor 时，此处要同步补动作名。
    terminalLog = DB.auditLogs.findOneWhere(
        contentId = contentId,
        action IN ['MANUAL_REJECT', 'AUTO_REJECT',
                   'AUTO_BLOCK', 'MANUAL_BLOCK'],
        orderBy = 'createdAt DESC'
    )
    IF terminalLog IS NULL:
        RETURN error(400, "CONTENT_NOT_APPEALABLE")

    deadline = terminalLog.createdAt + CONFIG.APPEAL_WINDOW (default 30d)

    IF now() > deadline:
        # 410 Gone 而非 403：明确告知"曾经可以，现已过期"，
        # 并在 detail 中给出人工客服入口，避免用户感到被无视。
        RETURN error(410, "APPEAL_WINDOW_EXPIRED",
                     detail = { deadline, blockedAt: terminalLog.createdAt,
                                supportUrl: '/support/contact' })

    RETURN NULL   # 在有效期内，允许继续


# 定时任务：每日执行一次。职责是**审计与告警**，不强行关闭用户申诉。
#
# 设计取舍说明：本函数刻意**不**把超期内容改为 DELETED。
#   · BLOCKED / APPEALING 不是"有时限的状态"——申诉窗口约束的是
#     用户**提交**的权利（由 checkAppealDeadline 服务端强制），
#     而一旦申诉已提交，处理时效是平台的义务，不能因为平台自己处理慢
#     就宣布用户的申诉失效。这与 06 节"申诉队列优先级高于新内容审核"
#     的产品意图一致。
#   · 因此这里只做两件事：暴露积压、暴露不一致。真正的状态收敛依赖
#     人工处理申诉（resolveAppeal），而非定时任务。
FUNCTION auditAppealHealth():
    now_ = now()

    # ① 积压超 SLA 的未结申诉：这是运营问题，需要告警而非静默清理
    overdueAppeals = DB.appeals.findWhere(
        status IN ['PENDING', 'UNDER_REVIEW'],
        createdAt < now_ - CONFIG.APPEAL_PROCESSING_SLA (default 7d)
    )
    IF COUNT(overdueAppeals) > 0:
        Alerting.fire('APPEAL_BACKLOG_BREACHED_SLA',
                      detail = {
                          count: COUNT(overdueAppeals),
                          oldestAgeDays: daysBetween(
                              MIN(a.createdAt FOR a IN overdueAppeals), now_),
                          slaDays: days(CONFIG.APPEAL_PROCESSING_SLA)
                      })

    # ② 已通过但内容仍不可见 → 恢复流程失败，必须人工介入
    approvedStale = DB.appeals.findWhere(
        status: 'APPROVED',
        resolvedAt < now_ - 1d          # 给异步恢复留 1 天缓冲
    )
    FOR appeal IN approvedStale:
        content = DB.contents.findOne(id = appeal.contentId)
        IF NOT isPubliclyVisible(content.status):
            Alerting.fire('APPEAL_APPROVED_BUT_CONTENT_NOT_RESTORED',
                          detail = { appealId: appeal.id, contentId: content.id,
                                     currentStatus: content.status,
                                     resolvedAt: appeal.resolvedAt,
                                     severity: 'HIGH' })

    # ③ 一致性校验：状态为 APPEALING 但已无未结申诉
    #    （说明某次流转中断，内容会被永久挂起——这是必须自动收敛的异常）
    danglingAppealing = DB.contents.findWhere(
        status: 'APPEALING', deletedAt IS NULL,
        NOT EXISTS (SELECT 1 FROM appeals a
                    WHERE a.content_id = contents.id
                      AND a.status IN ('PENDING', 'UNDER_REVIEW'))
    )
    FOR content IN danglingAppealing:
        # ⚠️ 逐条 TRY/CATCH：单条失败绝不能中断整个批次的收敛，
        #    否则一条坏数据会让其余所有悬挂内容永久得不到修复。
        TRY:
            # 回退到 BLOCKED 并重新开放申诉。
            # 这对用户是安全方向（可再申诉），对平台也是安全方向（不误放行）。
            # 依赖 STATE_MACHINE 中的 'APPEALING' → 'BLOCKED' 边。
            applyTransition(content.id, to = 'BLOCKED', actor = 'SYSTEM',
                            reason = 'appeal_state_reconciliation',
                            detail = { reason: 'no_open_appeal_found' })

            AuditLog.record(event = 'DANGLING_APPEALING_RECONCILED',
                            contentId = content.id, actorType = 'SYSTEM',
                            detail = { reconciledAt: now_ })

            Metrics.increment('audit.dangling_appealing_reconciled')
        CATCH e:
            # 单条修复失败只记录并继续，不让异常冒泡中断循环
            AuditLog.record(event = 'DANGLING_APPEALING_RECONCILE_FAILED',
                            contentId = content.id, actorType = 'SYSTEM',
                            detail = { error: e.message, fromStatus: 'APPEALING' })
            Metrics.increment('audit.dangling_appealing_reconcile_failed')
            CONTINUE
```


```
FUNCTION submitAppeal(user, contentId, reason, evidenceFiles):
    content = DB.contents.findOne(id = contentId)

    IF content.uploaderId != user.id:
        RETURN error(403, "FORBIDDEN")

    IF content.status NOT IN ['BLOCKED', 'REJECTED']:
        RETURN error(400, "CONTENT_NOT_APPEALABLE")

    # 服务端时效校验：前端倒计时只是提示，这里才是权威判定。
    # ⚠️ 必须接收返回值——若忽略它，超期申诉会被照常受理，校验即失效。
    deadlineError = checkAppealDeadline(contentId)
    IF deadlineError IS NOT NULL:
        RETURN deadlineError

    # ── 计数前先做全部前置校验 ──────────────────────────────────────
    # ⚠️ 顺序很重要：配额自增必须在所有可能失败的校验**之后**。
    #    若先自增后校验，被 409/400 拒绝的请求也会白扣一次每日额度，
    #    用户会看到"今天 3 次都用完了"但一次申诉都没提交成功。
    # 一个内容同时只能有一个未结申诉（由部分唯一索引兜底）
    existing = DB.appeals.findOne(contentId, status IN ['PENDING', 'UNDER_REVIEW'])
    IF existing:
        RETURN error(409, "APPEAL_ALREADY_OPEN")

    # 拒绝以完全相同理由重复提交（对应 06 节的产品约束）
    previousAppeals = DB.appeals.findWhere(contentId, appellantId: user.id,
                                           status IN ['REJECTED', 'WITHDRAWN'])
    IF ANY(normalizeReason(a.reason) == normalizeReason(reason) FOR a IN previousAppeals):
        RETURN error(409, "APPEAL_REASON_DUPLICATE",
                     detail = { hint: '请补充新的理由或证据后再提交。' })

    # ── 配额检查 + 占用（仅在此后的路径才真正消耗额度） ─────────────
    appealRateKey = "appeal:rate:{user.id}:" + today()
    current = Redis.GET(appealRateKey) OR 0
    IF current >= CONFIG.APPEAL_RATE_PER_DAY (default 3):
        RETURN error(429, "APPEAL_RATE_LIMITED",
                     detail = { limit: CONFIG.APPEAL_RATE_PER_DAY,
                                used: current, resetAt: tomorrow() })

    appeal = DB.appeals.insert({
        contentId, appellantId: user.id, reason,
        evidenceKeys: evidenceFiles.map(uploadToQuarantine),
        status: 'PENDING'
    })

    # 落库成功后才自增：撤回不退还额度，但失败提交不消耗额度
    Redis.INCR(appealRateKey)
    Redis.EXPIRE(appealRateKey, 86400)

    transitionStatus(contentId, to = 'APPEALING', actor = 'USER', actorId = user.id,
                     reason = 'user_submitted_appeal',
                     detail = { appealId: appeal.id })
    # 注：不再单独写一条 APPEAL_SUBMITTED 的 AuditLog.record。
    #     状态流转本身已写入该动作名且携带 from_status；重复写入会产生
    #     两条同 action 的记录，其中一条缺 from_status，从而干扰
    #     withdrawAppeal 依据 from_status 反查申诉前状态的查询。

    # 申诉队列优先级高于新内容审核（用户已在等待）
    MQ.publish('audit.appeal.required', { appealId: appeal.id, contentId, priority: 'HIGH' })

    # 申诉不重复扣**上传**配额（与上面的申诉日限额是两套独立的配额），
    # 且由不同审核员处理（避免同一人自我复核）
    RETURN { appealId: appeal.id, status: 'PENDING' }


# ── 用户撤回申诉 ─────────────────────────────────────────────────────
# appeal_status 枚举包含 WITHDRAWN（见 02 节）。若不实现撤回动作，
# 该枚举值永不可达，且用户提交后无法反悔——只能等审核员处理。
FUNCTION withdrawAppeal(user, appealId):
    appeal = DB.appeals.findOne(id = appealId)

    IF appeal.appellantId != user.id:
        RETURN error(403, "FORBIDDEN")

    IF appeal.status NOT IN ['PENDING', 'UNDER_REVIEW']:
        RETURN error(409, "APPEAL_ALREADY_RESOLVED")

    targetStatus = NULL

    DB.transaction(async tx => {
        # 行级锁：与全局流转纪律一致，防止与审核员并发决策冲突
        content = tx.contents.selectForUpdate(id = appeal.contentId)

        tx.appeals.update(appeal, {
            status: 'WITHDRAWN',
            resolvedAt: now(),
            reviewNote: 'Withdrawn by appellant.'
        })

        # ── 回到"申诉前"的状态 ────────────────────────────────────────
        # ⚠️ 不能靠 content.rejectedReason 判断：该字段一旦写入就永不清除，
        #    "曾经被驳回过"与"本次申诉前是 REJECTED"是两回事。
        #    权威依据是审计链中那条携带 from_status 的 APPEAL_SUBMITTED。
        #    该动作名现在只由一个写入者产生（transitionStatus 的
        #    ACTION_FOR_TRANSITION['APPEALING']），且必然带 from_status，
        #    因此查询是可靠的。过滤条件保留为纵深防御。
        priorLog = tx.auditLogs.findOneWhere(
            contentId = appeal.contentId,
            action = 'APPEAL_SUBMITTED',
            fromStatus IS NOT NULL,
            orderBy = 'createdAt DESC'
        )
        # 该记录的 from_status 即申诉前状态（BLOCKED 或 REJECTED）
        IF priorLog IS NULL:
            # 审计链缺失属异常数据，保守起见回退到 BLOCKED（对用户开放申诉入口）
            AuditLog.record(event = 'APPEAL_PRIOR_STATUS_MISSING',
                            contentId = appeal.contentId, actorType = 'SYSTEM',
                            detail = { appealId: appeal.id })
            targetStatus = 'BLOCKED'
        ELSE:
            targetStatus = priorLog.fromStatus

        # 双重保险：只接受两个可申诉的状态作为回退目标
        IF targetStatus NOT IN ['BLOCKED', 'REJECTED']:
            targetStatus = 'BLOCKED'

        affected = tx.contents.updateWhere(
            where = { id: appeal.contentId, status: 'APPEALING' },
            set   = { status: targetStatus, updatedAt: now() }
        )
        IF affected == 0:
            # 内容已不在 APPEALING（审核员刚决策完）→ 放弃撤回，整体回滚
            RAISE ConcurrentModification()

        # ── 审计链：必须补全 prev_hash / record_hash ──────────────────
        # record_hash 在 schema 中为 NOT NULL（见 02 节），缺省即插入失败。
        # 这里直接条件更新而不走 applyTransition，是为了与 appeals 的
        # 状态变更在同一事务内原子完成；代价是要手工维护哈希链。
        prevLog2 = tx.auditLogs.findLatest(contentId = appeal.contentId)
        prevHash = prevLog2 ? prevLog2.recordHash : ZERO_HASH

        record = {
            contentId: appeal.contentId,
            actorType: 'USER', actorId: user.id,
            action: 'APPEAL_WITHDRAWN',
            fromStatus: 'APPEALING', toStatus: targetStatus,
            reason: 'appeal_withdrawn: ' + appealId
        }

        tx.auditLogs.insert({
            ...record,
            prevHash,
            recordHash: SHA256(concat(prevHash, canonicalJSON(record)))
        })
    })

    Metrics.increment('audit.appeal_withdrawn')
    # 返回真实的 targetStatus，而非硬编码——REJECTED 分支下硬编码会与库中不符
    RETURN { appealId, status: 'WITHDRAWN', contentStatus: targetStatus }


FUNCTION resolveAppeal(reviewer, appealId, decision, note):
    appeal = DB.appeals.findOne(id = appealId)

    # ── 回避原则：原审核员不得复核自己驳回的申诉 ────────────────────
    originalReview = DB.auditLogs.findOne(
        contentId = appeal.contentId, action = 'MANUAL_REJECT')
    IF originalReview AND originalReview.actorId == reviewer.id:
        RETURN error(403, "CONFLICT_OF_INTEREST")

    IF decision == 'APPROVE':
        DB.appeals.update(appeal, { status: 'APPROVED', reviewerId: reviewer.id,
                                    reviewNote: note, resolvedAt: now() })
        # 重新读取最新快照：内容此刻应为 APPEALING（由 submitAppeal 写入）。
        # transitionStatus 内部会再次锁定并读状态，此处读取仅为后续统计使用。
        content = DB.contents.findOne(appeal.contentId)
        transitionStatus(appeal.contentId, to = 'PUBLISHED',
                         actor = 'MODERATOR', actorId = reviewer.id,
                         reason = 'appeal_approved: ' + note)

        # ── 误杀率统计的关键数据点 ─────────────────────────────────
        Metrics.increment('audit.false_positive_confirmed', tags = {
            originalRiskScore: content.riskScore,
            engineVersion: content.precheckEngineVersion
        })
        # 触发阈值复审：若某规则误杀率持续超限，自动告警
        IF ruleFalsePositiveRate(content.precheckDetail, window = '7d') > CONFIG.FPR_ALERT_THRESHOLD:
            Alerting.fire('RULE_FALSE_POSITIVE_SPIKE',
                          detail = { ruleIds: extractRuleIds(content.precheckDetail) })

    ELSE:
        DB.appeals.update(appeal, { status: 'REJECTED', reviewerId: reviewer.id,
                                    reviewNote: note, resolvedAt: now() })
        transitionStatus(appeal.contentId, to = 'REJECTED',
                         actor = 'MODERATOR', actorId = reviewer.id,
                         reason = 'appeal_rejected: ' + note)

    AuditLog.record(event = 'APPEAL_RESOLVED', contentId = appeal.contentId,
                    actorType = 'MODERATOR', actorId = reviewer.id,
                    detail = { appealId, decision, note })
```

---

## 3.5 签名 URL 与防盗链

### 3.5.1 下载签名 URL 签发

```
FUNCTION issueDownloadUrl(user, contentId, assetKind):
    content = DB.contents.findOne(id = contentId, deletedAt IS NULL)

    # ⚠️ 与播放一致：可见性判定必须包含 AUTO_PUBLISHED
    IF content IS NULL OR NOT isPubliclyVisible(content.status):
        RETURN error(404, "NOT_FOUND")

    # 权限：私有内容仅上传者与管理员可下载
    IF content.visibility == 'PRIVATE' AND content.uploaderId != user.id
       AND NOT RBAC.hasPermission(user, 'content:delete_any'):
        RETURN error(403, "FORBIDDEN")

    # ── 频次限制：防批量爬取（每用户每内容每小时 N 次） ──────────────
    rateKey = "download:rate:{user.id}:{contentId}"
    IF Redis.INCR(rateKey) > CONFIG.DOWNLOAD_RATE_PER_HOUR (default 5):
        RETURN error(429, "DOWNLOAD_RATE_LIMITED", retryAfter = Redis.TTL(rateKey))
    Redis.EXPIRE(rateKey, 3600)

    # ── 带宽配额：单用户当日累计下载字节上限 ────────────────────────
    # 与次数配额互补：次数限制挡不住"下载一个 50GB 的源文件"。
    bandwidthKey = "download:bytes:{user.id}:" + today()
    usedBytes = Redis.GET(bandwidthKey) OR 0

    IF usedBytes >= CONFIG.DOWNLOAD_BYTES_DAILY (default 10737418240):   # 10 GiB
        RETURN error(429, "DOWNLOAD_BANDWIDTH_QUOTA_EXCEEDED",
                     detail = { limit: CONFIG.DOWNLOAD_BYTES_DAILY,
                                used: usedBytes,
                                resetAt: tomorrow() })

    # 预估本次传输量；实际用量在下载网关按已传输字节累加
    estimatedBytes = resolveAssetSize(content, assetKind)
    IF usedBytes + estimatedBytes > CONFIG.DOWNLOAD_BYTES_DAILY:
        RETURN error(429, "DOWNLOAD_BANDWIDTH_QUOTA_EXCEEDED",
                     detail = { limit: CONFIG.DOWNLOAD_BYTES_DAILY,
                                used: usedBytes,
                                required: estimatedBytes,
                                resetAt: tomorrow() })

    # ── 创建授权记录（落到数据库，可审计、可吊销） ──────────────────
    grant = DB.downloadGrants.insert({
        contentId, userId: user.id, assetKind,
        maxDownloads: CONFIG.DEFAULT_MAX_DOWNLOADS (default 3),
        # maxBytes 为单次授权的传输上限，防止单个 URL 被用于无限制拉取
        maxBytes: MIN(estimatedBytes, CONFIG.MAX_SINGLE_GRANT_BYTES),
        expiresAt: now() + CONFIG.GRANT_TTL (default 30min)
    })

    objectKey = resolveAssetKey(content, assetKind)

    # ── 生成签名 URL ───────────────────────────────────────────────
    # ⚠️ 关键：对象存储（S3/OSS/MinIO）的 presigned URL **本身不支持把用户身份
    #    写入签名并强校验**——S3 的 presign 只保护 key + 过期时间 + 少量
    #    response 头覆盖项，metadata 不参与签名。因此"转发即失效"这一点
    #    必须由**下载网关**承担，不能寄希望于对象存储直接签发。
    #
    #    正确架构：客户端请求应用侧下载端点，由应用校验会话后再流式转发
    #    （或由应用签发带 HMAC 的 CDN 边缘校验 URL，见 3.5.3）。
    expiresAt = now() + CONFIG.SIGNED_URL_TTL (default 300s)

    # 方案 A（推荐，带边缘验签的 CDN 架构）：
    #   应用签发含会话绑定的 CDN URL，由边缘函数验签后回源。
    #   这与播放链路（3.5.2）使用同一套签名机制，保证行为一致。
    signedUrl = signCdnUrl(
        path = downloadPathFor(content, assetKind),
        expiresIn = CONFIG.SIGNED_URL_TTL,
        # 绑定会话与用户：URL 被转发后，他人无此会话 → 验签通过但会话校验失败
        claims = { sid: request.sessionId, uid: user.id, grant: grant.id },
        # 强制下载而非内联渲染，防止 HTML/SVG 触发存储型 XSS
        disposition = 'attachment; filename="' + sanitizeFilename(content.title) + '"'
    )

    # 方案 B（直连对象存储，无 CDN 边缘逻辑）：
    #   接受"URL 在有效期内可被转发"这一事实，改为用短 TTL + 次数配额 + 频次限制
    #   把风险压到可接受范围，并在下载网关侧记录实际下载者。
    #   ⚠️ 若采用方案 B，04 节与 08 节中"会话绑定"的表述必须同步修正，
    #      否则测试用例（08 节"签名 URL 盗用"）会与实现不符。

    AuditLog.record(event = 'DOWNLOAD_URL_ISSUED', actorType = 'USER',
                    contentId = contentId, actorId = user.id,
                    detail = { grantId: grant.id, assetKind, scheme: 'CDN_EDGE_SIGNED' })

    RETURN {
        url: signedUrl,
        expiresAt,
        maxDownloads: grant.maxDownloads,
        # 提示客户端：URL 为一次性短期凭据，勿分享
        notice: 'This URL is bound to your session and expires shortly.'
    }
```

### 3.5.2 播放鉴权（分片级短期签名）

```
FUNCTION issuePlaybackManifest(user, contentId):
    content = DB.contents.findOne(id = contentId)

    # ⚠️ 可见性判断必须包含 AUTO_PUBLISHED —— 这是对外可见的两个终态之一。
    #    只判 PUBLISHED 会导致自动放行的内容无法播放。
    IF NOT isPubliclyVisible(content.status) OR content.deletedAt IS NOT NULL:
        RETURN error(404, "NOT_FOUND")

    # ── 播放权限：支持会员/付费内容扩展点 ──────────────────────────
    IF NOT canPlay(user, content):
        RETURN error(403, "PLAYBACK_NOT_PERMITTED")

    # ── 播放会话 + 并发流限制（原子操作，避免竞态导致超限） ──────────
    sessionId = generatePlaybackSessionId()
    activeSetKey = "play:user:{user.id}:active"
    sessionKey   = "play:session:{sessionId}"

    # 用 Lua 脚本保证"检查 + 加入"原子，避免并发请求同时通过检查。
    # ⚠️ 脚本返回**数值** 1（成功）或 0（已达上限）。
    #    必须显式判断 `== 0`，不能用 `IF NOT acquired` —— 在多数语言中
    #    数值 0 是"有值"的，`NOT 0` 不成立，会把超限请求误判为成功。
    acquired = Redis.eval(LUA_ACQUIRE_STREAM, keys = [activeSetKey, sessionKey],
                          args = [sessionId, user.id, contentId,
                                  CONFIG.MAX_CONCURRENT_STREAMS (default 3),
                                  CONFIG.PLAYBACK_SESSION_TTL_SECONDS (default 14400)])
    IF acquired == 0:
        RETURN error(429, "TOO_MANY_CONCURRENT_STREAMS",
                     detail = { limit: CONFIG.MAX_CONCURRENT_STREAMS,
                                activeStreams: Redis.SCARD(activeSetKey) })

    # 会话记录：TTL 与 HLS 播放时长匹配
    Redis.EXPIRE(sessionKey, CONFIG.PLAYBACK_SESSION_TTL_SECONDS)

    # ── 会话结束时的清理 ────────────────────────────────────────────
    # 三个触发点，缺一不可（否则并发计数只增不减，最终谁都播不了）：
    #   ① 客户端显式调用 POST /playback/{sessionId}/end
    #   ② 心跳缺失：客户端每 30s 上报进度，超过 TTL 未上报由定时任务清理
    #   ③ Redis 过期事件（keyspace notification）触发 SREM
    registerSessionCleanup(user.id, sessionId)

    # ── 下发短期签名清单地址（TTL 短于转码完成时长） ─────────────────
    # 分片请求由 CDN 边缘校验签名，不回落源站，降低源站压力
    signedManifestUrl = signCdnUrl(
        path = content.hlsMasterKey,
        expiresIn = 3600,
        # 绑定播放会话，注销/顶号后旧地址立即失效
        claims = { sid: sessionId, uid: user.id }
    )

    RETURN {
        manifestUrl: signedManifestUrl,
        # 客户端据此做自适应码率切换
        renditions: loadRenditions(content.id),
        dashManifestUrl: signCdnUrl(content.dashManifestKey, 3600),
        subtitles: loadSubtitles(content.id),
        # 播放进度续播
        resumePositionMs: Redis.GET("progress:{user.id}:{contentId}") OR 0,
        danmakuToken: issueDanmakuToken(user, contentId),
        # 客户端须在离开页面/停止播放时调用此端点释放并发流额度
        endSessionEndpoint: '/api/v1/playback/' + sessionId + '/end',
        sessionId
    }
```

并发流额度获取的原子脚本（避免"检查通过后并发加入"导致的超限）：

```lua
-- KEYS[1] = play:user:{userId}:active     （活跃会话集合）
-- KEYS[2] = play:session:{sessionId}      （单个会话详情，用于过期自动清理）
-- ARGV[1] = sessionId
-- ARGV[2] = userId
-- ARGV[3] = contentId
-- ARGV[4] = maxConcurrent
-- ARGV[5] = sessionTtlSeconds
-- 返回 1 = 获取成功；0 = 已达上限

local active = redis.call('SCARD', KEYS[1])
if active >= tonumber(ARGV[4]) then
  return 0
end

redis.call('SADD', KEYS[1], ARGV[1])
redis.call('HSET', KEYS[2], 'userId', ARGV[2], 'contentId', ARGV[3],
                            'startedAt', redis.call('TIME')[1])
redis.call('EXPIRE', KEYS[2], tonumber(ARGV[5]))

-- 给集合本身一个较长的 TTL 兜底，防止异常情况下集合永久残留
redis.call('EXPIRE', KEYS[1], tonumber(ARGV[5]) * 2)

return 1
```

**释放逻辑**（与获取配对，缺一则计数只增不减）：

```
FUNCTION releaseStream(userId, sessionId):
    Redis.SREM("play:user:{userId}:active", sessionId)
    Redis.DEL("play:session:{sessionId}")
    Metrics.gauge('playback.active_streams', Redis.SCARD("play:user:{userId}:active"),
                  tags = { userId })

# 兜底清理任务：每 5 分钟扫描活跃集合，移除已过期的会话 ID
FUNCTION cleanupStaleStreams():
    FOR userId IN Redis.SCAN("play:user:*:active"):
        FOR sessionId IN Redis.SMEMBERS(userId):
            IF NOT Redis.EXISTS("play:session:{sessionId}"):
                # 会话详情已过期 → 说明客户端未正常结束，回收额度
                Redis.SREM(userId, sessionId)
```

### 3.5.3 防盗链的多层设计

| 层 | 机制 | 防的是什么 |
|---|---|---|
| 1. Referer 校验 | 白名单域名 | 基础盗链（可被伪造，仅作第一道） |
| 2. 短期签名 | URL 5 分钟有效 + 绑定 grantId | URL 被分享传播 |
| 3. 会话绑定 | 签名含 `sid`，顶号即失效 | 账号共享、长时挂机 |
| 4. 频次限制 | 单用户单内容每小时 5 次 | 脚本化批量下载 |
| 5. 次数配额 | 每 grant 最多 3 次下载 | 单 URL 无限复用 |
| 6. 并发流限制 | 单账户最多 3 路 | 一号多用 |
| 7. CDN 边缘校验 | 边缘验签，未签名请求 403 且不回源 | 绕过应用直连 CDN |

```
FUNCTION verifyCdnSignature(request):
    # 由 CDN 边缘函数执行，源站不参与
    sig  = request.query.signature
    exp  = request.query.expires
    sid  = request.query.sid
    path = request.path

    IF now() > exp:
        RETURN 403   # 过期

    # 签名密钥来自边缘安全配置，定期轮换，支持新旧并存
    expected = HMAC_SHA256(
        key = EdgeSecret.current(),
        msg = path + '|' + exp + '|' + sid
    )

    IF NOT secureCompare(hexToBytes(sig), expected):
        # 尝试上一版本密钥（轮换窗口内）
        IF NOT secureCompare(hexToBytes(sig), HMAC_SHA256(EdgeSecret.previous(), msg)):
            AuditLog.record(event = 'CDN_SIGNATURE_INVALID', actorType = 'SYSTEM',
                            detail = { path = path }, severity = 'WARN')
            RETURN 403

    # 会话有效性：顶号/登出后旧签名立即失效
    IF NOT PlaybackSession.exists(sid):
        RETURN 403

    RETURN allow()
```

---

## 3.6 令牌与会话安全

```
FUNCTION rotateRefreshToken(refreshToken):
    tokenHash = SHA256(refreshToken)
    session = DB.sessions.findOne(refreshTokenHash = tokenHash)

    IF session IS NULL:
        RETURN error(401, "INVALID_TOKEN")

    # ── 重放检测：已轮换过的令牌再次出现 → 判定为令牌泄露 ────────────
    IF session.rotatedAt IS NOT NULL:
        # 这是关键的安全信号：攻击者与合法用户同时持有同一令牌
        # 处置：吊销该用户全部会话，强制重新登录
        DB.sessions.updateWhere({ userId: session.userId, revokedAt: NULL },
                                { revokedAt: now() })
        AuditLog.record(event = 'REFRESH_TOKEN_REPLAY_DETECTED',
                        actorType = 'USER', actorId = session.userId,
                        severity = 'CRITICAL')
        Notify.send(session.userId, 'SECURITY_ALERT_SESSIONS_REVOKED')
        RETURN error(401, "TOKEN_REVOKED")

    IF session.expiresAt < now() OR session.revokedAt IS NOT NULL:
        RETURN error(401, "TOKEN_EXPIRED")

    # ── 轮换：旧令牌立即失效，签发新令牌 ────────────────────────────
    newToken = generateOpaqueToken()
    DB.transaction(tx => {
        tx.sessions.update(session, { rotatedAt: now() })
        tx.sessions.insert({
            userId: session.userId, refreshTokenHash: SHA256(newToken),
            deviceLabel: session.deviceLabel, ipHash: session.ipHash,
            mfaVerified: session.mfaVerified,      # MFA 状态跨轮换保持
            expiresAt: now() + 30d
        })
    })

    # 重新签发 Access Token 时携带最新权限，权限变更最多 15 分钟内生效
    permissions = RBAC.resolvePermissions(session.userId)
    RETURN { accessToken: signJWT({ sub: session.userId, perms: permissions }, '15m'),
             refreshToken: newToken }
```

### 3.6.1 权限变更的即时生效

```
# 权限变更时（如管理员被降权），需要立即让已签发的 Access Token 失效
FUNCTION onPermissionChanged(userId):
    # ① 加入令牌黑名单，TTL 覆盖最长 Access Token 有效期
    Redis.SET("revoked:user:{userId}", now(), ttl = 900)

    # ② 通过 Pub/Sub 通知所有应用实例清理本地权限缓存
    Redis.PUBLISH('rbac:invalidate', { userId, at: now() })

    # ③ 吊销全部 Refresh Token，强制重新登录
    DB.sessions.updateWhere({ userId, revokedAt: NULL }, { revokedAt: now() })

    AuditLog.record(event = 'PERMISSIONS_REVOKED', actorType = 'USER',
                    actorId = userId, severity = 'WARN')

# 每次请求的鉴权中间件
FUNCTION authGuard(request):
    token = extractBearer(request)
    claims = verifyJWT(token)

    IF claims IS NULL:
        RETURN 401

    # 黑名单检查（轻量 Redis 查询，命中率低）
    IF Redis.EXISTS("revoked:user:{claims.sub}"):
        RETURN 401

    # 特权接口额外要求 MFA 已在本会话中验证过
    IF request.route.requiresMfa AND NOT claims.mfa:
        RETURN 403, "MFA_REQUIRED_FOR_THIS_OPERATION"

    request.user = { id: claims.sub, perms: claims.perms, sessionId: claims.sid }
    RETURN next()
```

---

## 3.7 安全要点速查表

| 威胁 | 防护措施 | 落点 |
|---|---|---|
| 密码离线破解 | Argon2id (64MiB, t=3) + 独立盐 | 3.1 |
| 账户枚举 | 统一错误码 + 恒定时间 + 假哈希计算 | 3.2.2 第 2 层 |
| 暴力破解 | IP 限流 + 账户锁定 + CAPTCHA + 指数退避 | 3.2.4 |
| 时序攻击 | `timingSafeEqual` 用于所有机密比对 | 3.2.3 |
| 密钥泄露 | 信封加密 + KMS/HSM + 明文密钥内存清零 | 3.3.2 |
| 密文挪用 | AES-GCM 的 AAD 绑定字段用途 | 3.3.2 |
| JWT 算法混淆 | 强制校验 `alg`，拒绝 `none` | 3.3.3 |
| 上传恶意文件 | 魔数嗅探 + ffprobe 沙箱 + quarantine 标签 | 3.4.2 |
| 存储型 XSS | `Content-Disposition: attachment` + 类型白名单 | 3.5.1 |
| URL 盗链 | 短期签名 + 会话绑定 + CDN 边缘验签 | 3.5.3 |
| 令牌重放 | Refresh Token 轮换 + 重放检测 + 全量吊销 | 3.6 |
| 权限提升 | 默认 USER + 双因子激活 + 权限变更即时吊销 | 3.2.2 / 3.6.1 |
| 审核日志篡改 | append-only 触发器 + 哈希链 | 2.5 |
| 过度审核 | 三档阈值 + 申诉 + 误杀率监控 + 灰度 | 3.4.4 / 3.4.6 |
