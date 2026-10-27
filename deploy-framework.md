# 自托管部署框架：设计思想

> **本文件只讲框架本身的设计思想与原则，不讲单个项目怎么落地。**
> 单项目的操作细节（本项目实际值、首次部署步骤、GitHub 配置清单）见各仓库自己的 `README-deploy.md`，
> 参考实现见 [`elicloud-deploy-test/README-deploy.md`](../elicloud-deploy-test/README-deploy.md)。
>
> 框架的规范原文：[`deepseek_text_20261002_729729.txt`](./deepseek_text_20261002_729729.txt)（DevOps 初始化提示词）。
> 它的可执行摘要与模板：`.skill-build/init-deploy-framework/`（SKILL.md、`references/spec.md`、`references/templates/`）。
> 端到端验证证据：[`elicloud-deploy-test/VALIDATION.md`](../elicloud-deploy-test/VALIDATION.md)。

---

## 1. 一句话定义

```text
本地开发 → push GitHub → PR 合入 prod → GitHub Actions 通过 SSH 调用服务器上的
/srv/<app>/deploy.sh <ref> → 服务器按项目自己的方式运行应用
```

**核心思想：触发器与部署器分离。**

- **触发器**（框架、GitHub Actions）：知道"什么时候该部署"和"去调用谁"。
- **部署器**（`/srv/<app>/deploy.sh`）：知道"这个项目具体怎么部署"。
- 两者之间只有**一个契约**：`/srv/<app>/deploy.sh <ref>`，一个参数、非 0 表示失败。

一句话概括框架的自我定位：**框架只负责「触发和调用」，不关心项目具体怎么构建、怎么运行。**

---

## 2. 核心设计思想

### 2.1 触发器与部署器分离（本框架的根本原则）

这是所有其它设计决定的来源。它把"部署"拆成两个可以独立变化的部分：

| | 触发器 | 部署器 |
|---|---|---|
| 是什么 | 分支模型 + `ci.yml` + `deploy-prod.yml` | `/srv/<app>/deploy.sh` |
| 在哪运行 | GitHub Actions | 目标服务器 |
| 谁维护 | 框架（对所有项目一样） | 项目自己 |
| 关心什么 | 何时部署、部署哪个 ref、如何安全地触达服务器 | 如何构建、如何切换、如何重启 |
| 不该关心什么 | 项目用什么语言、什么容器、什么进程管理器 | CI 怎么触发、密钥怎么从 GitHub 来 |

**分离带来的三个直接收益：**

1. **触发器可以完全通用。** 换成 Python、Go、.NET、静态站点，`deploy-prod.yml` 一个字不用改。
2. **部署器可以完全自由。** 用 systemd、pm2、Docker、裸进程都行，框架不预设。
3. **故障边界清晰。** "Actions 跑红了"与"服务器上应用没起来"是两个可以分别定位的问题。

### 2.2 为什么部署逻辑必须留在服务器，而不是写在 workflow 里

这是分离原则最重要的一条推论，也是硬性约束里"**Actions 不得包含任何项目特定的构建或运行命令**"的原因：

| 如果部署逻辑写在 workflow | 后果 |
|---|---|
| 每次改部署步骤（换端口、加依赖、改重启方式） | 都要改 YAML、走一次完整 CI、再走一次人工审批 |
| 部署细节散在 GitHub 侧 | 无法在服务器上直接复现和调试——而部署问题天然只能在服务器上定位 |
| 服务器环境与仓库耦合 | 想换构建方式要动两个地方（workflow + 服务器） |
| 调试必须依赖 GitHub 环境 | 没有一个"能在服务器上直接跑一遍"的入口 |

反过来，把逻辑收进 `/srv/<app>/deploy.sh` 之后：**运维可以直接 `sudo -u deploy /srv/<app>/deploy.sh <ref>` 手工重放一次部署**，与 Actions 走的是同一条路径。这条"手工可重放"是框架可运维性的基石。

### 2.3 为什么框架不预设运行方式

框架**明确不规定**应用用什么语言、什么框架，也不规定用 systemd / pm2 / Docker / 裸进程。

**理由**：这些选择高度依赖项目（一个 Node 小服务和一个 .NET 服务的最优运行方式完全不同）。一旦框架替项目选了，就会出现"框架比项目本人更懂项目"的错位，而框架又不在服务器上运行——改起来最麻烦的恰恰是它。

**代价（必须承认）**：框架给不出"开箱即用"的部署体验，`deploy.sh` 的部署逻辑区在初始化后是**空骨架**。这是有意的：
模板 `deploy.sh` 只做 `set -euo pipefail`、解析 ref、`cd`、打印，逻辑区留给项目。

### 2.4 配置与密钥分离：三处归属

