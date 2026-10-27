# MC 服务器白名单服务（`mc`）设计规格

> **定位**：本文件是 [`architecture.md`](architecture.md) 第 7 章「服务端部署框架」下的一个**新增业务服务**规格，
> 与 `sso/`（`/auth/*`）、将来的 `main-api`（`/core/*`）、`pdf-decrypt`（`/pdf-decrypt/*`）并列。
> 接入约定沿用 `architecture.md` §8（业务服务只验签 SSO 的 JWT，按 `sub` 隔离数据）与 §7.8（新增一个服务的标准动作）。
>
> **状态**：设计已与需求方逐条对齐（见 §1.2 的决策表）；**代码已完成**（`mc/`，
> 92 个契约测试通过，见 §9.1 与 §12.5），**尚未部署**。本文档是写代码前的唯一真源。
> 文中的「✅ 实测」标记表示该结论来自线上真实环境验证，不是推断；实测方法与证据见 §9。

---

## 1. 目标与边界

### 1.1 一句话

**已经在 EliCloud SSO 注册过的用户，通过本服务的 API 提交自己的 Minecraft 用户名（每个账号最多 2 个）并附必填备注；
服务校验通过后立即通过 RCON 把该用户名写入 MC 服务器白名单，全过程留审计。**

```text
外部前端/客户端 ──Bearer <SSO access_token>──► mc :8000 ──RCON──► urania-mc:25575
  (OIDC 授权码+PKCE)                              │                          (whitelist add/remove/list)
                                                  └── SQLite: 名额 / 审计 / 快照
```

### 1.2 已对齐的决策（**写代码时不得擅自更改**）

| # | 决策项 | 定值 | 备注 |
|---|---|---|---|
| 1 | 身份 | **纯 API，无登录页**；`Authorization: Bearer <SSO access_token>`，JWKS 验签 | OIDC 授权码流程属于前端与 SSO 之间的事，**本服务不是 OIDC 客户端**，不需要在 SSO 注册 |
| 2 | 生效方式 | **自助即时生效**，无审批状态机 | 全量审计留痕替代事前审核 |
| 3 | 名额 | 每个 `sub` **终身最多绑定 2 个** MC 用户名（并发上限，删一个才能加一个） | 上限可配 `MAX_NAMES_PER_USER`，默认 2 |
| 4 | 归属 | MC 用户名**全局唯一**：已被 A 占用时 B 返回 `409` | 与 SSO 账号一对一，撤销时责任清晰 |
| 5 | 备注 | **必填**，长度 1–200；入库，管理员可查 | 不参与任何判定 |
| 6 | 撤回 | 用户可删除自己的条目，**立即 `whitelist remove`** | 只允许删 `sub` 自己的条目 |
| 7 | 写入通道 | **只走 RCON**，不直接改文件 | 与「MC 通过 RCON 内部执行指令」的运维方式一致 |
| 8 | UUID | **不手工构造**，交给服务端 `/whitelist add` 自己决定 | 见 §5.6 的实测形态 |
| 9 | 路由 | 对外 `/mc/*` → 容器 `:8000`；**新开独立 Caddy 站点块** | 不能并进面板站点块（会被 `basic_auth` 兜底拦下） |
| 10 | 服务内路径 | 带 `/v1`（如 `/v1/names`） | 沿用平台 §2.2 约定 |
| 11 | 界面 | **无 UI**，只提供 REST API | 前端由别处实现 |
| 12 | 网络 | 服务加入 `dsh-nas_dsh-net`（给网关）+ `mc_default`（给 RCON） | 不发布任何端口 |
| 13 | CORS | **可配**，当前阶段填开发来源 | 为将来浏览器直连预留 |
| 14 | RCON 密码 | 落在服务器 `/srv/mc/app.env`（600），**不进仓库、不进镜像、不进日志** | 由需求方从 `mc/docker-compose.yml` 提取后填入 |
| 15 | 白名单开关 | MC 侧**直接启用** `WHITELIST=TRUE` + `ENFORCE_WHITELIST=TRUE`，**不需要预加现有玩家** | 当前处于停服维护状态 |
| 16 | 令牌 scope（T1） | **必须含 `mc:whitelist`**，否则 `403 insufficient_scope` | ⚠️ 需要先给 SSO 加上该 scope（**§12.4**），否则全员 403 |
| 17 | 管理面（T2） | **容器内 CLI + HTTP 管理接口都要**；HTTP 用 `ADMIN_TOKEN`，未配置时 403 | 与 SSO §13 同构 |
| 18 | 撤回冷却期（T3） | **不加**，只靠限流 | — |
| 19 | 提交后通知（T4） | **不做** | 备注必填 + 全量审计已覆盖可查性 |
| 20 | 服务器范围（T5） | **只服务 `urania-mc`** | 表结构将来加 `server_id` 演进 |
| 21 | 部署方式（T7） | **先手工部署**，`deploy/deploy.sh` 按框架写，CI 后补 | — |

### 1.3 本服务明确不做

| 不做 | 原因 |
|---|---|
| 用户/密码/注册 | 身份完全由 SSO 负责（`architecture.md` §1） |
| 审批流、邮件/QQ 通知 | 已选自助即时生效；通知属以后的事 |
| 前端页面、OIDC 客户端注册 | 已选 API-only |
| 直接读写 `whitelist.json`、操作 world 数据 | 只走 RCON；world 由 MC 自己管 |
| 绑定/解绑 Mojang 账号校验、正版验证 | 见 §5.6：离线模式下这是假的安全性 |
| 多 MC 服务器 | 一期只服务一台 `urania-mc`；多实例见 §12 |

---

## 2. 环境事实（✅ 均为线上实测/只读侦察结论）

### 2.1 MC 服务器

| 项 | 实际值 | 来源 |
|---|---|---|
| 容器 | `urania-mc`，镜像 `urania-fabric:1.21.1`（itzg 系），`restart: unless-stopped` | `docker ps -a` |
| 运行身份 | `user: "1000:1000"`；容器内 uid 1000 = `minecraft` | compose + `docker exec id` |
| 版本 | Fabric 1.21.1 + Fabric Loader 0.19.3，JDK 21 | compose + 日志 |
| 网络 | `mc_default`（bridge），容器 IP `172.18.0.2` | `docker network inspect` |
| 端口 | 仅 `25565:25565` 发布；**RCON 25575 未发布** | `docker ps` |
| RCON（MC 内） | `enable-rcon=true`，`rcon.port=25575`，监听 `0.0.0.0:25575` | ✅ 容器内 `server.properties` + 日志 |
| RCON 账号 | `rcon.password` 由 MC 容器的 `RCON_PASSWORD` 环境变量生成 | ✅ `docker exec printenv` 与 properties 一致 |
| 模式 | `online-mode=false`（离线模式） | ✅ 容器内 `server.properties` + `ops.json` 全是 v3 UUID |
| 白名单（**改动前**） | `white-list=false`、`enforce-whitelist=false` | ✅ 容器内 `server.properties` |
| 白名单文件 | `/data/whitelist.json`，**bind mount 自宿主机 `/home/docker-admin/mc/whitelist.json`** | ✅ 两边内容逐字一致 |
| 部署目录 | `/home/docker-admin/mc/`（compose、world `urania-1437/`、日志、备份） | 只读侦察 |

