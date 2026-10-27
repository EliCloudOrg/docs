# EliCloud SSO：OIDC 扩展实现规格

> **本文件是 `docs/architecture.md` 的续篇**，只讲一件事：把现有 SSO（密码直连 + JWKS）扩成**标准 OIDC Provider**。
> `architecture.md` 的 §0.4 硬约束（issuer 逐字一致、RS256、`PUBLIC_BASE_URL` 单一真源、IP 阶段）**全部继续有效**，本文件不重复。
> 基线实现：`sso/`（已部署在 `https://146.56.237.33/auth`，README 见 `sso/README.md`）。

---

## 1. 为什么做这次扩展（结论）

现有实现的认证流程是**密码直连**（`/v1/login` 收用户名密码 → 直接发 token）。它覆盖不了已确认的四类客户端：

| 客户端 | 密码直连能否覆盖 | OIDC 方案 |
|---|---|---|
| 自家 Web 前端 | 能（但密码经前端转发） | 授权码 + PKCE |
| 自家手机 App（内嵌表单） | 能 | 保留密码直连作为私有兼容端点；或改用系统浏览器 |
| **自家 App 用系统浏览器/WebView 登录** | **不能** | 授权码 + PKCE + 自定义 scheme 回跳 |
| **第三方 / 自研 CLI、桌面工具** | **不能**（不提供密码） | **设备授权流程（RFC 8628）** |

另外 OAuth 2.1 已**移除** Resource Owner Password Credentials，所以"以后认真做多客户端"早晚要走授权码流程。因此：**保留现有端点，补齐标准 OIDC**。

### 1.1 本次决策（已定）

| 决策 | 取值 |
|---|---|
| 实现方式 | **在现有 FastAPI 服务上补**，不换 Casdoor/Keycloak（复用已打通的 JWKS、密钥轮换、refresh 轮换、网关路径、IP→域名切换设计） |
| 必须支持的流程 | `authorization_code` + **PKCE(S256)**、`refresh_token`、**`device_code`(RFC 8628)** |
| 保留的私有流程 | **密码直连** `/v1/login`（给自家 App/前端用，标记为非标准私有端点） |
| 客户端注册 | **静态注册**（管理接口 + CLI），**不实现** RFC 7591 动态注册 |
| 同意界面 | 一期**自动同意**已注册的 scope；保留 `consents` 表，二期再做交互式同意页 |
| issuer | 不变，仍由 `PUBLIC_BASE_URL` 派生（IP 阶段 `https://146.56.237.33/auth`） |

### 1.3 实现期对齐记录（**与前文冲突时以本小节为准**）

开工前把本文档与线上现状对了一遍，确认三处冲突并已拍板：

| 事项 | 结论 | 覆盖了前文哪里 |
|---|---|---|
| 私有 `POST /v1/logout` | **废弃**：登出统一走标准 RP-Initiated Logout（`/logout`）。自家 App/前端将来也走标准流程 | 覆盖 §1.2、§2.1、§2.8 里"保留私有 logout"的说法；§2.1 中"两个路由抢同一外部路径"的冲突随之消失 |
| `/v1/userinfo` 的 claim | **严格标准**：只返回 `preferred_username`，**不再返回 `username`**；且按 scope 过滤 | 覆盖 §1.2 的"不改语义"；需同步修改现有测试里对 `username` 的断言 |
| `refresh_tokens` 的列 | 除本文档要求的 `client_id`、`scope` 外，**再加一列 `session_id`** | §2.7 要求"撤销该会话的 refresh 链"，没有这一列只能退化成"撤销该用户全部链"（会误伤其它设备） |

另有两处**与现状不符的事实**已就地修正：§6.2 的 Cookie `Secure`、§12 的"自签证书"。

### 1.2 复用 / 新增 / 废弃 一览

| 现有代码 | 处置 |
|---|---|
| `app/keys.py`（RS256 + kid + JWKS + 轮换） | **原样复用**，OIDC 的 `id_token` 用它签 |
| `refresh_tokens` 轮换 + family 整链撤销 | **复用**，加 `client_id`、`scope` 两列 |
| `users`、bcrypt、登录限流、错误结构 | **复用** |
| `oauth_clients` 表（当前"只建表不使用"） | **启用并扩列**（见 §6.1） |
| `app/routers/wellknown.py` | **改写**：删掉 `/authorize` 死链、`token_endpoint` 指向 `/token`，补齐标准字段 |
| 发现文档里的 `authorization_endpoint` 占位 | **落地为真端点**（README:278 的已知限制就此消除） |
| `/v1/login` `/v1/refresh` `/v1/register` `/v1/userinfo` | **保留**，不改语义（自家客户端在用）；`/v1/userinfo` 的 claim 按 §1.3 收紧 |

---

## 2. 端点契约

### 2.0 为什么有些路径带版本、有些不带（动手前先看这条）

**判断规则：路径归谁定义 —— 协议定义的不能带版本，我们自己定义的必须带版本。**

| 路径类别 | 例子 | 谁定义 | 带版本？ | 原因 |
|---|---|---|---|---|
| 协议端点 | `/authorize`、`/token`、`/device_authorization`、`/logout` | OAuth 2.0（RFC 6749）、OIDC Core、设备流程（RFC 8628） | **❌ 禁止** | 路径被固化在协议里；`AppAuth`、`oidc-client`、`google-auth-library` 等标准库会硬编码或校验最后一段，改了就接不通 |
| 协议发现 | `/.well-known/openid-configuration`、`/.well-known/jwks.json` | RFC 8414 / OIDC Discovery / RFC 7517 | **❌ 禁止** | 位置必须严格是 `issuer + /.well-known/<固定名>`，**连父路径都不能加**（所以网关必须剥掉 `/auth`） |
| 自有私有 API | `/v1/login`、`/v1/refresh`、`/v1/register`、`/v1/clients` | **我们自己** | **✅ 必须** | 请求体/字段/错误码将来可能不兼容变更，`/v1` 是留给自己的演进退路 |

