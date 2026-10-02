# Git 工作流与发布流程

| 项目 | 内容 |
|---|---|
| 文档名称 | 分支模型、提交规范与发布流程 |
| 文档版本 | V1.0 |
| 编写日期 | 2026-10-02 |
| 关联文档 | `docs/技术架构.md` 第 10.2 节、`docs/环境与运行说明.md` |

本文件把 `docs/技术架构.md` 第 10.2 节的分支策略落成可执行的日常操作规范，供一期 15 天开发和后续发布使用。

---

## 1. 远程仓库

| 项目 | 内容 |
|---|---|
| 仓库地址 | `git@github.com:Hmi666/ai-english-learning.git` |
| 远端名 | `origin` |
| 协议 | SSH |
| 可见性 | Private |
| 默认分支 | `master` |

```bash
git remote -v
# origin  git@github.com:Hmi666/ai-english-learning.git (fetch)
# origin  git@github.com:Hmi666/ai-english-learning.git (push)
```

上游脚手架 `1024-lab/smart-admin` 不作为本仓库的 remote 保留。需要比对上游代码时，按 `docs/脚手架来源说明.md` 记录的 commit SHA 临时克隆，避免误把本仓库当成上游的 fork。

## 2. 分支模型

| 分支 | 生命周期 | 用途 | 保护 |
|---|---|---|---|
| `master` | 永久 | 稳定可验收版本，任何提交都必须可构建可验收 | 禁止直接推送 |
| `develop` | 永久 | 一期开发集成分支，日常提交目标 | 禁止直接推送，只接受 PR/MR |
| `feature/*` | 临时 | 单个功能开发 | 无 |
| `fix/*` | 临时 | 非紧急缺陷修复 | 无 |
| `release/*` | 临时 | 发布准备与验收 | 无 |
| `hotfix/*` | 临时 | 已发布版本的紧急修复 | 无 |

命令示例：

```bash
# 功能开发
git switch develop && git pull
git switch -c feature/FR-CONTENT-02-content-crud
# ... 开发与提交 ...
git switch develop && git merge --no-ff feature/FR-CONTENT-02-content-crud
git branch -d feature/FR-CONTENT-02-content-crud
```

分支命名带上需求编号，便于把代码变更和 `docs/一期开发产品需求书.md` 里的功能项对应起来，例如：

- `feature/FR-AUTH-02-first-login-password`
- `feature/FR-TASK-01-create-task`
- `feature/FR-QUIZ-04-submit-quiz`
- `fix/student-cross-class-leak`

## 3. 提交信息规范

统一使用 `type(scope): subject` 形式：

```text
feat(task): 支持教师向多个授权班级布置任务
fix(quiz): 修复同一学生可重复提交测验的问题
docs(arch): 补充测验提交的幂等约束说明
refactor(auth): 提取登录失败提示的统一处理
test(student): 补充跨班级数据隔离用例
chore(deps): 锁定 MySQL 驱动到 9.3.0
build(deploy): 补充验收环境 docker-compose
```

| 字段 | 取值 |
|---|---|
| `type` | `feat`、`fix`、`docs`、`refactor`、`test`、`chore`、`build`、`perf` |
| `scope` | `auth`、`org`、`content`、`task`、`quiz`、`record`、`log`、`arch`、`deps`、`deploy`、`repo` |
| `subject` | 中文描述，动词开头，不加句号，不超过 50 字 |

**要求：**

- 一次提交只做一件事，不把格式化、重命名和功能改动混在一起；
- 不提交无法构建或无法启动的代码；
- 提交信息写「为什么」，具体「改了什么」由 diff 说明；
- 禁止提交真实学生姓名、真实手机号、密码和生产密钥。

## 4. 日常开发流程

```bash
# 1. 同步集成分支
git switch develop && git pull --ff-only

# 2. 开功能分支
git switch -c feature/<需求编号>-<简述>

# 3. 小步提交
git add <files> && git commit -m "feat(scope): ..."

# 4. 推送到远端
git push -u origin feature/<需求编号>-<简述>

# 5. 提 PR 到 develop，至少 1 人评审
# 6. 评审通过后合并（--no-ff 保留分支历史），删除功能分支
```

**合并前自查：**