> ⚠️ **`/data` 的挂载结构有个坑**（写代码/写运维命令前必须知道）：
> `docker inspect` 显示 `/data` 是一个**匿名卷**，而 `urania-1437/`、`ops.json`、`logs/`、`whitelist.json` 都是它的子路径 bind。
> 其中 **`whitelist.json` 与宿主机文件是同一个文件**（路径没有冲突，实测两边内容一致）；
> 但**匿名卷的目录 `docker-admin` 用户无权读取**（snap Docker，`Permission denied`），
> 也就是说「从宿主机直接编辑 MC 数据目录」这条路走不通 —— 要用容器内路径（`docker exec`）或 RCON。
> 本服务选的是 RCON，所以不受影响。

### 2.2 平台侧（本服务要接入的东西）

| 项 | 实际值 |
|---|---|
| SSO issuer | `https://146.56.237.33/auth`（`sso/.env` 的 `PUBLIC_BASE_URL`） |
| JWKS | `https://146.56.237.33/auth/.well-known/jwks.json` |
| access token `aud` | `elicloud-services` |
| `id_token` 的 `aud` | 是 `client_id`，**必须拒绝**（`architecture.md` §16.1） |
| 网关 | `dsh-caddy`（Caddy 2.11.4，**静态 Caddyfile**），配置 `/home/docker-admin/dsh-nas/caddy/Caddyfile` |
| 网关网络 | `dsh-nas_dsh-net`（external，SSO 已加入） |
| 现有服务部署目录 | `/home/docker-admin/elicloud/<app>`（snap Docker 不能拿 `/srv` 做 bind mount） |
| 部署框架 | `docs/deploy-framework.md`：`/srv/<app>/deploy.sh <ref>` 由 `deploy` 用户执行；运行时变量放 `/srv/<app>/app.env` |

### 2.3 现有玩家（启用白名单时需要知道的事实）

`ops.json` 里 8 个 OP（`ZyJ`、`Elipese568`、`RainHeart826`、`KasakuraNeko`、`Usqwq`、`sanli520`、`Uyania`、`Dreamlight`）——
**OP 不需要在白名单里**（`PlayerList.isWhiteListed` 对 OP 直接放行），所以「不预加现有玩家」不会把 OP 挡在外面。
非 OP 玩家在 `white-list=true` 后需要重新加白，这是启用白名单的**固有代价**，已与需求方确认接受。

---

## 3. 服务形状与部署形态

### 3.1 标准形状（照 `sso/` 复制，`architecture.md` §7.1）

```text
mc/
├── app/
│   ├── main.py          # FastAPI 装配、CORS、统一错误处理、访问日志
│   ├── config.py        # 全部配置来自环境变量（pydantic-settings 风格，与 sso 一致）
│   ├── db.py            # SQLite 引擎/会话/建表（CREATE TABLE IF NOT EXISTS）
│   ├── models.py        # mc_names / audit_logs / whitelist_cache
│   ├── errors.py        # 统一错误结构 + 进程内限流器（沿用 sso 风格）
│   ├── sso_auth.py      # SSO access token 验签（JWKS 缓存、RS256 白名单、iss/aud 校验）
│   ├── rcon.py          # 最小 RCON 客户端（见 §5）
│   ├── whitelist.py     # 业务编排：名额/唯一性/幂等/回读校验
│   ├── deps.py          # 依赖注入、客户端 IP、Bearer 鉴权
│   ├── cli.py           # 管理员命令（list / force-remove / reconcile）
│   └── routers/
│       ├── names.py     # /v1/names、/v1/me
│       └── ops.py       # /v1/admin/*、/healthz
├── tests/               # 契约测试 + 假 RCON 服务器
├── Dockerfile
├── requirements.txt / requirements-dev.txt
├── docker-compose.yml   # 双网络、无 ports、healthcheck
├── .env.example
└── README.md
```

### 3.2 Dockerfile 与运行约束（照抄 SSO 的取舍）

- 容器内监听 `8000`（避开既有容器占用的 `8080`）。
- 以宿主机用户 `1002:1003`（`docker-admin`）运行，**非 root**。
- 数据挂卷 `/data`（SQLite 落在这里），`./data` 必须属于 `1002:1003`。
- **不写 `ports:`**，公网入口只经网关。
- `healthcheck`：`GET /healthz`（**必须包含 RCON 可达性**，见 §5.7，否则"容器在跑但 RCON 挂了"不会被摘除）。

### 3.3 docker-compose.yml（目标形态）

> ⚠️ **compose 项目名不能是 `mc`**（2026-10-06 实际事故）：MC 服务器那套栈的编排在
> `/home/docker-admin/mc/`，**它的 compose 项目名就是 `mc`**（由目录名推导）。`docker compose`
> 按**项目名**匹配已有容器，撞名时会用本服务的编排去重建**对方的**容器 —— 实测把
> `urania-mc` 的 `/data` 匿名卷换成了本服务的 `./data`，MC 因此崩溃循环、服务器一度不可用。
> 因此本项目用 **`COMPOSE_PROJECT_NAME=mc-svc`**，容器名/镜像/`/srv` 路径仍是 `mc`。
> `deploy.sh` 已加执行前校验（核对服务名与 `container_name`）。

```yaml
services:
  mc:
    build: .
    image: elicloud-mc:1.0.0
    container_name: mc
    restart: unless-stopped
    user: "1002:1003"
    volumes:
      - ./data:/data
    environment:
      SSO_ISSUER: "https://146.56.237.33/auth"
      SSO_JWKS_URL: "https://146.56.237.33/auth/.well-known/jwks.json"
      JWT_AUDIENCE: "elicloud-services"
      # T1：令牌必须携带的 scope；空串 = 关闭校验（仅本地调试）
      REQUIRED_SCOPE: "mc:whitelist"
      DATABASE_URL: "sqlite:////data/mc.db"
      RCON_HOST: "urania-mc"
      RCON_PORT: "25575"
      # 密码不写在这里：由 app.env 注入（见 §3.5）
      MAX_NAMES_PER_USER: "2"
      CORS_ORIGINS: "http://localhost:5173,http://127.0.0.1:5173"
      LOG_LEVEL: "info"
    networks:
      - dsh-nas_dsh-net     # 网关按容器名 mc:8000 反代
      - mc_default          # 连 urania-mc:25575 的 RCON
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request;urllib.request.urlopen('http://127.0.0.1:8000/healthz',timeout=3)"]
      interval: 30s
      timeout: 5s
      retries: 3

networks:
  dsh-nas_dsh-net:
    external: true
  mc_default:
    external: true
```

> **为什么是两张网**：`dsh-nas_dsh-net` 上只有网关与 SSO，`mc_default` 上只有 MC。
> 本服务是唯一同时属于两张网的容器，这样**既不用把 MC 拉进平台网络，也不用给 MC 发布 RCON 端口**（RCON 是弱认证协议，绝不暴露公网）。
> 代价：`mc` 若被 `docker compose down` 重建，`mc_default` 会消失，本服务需重启（写进 §11 排错清单）。

### 3.4 服务在平台里的登记（`architecture.md` §7.8 第 6、7 步）

| # | 动作 | 落点 |
|---|---|---|
| 1 | 网关：新增 `/mc/*` 站点块（§6.3） | `dsh-nas/caddy/Caddyfile` + `/home/docker-admin/elicloud/mc-caddy-block.Caddyfile` |
| 2 | 服务登记进"平台服务清单" | 将来 `main-api` 的 `GET /core/v1/services` 加一行 `{"id":"mc","name":"MC 白名单","path":"/mc"}` |
| 3 | 前端入口 | 前端按 `online` 显示/置灰 |

### 3.5 配置与密钥三处归属（`deploy-framework.md` §2.4）