**三条推论**：

1. **绝不要"统一风格"把协议端点改成 `/v1/token`** —— 这是本次扩展最容易犯的错，会让所有标准客户端失效。
2. **版本号管不到协议端点**：私有 API 将来出 `/v2` 时，OIDC 端点**仍然是 `/auth/token`**。协议的不兼容升级靠发现文档新增能力字段 + JWKS 密钥轮换，不靠 URL 版本。
3. **`issuer` 天然无版本**（`.../auth`），且不可在阶段切换之外变动；`jwks_uri` 由它派生、客户端会缓存，同样不能带版本（见 §0.4 硬约束 1）。

**这条规则直接决定网关的分块**（§8）：类别 ① 必须只剥前缀、类别 ③ 必须改写成 `/v1/...`。**分错块的表现就是 `/auth/token` 被改写成 `/v1/token` → 404**，正是 §8 原稿踩过的坑。

**一处"别扭但不冲突"**：`/auth/userinfo`（标准位置）与 `/auth/v1/userinfo`（一期私有路径）指向**同一实现** —— 这是兼容层：标准路径给第三方与新客户端，`/v1` 路径给已在运行的自家客户端；两者 claim 语义一致（按 §1.3 都只返回 `preferred_username`），等自家客户端迁移完再决定是否下线旧的。

### 2.1 路径总表

内部路径**不带 `/auth`**（网关剥离/重写，见 §8）。新增的是前 4 行：

| 方法 | 服务内路径 | 对外路径 | 性质 | 说明 |
|---|---|---|---|---|
| GET/POST | `/authorize` | `/auth/authorize` | **新增** | 授权端点；渲染登录页或 302 带 `code` 回跳 |
| POST | `/token` | `/auth/token` | **新增** | 令牌端点；`authorization_code` / `refresh_token` / `device_code` |
| POST | `/device_authorization` | `/auth/device_authorization` | **新增** | 设备流程第一步，返回 `user_code` |
| GET/POST | `/logout` | `/auth/logout` | **新增** | OIDC 会话登出（RP-Initiated Logout） |
| GET | `/v1/clients` 等 | `/auth/v1/clients` | **新增** | 客户端管理（需管理员令牌） |
| POST | `/v1/register` | `/auth/register` | 保留 | 私有注册 |
| POST | `/v1/login` | `/auth/login` | **保留（私有）** | 密码直连，非标准 |
| POST | `/v1/refresh` | `/auth/refresh` | **保留（私有）** | 与 `/token` 的 `refresh_token` 行为一致 |
| GET | `/v1/userinfo` | `/auth/userinfo` | 保留 | 标准 UserInfo；**按 scope 过滤 claim**，只返回 `preferred_username`（§1.3） |
| ~~POST~~ | ~~`/v1/logout`~~ | ~~`/auth/logout`~~ | **已废弃** | 按 §1.3 的 A3 决策**已删除**；登出统一走标准 `GET|POST /logout` |
| GET | `/.well-known/jwks.json` | `/auth/.well-known/jwks.json` | 保留 | 公钥集 |
| GET | `/.well-known/openid-configuration` | `/auth/.well-known/openid-configuration` | **改写** | 见 §2.2 |

> ⚠️ **§1.3 已覆盖本节**：私有 `/v1/logout` **已删除**（线上实测 404），登出只剩标准 `GET|POST /logout`。
> 标准登出撤销的是"该浏览器会话派生的 refresh 链"（靠 `refresh_tokens.session_id` 精确定位），
> 同账号其它设备不受影响。代价：密码直连签发的令牌没有会话绑定，无法被标准登出撤销 —— 见 `sso/README.md` §18.4。

### 2.2 发现文档（`openid-configuration`）目标形态

```json
{
  "issuer": "https://146.56.237.33/auth",
  "authorization_endpoint": "https://146.56.237.33/auth/authorize",
  "token_endpoint": "https://146.56.237.33/auth/token",
  "userinfo_endpoint": "https://146.56.237.33/auth/userinfo",
  "jwks_uri": "https://146.56.237.33/auth/.well-known/jwks.json",
  "end_session_endpoint": "https://146.56.237.33/auth/logout",
  "device_authorization_endpoint": "https://146.56.237.33/auth/device_authorization",

  "response_types_supported": ["code"],
  "response_modes_supported": ["query"],
  "grant_types_supported": [
    "authorization_code",
    "refresh_token",
    "urn:ietf:params:oauth:grant-type:device_code"
  ],
  "code_challenge_methods_supported": ["S256"],
  "token_endpoint_auth_methods_supported": ["client_secret_basic", "client_secret_post", "none"],
  "id_token_signing_alg_values_supported": ["RS256"],
  "subject_types_supported": ["public"],
  "scopes_supported": ["openid", "profile", "email", "offline_access", "pdf:read", "pdf:write", "mc:whitelist", "files:read", "files:write", "userinfo:read", "userinfo:write"],
  "claims_supported": ["sub", "preferred_username", "email", "email_verified", "name"]
}
```

**两条硬规则**：

1. **发现文档里声明的每个端点都必须真的存在。** 现在是 `/authorize` 不存在、`token_endpoint` 指向 `/refresh` —— 标准客户端读完发现文档直接 404。**这是本次扩展第一件要修的事**（改起来 5 分钟）。
2. 所有 URI 都从 `settings.external_url(...)` 派生，**零硬编码**（沿用 `config.py` 现有做法）。

### 2.3 `GET|POST /authorize` — 授权端点

请求参数（query，POST 时可用表单）：