- [ ] `cd backend && mvn -DskipTests clean package` 通过；
- [ ] `cd frontend && pnpm build:prod` 通过；
- [ ] 涉及数据库的改动已新增迁移脚本，未直接改 `V2026xxxx` 基线文件；
- [ ] 涉及接口权限的改动已按 `docs/一期开发产品需求书.md` 第 10.2 节补越权测试；
- [ ] 新增第三方依赖已确认许可证。

## 5. 版本号规则

采用语义化版本 `MAJOR.MINOR.PATCH`，一期处于 0.x 阶段：

| 版本 | 含义 |
|---|---|
| `v0.1.0` | 工程基线：脚手架初始化、依赖锁定、可启动（**已完成**） |
| `v0.x.0` | 每完成一个可演示的里程碑（登录鉴权、组织账号、内容、任务、测验、结果） |
| `v0.x.y` | 里程碑内的问题修复 |
| `v1.0.0` | 一期验收通过，可交付版本 |

一期不引入 `MAJOR` 升级，`v1.0.0` 之后如出现不兼容变更再升 `MAJOR`。

## 6. 发布流程

```bash
# 1. 从 develop 切发布分支
git switch develop && git pull
git switch -c release/v0.2.0

# 2. 发布准备：只允许修 bug、改文档、改版本号
#    更新 README / 变更记录，禁止在 release 分支加新功能

# 3. 在验收环境部署并执行验收用例
#    deploy/docker-compose.yml 或 Nginx + JAR 方式

# 4. 验收通过后合并到 master 并打 tag
git switch master && git pull
git merge --no-ff release/v0.2.0
git tag -a v0.2.0 -m "v0.2.0 <一句话说明本版交付内容>"
git push origin master --follow-tags

# 5. 回合并到 develop，避免修复丢失
git switch develop
git merge --no-ff release/v0.2.0
git push origin develop

# 6. 删除发布分支
git branch -d release/v0.2.0
```

**tag 规范：**

- 使用附注标签 `git tag -a`，不用轻量标签；
- 标签说明写清本版交付范围和验收依据；
- 只给 `master` 上的提交打 tag，不给 `develop` 或功能分支打 tag。

## 7. 紧急修复流程

```bash
git switch master && git pull
git switch -c hotfix/v1.0.1-cross-class-leak
# ... 修复并提交 ...
git switch master && git merge --no-ff hotfix/v1.0.1-cross-class-leak
git tag -a v1.0.1 -m "v1.0.1 修复教师可查看未授权班级数据的问题"
git push origin master --follow-tags

# 必须回合并到 develop
git switch develop && git merge --no-ff hotfix/v1.0.1-cross-class-leak
git push origin develop
```

越权类缺陷必须同时补上回归用例，并在 `docs/一期开发产品需求书.md` 第 10.2 节对应条目下记录。

## 8. 建议的分支保护规则

在 GitHub 仓库设置中对 `master` 和 `develop` 启用：

- 禁止直接 push，必须走 Pull Request；
- 至少 1 个 approving review；
- 合并前要求分支与目标分支同步（Require branches to be up to date）；
- 禁止 force push 和删除分支；
- 建议把「后端构建通过」「前端构建通过」作为必需的状态检查接入 CI。

CI 流水线接入后可执行的最小校验：

```bash
# 后端
cd backend && mvn -B -DskipTests clean package

# 前端
cd frontend && CI=true pnpm install --frozen-lockfile && pnpm build:prod
```

## 9. 首次建仓记录

| 步骤 | 内容 |
|---|---|
| 1 | 在 GitHub 创建私有空仓库 `Hmi666/ai-english-learning`（**不勾选** README / .gitignore / License，避免与本地历史冲突） |
| 2 | 本地配置远端：`git remote add origin git@github.com:Hmi666/ai-english-learning.git` |
| 3 | 本地建立 `develop` 分支并打 `v0.1.0` 标签 |
| 4 | 推送分支与标签：`git push -u origin master develop --follow-tags` |
| 5 | 在仓库设置中把默认分支设为 `master`，并按第 8 节开启分支保护 |

> 如果远端仓库在创建时已经初始化了 README 或 License，推送会被拒绝。此时先执行
> `git pull --rebase origin master` 合并远端历史，或删除远端仓库重新创建空仓库。

## 10. 变更记录

| 日期 | 版本 | 变更内容 |
|---|---|---|
| 2026-10-02 | V1.0 | 建立分支模型、提交规范、语义化版本规则、发布与热修复流程、首次建仓记录。 |