| 放什么 | 放哪 | 谁会读 |
|---|---|---|
| `RCON_PASSWORD` | 服务器 `/srv/mc/app.env`（600），由 `deploy.sh` 读出后以环境变量传给容器 | 只有本服务 |
| 其余运行时变量 | 同上 `app.env`（可版本化的部分才写进 compose） | 本服务 |
| 部署脚本 | 仓库 `deploy/deploy.sh`（版本管理）→ `scp` 覆盖 `/srv/mc/deploy.sh` | `deploy` 用户 |

**密码不得出现在**：Git 仓库、镜像层、compose 的 `environment:` 明文、日志、错误响应、本对话记录。

---

## 4. 数据模型（SQLite，TEXT + ISO8601，与 SSO 同风格）

```sql
-- 用户名 ↔ SSO 账号 的绑定；一行 = 一次申请
CREATE TABLE IF NOT EXISTS mc_names (
  id             TEXT PRIMARY KEY,            -- 如 mn_0001
  user_id        TEXT NOT NULL,               -- SSO 的 sub（不建外键：不跨库）
  name           TEXT NOT NULL,               -- 归一后的 MC 用户名（小写）
  name_display   TEXT NOT NULL,               -- 用户提交的原始写法，仅展示用
  note           TEXT NOT NULL,               -- 必填备注
  status         TEXT NOT NULL,               -- active / removed
  uuid           TEXT,                        -- 服务端写入白名单后的 UUID（只记录，不构造）
  request_ip     TEXT,
  user_agent     TEXT,
  created_at     TEXT NOT NULL,
  removed_at     TEXT,
  removed_by     TEXT                         -- self / admin / reconcile
);

-- 全局唯一：同一个 MC 用户名只能有一个 active 绑定（大小写已归一）
CREATE UNIQUE INDEX IF NOT EXISTS ux_mc_names_active_name
  ON mc_names(name) WHERE status = 'active';
-- 每个账号的 active 数量由应用层按 MAX_NAMES_PER_USER 校验（有测试锁住）

-- 全量审计：每一次尝试都留痕，包括失败
CREATE TABLE IF NOT EXISTS audit_logs (
  id           TEXT PRIMARY KEY,
  user_id      TEXT,                          -- 失败且未认证时为空
  action       TEXT NOT NULL,                 -- add / remove / reconcile / deny
  target       TEXT,                          -- MC 用户名
  result       TEXT NOT NULL,                 -- ok / conflict / quota_exceeded / rcon_error / ...
  detail       TEXT,                          -- JSON：RCON 原文、错误原因（不含令牌）
  request_ip   TEXT,
  created_at   TEXT NOT NULL
);

-- 白名单快照：用于对账（发现"库里以为加了、白名单里没有"的漂移）
CREATE TABLE IF NOT EXISTS whitelist_cache (
  name        TEXT PRIMARY KEY,               -- 小写
  uuid        TEXT,
  synced_at   TEXT NOT NULL
);
```

**约束落点**：
- 「最多 2 个」= `COUNT(*) WHERE user_id=? AND status='active' < MAX_NAMES_PER_USER`，**在写 RCON 前判定**，并且写库与写 RCON 之间有锁（§7）。
- 「全局唯一」由**部分唯一索引**兜底（并发下也不会写进两条 active），应用层再返回 `409` 而不是 `500`。

---

## 5. RCON 管道（**本节全部来自线上实测**，见 §9.2）

### 5.1 协议要点

- Minecraft RCON（Valve RCON）：`int32 长度 | int32 请求 id | int32 类型 | body | 0x00 0x00`，小端。
- 类型：`3=AUTH`、`2=EXECCOMMAND`、`0=RESPONSE_VALUE`。
- 认证失败时服务端回 **`id = -1`**（不是关闭连接）。

### 5.2 ✅ 实测到的四个必须照做的行为

| # | 行为 | 证据 |
|---|---|---|
| 1 | **一条连接只跑一条命令**：MC 在响应后**主动关闭连接**，同一连接里连发第二条命令会读到 EOF | 探针在同一连接里连发命令时收到 `RCON 连接被对方关闭`；改成一条命令一条连接后 7 条命令全部正常 |
| 2 | **每条命令服务端先回一个空包，再回真实内容**（两个包都可能带同一个 id） | 原始单帧探针：认证返回 `len=10 id=1 type=2 body=""` |
| 3 | **用户名一律被转成小写**：提交 `ProbeTester` → 响应 `Added probetester to the whitelist`，`whitelist.json` 里存 `probetester` | ✅ 见 §9.2 |
| 4 | **离线模式下 UUID 形态是变体**：`whitelist add ProbeTester` 写入的 UUID 是 `d36ee00d-f656-3501-a868-760cf20eb1d5`（**v5**），另一个用户是 `dc56f978-88d2-4d88-9279-84e7af2ea3de`（**v4，随机**）。**不是** `ops.json` 里那种 v3 离线 UUID | ✅ 见 §9.2 与 §12.1 |

> 结论 3 决定：**服务内部一律用小写做键**，展示时才回显用户原始写法。
> 结论 4 决定：**绝不自己构造 UUID**（决策 8 的实证依据）——服务器按它自己的缓存状态决定写 v5 还是随机 v4，服务无从预测，硬拼只会写出"看起来对但不生效"的条目。

### 5.3 ✅ 实测的命令响应文本（服务的判定依据）

| 命令 | 响应 | 服务应如何解释 |
|---|---|---|
| `whitelist list`（空） | `There are no whitelisted players` | 白名单为空 |
| `whitelist list`（有） | `There are 1 whitelisted player(s): probetester` | 解析出名字列表；**这是唯一可信的真源** |
| `whitelist add ProbeTester` | `Added probetester to the whitelist` | 成功 |
| `whitelist add Probetester`（已存在） | `Player is already whitelisted` | **幂等**：当成成功，不重复写库 |
| `whitelist add NonexistentGuy` | `Added Nonexistentguy to the whitelist` | ⚠️ **离线模式下不校验账号是否存在**，任意字符串都会"成功" → 输入必须靠本地正则严格约束（§5.4） |
| `whitelist remove probetester` | `Removed probetester from the whitelist` | 成功 |

> **判定原则：不信 add/remove 的回执，以 `whitelist list` 为准。**
> 流程固定为：`list`（前置检查）→ `add`/`remove` → `list`（回读确认）→ 落库 → 审计。
> 回执文本只用于日志与排错，不参与正确性判断。

### 5.4 输入约束（防注入是硬要求）

```text
name 必须匹配 ^[A-Za-z0-9_]{3,16}$        # MC 官方用户名规则
note 必须为 1–200 字符，禁止控制字符
```

**绝不拼接命令字符串**：虽然 RCON 的 body 是整条控制台命令，名字也必须先过正则；
不满足即 `400`，**不要把用户输入塞进命令里再靠转义兜底**（转义迟早出错，正则只有一次机会）。
备注**不进任何 RCON 命令**。

### 5.5 连接与超时

| 参数 | 默认 | 说明 |
|---|---|---|
| `RCON_TIMEOUT` | 5s | 建连 + 读写各自的超时 |
| 重试 | **0 次**（写操作不自动重试） | 写入是否成功必须由回读确认，盲目重试会造成状态歧义 |
| 串行化 | 进程内锁，一时刻只有一条 RCON 命令 | MC 的 RCON 监听本身是串行的，并发只会排队+超时 |

### 5.6 与「服务端行为」的分工

- 服务**只发名字**，UUID 由服务端决定并落盘；服务在回读 `whitelist list` 之后，另读一次 `whitelist.json`（或从 `list` 结果）把 UUID 记进 `mc_names.uuid` **作为记录**，不做任何推断。
- 若要把 UUID 当作判定依据，必须先证明服务端行为稳定 —— 当前实测已表明它**不稳定**（v5 与随机 v4 并存），所以**不采用**。