| 参数 | 必填 | 说明 |
|---|---|---|
| `response_type` | ✅ | 只接受 `code`，其它返回 `unsupported_response_type` |
| `client_id` | ✅ | 必须已注册 |
| `redirect_uri` | ✅ | **与注册值精确字符串匹配**（不做前缀/通配匹配） |
| `scope` | ✅ | 必须含 `openid`；否则按 OAuth 错误处理 |
| `state` | 强烈建议 | **原样回传**，用于 CSRF |
| `nonce` | 建议 | 写入 `id_token`，客户端校验 |
| `code_challenge` | 公开客户端必填 | PKCE |
| `code_challenge_method` | 公开客户端必填 | **只接受 `S256`**，`plain` 拒绝 |
| `prompt` | 可选 | `login`（强制重新认证）/ `none`（无会话时不返回登录页） |

行为：

```text
1. 校验 client_id / redirect_uri / response_type / scope / PKCE 参数
   - redirect_uri 或 client_id 非法 → 不重定向，直接渲染错误页（防开放重定向）
   - 其它参数非法 → 302 回 redirect_uri 带 error=...
2. 查会话 cookie（user_sessions）
   - 有效 → 跳过登录
   - 无效且 prompt=none → 302 回 redirect_uri?error=login_required
   - 无效 → 渲染登录页（HTML 表单 POST 回 /authorize）
3. 生成授权码（随机 32 字节，库里只存 SHA-256），TTL 60 秒，一次性
   - 绑定 client_id / user_id / redirect_uri / scope / nonce / code_challenge
4. 302 → redirect_uri?code=...&state=<原样>
```

登录页要求（一期最小实现）：

- 一个 HTML 表单：用户名 + 密码 + 隐藏字段（`state`、`nonce`、`code_challenge`… 全部透传）
- **表单带 CSRF token**，校验来源；不依赖第三方 JS
- 文案与配色跟随站点，但**一期不做精细品牌化**
- 复用现有登录限流与"不区分错误原因"的防枚举逻辑

### 2.4 `POST /token` — 令牌端点

`Content-Type: application/x-www-form-urlencoded`。客户端认证支持三种：`client_secret_basic`（HTTP Basic）、`client_secret_post`（表单字段）、`none`（公开客户端 + PKCE）。

#### (a) `grant_type=authorization_code`

```text
grant_type=authorization_code
code=<授权码>
redirect_uri=<与 authorize 时一致>
client_id=<公开客户端必填>
code_verifier=<PKCE 原文>
```

成功 `200`：

```json
{
  "access_token": "eyJ...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "rt_...",
  "id_token": "eyJ...",
  "scope": "openid profile email"
}
```

校验（**每条都必须实现，缺一条就是漏洞**）：
- 授权码存在、未使用、未过期（60 秒）→ 失败 `invalid_grant`
- 授权码归属的 `client_id` == 请求的 `client_id`
- `redirect_uri` 与授权时**逐字相同**
- PKCE：`BASE64URL(SHA256(code_verifier)) == code_challenge` → 失败 `invalid_grant`
- 授权码**一次性**：一旦开始兑换立即标记已用；**已用过的码再次兑换 → 撤销该码派生的整条 refresh 链**（与现有 refresh 重放检测同一套思路）
- 授权码是**保密客户端**还是公开客户端 → 决定是否要求 client 认证

#### (b) `grant_type=refresh_token`

```text
grant_type=refresh_token
refresh_token=rt_...
client_id=<必填>
```

成功 `200`：新的 `access_token` + **轮换后的 `refresh_token`** + 新的 `id_token`（若 scope 含 openid）。
沿用现有实现：**旧令牌立即失效、重放判定后整链撤销**（`auth.py:129` 的 `_revoke_family`）。

#### (c) `grant_type=urn:ietf:params:oauth:grant-type:device_code`

```text
grant_type=urn:ietf:params:oauth:grant-type:device_code
device_code=<设备码>
client_id=<必填>
```

- 用户尚未确认 → `400` + `{"error":"authorization_pending"}`
- 轮询过快 → `400` + `{"error":"slow_down"}`（此后 `interval` +5 秒）
- 用户拒绝 → `400` + `{"error":"access_denied"}`
- 设备码过期 → `400` + `{"error":"expired_token"}`
- 成功 → 与 (a) 相同的响应体

#### 错误响应统一

```json
{ "error": "invalid_grant", "error_description": "授权码无效或已过期" }
```

`400` + `Cache-Control: no-store`；**令牌端点响应一律不得被缓存**。
错误码范围：`invalid_request`、`invalid_client`、`invalid_grant`、`unauthorized_client`、`unsupported_grant_type`、`invalid_scope`、`authorization_pending`、`slow_down`、`access_denied`、`expired_token`。

### 2.5 `POST /device_authorization` — 设备流程入口（RFC 8628 §3.1–3.2）

```http
POST /auth/device_authorization
Content-Type: application/x-www-form-urlencoded

client_id=elicloud-cli&scope=openid profile offline_access
```

```json
{
  "device_code": "dc_9f1c...",                       // 高熵随机串，只给客户端
  "user_code": "WDJB-MJHT",                          // 人读短码，用户在浏览器输入
  "verification_uri": "https://146.56.237.33/auth/device",
  "verification_uri_complete": "https://146.56.237.33/auth/device?user_code=WDJB-MJHT",
  "expires_in": 600,
  "interval": 5
}
```

- `user_code` 用**无歧义字符集**（去掉 `0/O/1/I/L`），形如 `XXXX-XXXX`，8 位左右即可
- `expires_in` 默认 600 秒，`interval` 默认 5 秒
- 暴力猜测防护：短码尝试次数限制 + 短 TTL + 单 IP 限流

配套两个端点（**必须实现，否则设备流程走不通**）：

`GET /device` — 渲染"输入设备码"页面；带 `user_code` 参数时预填。
`POST /device` — 提交短码；已登录则要求确认，未登录则先走登录页；成功后页面提示"可以回到设备了"。

