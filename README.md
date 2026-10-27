# EliCloud 文档

EliCloud 自托管服务体系的架构与运维文档集。

## 目录

| 文件 | 内容 |
|---|---|
| [`architecture.md`](architecture.md) | 整体架构：SSO、网关、各业务服务的边界与数据流 |
| [`sso-oidc.md`](sso-oidc.md) | SSO 的 OIDC 实现细节、端点契约、令牌与 scope 设计 |
| [`mc.md`](mc.md) | MC（Minecraft）白名单服务与游戏服务器的联动 |
| [`deploy-framework.md`](deploy-framework.md) | 自托管部署框架的设计思想：触发器与部署器分离 |
| [`deepseek_text_20261002_729729.txt`](deepseek_text_20261002_729729.txt) | 部署框架的规范原文（DevOps 初始化提示词） |

## 模块与仓库

| 模块 | 仓库 |
|---|---|
| SSO（认证中心） | [EliCloudOrg/sso](https://github.com/EliCloudOrg/sso) |
| 邮箱验证服务 | [EliCloudOrg/email](https://github.com/EliCloudOrg/email) |
| 文件服务 | [EliCloudOrg/file](https://github.com/EliCloudOrg/file) |
| 用户资料服务 | [EliCloudOrg/userinfo](https://github.com/EliCloudOrg/userinfo) |
| MC 白名单服务 | [EliCloudOrg/mc](https://github.com/EliCloudOrg/mc) |
| .NET SDK | [EliCloudOrg/EliCloud.Sdk](https://github.com/EliCloudOrg/EliCloud.Sdk) |
| Windows 桌面端 | [EliCloudOrg/EliCloudDesktop](https://github.com/EliCloudOrg/EliCloudDesktop) |

## 说明

- 每个服务仓库自带 `README-deploy.md`，是**该服务**的部署操作手册（实际值、首次部署步骤、GitHub 配置清单）；
  本仓库的 `deploy-framework.md` 只讲框架本身的设计原则，不重复具体项目细节。
- 文中少数链接指向 `elicloud-deploy-test/`（部署框架的端到端验证项目）和 `.skill-build/`（生成框架的技能），
  这两个目录目前只在本地工作区，尚未独立成仓库；对应内容已在 `deploy-framework.md` 第 7 节说明。