### 5.7 健康检查语义

`GET /healthz` 返回：

```json
{ "status": "ok", "rcon": { "reachable": true, "latency_ms": 12 }, "whitelist_count": 3 }
```

- RCON 不可达 → **HTTP 503**（让网关与运维立刻看见），但服务进程保持运行、只读接口（如查自己的条目）仍可用。
- 这么定的理由：`architecture.md` §7.9 第 5 条要求 healthcheck 能反映"容器在跑但服务已挂"；对本服务，最可能的故障模式恰恰是 RCON 断了。

---

## 6. API 契约

**约定**：服务内路径；对外加 `/mc` 前缀（网关剥离，见 §6.3）。错误结构沿用平台风格：
`{"error": "...", "error_description": "..."}`，状态码 `400/401/403/404/409/429/503`。
日志与响应**绝不出现令牌原文**。

### 6.1 鉴权（所有 `/v1/*`）

| 校验 | 规则 | 失败 |
|---|---|---|
| 存在性 | `Authorization: Bearer <jwt>` | `401 invalid_token` |
| 签名 | 用 JWKS（`kid` 匹配，缓存尊重 `Cache-Control`）验签，**算法白名单只有 RS256** | `401 invalid_token` |
| `iss` | 必须逐字等于 `SSO_ISSUER` | `401 invalid_token` |
| `aud` | 必须等于 `JWT_AUDIENCE`（`elicloud-services`）→ **`id_token` 的 `aud` 是 client_id，会在这里被拒** | `401 invalid_token` |
| `exp`/`nbf` | 标准校验，容忍 30s 时钟偏移 | `401 invalid_token` |
| `scope` | **必须含 `mc:whitelist`**（空格分隔的 scope 串里整词匹配，与 SSO `validate_scope_subset` 同语义） | `403 insufficient_scope` |

**scope 要求（已按 T1 定死为「必须含」）**：
- `REQUIRED_SCOPE=mc:whitelist`，默认**非空**；配成空串等于关闭校验，只允许在本地调试时这样做。
- ⚠️ **这是一个跨服务前置条件**：`mc:whitelist` 目前 SSO 里不存在，必须先在 `sso/` 加上并部署，见 **§12.4**。
  在 SSO 加完之前，本服务对任何令牌都会回 `403 insufficient_scope` —— 这是**刻意**的 fail closed，
  免得出现「服务能调但权限模型是假的」。
- 失败用 **403**（不是 401）：身份是有效的，缺的是权限，语义上更准确，也便于调用方区分「重新登录」与「申请授权」。

### 6.2 端点

#### `POST /v1/names` — 提交一个 MC 用户名

```http
POST /mc/names
Authorization: Bearer <access_token>
Content-Type: application/json

{ "name": "ProbeTester", "note": "帮朋友申请，主号被封禁" }
```

```json
// 201：白名单已确认写入
{
  "id": "mn_0007",
  "name": "probetester",
  "name_display": "ProbeTester",
  "note": "帮朋友申请，主号被封禁",
  "status": "active",
  "uuid": "d36ee00d-f656-3501-a868-760cf20eb1d5",
  "created_at": "2026-10-06T00:50:00Z"
}
```

处理顺序（**每一步的失败都必须可区分**）：

```text
1. 验签 → 取 sub
2. 校验 scope 必须含 mc:whitelist                 失败 → 403 insufficient_scope
3. 校验 name 正则 / note 必填与长度                失败 → 400 invalid_request
4. name 归一为小写；查库/查白名单：
   - 库里 active 已被本 sub 占用                   → 200（幂等，直接返回已有条目，不重复调 RCON）
   - 库里 active 属于别人                           → 409 name_taken
   - 白名单里已存在但库里没有（对账缺口）            → 409 name_taken（提示联系管理员）
5. 名额：本 sub 的 active 数 >= MAX_NAMES_PER_USER  失败 → 409 quota_exceeded
6. whitelist list（前置确认同一把锁内）
7. whitelist add <归一后的名字>
8. whitelist list 回读确认名字在列表里             失败 → 503 rcon_error，不落库
9. 落库 + 审计（action=add, result=ok）
```

#### `GET /v1/names` — 我的条目与名额

```json
{
  "sub": "user_0001",
  "quota": { "limit": 2, "used": 1, "remaining": 1 },
  "names": [ { "id": "mn_0007", "name": "probetester", "name_display": "ProbeTester",
               "note": "...", "status": "active", "created_at": "..." } ]
}
```

默认只返回 `active`；`?include_removed=true` 时带上历史（含 `removed_at`）。

#### `DELETE /v1/names/{id}` — 撤回

```text
1. 验签 → sub
2. 条目必须存在、属于该 sub（否则 404 —— 不泄露他人资源是否存在，沿用 §8 约定）
3. 若已 removed → 200 幂等
4. whitelist remove <name> → whitelist list 回读确认已移除
5. 落库 status=removed + 审计（action=remove, removed_by=self）
```

```json
{ "id": "mn_0007", "name": "probetester", "status": "removed", "removed_at": "2026-10-06T01:00:00Z" }
```

#### `GET /v1/me` — 当前身份

```json
{ "sub": "user_0001", "username": "alice", "issuer": "https://146.56.237.33/auth" }
```
用于前端登录后自检（`username` 来自令牌 claim，**不查 SSO 数据库**）。

#### `GET /healthz` — 见 §5.7（不经网关也可访问，供容器内 healthcheck）