### 2.6 `GET /userinfo` — 按 scope 过滤 claims

```http
GET /auth/userinfo
Authorization: Bearer <access_token>
```

```json
{ "sub": "user_0001", "preferred_username": "alice", "email": "alice@example.com", "email_verified": true }
```

规则：`sub` 永远返回；`profile` scope 才有 `preferred_username`/`name`；`email` scope 才有 `email`。
**access token 的 `scope` 里没有的 claim 一律不返回**（现在是无条件返回，需收紧）。
接受 `GET` 与 `POST`（OIDC 要求两者都支持）。

### 2.7 `GET|POST /logout` — RP-Initiated Logout

```text
GET /auth/logout?post_logout_redirect_uri=<已注册的回跳>&id_token_hint=<可选>&state=<可选>
```

行为：清会话 cookie → 撤销该会话的 refresh 链 → 若 `post_logout_redirect_uri` 在该客户端注册值内则 302 回跳（带 `state`），否则渲染"已登出"页面。
**`post_logout_redirect_uri` 必须精确匹配注册值**，否则不跳转（防开放重定向）。

### 2.8 私有端点（保留，不参与 OIDC 发现）

`/v1/register`、`/v1/login`、`/v1/refresh`、`/v1/userinfo` 语义**保持**（`/v1/userinfo` 的 claim 按 §1.3 收紧），
继续给自家 App/前端用。**`/v1/logout` 已按 §1.3 的 A3 决策删除**（线上实测 404）。
**在 README 与发现文档之外的文档中明确标注为"非标准私有接口，不要给第三方用"**。

---

## 3. 三种流程时序

### 3.1 授权码 + PKCE（Web 前端、手机 App 系统浏览器）

```text
客户端                                     SSO                       用户
  │  生成 code_verifier，算出 S256 challenge
  │  ── 302 /auth/authorize?client_id&redirect_uri&scope=openid&state&nonce&code_challenge ──►
  │                                        │  无会话 → 渲染登录页 ──►  输入账号密码
  │                                        │  校验 → 建会话 cookie
  │                                        │  生成 code（存哈希，60s 一次性）
  │  ◄── 302 redirect_uri?code=...&state=... ──
  │  ── POST /auth/token (code + code_verifier + client_id) ──►
  │                                        │  校验 code/PKCE/redirect_uri 一致
  │  ◄── access_token + refresh_token + id_token ──
  │  校验 id_token：iss/aud/exp/nonce + 用 JWKS 验签
```

手机 App 用自定义 scheme 回跳（如 `elipese://callback`）或 App Link；`redirect_uri` 需在客户端注册时登记。**放行 `http://127.0.0.1:<随机端口>/...` 前必须确认该客户端可信** —— 这是 CLI 的标准做法，但也是开放重定向的经典触发点。

### 3.2 设备码（CLI / 桌面工具）

```text
CLI                                        SSO                        用户
  │  ── POST /auth/device_authorization ──►
  │  ◄── device_code, user_code, verification_uri, interval ──
  │  打印："请访问 https://…/auth/device 输入 WDJB-MJHT"
  │                                                            │  浏览器打开 verification_uri
  │                                                            │  登录（如未登录）→ 输入短码 → 确认
  │  ── POST /auth/token (device_code) ──►  (每 interval 轮询)
  │  ◄── authorization_pending / slow_down … 直到
  │  ◄── access_token + refresh_token (+ id_token) ──
```

### 3.3 刷新

```text
客户端 ── POST /auth/token (grant_type=refresh_token) ──►
       ◄── 新 access_token + 轮换后的 refresh_token
       （旧 refresh 再用 → 整链撤销，强制重新登录）
```

---

## 4. 令牌语义

### 4.1 `access_token`（API 访问用）

```json
{
  "iss": "https://146.56.237.33/auth",
  "sub": "user_0001",
  "aud": "elicloud-services",
  "client_id": "elipese-web",
  "scope": "openid profile email pdf:read",
  "username": "alice",
  "jti": "...",
  "iat": 1760000000,
  "exp": 1760003600
}
```

**关键**：access token 的 `aud` **保持 `elicloud-services`**（业务服务按它校验），不要改成 client_id —— 否则所有业务服务验签都要改。新增 `client_id` claim 用于审计。

### 4.2 `id_token`（身份证明，绝不用来访问 API）

```json
{
  "iss": "https://146.56.237.33/auth",
  "sub": "user_0001",
  "aud": "elipese-web",
  "exp": 1760000300,
  "iat": 1760000000,
  "auth_time": 1759999000,
  "nonce": "<原样回传 authorize 的 nonce>",
  "preferred_username": "alice",
  "email": "alice@example.com"
}
```

- **`aud` = client_id**（与 access token 不同，这是最容易搞混的一点）
- TTL 短（建议 300~600 秒），`aud` 已限定，过期后靠 refresh 重新获取
- `nonce` 仅在请求带 nonce 时出现；`auth_time` 是用户实际认证时间（非签发时间）
- 签名用现有 `keys.py`，`kid` 与 JWKS 一致
- 一期**不实现** `at_hash`/`c_hash`（非必需），若要实现需在文档中说明

### 4.3 `refresh_token`

沿用现有不透明串 + SHA-256 存储 + 轮换 + family 整链撤销，**新增绑定 `client_id` 与 `scope`**（防止 A 客户端的刷新令牌被 B 客户端使用、防止 scope 提权）。

---

## 5. 客户端注册模型

### 5.1 客户端类型与认证方式

| 类型 | `token_endpoint_auth_method` | PKCE | 适用 |
|---|---|---|---|
| `public` | `none` | **强制 S256** | 手机 App、CLI、桌面工具、SPA |
| `confidential` | `client_secret_basic` / `client_secret_post` | 建议（不强制） | 有后端能保密的前端、服务端集成 |