框架把"配置"按**敏感度**和**变更频率**拆到三个位置，边界不重叠：

| 放什么 | 放在哪 | 谁改 | 为什么放这 |
|---|---|---|---|
| 部署凭据（SSH 私钥、主机、用户、端口） | **GitHub Secrets** | 仓库管理员 | 只有执行部署的那个动作需要它；绝不能进仓库、绝不能上服务器 |
| 应用运行时变量（端口、版本、各类 token） | **服务器 `/srv/<app>/app.env`** | 运维 | 运行时才需要；服务器本地可读，不进仓库、不进镜像 |
| 部署脚本本身 | **仓库 `deploy/deploy.sh`（版本管理）** | 开发 | 是代码，需要 review、需要回滚、需要审计历史 |

三个位置对应三种生命周期：**凭据一次性配置、运行时变量偶尔调整、部署脚本随代码演进**。混在一起（比如把运行时变量写进 `deploy.sh`、把脚本只放在服务器上不版本化）立刻会丢失其中一种能力。

### 2.5 为什么部署脚本随仓库版本管理，却又上传覆盖服务器上的那份

这看着矛盾，其实是"**单一真源 + 运行位置分离**"：

- **单一真源在仓库**：`deploy/deploy.sh` 走 PR review 和 Git 历史，任何人不能偷偷在服务器上改部署行为。
- **运行副本在服务器**：每次部署时 `scp` 上传覆盖 `/srv/<app>/deploy.sh`，保证**实际执行的正是这次 ref 里的那份脚本**。

因此部署脚本模板里有一句看似多余、实则关键的注释：

```bash
# 本文件由仓库 deploy/deploy.sh 同步而来，不要在服务器上直接改；
# 改动请提交到仓库，再由 deploy-prod.yml 上传
```

> **一个刻意的设计后果**：脚本与它所部署的 ref 是**绑定**的——部署 `prod` 时执行的就是 `prod` 的部署脚本。
> 这也意味着"用旧 ref 回滚"会连脚本一起回到旧版本（见 §8 回滚）。

---

## 3. 硬性约束（不可更改）

| # | 约束 | 存在理由 |
|---|---|---|
| 1 | **分支模型**：`main` 集成分支（日常 PR、跑 CI）；`prod` 发布分支（受保护，只允许从 `main` 发 PR 合并）；`feature/*` 从 `main` 拉出、PR 回 `main` | 让"合并到 prod"成为唯一且可审计的发布动作 |
| 2 | **远程目录统一 `/srv/<app>/`**，`<app>` 由项目决定 | 不同项目在服务器上形状一致，运维心智负担最小 |
| 3 | **远程入口统一 `/srv/<app>/deploy.sh`**：由 `deploy` 用户执行，接收一个参数（Git ref，默认 `prod`），幂等、可重复执行，失败必须非 0 退出 | 这就是触发器与部署器之间的**唯一契约** |
| 4 | **Actions 只做三件事**：(a) 可选——构建并推送产物到镜像仓库 / Release / 对象存储；(b) SSH 到服务器；(c) 执行 `/srv/<app>/deploy.sh <ref>`。**不得含任何项目特定的构建或运行命令** | §2.2 |
| 5 | **配置与密钥分离**：密钥走 GitHub Secrets；运行时变量放 `/srv/<app>/app.env`；部署脚本随仓库版本管理，路径固定 `deploy/deploy.sh` | §2.4 / §2.5 |

**关于"幂等"与"失败非 0"的额外要求**：
- **幂等**：重复跑同一 ref 不产生破坏性副作用——这既让重试安全，也让"手工重放"（§2.2）成为可用的排障手段。
- **失败必须非 0 退出**：这是失败能一路传播回 Actions 的唯一机制（实测：`deploy.sh` 里 `exit 3` → step failure → job failure）。

---

## 4. 数据流与职责分层

```text
      feature/* ──PR──▶ main ──PR──▶ prod
                         │            │
                    push │            │ push / workflow_dispatch(ref)
                         ▼            ▼
              ┌──────────────────┐  ┌───────────────────────────────────────┐
              │ CI  ci.yml       │  │ Deploy  deploy-prod.yml               │
              │  检出 → 装依赖    │  │  ① (可选) 构建镜像 → ghcr.io          │
              │  → 测试 → 构建    │  │     仅当 vars.DEPLOY_MODE == image    │
              └──────────────────┘  │  ② scp deploy/deploy.sh → /srv/<app>/ │
                                    │  ③ ssh 执行 /srv/<app>/deploy.sh REF  │
                                    │  environment: production（人工审批）  │
                                    └──────────────────┬────────────────────┘
                                                       │ SSH（deploy 用户 + 私钥）
                                                       ▼
                                    ┌───────────────────────────────────────┐
                                    │ 服务器  /srv/<app>/                    │
                                    │   deploy.sh   ← 随仓库版本管理         │
                                    │   app.env     ← 运行时环境变量（600）  │
                                    │   项目自己的构建产物 / 容器 / 进程      │
                                    └───────────────────────────────────────┘
```