#### 管理端点（`ADMIN_TOKEN` 保护，fail closed：未配置时整组 403）

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/v1/admin/names` | 列出全部绑定（含 `sub`、备注、时间、UUID） |
| GET | `/v1/admin/audit` | 审计日志（分页、可按 `user_id`/`action` 过滤） |
| DELETE | `/v1/admin/names/{id}` | 管理员强制移除（`removed_by=admin`） |
| POST | `/v1/admin/reconcile` | 用 `whitelist list` 与库对账，输出并修复漂移（仅报告 + 可选 `apply=true`） |

### 6.3 网关（Caddyfile 片段，**独立站点块**）

```caddyfile
# 追加于 /home/docker-admin/dsh-nas/caddy/Caddyfile 末尾
# 为什么独立站点块：面板站点块有无 matcher 的 basic_auth 兜底指令，会把并进来的 /mc/* 也拦下
# （同一个坑在 SSO 上踩过一次，见 docs/sso-oidc.md §8 的实测补充）
https://146.56.237.33/mc/* {
	tls { issuer acme { profile shortlived } }
	log
	handle /mc/* {
		uri strip_prefix /mc
		reverse_proxy mc:8000
	}
}
```

映射（纯剥离，服务内就是 `/v1/...`，不再 rewrite）：

```text
POST /mc/v1/names     -> POST /v1/names
GET  /mc/healthz      -> GET  /healthz
```

**改动网关的铁律**（`sso/README.md` §4.3 已经踩过）：先 `caddy validate`，再 `caddy reload`；
绝不用 `docker restart dsh-caddy`（配置写错会进崩溃循环、公网全 502）。

### 6.4 错误码总表

| HTTP | `error` | 触发 |
|---|---|---|
| 400 | `invalid_request` | name 不符合 `^[A-Za-z0-9_]{3,16}$`；note 缺失/超长；JSON 含未知字段 |
| 401 | `invalid_token` | 令牌缺失/过期/签名错/iss 或 aud 不符（含误用 `id_token`） |
| 403 | `forbidden` | 管理端点未配置 `ADMIN_TOKEN` 或令牌错误 |
| 403 | `insufficient_scope` | 令牌有效但 scope 里没有 `mc:whitelist`（T1） |
| 404 | `not_found` | 条目不存在或不属于当前 `sub` |
| 409 | `name_taken` | 该 MC 用户名已被别的账号绑定 |
| 409 | `quota_exceeded` | 本账号已达 2 个上限 |
| 429 | `rate_limited` | 触发限流（带 `Retry-After`） |
| 503 | `rcon_unavailable` | RCON 不可达 / 回读未确认 |

---

## 7. 并发、幂等与一致性

| 场景 | 处理 |
|---|---|
| 同一账号并发提交同一个名字 | 进程内锁 + 部分唯一索引；一个成功、另一个走幂等分支返回同一条目 |
| 两个账号并发抢同一个名字 | 部分唯一索引保证只有一个 active；另一个 `409 name_taken` |
| MC 在 add 与回读之间挂掉 | 回读失败 → **不落库**，返回 `503`；审计记 `rcon_error`，真实状态交由对账 |
| 库里 active 但白名单里没有（漂移） | `POST /v1/admin/reconcile` 检测并修复；`whitelist_cache` 只做辅助，**真源永远是 MC 的 `whitelist list`** |
| 白名单里有、库里没有（OP 手动加的） | 视为「已被占用」，申请返回 `409 name_taken`，提示联系管理员；**不自动删除**（避免误删别人手工加的白名单） |
| 撤回时 RCON 挂了 | 回读不确认 → **不改库状态**，返回 `503`（宁可状态不动，也不要出现"库里已移除、白名单里还在"） |

---

## 8. 上线与启用白名单（MC 侧唯一需要改的东西）

### 8.1 MC 侧改动（在 `/home/docker-admin/mc/`）

itzg 镜像的 `server.properties` **由环境变量生成**（改宿主机那份文件没有用），正确改法：

```yaml
# docker-compose.yml 的 mc 服务，environment 段追加：
      WHITELIST: "TRUE"
      ENFORCE_WHITELIST: "TRUE"
      # RCON 已经是开的（RCON_PASSWORD 存在即自动 enable-rcon=true），无需改动
```

**后续动作**（敏感操作，逐条确认后执行）：

```bash
docker compose up -d mc          # 重建容器使环境变量生效
docker exec urania-mc grep -E '^white-list|^enforce-whitelist' /data/server.properties
# 期望：white-list=true / enforce-whitelist=true
```

**影响面（已确认接受）**：
- 生效瞬间，**不在白名单里的非 OP 玩家会被踢出且无法进服**。当前处于停服维护状态，故不预加。
- OP（`ops.json` 中 8 人）不受影响。

### 8.2 服务上线顺序（幂等、可回滚）

```text
1. 服务器建目录 /home/docker-admin/elicloud/mc/{data}，data 属主 1002:1003
2. scp 代码与 docker-compose.yml → 该目录
3. 写 /srv/mc/app.env（600，只含 RCON_PASSWORD 等密钥），scp deploy/deploy.sh → /srv/mc/
4. docker compose up -d --build
5. docker compose ps（应为 healthy）
6. 网关：追加 mc 站点块 → caddy validate → caddy reload
7. 验收：§9.3 的清单
```

**回滚**：`docker compose stop mc` → 网关块注释掉 → `caddy validate && reload`。
白名单是**附加式**的，停服不停白名单；要放弃已加的白名单用管理端点或 `POST /v1/admin/reconcile`。

### 8.3 与部署框架的衔接

按 `deploy-framework.md`：`deploy/deploy.sh <ref>` 放在仓库版本管理，`scp` 覆盖到 `/srv/mc/deploy.sh`，由 `deploy` 用户执行。
**但注意本机的两个既有约束**：
1. **snap Docker 看不到 `/srv`** → compose 的 bind mount 与构建上下文必须用 `/home/docker-admin/elicloud/mc`，`/srv` 只放 `deploy.sh` 与 `app.env`；
2. `deploy.sh` 里用 `--env-file` 传 `/srv/...` 同样会失败 → 应由 shell 读出 `app.env` 再用环境变量传给 `docker compose`。

---

## 9. 验收（Definition of Done）

### 9.1 单元 / 契约测试（一条命令跑通，不依赖真实 MC）

用一个**假 RCON 服务器**（本地 socket，复刻 §5.2 的四条行为：响应后关连接、先空包后内容、用户名转小写、`Player is already whitelisted`）覆盖：

- [ ] 无令牌 / 过期令牌 / 错签名 / 错 `iss` / 错 `aud` → 401；**`id_token` 当 access token 用 → 401**
- [ ] **scope 里没有 `mc:whitelist` → 403 `insufficient_scope`**（含「有其它 scope 但没有它」与「scope 为空」两种）
- [ ] `REQUIRED_SCOPE=""`（显式关闭）时同一令牌可以正常调用 —— 锁住这个调试开关的行为
- [ ] name 正则：`ab`（太短）、`名字带中文`、`a b`、`a;rm -rf`、17 字符 → 全部 400
- [ ] note 缺失 / 201 字符 → 400
- [ ] 首次提交 → 201，且假 RCON 收到恰好 `list` + `add` + `list` 三条命令
- [ ] 重复提交同名 → 幂等，不产生第二次 `add`
- [ ] 第 3 个名字 → 409 `quota_exceeded`；删一个后可以再加
- [ ] 别人的名字 → 409 `name_taken`；自己删别人的条目 → 404
- [ ] 大小写：提交 `ProbeTester` 与 `probetester` 视为同一个名字
- [ ] RCON 挂掉 → 503，且**库里没有"已生效"的记录**
- [ ] 撤回后可以立即重新申请同名（冷却期不在本期范围，见 T3）
- [ ] 审计：成功与失败都有 `audit_logs` 行；日志里**不含令牌原文**

### 9.2 ✅ 已完成的线上实测（本次已做，证据在此）

**方法**：不安装任何新镜像（镜像源无 `itzg/mcrcon`），用一个 ~50 行的 Python RCON 探针，
从 **MC 容器内部**连自己的 `urania-mc:25575` 执行命令；密码从 600 权限的文件读取，不进命令行、不进日志。

| 实测命令 | 实际响应 |
|---|---|
| `whitelist list`（初始） | `There are no whitelisted players` |
| `whitelist add ProbeTester` | `Added probetester to the whitelist` |
| `whitelist list` | `There are 1 whitelisted player(s): probetester` |
| `whitelist add Probetester` | `Player is already whitelisted` |
| `whitelist add NonexistentGuy` | `Added Nonexistentguy to the whitelist` ← 离线模式不校验账号存在 |
| `whitelist remove probetester` | `Removed probetester from the whitelist` |
| `whitelist list`（最终） | `There are no whitelisted players` |

`/data/whitelist.json` 落盘形态（解释见 §5.2 结论 4）：

```json
[ { "uuid": "d36ee00d-f656-3501-a868-760cf20eb1d5", "name": "probetester" } ]   // add ProbeTester 的结果（v5）
[ { "uuid": "dc56f978-88d2-4d88-9279-84e7af2ea3de", "name": "Nonexistentguy" } ] // 另一个（v4，随机）
```

**收尾状态（已核对）**：`whitelist.json` 已回到 `[]`；
`server.properties` 的 `white-list` / `enforce-whitelist` / `online-mode` / `enable-rcon` **全部保持原值未动**；
`urania-mc` 从"已停止"被我启动为运行中（需求方同意，停服维护状态），**world 数据未做任何改动**。

### 9.3 部署后线上冒烟（部署完成后做，打公网 `/mc`）

- [ ] `GET /mc/healthz` → 200 且 `rcon.reachable=true`
- [ ] 取一个真实 SSO access token：`POST /mc/v1/names` → 201；MC 内 `whitelist list` 能看到该名字
- [ ] `GET /mc/v1/names` → 列出 1 条、`quota.remaining=1`
- [ ] 第 3 个名字 → 409 `quota_exceeded`
- [ ] `DELETE /mc/v1/names/{id}` → MC 内 `whitelist list` 里该名字消失
- [ ] 用一个**别人已占用**的名字 → 409 `name_taken`
- [ ] `id_token` 冒充 access token → 401
- [ ] 网关：`/mc/v1/names` 无令牌 → 401（不是 404，证明路由通）；容器无公网端口
- [ ] 日志中 grep 令牌原文 → 零命中

### 9.4 白名单真正生效的验收（与 §8.1 同一次操作）

- [ ] 启用 `WHITELIST=TRUE`/`ENFORCE_WHITELIST=TRUE` 并重建 MC 后，`white-list=true`
- [ ] 一个**不在白名单**的测试账号被拒绝进入（`You are not whitelisted on this server!`）
- [ ] 服务加白之后，同一个账号可以进入
- [ ] OP 账号始终可进（不受白名单影响）

---

## 10. 安全清单（逐条自查）

| # | 项 | 做法 |
|---|---|---|
| 1 | 命令注入 | 用户名过 `^[A-Za-z0-9_]{3,16}$`，备注不进 RCON；**不拼接、不转义依赖** |
| 2 | 越权（IDOR） | `DELETE` 只允许 `user_id == sub`；不属于自己一律 404（不泄露存在性） |
| 3 | 身份伪造 | RS256 白名单；`iss`/`aud` 逐字校验；拒绝 `alg:none`/`HS256`；拒绝 `id_token` |
| 4 | 密钥泄漏 | JWKS 只读公钥；RCON 密码只在服务器 `app.env`，不进仓库/镜像/日志/响应 |
| 5 | 日志安全 | 只记 `sub`、`name`、结果、耗时；**绝不记令牌原文与备注全文**？备注可按需记录（管理员可见是需求），但不进第三方日志 |
| 6 | 限流 | 提交/撤回按 `sub` + IP 限流（如 10 次/10 分钟），防脚本刷白名单 |
| 7 | 网络暴露 | 容器**不发布端口**；RCON **不发布端口**（只有 `mc_default` 内可达） |
| 8 | 内部接口 | 无 Swagger/`/docs` 暴露到公网，或至少不泄露内部错误细节 |
| 9 | 请求体 | 未知字段直接 400（`extra=forbid`），不静默忽略 |
| 10 | 审计不可抵赖 | 每次成功/失败都写 `audit_logs`（含 IP、UA、RCON 原文） |
| 11 | 拒绝服务 | RCON 串行 + 超时；避免请求把 MC 的 RCON 打满 |
| 12 | 备注内容 | 长度上限 + 控制字符过滤；渲染侧（未来前端）需转义 |

---

## 11. 运维与排错

```bash
cd /home/docker-admin/elicloud/mc
docker compose ps
docker compose logs -f mc
curl -s http://127.0.0.1:8000/healthz            # 需在 compose 里保留 127.0.0.1:8000 调试映射，或 docker exec
docker compose exec mc python -m app.cli list-names
docker compose exec mc python -m app.cli reconcile
```

| 现象 | 排查顺序 |
|---|---|
| 所有请求 401 | 比对服务 `SSO_ISSUER` 与 SSO `PUBLIC_BASE_URL` 是否逐字相同；确认客户端送的是 access token 而不是 id_token |
| `/mc/*` 返回 404 | 网关站点块是否生效（`caddy validate` / `reload`）；站点块地址是否也带 `/mc/*` 路径 |
| `/healthz` 报 `rcon.reachable=false` | MC 容器是否在跑；`mc_default` 网络是否还存在（`mc` 被 `compose down` 重建会让它消失 → 重启本服务）；容器是否同时加入了两张网 |
| `whitelist add` 成功但玩家进不去 | 白名单是否真的启用（`white-list`/`enforce-whitelist`）；UUID=随机 v4 的行不生效（§12.1） |
| 库里说加了、白名单里没有 | `POST /v1/admin/reconcile`；`audit_logs` 里找该条的 RCON 原文与 `result` |
| 重建 MC 容器后服务连不上 RCON | 网络重建导致；`docker compose up -d --force-recreate mc` |

**备份**：`./data/mc.db`（绑定关系 + 审计）。白名单本身由 MC 侧备份（`itzg/mc-backup`，已有 5GB/12h 的历史备份）。

---

## 12. 已知限制与待确认项

### 12.1 ⚠️ 已知限制一：离线模式下的 UUID 语义

实测表明 `online-mode=false` 时，`whitelist add` 写入的 UUID **不一定是离线（v3）UUID**：
未登录过的用户名拿到**随机 v4**，另一条拿到 v5。这带来两个后果：

1. 随机 v4 的行**可能不生效**（客户端自报 UUID 无法匹配）；
2. **一旦有人把 `online-mode` 改回 `true`，整个白名单文件都要重建** —— 现在里面很可能混着 v4/v5/离线各自形态的行。

**应对**：本文档不解决它（属于 MC 侧运维决策），但**明确记录**，并在 §9.4 的验收里把「加白后真的能进服」列为必测项 —— 如果届时发现随机 v4 的行不生效，正确的修法是让玩家先登录一次（建立 name→UUID 缓存）再加白，而不是让本服务去猜 UUID。

### 12.2 已确认项（2026-10-06 全部拍板，原「待确认项」）

| # | 事项 | **定值** | 落地要点 |
|---|---|---|---|
| T1 | 令牌是否必须含 `mc:whitelist` scope | **必须含**（硬要求，不再做「可配开关」的默认关） | ⚠️ **跨服务前置条件**：该 scope 目前 SSO 里不存在，必须先在 `sso/` 加上（§12.4），否则服务上线即全员 403 |
| T2 | 管理面 | **容器内 CLI + HTTP 管理接口都要** | HTTP 侧用 `ADMIN_TOKEN` Bearer；未配置时整组 403（fail closed），沿用 SSO §13 的做法 |
| T3 | 撤回后的冷却期 | **本期不加** | 防刷只靠限流（`sub` + IP） |
| T4 | 提交后通知管理员 | **本期不做** | 备注必填 + 全量审计已覆盖「事后可查」 |
| T5 | 多 MC 服务器 | **本期只服务 `urania-mc`** | 表结构将来加 `server_id` 演进；届时「全局唯一」变成「每服务器唯一」 |
| T6 | 对外路径前缀 | **定死 `/mc`** | 网关站点块、`CORS_ORIGINS`、前端与文档全部按 `/mc` 写 |
| T7 | 部署方式 | **先手工部署**，`deploy/deploy.sh` 按框架写 | 交付物含 Dockerfile + compose + `deploy.sh` + README；CI/CD 验收通过后再补 |
| T8 | 备注在管理接口的可见性 | **管理员可见全文** | 只出现在 `GET /v1/admin/names` 与审计里；不进日志明文、不进 RCON |

### 12.3 本期明确不做

交互式同意页、提交后通知、多 MC 服务器、撤回冷却期、用户自服务前端、正版 UUID 校验、白名单容量分析、Prometheus 指标、CI/CD 流水线（T7 后补）。

### 12.4 ⚠️ T1 带来的跨服务前置条件：给 SSO 加 `mc:whitelist` scope

`scope` 的单一真源在 SSO（`sso/app/constants.py::SUPPORTED_SCOPES`），客户端与令牌都从那里派生，
**本服务无法自行引入一个新 scope** —— 不改 SSO 就要求它，结果是所有令牌都验证失败。需要动 4 处：

| # | 文件 | 改动 | 为什么必须改 |
|---|---|---|---|
| 1 | `sso/app/constants.py` | 加 `SCOPE_MC_WHITELIST = "mc:whitelist"` 并加入 `SUPPORTED_SCOPES` | `SUPPORTED_SCOPES` 是发现文档 `scopes_supported`、`/authorize` 与 `/token` 的校验源；不在里面，客户端请求该 scope 会被 `invalid_scope` 拒掉 |
| 2 | `sso/app/config.py` | `default_scope` 追加 `mc:whitelist` | 密码直连 `/v1/login`（`routers/auth.py:184`）与无 scope 场景（`security.py:147`）都用它；不追加则新签发的令牌里没有该 scope |
| 3 | `sso/app/device.py` | `SCOPE_LABELS` 加 `"mc:whitelist": "MC 白名单：提交游戏名"` | 设备流程确认页要显示中文说明，漏了会显示原始 scope 串 |
| 4 | `sso/tests/conftest.py` | 测试用 `DEFAULT_SCOPE` 同步追加 | 否则测试环境与线上配置漂移（现有断言用 `settings.default_scope`，不会红，但覆盖不到新 scope） |

**部署后的两个副作用（必须先告知用户，不是 bug）**：
1. **现有 access token 里没有该 scope**，改完 SSO 后用户需**重新登录**（拿新令牌）才能调用本服务；
2. 走**授权码流程**的前端，除了要注册客户端时允许该 scope，**请求时也必须显式带上** `scope=... mc:whitelist`。

**建议的实施顺序**（避免出现「服务上线了但没人能调」的窗口）：
```text
1. 改 SSO 4 处 → 跑 sso 的 pytest → 部署 SSO → 给前端客户端 PATCH allowed_scopes 加上 mc:whitelist
2. 验收：取一个新令牌，解出 scope 确认含 mc:whitelist
3. 再上线 mc 服务
4. 反例验收：拿一个不含该 scope 的旧令牌调用 → 必须 403
```

#### ✅ 已落地（2026-10-06，SSO 侧全部完成）

| 项 | 结果 |
|---|---|
| 代码改动 | 上表 4 处 + **新增 2 条测试**（`test_mc_whitelist_scope_is_declared_and_accepted`、`test_default_scope_covers_mc_whitelist`） |
| 测试 | 改前 **172 passed**，改后 **174 passed**（在 sso 容器的只读挂载下跑，未污染仓库） |
| 部署（第一轮，09:04） | `docker compose up -d --build`，本地镜像 `elicloud-sso:1.0.0` 由 `041a94f52672` → `b91359d0c866`；容器 `healthy`；**issuer 未变** |
| 线上验收（第一轮） | 发现文档 `scopes_supported` 已含 `mc:whitelist`；`/authorize` 用 `scope=openid profile email mc:whitelist` 能渲染登录页；不支持的 scope 仍 `302 invalid_scope`；public 客户端缺 PKCE 仍被拒 |
| 客户端 | 线上**已有 3 个客户端**（`elipese-web`、`elipese-app`、`elicloud-cli`）已全部追加 `mc:whitelist` 到 `allowed_scopes` |

##### ⚠️ 第一轮验收漏掉的一层：`DEFAULT_SCOPE` 有**第二个真源**

第一轮只验到「发现文档 / 授权端点 / 客户端 `allowed_scopes`」都对了，
**没有验「签出来的令牌里到底有没有该 scope」**。做 MC 白名单服务的端到端验收时才暴露：

```text
线上令牌实际 scope：openid profile email pdf:read pdf:write        ← 没有 mc:whitelist
```

根因链条（**本项目最容易漏的一处**）：

1. 线上 SSO 是 `DEPLOY_MODE=image`：跑的是 `ghcr.io/elicloudorg/sso:prod`，
   容器环境变量由服务器的 **`/srv/sso/deploy.sh`** 生成，而它由仓库的 **`deploy/deploy.sh`** 同步覆盖；
2. `deploy/deploy.sh` 内嵌的那份 compose 里写死了
   `DEFAULT_SCOPE: "openid profile email pdf:read pdf:write"`；
3. 环境变量**覆盖** `app/config.py` 的 `default_scope`，于是代码里的 `mc:whitelist` 被压掉。

**结论：`scope` 有三个位置要改，不是两个。**

| # | 位置 | 作用 |
|---|---|---|
| 1 | `app/constants.py::SUPPORTED_SCOPES` | 平台是否"声明/接受"该 scope（发现文档 + `/authorize` + `/token` 校验） |
| 2 | `app/config.py::default_scope` | 代码默认值 |
| 3 | **`deploy/deploy.sh` 内嵌 compose 的 `DEFAULT_SCOPE`**（以及 `docker-compose.yml`） | **实际生效值**（环境变量覆盖代码默认） |

修复：三处同步补上 + 加注释说明这是易漏的第二真源，走 PR 合并到 `prod` 触发 `deploy-prod.yml` 重建镜像。

> **教训（写进验收清单）**：验证 scope 必须验到**令牌原文里的 `scope` claim**，
> 只验发现文档与客户端配置会漏；而"用户能调通业务服务"才是最终真源。

> ⚠️ **将来新建客户端时别忘了**：`create-client --scope` 里必须带 `mc:whitelist`，
> 否则那个客户端请求该 scope 会在 `/authorize` 被 `invalid_scope` 拒掉（`allowed_scopes` 是硬上限，
> `SUPPORTED_SCOPES` 只是"平台支持"）。已加的两个验证脚本可复用：
> `.pytest-runner/verify-deploy.py`（发现文档 + 客户端 scope）与
> `.pytest-runner/verify-authorize-scope.py`（授权端点端到端）。

> 📌 **仍待做的两件（不在 SSO 侧）**：① 用户需重新登录一次以拿到带 `mc:whitelist` 的令牌；
> ② ~~`mc` 服务本身尚未写码~~ → **代码已于 2026-10-06 完成**（见 §12.5），下一步是部署。

### 12.5 ✅ 已落地（2026-10-06，mc 服务侧）

| 项 | 结果 |
|---|---|
| 代码 | `mc/`：`app/` 十个模块（`config` / `db` / `models` / `errors` / `sso_auth` / `rcon` / `whitelist` / `deps` / `cli` + `routers/{names,ops}`），加 `Dockerfile` / `docker-compose.yml` / `.env.example` / `README.md` |
| 测试 | **92 passed**（`pip install -r requirements-dev.txt && pytest -q`）：§9.1 清单逐条覆盖。用一个**假 RCON 服务器**复刻 §5.2 的四条实测行为，并额外提供 `lying_add` 故障注入 —— 专门证明「不信 `add` 回执、以 `whitelist list` 为准」；另有一个真实 HTTP 的假 JWKS 服务，走**真实验签路径** |
| 部署脚本 | `deploy/deploy.sh`（框架形状）：`/srv` 读 `app.env` → `/home` 写 `.env`（snap Docker 读不到 `/srv`）→ `compose build && up -d` → 健康检查（含 RCON 可达性）→ 回环自检 |
| 部署 | ✅ **已上线**（2026-10-06）：镜像 `elicloud-mc:1.0.0`，容器 `mc`，`healthy`；同时接入 `dsh-nas_dsh-net`（网关反代）与 `mc_default`（RCON），仅 `127.0.0.1:8001` 回环映射；`GET /mc/healthz` → `200` 且 `rcon.reachable=true`（实测 1~3ms） |
| 网关 | ✅ 已追加 `/mc/*` 独立站点块到 `dsh-nas/caddy/Caddyfile`：`caddy validate` = `Valid configuration` → `caddy reload` 成功；回归无影响（`/auth/*` 与面板 `401` 均正常）。注意 `ubuntu` 账号读不到 `/home/docker-admin`（750），追加/备份必须在 `sudo -n sh -c` 里做重定向 |
| 部署方式 | 因本机 `docker-admin` **没有免密 sudo**（只有 `ubuntu` 有），`/srv/mc/deploy.sh` 那条路走不通，改为等价的直接部署（构建镜像 → `docker run` → 接第二张网）。`deploy/deploy.sh` 仍按框架写好，等 CI/CD 接入时启用 |
| 端到端验收 | 真令牌走通：未认证 401、输入校验与注入尝试 400、首次 201、重复 200 幂等、第三个名字 409 `quota_exceeded`、`GET /v1/names` 名额视图、`DELETE` 后本人条目清空、不存在的条目 404、管理接口 403→200、**`id_token` 冒充 access token → 401 而同一流程的 access token → 200**、日志无 JWT 原文与 RCON 密码 |
| **真源证据链**（最关键的一条） | 加白 → `POST /mc/v1/names` = 201；**MC 侧 `whitelist list` 显示 `mcwtruthcheck`**；撤回 → MC 侧变为 `[]`；名额回到 0。证明"自动纳入白名单"是真的落到 MC 服务端，而不是只改了库 |
| ⚠️ 验收抓到的问题 1 | 令牌里**没有** `mc:whitelist` → 所有业务请求 403。根因见 §12.4 的「第二个真源」：`deploy/deploy.sh` 内嵌 compose 写死了旧 `DEFAULT_SCOPE`。已修（PR `EliCloudOrg/sso#1` 合并到 prod），修复后**令牌原文里已含 `mc:whitelist`** |
| ⚠️ 验收抓到的问题 2 | `ghcr.io` 拉取**卡死**（`docker pull` 挂 10 分钟无进展，日志一次都没打印）→ `deploy-prod.yml` 的「Trigger remote deploy」整步卡住。因本次提交只改 compose/deploy.sh（**镜像层没变**），改用已生成的 compose 直接重建容器即完成部署。这是 `deploy-framework.md` §7.10 已记录的环境约束（ghcr.io 在部分网络不可达）的又一次实测 |
| ⚠️ 验收脚本自身的坑（记录下来免得重踩） | ① 网关对**默认 Python UA**（`Python-urllib/3.x`）返回 `401 Basic realm="restricted"`——请求必须带显式 `User-Agent`；② SSO 的 CSRF 是**双重提交**，POST 表单必须带 cookie（要用 `HTTPCookieProcessor`），否则 403「表单已失效」；③ URL 少写 `/auth` 前缀会掉进面板站点块（401）；④ `deploy.sh` 之类含 `$(...)` 的命令会触发敏感命令拦截 |
| ⚠️ 验收中发现并修掉的配置不一致 | `.env` 的 `ADMIN_TOKEN` 曾被写坏成 `ADMIN_TOKEN=ADMIN_TOKEN=<hex>`（76 字符），**磁盘与运行容器不一致**（下次重建就会丢管理令牌）。已以"容器内实际生效的 64 字符令牌"为真源回写 `.env`，两侧 `sha256[:8]` 现已一致（`3ca0bb6b`） |
| 未做 | ~~CI/CD（T7「先手工部署」）~~ → **已于 2026-10-06 完成框架化，见 §12.6**；容器内 CLI 与 HTTP 管理接口均可用 |
| 实现中的三处取舍（规格未明写、需记录） | ① `mc_names.uuid` 恒为 `null`：只走 RCON 而 `whitelist list` 不返回 UUID，读 `whitelist.json` 又因匿名卷不可行（§5.6 允许「只作记录」）；② **被拒绝的请求也写审计**（§10 第 10 条），但按 IP 限流以防坏令牌刷爆审计表；③ **`/v1/me` 同样要求 `mc:whitelist`**（§6.1 写的是「所有 `/v1/*`」） |
| 已知易错点（实测踩到并已修） | 测试里 `from tests.conftest import ...` 会**再导入一份 conftest**（pytest 以顶层名 `conftest` 导入），产生两把不同测试密钥 → `InvalidSignatureError`。共享常量因此放在 `tests/helpers.py` |

### 12.6 ✅ 部署框架化完成（2026-10-06，`/init-deploy-framework`）

按技能要求全量落地，与 `sso` 现在是同一套形态。仓库：<https://github.com/EliCloudOrg/mc>（public）。

| 项 | 结果 |
|---|---|
| 分支模型 | `main`（集成，4 个提交）+ `prod`（发布，与 main 同步）；**两分支均有分支保护**（`main`: `Test and build`；`prod`: + `Guard prod source`，strict、禁 force push/删除） |
| CI | `.github/workflows/ci.yml`（Python 3.12 + pip）：compileall 语法检查 → 导入 `app.main` → `pytest -q`；含 `guard-prod-source` job |
| 部署 | `.github/workflows/deploy-prod.yml`（`DEPLOY_MODE=image` → 推 `ghcr.io/elicloudorg/mc:prod` 与 `:sha-<short>` → scp `deploy/deploy.sh` → ssh 执行） |
| 部署器 | `/srv/mc/deploy.sh`（由流水线同步覆盖）+ `/srv/mc/app.env`（600, deploy:deploy） |
| 项目目录 | `/home/deploy/elicloud-mc`（**snap docker 看不到 `/srv`**；`/home/docker-admin` 对 `deploy` 不可遍历） |
| 服务器前置 | `deploy` 用户已存在且在 docker 组；`deploy` 免密 SSH 打通；bootstrap 幂等执行 |
| **端到端实测** | `push prod` → CI 33s → Deploy 1m6s（镜像 21s + 远程部署 39s）→ 容器切到 `ghcr.io/elicloudorg/mc:prod`，`healthy`，公网 `/mc/healthz` 200，SQLite **33 条审计全部保留** |
| 数据迁移 | 从旧目录 `cp -a` 过来；`data/` 属主保持 `1002:1003`（容器运行用户），否则容器无法写库 |
| 三个受限项（免费账号） | ① Environment `production` 建好了但**加不了 Required reviewers**（422 计费限制）；② private 仓库**用不了分支保护**（403）→ 已设为 public（与 `EliCloudOrg/sso` 一致）；③ 单账号不能自审 → `required_approving_review_count=0`（仍需走 PR） |
| ⚠️ ghcr 实测（与直觉不同） | `denied` = **包还不存在**（非权限问题）；pull 会长时间停在 `Pulling fs layer`（数分钟）**但最终成功**；`/v2/` 返回 401 是正常匿名响应。因此**必须先推镜像再部署**，且 `deploy.sh` 的 pull 超时回退逻辑是必需的 |

---

## 13. 相关文档

| 文档 | 关系 |
|---|---|
| [`architecture.md`](architecture.md) | 平台总纲：§2.2 路径规范、§4 数据隔离、§7 部署形态与网关、§8 业务服务接入约定 |
| [`deploy-framework.md`](deploy-framework.md) | 部署框架（触发器/部署器分离、`/srv/<app>/app.env`、snap Docker 约束） |
| [`sso-oidc.md`](sso-oidc.md) | SSO 的 OIDC 端点；前端如何拿到 access token |
| `sso/README.md` | 网关改动的验证流程（`caddy validate` 再 `reload`）、IP 阶段地址 |
| 本文件 | MC 白名单服务规格（契约、RCON 管道、实测证据、部署与验收） |