`client_secret` 只在校验时用哈希比对，**明文只在创建那一次返回**（现有 `oauth_clients.client_secret_hash` 已是这个思路）。

### 5.2 管理接口（管理员令牌保护）

```http
POST   /v1/clients           # 创建客户端 → 返回 client_id + client_secret（仅此一次）
GET    /v1/clients           # 列出（不回显 secret）
GET    /v1/clients/{id}      # 详情
PATCH  /v1/clients/{id}      # 改 redirect_uris / scopes / 名称
DELETE /v1/clients/{id}      # 删除（吊销）
POST   /v1/clients/{id}/rotate-secret
```

保护方式：复用 `app/cli.py` 的管理员机制（一期用 `ADMIN_TOKEN` 环境变量 + `Authorization: Bearer`，别新增用户角色系统）。
请求/响应示例与错误码遵循 `architecture.md` §5 的既有风格。

### 5.3 CLI 子命令

```bash
docker compose exec sso python -m app.cli create-client \
  --client-id elipese-web --name "Elipese Web" --type public \
  --redirect-uri https://elipese.cloud/callback \
  --redirect-uri http://127.0.0.1:5173/callback \
  --scope "openid profile email pdf:read"

docker compose exec sso python -m app.cli list-clients
docker compose exec sso python -m app.cli rotate-client-secret --client-id elipese-web
```

### 5.4 一期预置客户端（建议）

| client_id | 类型 | redirect_uris | scopes |
|---|---|---|---|
| `elipese-web` | public | `https://elipese.cloud/callback`、`http://localhost:5173/callback` | `openid profile email` |
| `elipese-app` | public | `elipese://callback` | `openid profile email offline_access` |
| `elicloud-cli` | public | 无（走设备码） | `openid profile offline_access` |

---

## 6. 数据模型

### 6.1 表结构（在现有三张表基础上新增/扩展）

```sql
-- 扩展现有表（原本"只建表不使用"）
CREATE TABLE IF NOT EXISTS oauth_clients (
  client_id              TEXT PRIMARY KEY,
  client_secret_hash     TEXT,                 -- public 客户端为 NULL
  client_type            TEXT NOT NULL,        -- public / confidential
  name                   TEXT NOT NULL,
  redirect_uris          TEXT NOT NULL,        -- JSON 数组，精确匹配
  post_logout_redirect_uris TEXT,              -- JSON 数组
  allowed_scopes         TEXT NOT NULL,        -- JSON 数组
  allowed_grant_types    TEXT NOT NULL,        -- JSON 数组
  token_endpoint_auth_method TEXT NOT NULL,    -- none / client_secret_basic / client_secret_post
  created_at             TEXT NOT NULL,
  updated_at             TEXT NOT NULL
);

-- 新增：授权码（只存哈希，一次性，60 秒）
CREATE TABLE IF NOT EXISTS authorization_codes (
  code_hash             TEXT PRIMARY KEY,      -- sha256(code)
  client_id             TEXT NOT NULL REFERENCES oauth_clients(client_id),
  user_id               TEXT NOT NULL REFERENCES users(id),
  redirect_uri          TEXT NOT NULL,
  scope                 TEXT NOT NULL,
  nonce                 TEXT,
  code_challenge        TEXT,
  code_challenge_method TEXT,                  -- 只存 S256
  auth_time             TEXT NOT NULL,
  session_id            TEXT,                  -- 关联登录会话
  expires_at            TEXT NOT NULL,
  used_at               TEXT,
  created_at            TEXT NOT NULL,
  ip                    TEXT,
  user_agent            TEXT
);

-- 新增：设备码（RFC 8628）
CREATE TABLE IF NOT EXISTS device_codes (
  device_code_hash TEXT PRIMARY KEY,
  user_code        TEXT NOT NULL UNIQUE,       -- 用于索引查找
  client_id        TEXT NOT NULL REFERENCES oauth_clients(client_id),
  scope            TEXT NOT NULL,
  status           TEXT NOT NULL DEFAULT 'pending',  -- pending/approved/denied/expired
  user_id          TEXT REFERENCES users(id),  -- 用户确认后写入
  interval_seconds INTEGER NOT NULL DEFAULT 5,
  last_polled_at   TEXT,                       -- 实现 slow_down
  expires_at       TEXT NOT NULL,
  created_at       TEXT NOT NULL,
  ip               TEXT
);

-- 新增：登录会话（让 /authorize 不必每次都重登）
CREATE TABLE IF NOT EXISTS user_sessions (
  id            TEXT PRIMARY KEY,
  user_id       TEXT NOT NULL REFERENCES users(id),
  session_hash  TEXT NOT NULL UNIQUE,          -- sha256(cookie 值)
  auth_time     TEXT NOT NULL,
  expires_at    TEXT NOT NULL,
  revoked_at    TEXT,
  created_at    TEXT NOT NULL,
  ip            TEXT,
  user_agent    TEXT
);

-- 新增：同意记录（一期自动写入，二期做交互式确认）
CREATE TABLE IF NOT EXISTS consents (
  user_id    TEXT NOT NULL REFERENCES users(id),
  client_id  TEXT NOT NULL REFERENCES oauth_clients(client_id),
  scope      TEXT NOT NULL,
  created_at TEXT NOT NULL,
  PRIMARY KEY (user_id, client_id)
);

-- 扩展现有 refresh_tokens：加 client_id（绑定客户端）与 scope（防提权）
ALTER TABLE refresh_tokens ADD COLUMN client_id TEXT;
ALTER TABLE refresh_tokens ADD COLUMN scope TEXT;
```

> 沿用现有约定：全 TEXT + ISO8601 字符串，SQLite 起步、换 PostgreSQL 不改表；启动时 `CREATE TABLE IF NOT EXISTS`。
> `ALTER TABLE` 需在迁移里做成**幂等**（先查 `PRAGMA table_info` 再决定是否加列），避免重启报错。