**分层读法：**

| 层 | 属于 | 职责 | 变更频率 |
|---|---|---|---|
| 分支与审批 | 流程 | 决定"什么算发布" | 极低 |
| CI（`ci.yml`） | 触发器 | 验证代码可用 | 随技术栈 |
| Deploy（`deploy-prod.yml`） | 触发器 | 把 ref 安全地送达服务器并调用入口 | 极低 |
| `/srv/<app>/deploy.sh` | 部署器 | 真正让应用跑起来 | 随项目 |
| `/srv/<app>/app.env` | 配置 | 运行时变量 | 偶尔 |

> 注意：**CI 与部署是两条独立的链路**。`ci.yml` 不参与部署决策，`deploy-prod.yml` 也不跑测试。
> 它们的连接点只有"同一个 ref"。

---

## 5. 非目标（框架明确不做）

| 不做 | 原因 |
|---|---|
| 不规定语言与框架 | §2.3 |
| 不规定 systemd / pm2 / Docker / 裸进程 | §2.3 |
| 不规定构建产物形态（镜像 / 二进制 / 静态文件） | §2.3 |
| 不实现具体部署逻辑，只给 `deploy.sh` 骨架 | §2.1 的分离原则 |
| 不引入 Kubernetes / Terraform / Ansible | 目标规模是自托管单机；引入即失去"薄"的价值 |

**第三方 Action 白名单**（与之配套的"薄"）：只用 `actions/*`（GitHub 官方）、`appleboy/ssh-action`、`appleboy/scp-action`、`docker/*`。
其它市场 Action 一律不用——每引入一个就多一个供应链与长期维护面。CI 的"装依赖"必须用 `actions/setup-*` 等官方动作。

---

## 6. 扩展点

框架留出的自由度全部集中在两处，且都**不需要改触发器**：

1. **`deploy.sh` 的"项目自定义部署逻辑"段**——拉代码 / 装依赖 / 构建 / 重启服务等，全在项目手里。
2. **可选阶段一（`DEPLOY_MODE`）**——需要镜像时把它设为 `image`，触发器会先用 `docker/build-push-action`
   推送到 `ghcr.io` 并打 **`prod`** 与 **`sha-<short>`** 两个 tag，然后才执行部署；
   不需要镜像时（如静态站点、直接 scp 二进制）这一阶段被跳过，部署照常进行。

对应的开关语义：**阶段二只在阶段一"成功或被跳过"时执行**（`needs: [image]` + `always()` + 结果判断），
因此"用不用镜像"不会影响"能不能部署"。

---

## 7. 落地与验证证据

