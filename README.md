# 基于生成式大模型的小学英语辅助学习系统

小学英语辅助学习系统的一期工程仓库。一期目标是交付一个可演示、可验收的核心学习闭环：

> 管理员准备学习内容 → 教师布置任务和客观题测验 → 学生完成学习和测验 → 系统记录结果 → 教师和学生查看反馈。

一期范围为单学校部署、三类角色（`admin` / `teacher` / `student`），**不包含任何 AI 能力**，AI 相关功能属于二期规划。

## 文档

需求与技术方案的唯一依据在 `docs/` 目录，编码前请先阅读：

| 文档 | 说明 |
|---|---|
| [一期开发产品需求书](docs/一期开发产品需求书.md) | 一期开发基线，功能边界、业务规则、验收标准 |
| [技术架构](docs/技术架构.md) | 技术选型、分层规范、数据库与接口约束、分支策略 |
| [需求评审结论](docs/需求评审结论.md) | 一期范围收缩的评审依据 |
| [产品需求书](docs/产品需求书.md) | 完整产品规划，包含二期及后续方向 |
| [脚手架来源说明](docs/脚手架来源说明.md) | 脚手架来源 commit、导入范围、许可证义务 |
| [环境与运行说明](docs/环境与运行说明.md) | 工具链版本、启动步骤、初始化阶段发现的问题 |

## 目录结构

```text
ai-english-learning/
├── docs/                  # 需求、架构、评审与工程说明文档
├── backend/               # Java 17 + Spring Boot 3 后端（源自 SmartAdmin）
│   ├── sa-base/           # 脚手架公共基础能力
│   └── sa-admin/          # 脚手架系统管理能力 + 一期业务开发入口
├── frontend/              # Vue 3 + TypeScript + Vite 前端（源自 SmartAdmin）
├── database/
│   ├── migrations/        # 数据库版本迁移脚本
│   └── seed/              # 演示与测试数据（虚构数据）
├── deploy/
│   └── docker-compose.yml # 本地开发用 MySQL 8 + Redis 7
├── .node-version          # 前端 Node 版本锁定
└── LICENSE                # 上游 SmartAdmin 的 MIT 许可证原件
```

## 技术栈

| 层次 | 技术 | 版本 |
|---|---|---:|
| 后端语言 | Java | 17 LTS |
| 后端框架 | Spring Boot（Spring MVC，非 WebFlux） | 3.5.4 |
| ORM | MyBatis-Plus | 3.5.12 |
| 数据库 | MySQL | 8.0+（`utf8mb4`） |
| 权限认证 | Sa-Token | 1.44.0 |
| API 文档 | Knife4j | 4.6.0 |
| 后端构建 | Maven | 3.9+ |
| 前端框架 | Vue（Composition API + `<script setup>`） | 3.4.27 |
| 前端语言 | TypeScript | 5.6.3 |
| 前端构建 | Vite | 5.2.12 |
| UI 组件 | Ant Design Vue | 4.2.5 |
| 状态管理 | Pinia | 2.1.7 |
| HTTP 客户端 | Axios | 1.6.8 |
| 前端运行环境 | Node.js | 22.23.2（见 `.node-version`） |
| 前端包管理 | pnpm | 统一使用 pnpm，不使用 npm/yarn |

一期不引入微服务、网关、注册中心、配置中心和消息队列。

## 环境准备

```bash
# 1. JDK 17（示例使用 sdkman，也可用 brew install --cask temurin@17）
sdk install java 17.0.20-tem && sdk use java 17.0.20-tem

# 2. Node 22（示例使用 fnm，仓库已提供 .node-version）
fnm install && fnm use

# 3. pnpm（要求 Node >= 22.13）
npm install -g pnpm

# 4. Maven 3.9+
mvn -v
```

## 快速开始

启动本地依赖（MySQL 8 + Redis 7），首次启动会自动导入建库脚本：

```bash
docker compose -f deploy/docker-compose.yml up -d
docker compose -f deploy/docker-compose.yml ps   # 等待 mysql 变为 healthy
```

启动后端（默认端口 `1024`，profile 为 `dev`）：

```bash
cd backend
mvn -pl sa-admin -am spring-boot:run
```

启动前端（默认端口 `8081`）：

```bash
cd frontend
pnpm install
pnpm dev
```

更详细的启动参数、默认账号和本地配置覆盖方式见 [环境与运行说明](docs/环境与运行说明.md)。

## Git 工作流

分支策略遵循 [技术架构](docs/技术架构.md) 第 10.2 节：

| 分支 | 用途 | 说明 |
|---|---|---|
| `master` | 稳定可验收版本 | 只接受来自 `release/*` 或 `hotfix/*` 的合并，每个版本打 tag |
| `develop` | 一期开发集成分支 | 功能开发的集成分支，日常提交目标 |
| `feature/*` | 功能开发 | 从 `develop` 切出，完成后合并回 `develop` |
| `fix/*` | 问题修复 | 非紧急缺陷修复 |
| `release/*` | 发布准备 | 从 `develop` 切出，验收通过后合并到 `master` 并回合并 `develop` |
| `hotfix/*` | 线上紧急修复 | 从 `master` 切出，修复后同时合并 `master` 与 `develop` |

提交信息使用 `type(scope): subject` 形式，`type` 取值 `feat` / `fix` / `docs` / `refactor` / `test` / `chore` / `build`。完整约定与发布流程见 [环境与运行说明](docs/环境与运行说明.md)。

## 许可证

本项目基于 [1024-lab/smart-admin](https://github.com/1024-lab/smart-admin)（MIT License，Copyright (c) 2020 1024-lab）二次开发。

- 根目录 [`LICENSE`](LICENSE) 为上游许可证原件，必须保留；
- 引入的 commit 与目录范围见 [脚手架来源说明](docs/脚手架来源说明.md)；
- 新增第三方依赖需逐项确认许可证兼容性；
- 不得对外宣称脚手架原作者为本项目作者。

学习内容、图片、音频等业务素材的版权需单独确认，许可证确认不覆盖业务素材。