### 6.2 会话与 Cookie 要求

- Cookie 值用 `secrets.token_urlsafe(32)`，库里只存 SHA-256（**与 refresh token 同等对待**）
- 属性：`HttpOnly`、`SameSite=Lax`、`Path=/auth`、`Secure`。**IP 阶段也照开 `Secure`** —— 线上是 Let's Encrypt 的 IP 证书（见 §12），不是自签，HTTPS 由 `PUBLIC_BASE_URL` 的 scheme 决定（`Settings.cookie_secure`）
- 登录页表单带 **CSRF token**（双重提交或与会话绑定的随机值）
- 会话 TTL 建议 12 小时，登出即撤销

---

## 7. 安全清单（逐条自查）

| # | 项 | 要求 |
|---|---|---|
| 1 | 开放重定向 | `redirect_uri` **精确匹配**注册值；非法则**不重定向**、直接渲染错误页 |
| 2 | PKCE | 公开客户端强制 `S256`；拒绝 `plain`；`code_verifier` 长度 43–128 |
| 3 | 授权码 | 只存哈希、一次性、60 秒 TTL、绑定 `client_id`+`redirect_uri`+`code_challenge` |
| 4 | 授权码重放 | 已用过的码再次兑换 → 撤销其派生的整条 refresh 链 |
| 5 | refresh 绑定 | 校验 `client_id` 与 `scope`，防止跨客户端使用与 scope 提权 |
| 6 | id_token | 校验 `aud=client_id`、`nonce` 原样回传、TTL ≤10 分钟、**不可用于访问 API** |
| 7 | access token | `aud` 保持 `elicloud-services`；业务服务仍只认 RS256 |
| 8 | client_secret | 只存哈希；明文仅在创建时返回一次；比较用常量时间 |
| 9 | 设备短码 | 无歧义字符集；尝试次数限制 + 短 TTL + 单 IP 限流 |
| 10 | 缓存 | `/token`、`/device_authorization` 响应 `Cache-Control: no-store`；JWKS 保持 `max-age=300` |
| 11 | CSRF | 登录页与 `/device` 确认表单带 token；Cookie `SameSite=Lax` |
| 12 | 限流 | `/authorize`、`/token`、`/device_authorization`、`/device` 都要限流（沿用现有进程内限流器，多副本时换 Redis） |
| 13 | 日志 | 不记录密码、令牌原文、授权码、`client_secret`；只记 `client_id`/`user_id`/结果/耗时 |
| 14 | 枚举 | 登录失败响应逐字相同（沿用现有实现）；`user_code` 无效与过期返回相同提示 |
| 15 | 会话固定 | 登录成功后**重新生成**会话 ID，不复用登出前的 cookie |
| 16 | 会话 cookie | `HttpOnly` + `SameSite=Lax` + `Secure`（https 阶段）+ `Path=/auth` |

---

## 8. 网关配置（Caddy）

现在是**路径重写**（`/auth/<x>` → `/v1/<x>`），不是纯剥离。新增标准端点后要保证 **`/auth/token` 不被重写成 `/v1/token`** —— 也就是把 §2.0 的三类路径分派到正确的块里。

> ⚠️ **本节原稿的 matcher 有误，已修正。** 站点块地址是 `.../auth/*`，但**地址里的路径只用于选中站点块，块内请求路径不会被剥离**（已实测确认）。因此块内匹配器必须写完整路径 `/auth/token`：原稿的 `handle /.well-known/*` 与 `handle /token /authorize …` **永远不会匹配**，请求会掉进兜底块、被改写成 `/v1/token` → 404 —— 正好违反 §2.2 硬规则 1。

```caddyfile
https://146.56.237.33/auth/* {
    tls { issuer acme { profile shortlived } }
    log

    # ① 发现文档：落在服务根，只剥 /auth
    handle /auth/.well-known/* {
        uri strip_prefix /auth
        reverse_proxy sso:8000
    }

    # ② 标准 OIDC 端点：只剥 /auth，服务内就是 /token、/authorize…
    #    ⚠️ 多路径**必须用具名 matcher**：`handle a b { }` 这种简写只接受一个 matcher，
    #       Caddy 会报 `wrong argument count ... after '/auth/token'` 并拒绝启动（已实测）。
    #    这份列表必须与 app/constants.py 的 STANDARD_ENDPOINT_PATHS 保持一致。
    @standard_oidc path /auth/authorize /auth/token /auth/device_authorization /auth/device /auth/logout
    handle @standard_oidc {
        uri strip_prefix /auth
        reverse_proxy sso:8000
    }

    # ③ 兼容写法：/auth/v1/<x> → /v1/<x>（客户端管理接口的外部形态）
    handle /auth/v1/* {
        uri strip_prefix /auth
        reverse_proxy sso:8000
    }

    # ④ 私有端点：/auth/<x> → /v1/<x>
    handle_path /auth/* {
        rewrite * /v1{uri}
        reverse_proxy sso:8000
    }
}
```

排序依据：Caddy 对 `handle` 类路由按 **path matcher 的具体程度**排序，无 matcher 的兜底块排最后 —— 所以 ② 的精确路径一定先于 ④ 的 `/auth/*` 命中，**`/auth/logout` 走标准语义**（与 §1.3 废弃私有 logout 的决策一致）。

> **实现期实测结论**（`sso/scripts/e2e_endpoints.py` 会逐条线上验证）：
>
> ```text
> /auth/authorize            -> 400（缺参数时渲染错误页，不是 404）✓
> /auth/token                -> 405（GET 不允许；POST 才是入口）✓
> /auth/device_authorization -> 405 ✓
> /auth/device               -> 200 ✓
> /auth/logout               -> 200 ✓
> /auth/v1/clients           -> 401（走到私有 /v1/clients，受 ADMIN_TOKEN 保护）✓
> /auth/login                -> 400/401（走到私有 /v1/login）✓
> /auth/v1/logout            -> 404（A3 已删除）✓
> ```
>
> 其中「标准端点不是 404」这一组特别关键：一旦 ② 的 matcher 写错，它们会全部变成 404，
> 而这正是本次扩展要修的原始问题。