| 内容 | 位置 |
|---|---|
| 规范原文（提示词） | [`deepseek_text_20261002_729729.txt`](./deepseek_text_20261002_729729.txt) |
| 可执行摘要与目录结构要求 | `.skill-build/init-deploy-framework/references/spec.md` |
| 技能说明（含前置检查、白名单、产出清单） | `.skill-build/init-deploy-framework/SKILL.md` |
| 模板（`ci.yml` / `deploy-prod.yml` / `deploy.sh` / `bootstrap-server.sh` / `README-deploy.md`） | `.skill-build/init-deploy-framework/references/templates/` |
| 脚手架（幂等生成、LF 换行） | `.skill-build/init-deploy-framework/scripts/scaffold.ps1` |
| 端到端验证项目 | [`elicloud-deploy-test/`](../elicloud-deploy-test/) |
| 验证报告（含失败传播、幂等、权限、落点等实测项） | [`elicloud-deploy-test/VALIDATION.md`](../elicloud-deploy-test/VALIDATION.md) |
| **第一个真实业务服务**（SSO，2026-10-06 接入） | GitHub [`EliCloudOrg/sso`](https://github.com/EliCloudOrg/sso) · 本地 [`sso/`](https://github.com/EliCloudOrg/sso) · 部署手册 [`sso/README-deploy.md`](https://github.com/EliCloudOrg/sso/blob/main/README-deploy.md) |

**已验证的框架自身行为**（详见 `VALIDATION.md`）：生成物与模板逐字节一致；shell 语法通过；
scp 落点与仓库脚本一致（无 `strip_components` 嵌套）；`deploy.sh` 连跑 3 次输出一致（幂等）；
失败能传播为 job failure；全链路 `push → PR → prod → Actions（人工审批）→ SSH → deploy.sh → 容器` 跑通。

> **注意区分**：验证项目里 `deploy/deploy.sh` **按框架约定仍是空骨架**，真实部署逻辑只在 `VALIDATION.md` 第 6 节作为参考实现给出（未入库）。
> 因此"部署成功"目前意味着"脚本被正确调用并成功退出"，而不是"应用已更新"。

---

## 8. 已知边界与框架层未定论项

### 8.1 回滚：框架层不做统一定论（风险已记录）

`VALIDATION.md` D4 记录了一个真实隐患：

```text
用 workflow_dispatch 传旧 ref 回滚时：
  阶段一始终 checkout 触发分支、按当前 SHA 打 :prod tag（不引用 inputs.ref）
  → 只有 deploy.sh 收到了旧 ref
  → 若 deploy.sh 拉 :prod，实际部署的是"最新代码"，而不是回滚目标
```

> **✅ 该缺陷已在模板 v2 修掉（2026-10-06 更新）**：`templates/deploy-prod.yml` 阶段一的
> checkout 改为 `ref: ${{ github.event.inputs.ref || github.ref }}`，`sha-<short>` 也改为
> `git rev-parse --short=7 HEAD`（取自刚检出的那个 ref）。因此 `:prod` 会被重新指向**回滚目标**，
> `deploy.sh` 固定拉 `:prod` 就是正确的回滚语义。
> 下面这段「立场」与 D4 的描述记录的是修复前的状态；
> 另注：`elicloud-deploy-test/README-deploy.md` 的「回滚流程」一节仍写着旧行为，属该示例文档未同步。

**框架层的立场：回滚方式由项目在 `deploy.sh` 里自行决定，框架不统一规定。** 原因是回滚策略与产物形态强绑定
（镜像是按 tag 回退还是按 digest 回退、静态文件是切软链还是覆盖），框架无法给出对所有项目都正确的答案。

**但风险必须显式记录，且作者有责任择一规避**：

| 候选做法 | 效果 |
|---|---|
| 阶段一改为 `checkout` 指定的 ref，并让 `sha-<short>` 取自该 ref | 让镜像与 ref 一致，回滚落到确定版本（模板层修复） |
| `deploy.sh` 只按 `sha-<short>` 拉取，不依赖 `:prod` | 部署器侧保证确定性，不要求改触发器 |
| 明确声明"本项目不支持用旧 ref 回滚" | 至少不制造"以为能回滚"的错觉 |

> 采用 `DEPLOY_MODE=image` 的项目**必须**处理这一条。

### 8.2 其它已记录的部署环境约束（写给 `deploy.sh` 的作者）

这些不是框架缺陷，而是目标环境的真实限制，框架的作用是把它们**提前暴露**而不是上线时才发现：

| 约束 | 影响 |
|---|---|
| **snap 版 Docker 看不到 `/srv`** | `deploy.sh` 里任何把 `/srv` 路径交给 docker 的写法都会失败（`-v /srv/...`、`--env-file /srv/...`、`docker build /srv/...`）。不影响 scp 上传与脚本自身读 `app.env`。 |
| `ghcr.io` 在部分网络不可达 | 需给 dockerd 配 `registry-mirrors`，或让 `deploy.sh` 支持镜像源变量 |
| GitHub 免费账号的 private 仓库 | 配不了分支保护与 Required reviewers——而分支模型恰恰依赖它们 |
| `deploy.sh` 出现 `bad interpreter: ...^M` | 上传/提交时带了 CRLF；`.gitattributes` 里需有 `*.sh text eol=lf` |
| `deploy.sh: Permission denied` | 服务器上文件缺可执行位（workflow 已 `chmod +x`，手工上传时要自己加） |

### 8.3 尚未内置的能力（按需再加，不要默认引入）

- **多环境**（staging/preview）：当前只有 `prod` 一条发布链。
- **构建缓存与产物留存策略**：由各项目自行决定。
- **部署通知/回执**：目前只有 Actions 的 run 结果。

---

## 9. 框架的由来（便于日后追溯）

```text
一份 DevOps 初始化提示词（deepseek_text_20261002_729729.txt）
        │  抽取为可复用技能
        ▼
skill: init-deploy-framework（.skill-build/init-deploy-framework/）
        │  在真实服务器上端到端验证
        ▼
验证项目 elicloud-deploy-test + 验证报告 VALIDATION.md
        │  思想沉淀
        ▼
本文件（docs/deploy-framework.md）
```

三者的分工：**提示词是需求、skill 是可执行规范、验证项目是证据，本文件是"为什么"。**
改框架时按此顺序追溯：先看本文件的原则，再看 `references/spec.md` 的规范，最后改模板与 skill。
