# EliCloud SSO API 实现规格（单文件交付）

> **给实现 agent：先把本节 0 读完再动代码。** 本节说明你的任务边界、必读章节、硬约束和验收标准；
> 第 1~11 节是设计与契约正文，第 12~13 节是现状盘点与背景。
>
> 📌 **续篇**：SSO 一期已落地；其后决定把它扩成**完整 OIDC Provider**（授权码 + PKCE、设备流程、`id_token`、客户端注册）。
> **OIDC 部分的实现细节以 `docs/sso-oidc.md` 为准**，本文件的 §3 与 §5.7 只保留一期形态并已标注取代关系。
> 本文件的 §0.4 硬约束对 OIDC 扩展**同样有效**。
>
> 🔌 **服务端部署框架**：服务以什么形态上线、动态网关（容器启停 = 服务上下线）、以后怎么新增一个服务，见 **§7**；
> 框架层的「触发器与部署器分离」见 [`deploy-framework.md`](deploy-framework.md)，两者的衔接点在 §7.10。
>
> 🧪 **本地联调**：`sso` / `email` / `file` / `userinfo` / `mc` 五个模块的一键拉起、
> 生产同款路径前缀、假 RCON 与端到端冒烟脚本，见
> [`EliCloudDesktop/local-dev/README.md`](https://github.com/EliCloudOrg/EliCloudDesktop/blob/main/local-dev/README.md)
> （独立仓库 `EliCloudOrg/devtools`，在桌面端里挂为 submodule；
> `EliCloudDesktop\local-dev\dev.ps1 up` + `... smoke`）。不需要 Docker，不碰生产。

---

## 0. 阅读指南与任务边界（必读）

### 0.1 你的任务

**实现 EliCloud 平台的 SSO 服务**（容器名 `sso`，对外路径前缀 `/auth/*`）：账号注册/登录、JWT 签发与刷新、`userinfo`、JWKS 发布。

> **一期已完成**（2026-09-25 落地并部署，见 §0.9）。当前进行的是**二期：把它扩成标准 OIDC Provider** ——
> 授权码 + PKCE(S256)、`refresh_token`、设备授权流程（RFC 8628）、`id_token`、客户端静态注册。
> **二期规格见 `docs/sso-oidc.md`**（含端点契约、三种流程时序、表结构、安全清单、实现顺序、测试清单）。

**你不需要实现**（属于其它任务，文档里出现只是为了让你理解上下文）：
- `main-api`（平台核心 API，`/core/*`）
- `pdf-decrypt`（PDF 解密服务，`/pdf-decrypt/*`）
- 前端 `main-web-front`、Caddy 网关容器本身

但你需要**按本文档约定与之对接**：路径前缀、issuer、JWKS 地址、JWT claim 结构、Caddy labels 必须逐字一致，否则后续服务接不进来。

### 0.2 必读章节（按此顺序）

| 顺序 | 章节 | 为什么必须读 |
|---|---|---|
| 1 | **§12 现有工程现状盘点** | 仓库里已有的三个项目**都跟本平台无关**，没有任何可复用的认证/数据库代码。不读这节会去翻 `lumi-backend`、`chatroom-server` 并浪费大量时间 |
| 2 | **`docs/sso-oidc.md`** | **二期 OIDC 的实现规格**：端点、流程、表结构、安全清单、实现顺序、测试清单 |
| 3 | **§2 域名与路径规划** | 路径里**不再有 `/api`**；网关会重写/剥离 `/auth` 前缀，你的服务内部只看到 `/v1/...` 与标准端点 |
| 4 | **§3.1 issuer 与 JWKS 地址** | 全文最容易踩的坑，见 §0.4 硬约束 1 |
| 5 | **§5 SSO API 契约** | 一期逐接口契约（私有端点保留不变），OIDC 端点见续篇 |
| 6 | **§6 SSO 服务模块设计** | 目录结构、技术选型、环境变量、建表 SQL、容器与 labels |
| 7 | **§4 认证与数据隔离模型** | 理解 `sub` 的作用与「SSO 不碰业务数据」的边界 |
| 8 | §10 待决策事项 + §0.9 已落地现状 | 确认哪些已定、哪些还需要问 |
| 9 | **§7 服务端部署框架** | 服务以什么形状上线、动态网关如何感知上下线、**新增一个服务的标准动作**（§7.8）。扩平台前必读 |
| 10 | 其余章节 | 选读，理解平台全貌 |

### 0.3 文档措辞含义

- **「已定」** = 讨论中已拍板，直接照做，不要改。
- **「建议 / 倾向」** = 推荐做法，可按实际调整，但要在 README 里写清你的选择与理由。
- **「待确认 / 待决策」** = **必须先问，不要猜**。集中在 §10。
- 标注 `✅` 的是已经用事实核对过的结论（例如 §10 第 6 项）。

### 0.4 硬约束（违反则整个平台对接失败）

1. **issuer 必须与运行时实际的对外访问地址逐字一致**：JWT 里签的 `iss`、`openid-configuration` 里的 `issuer`、`jwks_uri` 三处必须一致。由环境变量注入，**严禁硬编码**。
   > 原因：Caddy 用 `handle_path` 剥离 `/auth` 前缀，服务内部只看到 `/.well-known/...`；如果按「服务内部视角」写 issuer（漏掉或写错前缀），所有业务服务验签都会失败。
   > **⚠️ 当前域名暂不可用，issuer 必须是 `http://<IP>:<端口>/auth` 而不是域名 —— 见 §0.9，这是本任务最容易做错的地方。**
2. **JWKS 地址固定为 `<PUBLIC_BASE_URL>/.well-known/jwks.json`**（域名阶段即 `https://api.elipese.cloud/auth/.well-known/jwks.json`；当前 IP 阶段见 §0.9），且必须**匿名可访问、无需认证**。
3. **签名算法用 RS256**（非对称）。不要用 HS256 —— 业务服务只应拿到公钥，不能持有共享密钥。
4. **服务内部路由不带 `/auth` 前缀**：注册 `POST /v1/register`，而不是 `/auth/v1/register`。前缀由网关处理。
5. **私钥必须持久化**（挂卷），**绝不打进镜像层、绝不进 Git**。容器重建若重新生成密钥，所有已签发令牌立即全部失效。
6. **密码只存 bcrypt/argon2 哈希**，绝不明文、绝不 MD5/SHA1；日志与响应中**不得出现密码或令牌原文**。
7. **`sub` 是业务数据隔离的唯一依据**，一经签发永不变更。
8. **接口契约不可擅自改名**：路径、字段名、状态码按 §5 原样实现。要改先说明理由。

### 0.5 开工前必须确认的事项（先问，别猜）

见 **§10 待决策事项**，共 9 项。其中至少这 5 项会直接改变代码或配置，**必须先拿到答复**：

1. **当前阶段的服务器 IP、端口、scheme**（HTTP 或自签 HTTPS）—— 没有它就无法正确配置 `issuer`，见 §0.9
2. SSO 用 **FastAPI 自研** 还是 **Casdoor 品牌化部署**（决定是"写代码"还是"写部署配置"）
3. 数据库用 **SQLite** 还是 **PostgreSQL**
4. 是否**开放自助注册**（决定 `/v1/register` 是公开接口还是管理员接口）
5. **IP 阶段是否启用 Caddy**（不启用就直接映射端口，配置更简单）

其余 4 项（登录形式、Access Token 有效期、站点名统一、main-api 归属）可采用文档中的结论或倾向值先实现，后续再调。

### 0.6 建议的交付物

```text
sso/
├── app/                 # FastAPI 应用（结构见 §6.2）
├── tests/               # 至少覆盖：注册→登录→userinfo→refresh→logout 全链路
├── Dockerfile           # 见 §6.4 环境与 §6.3 选型
├── requirements.txt
└── README.md            # 必须有，见 §0.7
```

### 0.7 验收标准（Definition of Done）

**功能**
- [ ] `POST /v1/register` 能创建用户；重复用户名返回 `409`；密码按强度规则校验
- [ ] `POST /v1/login` 正确返回 `access_token` + `refresh_token` + `expires_in`；密码错误返回 `401` 且**不泄露用户是否存在**
- [ ] `GET /v1/userinfo` 带合法 Bearer 返回用户信息；无/错令牌返回 `401`
- [ ] `POST /v1/refresh` 能换新令牌，且**旧 refresh_token 立即失效**（轮换）
- [ ] 重放已使用过的 refresh_token 时，**该令牌链整体撤销**
- [ ] `POST /v1/logout` 撤销 refresh_token，返回 `204`
- [ ] `GET /.well-known/jwks.json` 匿名返回公钥，带 `kid`，支持密钥轮换时新旧并存
- [ ] `GET /.well-known/openid-configuration` 的 `issuer` / `jwks_uri` 与 §3.1 逐字一致

**契约与部署**
- [ ] 服务内路由**无 `/auth` 前缀**，仅 `/v1/...` 与 `/.well-known/...`
- [ ] `iss` claim 与 `PUBLIC_BASE_URL` 严格相等（**当前阶段为 `http://<IP>:<端口>/auth`，不是域名**，见 §0.9）；由环境变量注入，代码中零硬编码
- [ ] `issuer` 与 `jwks_uri` **都由 `PUBLIC_BASE_URL` 派生**，改这一个变量即可整体切换阶段
- [ ] 算法为 RS256；用 HS256 签的令牌必须被拒绝；`alg: none` 必须被拒绝
- [ ] 私钥与 SQLite 都在挂卷目录（如 `/data`）中持久化，重建容器后**令牌仍有效**
- [ ] compose labels 为 `caddy: api.elipese.cloud`（当前阶段可为 IP 或直接不启用 Caddy，见 §0.9）+ `caddy.handle_path: /auth/*` + `caddy.0_reverse_proxy: "{{upstreams 8000}}"`，端口 8000
- [ ] 健康检查可用（建议打 `/.well-known/jwks.json`）
- [ ] CORS 由 `CORS_ORIGINS` 环境变量驱动，当前阶段填 IP/`localhost` 来源也能正常调用（不是只写死域名）
- [ ] 登录接口有按 IP + 用户名的限流

**文档**
- [ ] `README.md` 写清：环境变量表、启动方式、**令牌语义与「注销后 access token 仍在 TTL 内有效」这一取舍**、表结构
- [ ] `README.md` 写清**「IP 阶段 → 域名阶段」的切换步骤**（改 `PUBLIC_BASE_URL` / `CORS_ORIGINS` → 重启 → 清空 refresh_tokens）
- [ ] 测试可一条命令跑通

### 0.8 术语与命名注意事项

- **站点名 vs 域名**：讨论中站点名用过「EliCloud」，实际域名是 **`elipese.cloud`**。**一切 URL、issuer、labels 以域名为准**；站点显示名待统一（§10 第 7 项），不影响 API 实现。
- **`sub` / `user_id`**：同一概念。SSO 侧叫 `sub`（JWT claim），业务服务侧叫 `user_id`（数据库列）。
- **前缀剥离**：外部 `/auth/login` → 服务内 `/v1/login`。看到「服务内」字样时，一律指剥离后的路径。

### 0.9 ⚠️ 当前部署阶段：域名暂不可用，必须用 IP（影响 issuer）

> ### ✅ 已落地现状（本轮实现，2026-09-25）——读本节请先看这里
>
> SSO 已按本文档部署完成，**当前阶段就是 HTTPS**，不再是待选方案：
> Caddy 用 Let's Encrypt 的 **IP 证书**（`profile shortlived`，6 天期自动续期）为
> `https://146.56.237.33` 提供 TLS，`/auth/*` 已按 §2.2 契约重写转发到 `sso:8000`。
>
> | 项 | **实际取值（以此为准）** |
> |---|---|
> | `PUBLIC_BASE_URL` | `https://146.56.237.33/auth` |
> | issuer（JWT 的 `iss`） | `https://146.56.237.33/auth` |
> | JWKS | `https://146.56.237.33/auth/.well-known/jwks.json` |
> | 业务服务 `SSO_ISSUER` | `https://146.56.237.33/auth` |
> | 业务服务 `SSO_JWKS_URL` | `https://146.56.237.33/auth/.well-known/jwks.json` |
>
> 结论与影响：
>
> - **§10 第 8 项（IP 阶段 HTTPS 方案）已被实现解决**：走的是"Caddy + 真实 IP 证书"，
>   既不是纯 HTTP 也不是自签，客户端无需任何额外信任配置。
> - 本节下文关于"先用 HTTP + IP"、"IP 阶段 Caddy 拿不到 Let's Encrypt 证书"的推断**已经过时**，
>   保留仅为决策过程存档。
> - ⚠️ 本机公网 IP 曾因服务器被攻击而更换：**被攻击前的那个旧 IP 已完全不可达**，
>   任何配置里都不要再用它；当前唯一有效的公网地址就是 `146.56.237.33`。
>   强烈建议把公网 IP 固定为**弹性 IP**，避免再次全量改地址。
> - 实现、部署与运维细节（含"再换 IP 时的完整清单"）见 `sso/README.md`。

**事实**：`elipese.cloud` / `api.elipese.cloud` **目前不可用**，**至少半个月后**才能用。**现在只能用 IP 访问。**

**因此，本任务当前阶段的真实运行参数是**：

| 项 | 域名阶段（半个月后） | **当前阶段（现在就要能跑）** |
|---|---|---|
| API 入口 | `https://api.elipese.cloud` | `http://<服务器IP>:<端口>` |
| **issuer** | `https://api.elipese.cloud/auth` | **`http://<服务器IP>:<端口>/auth`** |
| JWKS | `.../auth/.well-known/jwks.json` | `http://<服务器IP>:<端口>/auth/.well-known/jwks.json` |
| 前端来源（CORS） | `https://elipese.cloud` | `http://<服务器IP>:<前端端口>`（或开发期 `http://localhost:*`） |

**你必须这样实现（否则半个月后要重写）**：

1. **一切 URL 参数化，代码里零硬编码域名**。所有对外地址从环境变量读取，域名上线时**只改环境变量、重启容器**，代码一行不动：
   ```text
   PUBLIC_BASE_URL = http://<IP>:<端口>/auth      # 现在
   PUBLIC_BASE_URL = https://api.elipese.cloud/auth   # 半个月后
   ISSUER          = 与 PUBLIC_BASE_URL 相同（或由它派生）
   ```
   **issuer 与 JWKS URL 都必须由 `PUBLIC_BASE_URL` 派生**，不要写两份、不要写死。
2. **不要把 issuer 写成从请求 Host 推导**（不要信任 `X-Forwarded-Host`）。用显式环境变量，行为可预测、可测试。
3. **CORS 用 `CORS_ORIGINS` 环境变量配置**（逗号分隔），当前阶段填 IP 来源，上线后换成域名。不要只写死域名导致现在前端全部被拦。
4. **别为 IP 场景改协议设计**：接口路径、claim 结构、表结构、RS256 都与域名阶段完全相同，**变的只有 base URL 和 CORS 来源**。
5. **README 必须写清这套切换步骤**，并明确以下已知影响：
   - 切换 issuer 后，**旧 issuer 签发的 access/refresh token 全部失效**，用户需重新登录一次（半个月后的一次性代价，可接受）。**用户数据不受影响**。
   - 建议切换时**直接清空 `refresh_tokens` 表**，避免遗留无效令牌。
6. **注意 IP 场景下的 HTTPS**：Caddy 无法为裸 IP 自动申请 Let's Encrypt 证书，所以当前阶段有三种选择，**开工前确认用哪种**（已并入 §10 待决策）：
   - **A. 先用 HTTP + IP**（最简单，仅内网/自测，注意流量明文）
   - **B. Caddy 对 IP 用自签证书**（`tls internal`），客户端需信任自签 CA
   - **C. 暂不启用 Caddy**，直接映射服务端口到宿主机的非 80/443 端口（如 `8000`）
   > 无论选哪种，**`PUBLIC_BASE_URL` 的 scheme 必须与实际情况一致**（HTTP 阶段就别写 `https://`），否则客户端按 issuer 拼 JWKS 地址会连不上。

---

## 1. 平台定位

EliCloud = **统一身份认证中心（SSO）** + 若干**业务服务**。

- 用户注册/登录一次 EliCloud 账号，即可访问平台下所有服务。
- 每个服务的数据都归属到这个账号下（数据库里每条业务数据带 `user_id`）。
- 服务之间**不共享数据库**，只通过 **HTTPS + JWT（JWKS 验签）** 交互。
- 核心思想：**SSO 只负责「你是谁」，业务服务只负责「你能干什么」。**

目前规划中的服务：

| 服务 | 容器名 | 对外路径前缀 | 状态 |
|---|---|---|---|
| SSO / 账号中心 | `sso` | `/auth/*` | ✅ 已上线（本文档 §0.9 + `sso/README.md`） |
| 平台核心 API | `main-api` | `/core/*` | 规划 |
| PDF 解密 | `pdf-decrypt` | `/pdf-decrypt/*` | 规划（第一个业务服务） |
| MC 白名单 | `mc` | `/mc/*` | ✅ 代码已完成（`mc/`，92 个契约测试通过），待部署 |
| 文件服务 | `file` | `/file/*` | ✅ 代码已完成（`file/`，166 个测试通过），待部署 |
| 用户资料 | `userinfo` | `/userinfo/*` | ✅ 代码已完成（`userinfo/`，48 个测试通过），待部署；依赖 file |
| 主站前端 | `main-web-front` | 主域 `elipese.cloud` | 规划 |
| 未来服务 | `xxx` | `/xxx/*` | 按同一模式复制 |

> 上表中 **SSO 已实现并上线**（见第 12 节现状盘点）；`mc` 是**第二个**、也是第一个按 §8
> 接入 SSO 的业务服务（纯 API + Bearer 验签）——**代码已完成，尚未部署**（见 [`mc.md`](mc.md) §12.5）。
> `file` 是**第三个**业务服务：用户上传文件（id + 有效期 + `public`/`private`/`protected` 可见性），
> 内容存私有 COS 桶，元数据在本地 SQLite；同样**代码已完成，尚未部署**（见 `file/README-deploy.md`）。
> `userinfo` 是**第四个**业务服务：昵称/简介在 userinfo SQLite，头像/banner 由 file 管理
> （固定 `public` + 永久），对外 `/userinfo/v1/*`；**必须先部署 file**。

---

## 2. 域名与路径规划（已定）

### 2.1 域名划分

```text
https://elipese.cloud            → 前端 Web App（主站，也承担 www 跳转）
https://www.elipese.cloud        → 301 跳转到主域
https://api.elipese.cloud        → 统一 API 网关入口
    ├── /auth/*          → SSO（账号/令牌/JWKS）
    ├── /core/*          → main-api（平台核心：用户中心、服务列表）
    ├── /pdf-decrypt/*   → PDF 解密服务
    └── /xxx/*           → 未来服务
```

### 2.2 路径规范（已定，重要）

- **主机名已经表达了「这是 API」**，所以路径里**不再出现 `/api`**。
- **每个服务内部保留 `/v1` 版本段**：`api.` 说明「这是 API 入口」，`/v1` 说明「API 协议版本」，两者分工不同；将来不兼容升级可平滑切 `/v2`，老客户端不受影响。
- 网关用 `handle_path`（Caddy）**剥离服务前缀**，服务容器内**只看到 `/v1/...`**，服务代码与挂载路径完全解耦。以后想把 `pdf-decrypt` 拆到独立域名，服务代码一行不用改。

```text
外部请求                                                    服务容器内实际收到
─────────────────────────────────────────────────────────  ─────────────────────
POST   api.elipese.cloud/auth/register                  →  POST /v1/register
POST   api.elipese.cloud/auth/login                     →  POST /v1/login
POST   api.elipese.cloud/auth/refresh                   →  POST /v1/refresh
POST   api.elipese.cloud/auth/logout                    →  POST /v1/logout
GET    api.elipese.cloud/auth/userinfo                  →  GET  /v1/userinfo
GET    api.elipese.cloud/auth/.well-known/jwks.json     →  GET  /.well-known/jwks.json
GET    api.elipese.cloud/auth/.well-known/openid-configuration → GET /.well-known/openid-configuration

POST   api.elipese.cloud/pdf-decrypt/v1/jobs            →  POST /v1/jobs
GET    api.elipese.cloud/pdf-decrypt/v1/jobs            →  GET  /v1/jobs
GET    api.elipese.cloud/pdf-decrypt/v1/jobs/{id}       →  GET  /v1/jobs/{id}
DELETE api.elipese.cloud/pdf-decrypt/v1/jobs/{id}       →  DELETE /v1/jobs/{id}
```

> 注意：`.well-known/*` 是按标准要求挂在 **服务根**（服务内 `/.well-known/...`），所以网关剥离 `/auth` 前缀后不能再补任何段。
>
> ⚠️ 上表用的是**域名阶段**的地址形态。**当前阶段域名不可用，把 `api.elipese.cloud` 换成 `http://<服务器IP>:<端口>` 即可，路径与剥离规则完全不变**（见 §0.9）。

### 2.3 前端与 API 跨域

前端在 `elipese.cloud`，API 在 `api.elipese.cloud`，**浏览器调用属于跨域，需要 CORS**（手机 App 不受 CORS 限制）。CORS 在各服务的 FastAPI 里配置：

```python
import os

app.add_middleware(
    CORSMiddleware,
    # 必须从环境变量读，逗号分隔；当前阶段填 IP 来源，域名可用后改环境变量
    allow_origins=os.environ["CORS_ORIGINS"].split(","),
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

配置值（当前阶段 → 域名阶段）：

```text
CORS_ORIGINS = http://<服务器IP>:<前端端口>,http://localhost:3000     # 现在
CORS_ORIGINS = https://elipese.cloud,https://www.elipese.cloud        # 半个月后
```

因为采用 **JWT Bearer**（而非 Cookie 会话），跨域 + `allow_credentials` 不构成 Cookie 共享问题；**不要**设计成依赖子域 Cookie 的 SSO。

---

## 3. 认证协议选型

| 方案 | 说明 | 评价 |
|---|---|---|
| **A. 自研轻量 OIDC Provider（FastAPI + Authlib 或 PyJWT）** | 自己实现 `/authorize`(可选)、`/v1/login`、`/v1/refresh`、`/v1/userinfo`、`/.well-known/jwks.json`，签发 JWT | ⭐⭐⭐⭐ 服务不多、想完全掌控、可平滑演进 |
| B. Casdoor 品牌化部署 | Go 编写，Docker 一条命令，自带登录/注册 UI，支持 OIDC/OAuth2/SAML | ⭐⭐⭐⭐ 落地最快，但引入外部组件与 UI 依赖 |
| C. Authentik / Keycloak | 功能强但偏重（Java/多容器） | ⭐⭐⭐ 未来企业级再考虑 |
| D. 简单 JWT（无标准协议、共享密钥验签） | 最简单 | ⭐⭐ **不推荐**：扩展性差，业务服务要持有签发密钥 |

**业务服务的接入方式与选型无关**：都通过 `/.well-known/jwks.json` 拉公钥验签 Access Token。因此本任务只需保证**令牌格式、JWKS、issuer 三者稳定**，SSO 内部实现可替换。

**倾向（讨论中的建议）**：先用 **FastAPI 自研轻量版**起步（依赖少、可控、与其它服务的 FastAPI 技术栈一致）；如果只想最快跑起来则用 Casdoor。**这个决策需要在开工前确认**（见第 10 节）。

> ### ✅ 决策已定（后续更新）：选 A，并继续扩展为完整 OIDC Provider
>
> 一期已按方案 A 落地（`sso/`，见 §0.9「已落地现状」）。随后确认客户端不止自家前端/App，还包括
> **App 用系统浏览器登录**与**第三方/CLI/桌面工具**——这两类密码直连覆盖不了，因此决定**在现有服务上补齐标准 OIDC**：
> 授权码 + PKCE(S256)、`refresh_token`、**设备授权流程（RFC 8628）**、`id_token`、客户端静态注册。
> 不换 Casdoor/Keycloak。**本章与 §5.7 的端点形态以新规格为准 → 详见 `docs/sso-oidc.md`。**
> 现有私有端点（`/v1/login` 等）**保留不变**，继续给自家客户端用。

### 3.1 issuer 与 JWKS 地址（必须严格一致）

**域名阶段（半个月后）与当前 IP 阶段（现在）的取值如下，形态完全相同，只是 base URL 不同**：

```text
# 当前阶段（域名不可用，只能走 IP —— 具体 IP/端口开工时确认）
PUBLIC_BASE_URL = http://<服务器IP>:<端口>/auth
issuer        = http://<服务器IP>:<端口>/auth
JWKS URL      = http://<服务器IP>:<端口>/auth/.well-known/jwks.json
register      = http://<服务器IP>:<端口>/auth/register
login         = http://<服务器IP>:<端口>/auth/login
refresh       = http://<服务器IP>:<端口>/auth/refresh
userinfo      = http://<服务器IP>:<端口>/auth/userinfo

# 域名阶段（切换时只改环境变量，代码不动）
PUBLIC_BASE_URL = https://api.elipese.cloud/auth
issuer        = https://api.elipese.cloud/auth
JWKS URL      = https://api.elipese.cloud/auth/.well-known/jwks.json
```

**实现要求**：`issuer` 与 `jwks_uri` **都从 `PUBLIC_BASE_URL` 派生**（`PUBLIC_BASE_URL.rstrip("/")` + 路径），不要写两份常量、不要硬编码。

> 因为网关剥离前缀，服务内部认为自己挂在根路径，但 **JWT 里签的 `iss` 必须是上面这个带 `/auth` 的完整外部 URL**（否则业务服务按配置的 `SSO_ISSUER` 验签会失败）。`ISSUER` 一律由环境变量注入，**绝不硬编码**。
>
> ⚠️ **切换阶段时**：issuer 变化会让旧令牌全部失效（用户重新登录一次），业务服务端只改自己的 `SSO_ISSUER` 环境变量即可，**代码无需改动**。详见 §0.9。

---

## 4. 认证与数据隔离模型（架构核心）

```text
1. App/前端打开 → 未登录 → 跳 EliCloud SSO 登录（或 App 内嵌原生登录表单）
2. 用户提交 EliCloud 账号密码
3. SSO 校验通过 → 签发 Access Token(JWT) + Refresh Token
   Access Token payload：
   {
     "iss": "http://<IP>:<端口>/auth",     ← 当前阶段；域名可用后为 https://api.elipese.cloud/auth
     "sub": "user_0001",                  ← 全局唯一用户 ID，业务数据隔离的唯一依据
     "username": "elicloud_user",
     "scope": "pdf:read pdf:write",
     "aud": "elicloud-services",          ← 或按服务填 client_id
     "iat": 1760000000,
     "exp": 1760003600
   }
4. 客户端调用业务服务：Authorization: Bearer <access_token>
5. 业务服务从 SSO 的 JWKS 拉公钥验签（本地缓存，尊重 Cache-Control）
   → 通过 → 取出 sub
6. 业务服务所有查询都带 WHERE user_id = <sub>
   → 用户只能看到自己的任务与结果
```

### 4.1 数据隔离规则

```text
SSO 数据库                               各业务服务数据库（按服务独立）
──────────────────────────              ──────────────────────────────
users                                    jobs / dicts / ...
  id (PK, 对外即 sub)                       id (PK)
  username (unique)                        user_id  ← 就是 SSO 的 users.id
  email (可选)                             ...
  password_hash (bcrypt/argon2)
  status (active/disabled)
  created_at / updated_at
                                        ★ 业务库不存密码，不存用户档案，
oauth_clients (未来 OAuth2 客户端时)       只存 user_id 作为外键式引用
  client_id / client_secret_hash
  name / redirect_uris / scopes
  created_at
                                        ★ 业务服务之间不共享数据库
refresh_tokens
  id / user_id / token_hash (唯一)
  expires_at / revoked_at
  user_agent / ip (可选)
```

**关键约定**：`sub` 一经签发永不变更；用户名/邮箱可变但 `sub` 不变；业务服务**不查 SSO 数据库**，需要用户展示名时调 `/v1/userinfo` 或只依赖 JWT 里的 `username`。

---

## 5. SSO API 契约（本任务的实现目标）

所有接口 `Content-Type: application/json`（除注明外）。错误统一使用如下结构：

```json
{ "error": "invalid_grant", "error_description": "用户名或密码错误" }
```

HTTP 状态码：`400` 参数错误、`401` 认证失败/令牌无效、`403` 权限不足、`404` 资源不存在、`409` 冲突（用户名已存在）、`429` 限流。

> **本节示例中的 host 一律写作域名阶段的形式（`https://api.elipese.cloud/auth/...`）。**
> **当前域名不可用，实际请求的是 `http://<服务器IP>:<端口>/auth/...`；接口路径、请求体、响应体、状态码完全不变，只有 host 部分不同（见 §0.9）。**

### 5.1 `POST /v1/register` — 注册

```http
POST https://api.elipese.cloud/auth/register
Content-Type: application/json

{ "username": "alice", "password": "S3cret!pass", "email": "alice@example.com" }
```

```json
// 201
{ "user": { "id": "user_0001", "username": "alice", "email": "alice@example.com", "created_at": "2026-01-01T00:00:00Z" } }
```

规则：`username` 唯一（冲突返回 409）、长度/字符集校验、密码最低强度校验、`email` 可选。
> 是否开放自助注册、还是「管理员创建账号」需确认（见第 10 节）。若暂不开放，仍应实现该接口但可用配置开关关闭。

### 5.2 `POST /v1/login` — 登录（签发令牌）

```http
POST https://api.elipese.cloud/auth/login
Content-Type: application/json

{ "username": "alice", "password": "S3cret!pass" }
```

```json
// 200
{
  "access_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "rt_7f3c...",
  "scope": "pdf:read pdf:write",
  "user": { "id": "user_0001", "username": "alice" }
}
```

规则：失败信息**不区分**「用户不存在」与「密码错误」（防枚举）；失败计数限流；`refresh_token` 是**不透明随机串**（存哈希），不是 JWT。

### 5.3 `POST /v1/refresh` — 刷新令牌

```http
POST https://api.elipese.cloud/auth/refresh
Content-Type: application/json

{ "refresh_token": "rt_7f3c..." }
```

```json
// 200：返回新的 access_token，并轮换 refresh_token（refresh token rotation）
{ "access_token": "...", "token_type": "Bearer", "expires_in": 3600, "refresh_token": "rt_9a1d..." }
```

规则：旧 `refresh_token` 一经使用即失效（[refresh token rotation](https://auth0.com/blog/refresh-token-security-detecting-hijacking-and-misuse-with-auth0/)）；若已轮换过的旧令牌被再次使用（说明令牌被窃取／重放），撤销**该令牌所属整条链**的全部 refresh token，强制重新登录。实现上给 `refresh_tokens` 加 `family_id`（令牌链标识）与 `used_at` 即可，比只按 `user_id` 全撤更精确、不误伤其它设备。

### 5.4 `GET /v1/userinfo` — 当前用户信息

```http
GET https://api.elipese.cloud/auth/userinfo
Authorization: Bearer <access_token>
```

```json
{ "sub": "user_0001", "username": "alice", "email": "alice@example.com", "scope": "pdf:read pdf:write" }
```

### 5.5 `POST /v1/logout` — 注销

```http
POST https://api.elipese.cloud/auth/logout
Authorization: Bearer <access_token>
Content-Type: application/json

{ "refresh_token": "rt_9a1d..." }   // 可选；撤销该 refresh token
```

```json
// 204 No Content
```

> Access Token 是无状态 JWT，注销后在 `exp` 之前仍然「有效」；需要即时失效时必须引入吊销名单或把 access token 有效期设短（推荐 15~60 分钟）。**这个取舍要在实现时明确写进 README。**

### 5.6 `GET /.well-known/jwks.json` — 公钥集（业务服务验签用）

```http
GET https://api.elipese.cloud/auth/.well-known/jwks.json
```

```json
{
  "keys": [
    { "kty": "RSA", "use": "sig", "alg": "RS256", "kid": "2026-01", "n": "...", "e": "AQAB" }
  ]
}
```

要求：
- 支持 `kid`，**密钥轮换时新旧公钥并存**（否则轮换瞬间所有业务服务验签失败）。
- 建议 `Cache-Control: public, max-age=300`，业务服务按此缓存。
- 必须可**匿名访问**，且不经过任何认证。

### 5.7 `GET /.well-known/openid-configuration` — OIDC 发现（可选但推荐）

> ⚠️ **本节已被 `docs/sso-oidc.md` §2.2 取代**（现已决定实现完整 OIDC Provider）。下面保留的是**一期形态**，其
> `token_endpoint` 指向 `/refresh`、且声明了当时并不存在的 `/authorize` —— 这正是新规格第 1 步要修的问题。
> **实现时以 `docs/sso-oidc.md` §2.2 的发现文档为准。**

```json
{
  "issuer": "https://api.elipese.cloud/auth",
  "jwks_uri": "https://api.elipese.cloud/auth/.well-known/jwks.json",
  "authorization_endpoint": "https://api.elipese.cloud/auth/authorize",
  "token_endpoint": "https://api.elipese.cloud/auth/refresh",
  "userinfo_endpoint": "https://api.elipese.cloud/auth/userinfo",
  "response_types_supported": ["code"],
  "subject_types_supported": ["public"],
  "id_token_signing_alg_values_supported": ["RS256"],
  "scopes_supported": ["openid", "profile", "email"]
}
```

> 上面写的是**域名阶段**的取值；**IP 阶段由 `PUBLIC_BASE_URL` 自动派生**（实际为 `https://146.56.237.33/auth`，见 §0.9 已落地现状），不要手写域名。

### 5.8 平台级接口（归属 `main-api`，不是 SSO）

讨论中确定由 `main-api` 提供「当前哪些服务在线」，供前端/App 隐藏或置灰不可用入口：

```http
GET https://api.elipese.cloud/core/v1/services
```

```json
{
  "services": [
    { "id": "pdf-decrypt", "name": "PDF 解密", "path": "/pdf-decrypt", "online": true },
    { "id": "xxx", "name": "未来服务", "path": "/xxx", "online": false }
  ]
}
```

在线判定来源（推荐 A）：**A. 查询 Caddy Admin API `GET /config/`**（路由存在 = 真的可访问，最真实）；B. 只读 Docker Socket 查带约定 label 的运行中容器（容器在跑 ≠ 健康）。

---

## 6. SSO 服务模块设计（实现者读这一节）

### 6.1 模块边界

**SSO 负责**：账号存储、密码校验、注册、令牌签发/刷新/撤销、JWKS 发布、OIDC 发现、用户信息读取、登录限流与审计日志。

**SSO 不负责**：任何业务数据（PDF 任务、聊天记录等）、业务权限决策（只提供 `scope`）、前端页面（本期登录表单由 App/前端自己画，或后续再补一个最小登录页）。

### 6.2 目录结构建议

```text
sso/
├── Dockerfile
├── requirements.txt
├── app/
│   ├── main.py            # FastAPI 应用、CORS、异常处理、路由注册
│   ├── config.py          # 全部配置从环境变量读（pydantic-settings）
│   ├── db.py              # 数据库连接/会话/SQLite 或 PG
│   ├── models.py          # users / refresh_tokens / oauth_clients
│   ├── security.py        # 密码哈希、JWT 签发与验签、密钥加载
│   ├── keys.py            # RSA 私钥/公钥加载、kid 计算、JWKS 生成、轮换
│   └── routers/
│       ├── auth.py        # /v1/register /v1/login /v1/refresh /v1/logout
│       ├── userinfo.py    # /v1/userinfo
│       └── wellknown.py   # /.well-known/jwks.json、/.well-known/openid-configuration
├── tests/
│   └── test_auth_flow.py  # 注册→登录→userinfo→refresh→logout 全链路
└── README.md              # 启动方式、环境变量表、令牌语义与失效取舍
```

### 6.3 技术选型

| 项 | 选型 | 说明 |
|---|---|---|
| Web 框架 | FastAPI + Uvicorn | 与其它服务一致 |
| 数据库 | **SQLite** 起步（挂卷持久化），用户量上来换 PostgreSQL | SSO 数据量小；表结构保持可迁移，别用 SQLite 专有语法 |
| 密码哈希 | **bcrypt**（`passlib[bcrypt]` 或 `bcrypt`）或 argon2 | 每个密码独立盐；绝不明文/MD5/SHA1 |
| JWT | `PyJWT`（`python-jose` 亦可），算法 **RS256** | **RS256 而非 HS256**：业务服务只拿公钥，不必持有共享密钥 |
| 密钥 | 启动时从文件/环境变量加载 RSA 私钥；**持久化到挂卷** | 容器重建不能重新生成密钥，否则所有已签发令牌失效 |
| Refresh Token | `secrets.token_urlsafe(32)` + 数据库存哈希 | 不透明串，可即时撤销 |
| 迁移 | 起步 `CREATE TABLE IF NOT EXISTS` 即可 | 别为小项目引入 Alembic 的复杂度，但表结构写进 README |

### 6.4 容器与环境变量

```yaml
  sso:
    build: ./sso
    restart: unless-stopped
    volumes:
      - ./data/sso:/data                 # SQLite + RSA 密钥，必须持久化
    environment:
      # ⚠️ 当前域名不可用，先用 IP（见 §0.9）；域名上线时只改下面两行，代码不动
      PUBLIC_BASE_URL: "http://<服务器IP>:8000/auth"   # 域名阶段改为 https://api.elipese.cloud/auth
      CORS_ORIGINS: "http://<服务器IP>:<前端端口>,http://localhost:3000"  # 域名阶段改为 https://elipese.cloud,https://www.elipese.cloud
      # issuer 与 jwks_uri 由 PUBLIC_BASE_URL 派生，无需单独配置，避免两者不一致
      DATABASE_URL: "sqlite:////data/sso.db"
      JWT_ALG: "RS256"
      JWT_KID: "2026-01"
      JWT_PRIVATE_KEY_PATH: "/data/jwt_private.pem"
      ACCESS_TOKEN_TTL: "3600"           # 秒
      REFRESH_TOKEN_TTL: "2592000"       # 30 天
      ALLOW_REGISTRATION: "false"        # 是否开放自助注册
      LOG_LEVEL: "info"
    networks:
      - elicloud
    labels:                              # caddy-docker-proxy 自动生成路由
      caddy: api.elipese.cloud           # 当前阶段：不启用 Caddy 时删掉整个 labels 段，
                                         # 改为直接映射端口 ports: ["8000:8000"]，见 §0.9
      caddy.handle_path: /auth/*
      caddy.0_reverse_proxy: "{{upstreams 8000}}"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/.well-known/jwks.json"]
      interval: 30s
      timeout: 3s
      retries: 3
    # 不把 8000 暴露到公网，只经网关访问
    # 端口选 8000：现有 lumi-backend / nonebot / chatroom 容器内都占 8080，避开以免混淆
```

**安全要求**：
1. 私钥文件权限 `600`，绝不进 Git、绝不打进镜像层。
2. `SECRET_KEY`/私钥等敏感值只从环境变量或挂卷文件读取。
3. 登录接口按 IP + 用户名双维度限流（`slowapi` 或自带计数器）。
4. 日志**不记录**密码、令牌原文；只记 `user_id`、结果与耗时。
5. 响应里不回显 `password_hash` 等内部字段。

### 6.5 表结构（起步版）

```sql
CREATE TABLE IF NOT EXISTS users (
  id            TEXT PRIMARY KEY,          -- 如 "user_0001"，对外即 sub
  username      TEXT UNIQUE NOT NULL,
  email         TEXT UNIQUE,
  password_hash TEXT NOT NULL,
  status        TEXT NOT NULL DEFAULT 'active',
  created_at    TEXT NOT NULL,
  updated_at    TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS refresh_tokens (
  id          TEXT PRIMARY KEY,
  user_id     TEXT NOT NULL REFERENCES users(id),
  family_id   TEXT NOT NULL,               -- 令牌链（一次登录 = 一条链），重放时整链撤销
  token_hash  TEXT NOT NULL UNIQUE,
  expires_at  TEXT NOT NULL,
  used_at     TEXT,                        -- 已被轮换使用的时间，再次使用即判定为重放
  revoked_at  TEXT,
  created_at  TEXT NOT NULL,
  user_agent  TEXT,
  ip          TEXT
);

CREATE TABLE IF NOT EXISTS oauth_clients (      -- 本期可只建表不用
  client_id     TEXT PRIMARY KEY,
  client_secret_hash TEXT NOT NULL,
  name          TEXT NOT NULL,
  redirect_uris TEXT,
  scopes        TEXT,
  created_at    TEXT NOT NULL
);
```

---

## 7. 服务端部署框架（部署形态 · 动态网关 · 服务上下线）

> **本章定位**：前面几章讲的是"服务长什么样"（契约、认证、数据隔离），本章讲的是"服务怎么跑到服务器上"，
> 以及"服务上线 / 下线怎么体现在网关和前端"。
> 框架层的「触发器与部署器分离」见 [`deploy-framework.md`](deploy-framework.md)（代码怎么送到服务器、谁调用谁）；
> 本章讲的是**部署器以后的事**——容器起来之后，它凭什么成为一条能被外部访问的路由。两者的衔接点在 §7.10。
>
> 一句话概括本章：**服务模块化的可操作定义 = 容器边界 = 路由边界 = 功能入口边界。**

三条主线，一条比一条更"动态"：

| 主线 | 一句话 | 小节 |
|---|---|---|
| **部署形态** | 一个服务 = 一个容器 = 一份数据卷；服务之间不共享数据库、不互相访问 | §7.1 |
| **网关形态** | 容器的启停本身就是服务注册/注销，加服务不用改网关配置 | §7.2 ~ §7.6 |
| **服务可见性** | 路由变化必须能被前端 / App 感知，否则用户会点到已经 404 的入口 | §7.7 |

### 7.1 部署形态：单机 Docker Compose，一服务一容器一数据卷

目标规模是**自托管单机**（§11），所以部署形态刻意保持"薄"：不上 K8s、不上 Swarm、不引入服务网格。

| 维度 | 约定 | 理由 |
|---|---|---|
| 部署单元 | 每个服务**一个容器**：`sso`、`main-api`、`pdf-decrypt`、`main-web-front` … | 启停粒度 = 服务粒度，才能"装卸一个服务不影响其它服务" |
| 编排方式 | 一份 `docker-compose.yml` + **compose profiles**（§7.5） | 单机上 profile 就是"可选服务开关"，不需要额外编排层 |
| 数据 | 每个服务**自己的数据卷**（如 `./data/sso:/data`），**服务间不共享数据库** | 服务之间只通过 HTTPS + JWT 交互（§4） |
| 端口 | 服务容器内监听 `8000`（避开既有容器占用的 `8080`），**不向公网发布** | 外部世界只认网关的 80/443，服务端口是内网细节 |
| 调试 | 需要本机 `curl` 时把端口**绑到回环地址**（`127.0.0.1:8000:8000`） | 公网与 Docker 内网都不可达，不为"调试方便"开口子 |
| 持久化 | 私钥、数据库挂卷到 `/data`，**绝不进镜像层、绝不进 Git** | 容器重建不得重新生成密钥，否则已签发令牌全部失效（§0.4 硬约束 5） |

> ✅ **已在 SSO 上落地**：见 [`sso/docker-compose.yml`](https://github.com/EliCloudOrg/sso/blob/main/docker-compose.yml)——只加入外部网关网络、不发布公网端口、
> 8000 绑回环、`/data` 挂卷、带 healthcheck、以宿主机用户身份运行。**新服务照抄这个形状即可**，
> 这也是 §7.8「新增一个服务」清单里第 2、3 步的模板。

### 7.2 动态网关：容器启停即服务注册/注销

标准「动态网关 + 模块化服务」模式：**容器的启停本身就是服务注册/注销**。

```text
                 ┌────────────────────────────────────────┐
                 │   Caddy 动态网关（caddy-docker-proxy）    │
                 │   - 监听 Docker 容器启停事件              │
                 │   - 容器在线 → 自动添加路由               │
                 │   - 容器离线 → 自动移除路由               │
                 └──────┬──────────┬──────────┬────────────┘
                        │          │          │
        ┌───────────────▼┐  ┌──────▼───────┐ ┌▼──────────────┐
        │ main-web-front │  │ main-api     │ │ sso / pdf-decrypt│
        │ (elipese.cloud)│  │ (/core/*)    │ │ (/auth/*, /pdf-*)│
        └────────────────┘  └──────────────┘ └───────────────┘
```

为什么选它：

| 能力 | 说明 |
|---|---|
| 容器启动 → 路由自动生效 | 无需改 Caddyfile、无需 reload |
| 容器停止 → 路由自动消失 | 外部访问立即 404/503 |
| 自动 HTTPS | Caddy 原生，一张证书覆盖 `elipese.cloud` / `api.elipese.cloud` |
| 前缀剥离 | `handle_path` 在网关层剥离 `/auth/*`、`/pdf-decrypt/*`，服务内部无感知 |
| 健康检查 | 容器有 `healthcheck` 时，不健康自动摘除路由 |

备选对比：Traefik（成熟但引入新组件）⭐⭐⭐；静态 Caddyfile + `caddy reload`（每次装卸要改配置）⭐⭐；Consul + consul-template（当前规模没必要）❌。

### 7.3 网关容器与路由声明（labels）

网关本身只是一个 Caddy，**路由不写在 Caddyfile 里，而是写在每个服务容器的 labels 上**——这就是"加服务不用改网关"的全部秘密。
网关容器只需要三件事：只读地看 Docker、拿到证书、加入服务网络（下面是片段，完整文件见 §7.4）：

```yaml
  caddy:
    image: lucaslorentz/caddy-docker-proxy:ci-alpine
    restart: unless-stopped
    ports: ["80:80", "443:443"]
    environment:
      CADDY_INGRESS_NETWORKS: elicloud      # 只在服务网络内发现容器
      ACME_EMAIL: "you@elipese.cloud"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro   # 只读！
      - caddy_data:/data
      - caddy_config:/config
    networks: [elicloud]

networks:
  elicloud: { external: true }

volumes:
  caddy_data:
  caddy_config:
```

每个**服务**容器用三条 label 声明自己挂在哪（完整编排见 §7.4）：

| label | 作用 | 示例 |
|---|---|---|
| `caddy` | 这个服务属于哪个站点 | `caddy: api.elipese.cloud` |
| `caddy.handle_path` | 独占的路由前缀，**网关会把它剥掉**（§7.6） | `caddy.handle_path: /pdf-decrypt/*` |
| `caddy.0_reverse_proxy` | 反代到容器内的哪个端口 | `caddy.0_reverse_proxy: "{{upstreams 8000}}"` |

> labels 里**只写前缀和容器内端口**，不写 IP、不写服务发现：容器在网络里叫什么名字、地址是什么，
> 由 `{{upstreams}}` 从 Docker 元数据里自己解析。所以**同一份 compose 可以在任何主机上跑，不需要改地址**。

### 7.4 完整编排骨架（多服务 compose）

把 §7.1 的部署形态、§7.2 的动态网关、§7.3 的 labels 拼起来，就是平台编排的目标形状。
**注意只有一个容器发布端口**：

```yaml
services:
  # ---------- 动态网关（唯一发布端口的容器） ----------
  caddy:
    image: lucaslorentz/caddy-docker-proxy:ci-alpine
    restart: unless-stopped
    ports: ["80:80", "443:443"]
    environment:
      CADDY_INGRESS_NETWORKS: elicloud
      ACME_EMAIL: "you@elipese.cloud"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - caddy_data:/data
      - caddy_config:/config
    networks: [elicloud]

  # ---------- 主前端 ----------
  main-web-front:
    image: ...
    restart: unless-stopped
    networks: [elicloud]
    labels:
      caddy: elipese.cloud
      caddy.0_reverse_proxy: "{{upstreams 3000}}"

  # ---------- 平台核心 API ----------
  main-api:
    image: ...
    restart: unless-stopped
    networks: [elicloud]
    labels:
      caddy: api.elipese.cloud
      caddy.handle_path: /core/*
      caddy.0_reverse_proxy: "{{upstreams 8000}}"

  # ---------- SSO ----------
  sso:
    build: ./sso
    restart: unless-stopped
    volumes: ["./data/sso:/data"]
    networks: [elicloud]
    labels:
      caddy: api.elipese.cloud
      caddy.handle_path: /auth/*
      caddy.0_reverse_proxy: "{{upstreams 8000}}"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/.well-known/jwks.json"]
      interval: 30s
      timeout: 3s
      retries: 3

  # ---------- PDF 解密（可选服务，按需启用） ----------
  pdf-decrypt:
    image: ...
    profiles: ["pdf"]                 # 默认不启动；启用即上线
    restart: unless-stopped
    volumes: ["./data/pdf:/data"]
    networks: [elicloud]
    labels:
      caddy: api.elipese.cloud
      caddy.handle_path: /pdf-decrypt/*
      caddy.0_reverse_proxy: "{{upstreams 8000}}"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 3s
      retries: 3

networks:
  elicloud: { external: true }        # 所有服务与网关共用一张内网

volumes:
  caddy_data:
  caddy_config:
```

自查这张骨架时的三条硬要求：

1. 除 `caddy` 外**没有任何服务写 `ports:`**——写一个就把内网服务暴露成了公网攻击面；
2. 除 `caddy` 的 `docker.sock` 外**没有任何容器挂宿主 socket**；
3. 每个服务都有 `healthcheck`，否则"容器在跑但服务已挂"时网关不会摘路由（§7.7）。

### 7.5 模块化启停（compose profiles）

**profile 未启用 = 容器不存在 = 路由不存在**，所以"装卸服务"和"开关功能"是同一个动作：

```bash
docker compose --profile pdf up -d pdf-decrypt   # 启动 → /pdf-decrypt/* 立即可访问
docker compose --profile pdf stop pdf-decrypt    # 停止 → 该路由自动关闭，返回 404
docker compose --profile pdf down                # 完全卸载
```

| 动作 | 命令 | 外部表现 | 数据 |
|---|---|---|---|
| 上线一个可选服务 | `docker compose --profile pdf up -d pdf-decrypt` | 前缀立即可访问，其它服务无感 | 沿用已有卷 |
| 下线（保留数据） | `docker compose --profile pdf stop pdf-decrypt` | 该前缀立刻 404/503，其它路由不受影响 | **保留** |
| 彻底卸载 | `docker compose --profile pdf down` | 容器与网络成员消失 | 卷按需保留/删除 |

> 核心服务（`caddy` / `sso` / `main-api` / `main-web-front`）**不加 profile**，随 `docker compose up -d` 一起起；
> 只有"可装卸的业务服务"才进 profile。这条分界就是"平台核心 vs 可选模块"的部署侧表达。

### 7.6 前缀剥离与「服务内部无前缀」

`handle_path` 在网关层剥掉服务前缀，服务容器内只看到 `/v1/...` 与 `/.well-known/...`（契约见 §2.2）：

```text
外部（用户 / App 看到的）                                 容器内（服务代码看到的）
──────────────────────────────────────────────────────  ────────────────────────────
POST api.elipese.cloud/auth/login                    →  POST /v1/login
GET  api.elipese.cloud/auth/.well-known/jwks.json    →  GET  /.well-known/jwks.json
POST api.elipese.cloud/pdf-decrypt/v1/jobs           →  POST /v1/jobs
```

**这条约定的部署价值**：服务代码与"自己被挂在哪个前缀下"完全解耦——以后想把 `pdf-decrypt` 挪到独立域名
`pdf.elipese.cloud`，只改网关侧一行 labels，服务镜像与代码一行不动。

**代价**：服务内部视角的路径 ≠ 对外路径，因此**一切"对外地址"必须由环境变量注入**，
最典型的就是 `issuer` 与 `jwks_uri`（§0.4 硬约束 1、§3.1）。服务容器内永远不要用请求 Host 推导对外地址。

### 7.7 服务上下线如何被前端感知（服务列表 API）

容器装卸只改变路由，**不会自动通知前端**。所以平台需要一个"当前有哪些服务在线"的出口，
由 `main-api` 提供，契约见 §5.8。完整链路：

```text
docker compose --profile pdf up -d pdf-decrypt
    ↓  容器启动 + labels 生效
Caddy 监听到 Docker 事件 → 自动生成 /pdf-decrypt/* 路由
    ↓
main-api 的 GET /core/v1/services 里 pdf-decrypt online=true
    ↓
前端 / App 显示「PDF 解密」入口可用（online=false 时置灰或隐藏）
```

在线判定来源二选一：

| 来源 | 方式 | 优点 | 缺点 |
|---|---|---|---|
| **A. Caddy Admin API（推荐）** | 查询 `GET /config/`，解析当前生效路由 | **"路由存在 = 真的可达"**，与用户实际体验一致 | 需要 Caddy 与 `main-api` 之间有内网 admin 通道 |
| B. Docker Socket（只读） | 查询带约定 label 的运行中容器 | 实现直接 | **容器在跑 ≠ 健康检查通过 ≠ 路由已生效** |

> 无论选哪种，**服务容器都必须有 healthcheck**（§7.9 第 5 条）：没有它，网关无法摘除"容器在跑但服务已挂"的路由，
> 服务列表也就会撒谎。前端轮询该接口即可，暂不需要 WebSocket / SSE 推送。

### 7.8 新增一个服务的标准动作

§9「以后加新服务，复制同一接入模式」在部署侧展开就是这张清单——**只动新服务自己，不动网关配置、不动其它服务**：

| # | 动作 | 落在哪 | 参照 |
|---|---|---|---|
| 1 | 建仓库 / 目录，写 `deploy/deploy.sh`（本项目具体怎么部署） | 新服务仓库 | [`deploy-framework.md`](deploy-framework.md) |
| 2 | 写 `Dockerfile`：容器内监听 `8000`、数据挂 `/data`、`healthcheck` | 新服务仓库 | §6.4 |
| 3 | 加进平台编排：接 `elicloud` 网络、可选 `profiles`、不发布端口 | 平台 compose | §7.4 |
| 4 | 加三条 labels：**全局独占前缀** + `{{upstreams 8000}}` | 平台 compose | §7.3 |
| 5 | 接 SSO：同一套 JWKS 验签、按 `sub` 隔离数据 | 新服务代码 | §8 |
| 6 | 在 `GET /core/v1/services` 登记该项（id / 名称 / 路径） | `main-api` | §5.8 |
| 7 | 前端 / App 加入口（按 `online` 显示或置灰） | 前端 | §5.8 |

> 第 4 步的**前缀必须全局唯一**：`/auth`、`/core`、`/pdf-decrypt`、`/xxx` 各服务独占。
> 冲突时的表现是"路由被另一个服务悄悄覆盖"，而且**这是整条链路里唯一一处框架不会替你检查的错误**（见 §7.9 第 1 条）。

### 7.9 约定与安全底线（必须遵守）

1. **路由前缀唯一**：`/auth/*`、`/core/*`、`/pdf-decrypt/*` 各服务独占，避免冲突。
2. **网关统一剥离前缀**：服务容器内只看到 `/v1/...`。
3. **网络隔离**：所有服务加入同一 `elicloud` 内网，Caddy 用服务名访问；**不向公网暴露服务端口**。
4. **Docker socket 只读挂载**，并限制网关容器权限。
5. **每个服务都要 healthcheck**，否则「容器在跑但服务已挂」的路由不会被摘除。
6. **可观测**：Caddy 日志即服务上下线日志。
7. **调试端口只绑回环**（`127.0.0.1:8000:8000`）：任何"为了调试方便"而发布到公网的端口都是漏洞面。
8. **网关的证书卷必须持久化**（`caddy_data`）：否则容器重建会重新申请证书，容易撞上 ACME 的频率限制。

### 7.10 与自托管部署框架的衔接（触发器 / 部署器 / 网关）

[`deploy-framework.md`](deploy-framework.md) 讲的是「触发器与部署器分离」：GitHub Actions 只负责把 ref 送到服务器、
调用 `/srv/<app>/deploy.sh <ref>`，具体怎么构建、怎么重启由项目自己写在 `deploy.sh` 里。
把它与本章的动态网关拼起来，才是完整的服务端部署链路：

```text
   push / PR 合入 prod
        │
        ▼
   GitHub Actions（触发器）      —— 只做三件事：构建(可选) → scp deploy.sh → ssh 执行
        │  SSH
        ▼
   /srv/<app>/deploy.sh <ref>（部署器）—— 项目自己实现：拉代码/镜像 → docker compose up -d
        │  Docker
        ▼
   容器启停（动态网关的唯一输入）—— caddy-docker-proxy 监听到事件 → 路由自动增删
        │
        ▼
   外部可访问性变化 → GET /core/v1/services → 前端 / App 入口变化
```

三个直接结论：

1. **部署框架不需要知道网关**：`deploy.sh` 只要把容器起停做对，路由自动跟着变；触发器更是一个字都不用改。
2. **网关不需要知道 CI**：它唯一的输入是"Docker 里有哪些容器、容器上写了什么 labels"。两条链路只在这一处耦合。
3. **部署器必须幂等**（部署框架的硬性约束之一）：容器重建 = 路由重建，一次失败的重复部署会让服务在网关侧"闪断"。

> ⚠️ **本机环境的两个真实约束**（实测见 [`../elicloud-deploy-test/VALIDATION.md`](../elicloud-deploy-test/VALIDATION.md)）：
>
> - **snap 版 Docker 看不到 `/srv`**：`deploy.sh` 里把 `/srv/...` 交给 docker（bind mount、`--env-file`、构建上下文）都会失败。
>   所以"部署器放在 `/srv/<app>/`"与"容器数据落在哪里"必须分开：当前 SSO 的部署目录是
>   `/home/docker-admin/elicloud/sso`（Docker 可见），而 `/srv/<app>/` 只放 `deploy.sh` 与 `app.env`，
>   由 shell 读出来再以环境变量传给容器。
> - **`ghcr.io` 在部分网络不可达**：需要给 dockerd 配 `registry-mirrors`，或让 `deploy.sh` 支持镜像源变量
>   （SSO 的 `deploy.sh` 已支持：`app.env` 里改 `SSO_IMAGE_REPO` 即可换镜像源，例如 `ghcr.nju.edu.cn/elicloudorg/sso`）。
>
> ✅ **SSO 已于 2026-10-06 接入该框架**（第一个真实业务服务，不再是手工 `docker compose up -d --build`）：
>
> | 项 | 值 |
> |---|---|
> | 仓库 | [`EliCloudOrg/sso`](https://github.com/EliCloudOrg/sso)（public） |
> | 触发 | push 到 `prod` / `workflow_dispatch(ref)`，`DEPLOY_MODE=image` |
> | 部署入口 | `/srv/sso/deploy.sh`（框架上传）+ `/srv/sso/app.env`（运行时变量） |
> | docker 可见的项目目录 | `/home/deploy/elicloud-sso`（编排由 `deploy.sh` 生成，compose 项目名 `sso`） |
> | 数据目录 | `/home/docker-admin/elicloud/sso/data`（私钥 + SQLite，**不随部署移动**） |
> | 镜像 | `ghcr.io/elicloudorg/sso:prod` / `:sha-<short>` |
>
> 细节见 [`https://github.com/EliCloudOrg/sso/blob/main/README-deploy.md`](https://github.com/EliCloudOrg/sso/blob/main/README-deploy.md)。

### 7.11 现状与演进路径（静态 Caddyfile → 动态网关）

**当前 IP 阶段用的是"静态网关"**：网关是 dsh-nas 里既有的 Caddy 容器，路由写在 Caddyfile 里
（`handle_path /auth/*` → `sso:8000`），SSO 只加入它所在的 external 网络、不发布端口，
详见 [`sso/docker-compose.yml`](https://github.com/EliCloudOrg/sso/blob/main/docker-compose.yml) 与 `sso/README.md`。
**这保证了现在就能跑，但每加一个服务都要改网关配置并 reload。**

目标形态是本章的动态网关。两者的差别**只在"路由从哪来"**，服务侧的形状完全一样：

| 维度 | 现在：静态 Caddyfile | 目标：caddy-docker-proxy |
|---|---|---|
| 路由声明在哪 | 网关的 `Caddyfile` | **各服务容器的 labels** |
| 加一个服务要做什么 | 改 Caddyfile + reload（**要动网关，有全局风险**） | 起一个带 labels 的容器（**不碰网关**） |
| 停一个服务会发生什么 | 路由还在，打到空上游 → 5xx | 路由自动消失 → 干净的 404 |
| 健康摘除 | 需要在 Caddyfile 里手工配 upstream 健康检查 | 容器 `healthcheck` 自动生效 |
| 网络 | 复用既有网关网络（`dsh-nas_dsh-net`） | 独立 `elicloud` 内网 |
| 证书 | 沿用 dsh-nas 既有 Caddy 的证书（IP 阶段为 `profile shortlived` 的 IP 证书，§0.9） | 网关自己的 `caddy_data` 卷 |

**迁移时机与顺序（建议）**：

1. **域名阶段再做**：等 `elipese.cloud` / `api.elipese.cloud` 可用后一次性切换网关，
   避免在"IP 证书 + 复用别人 Caddy"的组合上叠加变更。
2. **服务侧提前按目标形状写好**：`main-api`、`main-web-front`、`pdf-decrypt` 无论先跑在静态 Caddyfile 还是动态网关上，
   编排里都按 §7.4 写全网络、端口、healthcheck、labels——这样切换网关时**只换网关容器，服务编排一行不改**。
3. **切换前**确认：独立 `elicloud` 网络已创建、`caddy_data` 卷已就绪、新网关能拿到证书。
4. **切换后回归**：`GET /core/v1/services` 的 `online` 与实际路由一致；停掉一个可选服务，确认返回 404 而不是 5xx。

> 一句话：**动态网关是"以后不用再改网关配置"的能力，不是"现在必须立刻上"的前提。**
> 现在唯一要提前做对的，是把每个服务容器的形状（网络、端口、healthcheck、labels）按目标形态写好。

---

## 8. 与业务服务的对接约定（写给 PDF 解密服务的实现者）

业务服务**完全不碰用户密码**，只做一件事：验签 SSO 签发的 JWT。

```python
# 域名阶段
SSO_ISSUER   = "https://api.elipese.cloud/auth"
SSO_JWKS_URL = "https://api.elipese.cloud/auth/.well-known/jwks.json"

# 当前 IP 阶段（域名不可用，见 §0.9）—— 这两项必须与 SSO 的 PUBLIC_BASE_URL 完全一致
SSO_ISSUER   = "http://<服务器IP>:<端口>/auth"
SSO_JWKS_URL = "http://<服务器IP>:<端口>/auth/.well-known/jwks.json"
```

> ⚠️ 业务服务的 `SSO_ISSUER` 是**由环境变量注入**的：域名上线时只改这个变量并重启，**验签代码一行不用改**。若两边 issuer 不一致，表现为「所有令牌都验签失败」——排查时先比对这两个值。

- 用 `kid` 匹配公钥，本地缓存 JWKS（尊重 `Cache-Control`），验签失败/未知 `kid` 时才重新拉取。
- 校验 `iss` == `SSO_ISSUER`、`exp` 未过期、算法白名单（只接受 RS256，**拒绝 `alg: none` 与 HS256**）。
- 从 `sub` 取用户身份，业务表所有读写都带 `WHERE user_id = <sub>`。
- 越权访问他人资源返回 **404**（而不是 403），避免泄露资源是否存在。
- 建议把验签逻辑抽成一个公共 Python 包（例如 `elicloud_auth`），所有业务服务共用，避免各写一份。

---

## 9. 建议落地顺序

```text
第一步：EliCloud SSO（注册/登录/refresh/userinfo/JWKS）   ← 基础设施，本任务
第二步：main-api（服务列表 API + 核心业务 + JWT 验签）
第三步：PDF 解密服务接入 SSO（任务/结果 API + hashcat 后台执行器）
第四步：caddy + docker-compose + HTTPS 上线（含 profiles 模块化）
第五步：前端/App（登录 + 上传 + 任务列表/结果页）
第六步：以后加新服务，复制同一接入模式
```

---

## 10. 待决策事项（开工前请确认）

| # | 决策 | 选项 | 讨论中的倾向 |
|---|---|---|---|
| 1 | SSO 实现方式 | FastAPI 自研轻量版 / Casdoor 品牌化 | **自研 FastAPI**（依赖少、与其它服务同栈、完全可控） |
| 2 | SSO 数据库 | SQLite / PostgreSQL | SQLite 起步（挂卷持久化），表结构保持可迁移 |
| 3 | 是否开放自助注册 | 开放 / 管理员建号 / 配置开关 | 先实现接口，用 `ALLOW_REGISTRATION` 开关控制 |
| 4 | 登录形式 | 跳转网页登录 / App 内嵌原生表单 | 内嵌原生表单（用户名密码只发给 SSO） |
| 5 | Access Token 有效期 | 短 TTL / 长 TTL + 吊销名单 | 短 TTL（15~60 分钟）+ refresh 轮换 |
| 6 | 平台核心 API 归属 | 新建 `main-api` / 由现有服务兼任 | **✅ 已有事实答案：必须新建**。现有三个项目都不是 Web API（见第 12 节），无人可兼任 |
| 7 | 站点名称 | 文档用「EliCloud」/ 域名 `elipese.cloud` | 以域名 `elipese.cloud` 为准，站点名待统一 |
| 8 | **IP 阶段的 HTTPS 方案**（域名不可用，至少半个月） | A. 纯 HTTP + IP ／ B. Caddy 自签（`tls internal`）／ C. 不用 Caddy，直接映射端口 | **✅ 已解决**：实际采用"复用既有 Caddy + Let's Encrypt **IP 证书**（`profile shortlived`）"，即 HTTPS 且非自签。详见 §0.9 已落地现状 |
| 9 | **当前阶段的服务器 IP 与端口** | 例如 `146.56.237.33` + 端口 443 | **✅ 已确认并落地**：`PUBLIC_BASE_URL = CORS 之外的对外地址 = https://146.56.237.33/auth` |

> 第 8、9 项曾是本轮的硬性阻塞项（**没有确切的 IP + 端口 + scheme，`issuer` 就无法正确配置**），
> 现已全部确认并落地，实际取值见 §0.9 的"已落地现状"表格。

---

## 11. 部署环境事实

- 远程主机：`ubuntu@146.56.237.33`（另有 `docker-admin@146.56.237.33` 容器管理账号），**远程只有 CPU，无 GPU**。
- **域名 `elipese.cloud` / `api.elipese.cloud` 暂不可用，至少还需半个月；当前只能用 IP 部署（见 §0.9）。**
- HTTPS/TLS 由 Caddy 自动申请（域名阶段：`elipese.cloud` / `api.elipese.cloud`）；**IP 阶段原本以为拿不到 Let's Encrypt 证书，实际用 `profile shortlived` 拿到了 IP 证书**，故当前阶段已是 HTTPS（见 §0.9 已落地现状）。
- ⚠️ **公网 IP 曾因服务器被攻击而更换**：被攻击前的旧 IP 已完全不可达，当前唯一有效地址是 `146.56.237.33`。建议固定为弹性 IP。
- 代码仓库：`git@elipese.cloud:22`（EliCloudGit）；另有 GitHub 组织 `github.com/EliCloudOrg`（已确认 `lumi-bot` 在其下）。
- 部署形态：Docker + Docker Compose，每服务独立容器、独立数据卷。
- 端口备忘：现有 `lumi-backend` / `lumi-bot(nonebot)` / `chatroom-server` 容器内均监听 **8080**；**SSO 用 8000**，避免混淆。

---

## 12. 现有工程现状盘点（只读侦察结论，SSO 实现者必读）

仓库 `D:\work\projects\EliCloud` 下**已有的三个项目都不属于 EliCloud 平台服务体系**，SSO 是全新工程，**不存在「复用现有用户表 / 与现有服务共用数据库」的场景**。

| 项目 | 实际角色 | 端口 / 镜像 | 已有认证 |
|---|---|---|---|
| `lumi-backend` | **QQ 机器人 OneBot11 后端**（.NET 10 + Sora 脚手架，全仓仅 10 行 `Program.cs`） | 容器内 `8080`（compose 未映射任何端口）／ 镜像 `${DOCKER_REGISTRY-}lumibackend` | 无。仅 OneBot 协议 `AccessToken="a"`，**不是用户认证** |
| `chatroom-server` | 独立简易聊天室（Node 22 + `ws`，与平台无集成点） | `23456:8080`／`chatroom-server:1.0.0`，容器名 `chatroom` | 假登录：用户名即身份，`README.md:276` 自述「无身份认证，任何人可冒用」；另有内存 uploadToken |
| `lumi-bot` | QQ 机器人（NoneBot2 + NapCat，与 `lumi-backend` 是同一目标的两种实现：Python vs C#） | nonebot `8080` + napcat `3000/3001/6099` | 无 |

**关键事实**：

1. `lumi-backend` **不是 Web API**：`lumi-backend.csproj:5` 为 `OutputType=Exe` / `net10.0`，无 ASP.NET Core、无 EF Core、无 `appsettings`、无 Controller、无用户表 —— 它**无法承载 `/core/*`**。因此 §10 第 6 项「平台核心 API 归属」的答案是**必须新建 `main-api`**。
2. **这三个项目里没有任何 SSO/JWT/JWKS/OAuth 实现**：所有 `jwks / issuer / oauth / jwt` 关键词命中只有两类 —— 本文档（设计规格）与 chatroom 的内存 uploadToken。
   > ⚠️ **本节写于 2026-09-25，当时的"全仓无部署脚本"结论已过时**：此后已落地自托管部署框架（`.github/workflows/`、`deploy/deploy.sh`、`scripts/bootstrap-server.sh`，见 [`deploy-framework.md`](deploy-framework.md)）与 SSO 一期（`sso/`，含网关 Caddyfile 片段）。**"没有可复用的认证代码"这一结论仍然成立**，但"没有部署脚本"不再成立。
3. 三个项目容器内**都监听 8080**。同主机部署时靠 Docker 网络与服务名区分；**SSO 建议用 8000**（本文档 §6.4 即按 8000 写），并注意 Caddy labels 里的 `{{upstreams 8000}}` 要与实际端口一致。
4. SSO 与这三个项目唯一的将来接触面是：**让它们按本文档 §3.1 / §8 验签 SSO 签发的 JWT**。所以本期 SSO 契约**只需与 §3.1 的 issuer / JWKS 路径严格一致，不必迁就这三个项目**。
5. 唯一的 EliCloud 组织痕迹：`lumi-bot` 的 git remote 为 `github.com/EliCloudOrg/lumi-bot.git`。

---

## 13. 附：PDF 解密服务要点（第二个任务，仅供上下文对齐）

SSO 实现者无需实现这些，但需知道平台整体形态：

- 链路：`PDF → pdf2hashcat.pl → hashcat → 密码`；远程仅 CPU，用 `pocl-opencl-icd` 跑 OpenCL，调用时加 `-D 1` 只选 CPU 设备。
- 典型需求是 **6 位纯数字**（100 万组合），CPU 足够；`max_runtime` 默认 600 秒超时。
- 任务状态机：`queued → running → finished / failed / timeout / cancelled`。
- 参数均可缺省：`hash_mode=auto`、`attack_mode=3`（掩码）、`mask=?d?d?d?d?d?d`、`max_runtime=600`。
- 后台执行器单线程串行（`ThreadPoolExecutor(max_workers=1)` + 全局锁），避免多个 hashcat 争抢资源。
- 安全：**绝不能拼 shell 字符串**调用 hashcat/pdf2hashcat，必须用参数数组；上传先做文件类型/病毒校验；任务结束清理临时文件。

---

## 14. 相关文档

| 文档 | 内容 |
|---|---|
| 本文件 | 平台架构与 SSO 一期契约（`/auth/*`、令牌语义、路径规范、issuer 硬约束）+ **§7 服务端部署框架**（部署形态、动态网关、服务上下线、新增服务清单） |
| [`mc.md`](mc.md) | **第二个业务服务**：MC 服务器白名单服务（`/mc/*`，容器 `mc`）。纯 API + SSO Bearer 验签，经 RCON 把用户提交的 MC 用户名写入 `urania-mc` 白名单。含线上实测的 RCON 行为、表结构、网关片段、部署与验收清单 |
| [`https://github.com/EliCloudOrg/file/blob/main/README.md`](https://github.com/EliCloudOrg/file/blob/main/README.md)、[`https://github.com/EliCloudOrg/file/blob/main/README-deploy.md`](https://github.com/EliCloudOrg/file/blob/main/README-deploy.md) | **第三个业务服务**：通用文件服务（`/file/*`，容器 `file`）。上传/下载/删除、id + 有效期、`public`/`private`/`protected` 可见性、按 SSO `sub` 授权；内容存私有 COS 桶（代理 / 预签名两种下载链路），元数据在本地 SQLite |
| [`sso-oidc.md`](sso-oidc.md) | SSO 二期：扩展为完整 OIDC Provider（端点契约、三种流程、表结构、安全清单） |
| [`deploy-framework.md`](deploy-framework.md) | **自托管部署框架的设计思想**（触发器与部署器分离）——与 §7.10 衔接：它管"代码怎么送到服务器"，§7 管"到了服务器之后怎么变成一条可访问的路由" |
| `sso/README.md` | SSO 一期的实现、部署与运维手册（环境变量、表结构、密钥轮换、阶段切换） |
| [`../elicloud-deploy-test/README-deploy.md`](../elicloud-deploy-test/README-deploy.md) | 部署框架的单个项目落地示例（操作手册） |
| [`../elicloud-deploy-test/VALIDATION.md`](../elicloud-deploy-test/VALIDATION.md) | 部署框架端到端验证报告（含回滚隐患 D4、snap Docker 约束） |