改完**先用一次性容器校验、再 reload**（校验失败会让网关进崩溃循环，本项目已实际踩过一次）：

```bash
docker run --rm --entrypoint caddy -v /home/docker-admin/dsh-nas/caddy:/etc/caddy:ro \
  caddy:2-alpine validate --config /etc/caddy/Caddyfile    # 必须看到 Valid configuration
docker exec dsh-caddy caddy reload --config /etc/caddy/Caddyfile
```

**验收**：`caddy validate` 通过；`/auth/.well-known/openid-configuration` 返回 200；`/auth/token` 不是 404。

> **实测补充（实现期）**：
>
> 1. **标准端点要一个一个加，不要提前声明**。② 的列表里每个端点落地时再加 ——
>    提前加会让标准客户端拿到 404，违反 §2.2 硬规则 1。目前列表里只有 `/auth/authorize`。
> 2. Caddy 2.11 对"站点地址里带路径"（`https://IP/auth/*`）会打一条 **deprecation 警告**：
>    `Using a path in a site address is deprecated; please use the 'handle' directive instead`。
>    功能仍正常（已实测），但将来要迁移。之所以暂时保留这种写法：把 SSO 路由并进面板站点块后，
>    面板块里的 `basic_auth`（无 matcher 的兜底指令）会先于 `handle` 执行，把 `/auth/*` 也拦下。
>    迁移时需要用 `route` 块强制顺序，或把面板与 SSO 拆到不同 host。

---

## 9. 配置新增

```bash
# ---- OIDC 新增 ----
AUTHORIZATION_CODE_TTL=60          # 授权码有效期（秒）
ID_TOKEN_TTL=600                   # id_token 有效期（秒）
SESSION_TTL=43200                  # 登录会话 12 小时
SESSION_COOKIE_NAME=elicloud_sso_session
DEVICE_CODE_TTL=600
DEVICE_POLL_INTERVAL=5
ADMIN_TOKEN=<随机长串>              # 客户端管理接口鉴权，仅从环境变量读
DEFAULT_SCOPE=openid profile email pdf:read pdf:write mc:whitelist files:read files:write userinfo:read userinfo:write  # 已追加 mc:whitelist / files:* / userinfo:*
```

> **`mc:whitelist` 是第二个业务服务（[`mc.md`](mc.md)）要求的 scope**，加它要动四处、
> 且**线上已有的 access token 里没有它** —— 用户需重新登录后才能调用 MC 白名单服务，
> 走授权码流程的客户端还要显式请求该 scope。完整清单与实施顺序见 [`mc.md`](mc.md) §12.4。
>
> **`files:read` / `files:write` 是第三个业务服务（`file/`，通用文件服务）要求的 scope**，
> 同样的「四处真源」规则：`constants.SUPPORTED_SCOPES`、`config.default_scope`、
> `device.SCOPE_LABELS`、以及**线上真正生效**的 `deploy.sh` 内嵌 `DEFAULT_SCOPE`
> （外加 `docker-compose.yml` 与 `tests/conftest.py`）。旧令牌同样需要重新登录。
>
> **`userinfo:read` / `userinfo:write` 是第四个业务服务（`userinfo/`，个人资料服务）要求的 scope**，
> 规则同上；头像 / banner 的读写还分别需要 `files:read` / `files:write`。

`config.py` 里保持现有风格：全部 `Settings` 字段 + 派生属性（`authorization_endpoint`、`token_endpoint`、`device_authorization_endpoint`、`end_session_endpoint` 都用 `external_url(...)`）。

---

## 10. 实现顺序（每步都可独立验证）

```text
第 1 步  修发现文档（删死链、token_endpoint → /token、补齐字段）
         → 验收：curl 发现文档，所有 endpoint 都能返回非 404（哪怕先返回 501）
第 2 步  启用 oauth_clients：表扩展 + 管理接口 + CLI create-client
         → 验收：能创建一个 public 客户端并拿到 client_id
第 3 步  /authorize + 登录页 + 会话 cookie + authorization_codes
         → 验收：浏览器手测能拿到 code 并回跳
第 4 步  /token 的 authorization_code 分支（含 PKCE、一次性、重放撤销）
         → 验收：用 code_verifier 换到三个令牌；错 challenge 必须失败
第 5 步  id_token 签发 + /userinfo 按 scope 过滤
         → 验收：用 JWKS 验签 id_token，核对 aud/nonce/auth_time
第 6 步  /token 的 refresh_token 分支（绑定 client_id/scope）
         → 验收：轮换 + 重放整链撤销仍成立
第 7 步  设备流程：/device_authorization + /device 页面 + device_code 分支
         → 验收：CLI 脚本能完成完整设备登录
第 8 步  /logout（标准）+ 会话撤销 + post_logout 回跳
         → 验收：登出后会话 cookie 失效、refresh 链被撤销
第 9 步  网关块扩展 + 线上验收
         → 验收：README §11.2 的冒烟清单扩到覆盖新端点
```

### 10.1 ✅ 实现完成情况（2026-09-25 全部落地并线上验收）

九步**全部完成**，线上入口 `https://146.56.237.33/auth`。可复现的验收证据：

| 证据 | 数量 | 位置 |
|---|---|---|
| 单元 / 契约测试 | **172 passed**（一条命令：`pytest -q`） | `sso/tests/` |
| 发现文档 ↔ 网关 ↔ 实现 三方一致性（打公网） | **23/23** | `sso/scripts/e2e_endpoints.py` |
| 授权码 + PKCE + 刷新 + id_token 全流程（打公网） | **29/29** | `sso/scripts/e2e_oidc.py` |
| 设备流程 RFC 8628（设备与浏览器两个角色） | **23/23** | `sso/scripts/e2e_device.py` |
| 原有私有端点冒烟（含 A3 后的注销检查） | **24/24** | `sso/scripts/e2e_smoke.py` |

实现期对本文档的**偏离与修正**都记在 §1.3 与本节的注释里，主要三类：
① 与线上事实不符（证书、matcher）；② 文档内部自相矛盾（logout 抢路径、userinfo 语义）；
③ 实现必需但文档未覆盖的扩展（`DEVICE_STATUS_USED`、`refresh_tokens.session_id`、
`device` 登录复用登录页）。运维与排错细节见 `sso/README.md`。

---

## 11. 测试清单（扩到现有 `tests/`）

现有 23 项契约测试 + 24 项冒烟验收必须**继续全绿**（私有端点语义不能回归）。新增：

**授权码流程**
- [ ] 正常流程拿到 access/refresh/id_token
- [ ] `redirect_uri` 不匹配 → 不重定向、不泄露信息
- [ ] `client_id` 不存在 → 错误页
- [ ] 缺 `code_challenge` 的公开客户端 → 拒绝
- [ ] `code_challenge_method=plain` → 拒绝
- [ ] `code_verifier` 错误 → `invalid_grant`
- [ ] 授权码重复兑换 → 第二次失败且**整链撤销**
- [ ] 授权码过期（>60 秒）→ `invalid_grant`
- [ ] `state` 原样回传
- [ ] `nonce` 出现在 id_token 中

**令牌与身份**
- [ ] id_token 的 `aud` == client_id；access token 的 `aud` == `elicloud-services`
- [ ] id_token 用 JWKS 验签通过、`kid` 正确
- [ ] 密钥轮换后**新旧 id_token 都能验签**（沿用现有轮换测试）
- [ ] `/userinfo` 在无 `email` scope 时**不返回** email

**客户端认证**
- [ ] confidential 客户端用 Basic 与 post 两种方式都能过
- [ ] 错误的 client_secret → `invalid_client`
- [ ] 公开客户端不要求 secret，但必须带 PKCE

**刷新**
- [ ] 轮换后旧 refresh 失效
- [ ] 旧 refresh 重放 → 整链撤销
- [ ] A 客户端的 refresh 不能给 B 客户端用
- [ ] scope 不能被刷新时提权

**设备流程**
- [ ] `authorization_pending` → 用户确认 → 成功
- [ ] 轮询过快 → `slow_down`
- [ ] 用户拒绝 → `access_denied`
- [ ] 设备码过期 → `expired_token`
- [ ] 未知 `user_code` 与过期 `user_code` 提示一致（防枚举）

**登出与会话**
- [ ] 标准 `/logout` 清会话 + 撤销 refresh + 按注册值回跳
- [ ] `post_logout_redirect_uri` 未注册 → 不回跳
- [ ] 登录成功后会话 ID 变化（防会话固定）
- [ ] Cookie 带 `HttpOnly` / `SameSite=Lax`

**发现的自我一致性（关键）**
- [ ] 遍历 `/auth/.well-known/openid-configuration` 里声明的**每一个 endpoint**，断言不是 404 —— 这条测试能防止再次出现"声明了却没实现"的问题

---

## 12. IP 阶段与域名阶段的注意事项

1. **issuer 不变**：仍是 `https://146.56.237.33/auth`。**不要**为了 OIDC 把 issuer 改成域名 —— 域名还没上线。
2. **证书：不是自签，是 Let's Encrypt 的 IP 证书**（`tls { issuer acme { profile shortlived } }`，SAN = `146.56.237.33`，TLS 1.3，6 天期自动续期，公共 CA 签发）。因此：
   - 浏览器授权码流程 **不需要**在测试机安装/信任任何 CA，`authorize` 页不会有证书告警；
   - 标准客户端库**不会**因自签而拒绝验签，**不要**用 `verify=False` / `--insecure` 绕过 TLS 校验；
   - 设备流程仍然更容易做自动化验收（无浏览器交互），但理由不再是"绕开证书"，而是"无需人工点登录页"。
3. **redirect_uri 登记**：IP 阶段前端回跳地址类似 `https://146.56.237.33/callback`；域名上线后要**同时登记或更新**为 `https://elipese.cloud/callback`。
4. **切换域名时**（沿用 `sso/README.md` §9，增加两项）：
   - 除了清空 `refresh_tokens`，还要**清空 `authorization_codes`、`user_sessions`、`device_codes`**（虽短命，但避免边界情况）
   - 更新各客户端的 `redirect_uris`
   - 业务服务的 `SSO_ISSUER` / `SSO_JWKS_URL` 改新地址，**代码不动**
5. **不要**在代码或配置里出现 `api.elipese.cloud` 的硬编码 —— 域名上线前它不可用，硬编码会让"现在的环境"直接坏掉。

---

## 13. 明确不做（一期范围外）

| 项 | 原因 |
|---|---|
| 动态客户端注册（RFC 7591） | 客户端数量可控，静态注册够用；管理接口已预留 |
| 交互式同意页 | 一期自动同意已注册 scope，`consents` 表已建，二期再做 UI |
| `id_token` 的 `at_hash` / `c_hash` | 非必需；若加需一并写进文档 |
| `implicit` / `hybrid` 流程 | OAuth 2.1 已移除，不要实现 |
| Token introspection（RFC 7662）/ Revocation（RFC 7009） | 业务服务用 JWKS 本地验签，不需要内省 |
| `PAR` / `JAR` / `DPoP` | 规模不需要 |
| 多副本共享限流与会话 | 当前单容器；需多副本时再换 Redis |
| 第三方登录（GitHub/微信等） | 与本次 OIDC Provider 是两件事，别混进来 |
