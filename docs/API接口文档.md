# 小学英语辅助学习系统 API 接口文档

| 项目 | 内容 |
|---|---|
| 文档版本 | V1.0（接口设计基线） |
| 状态 | 前后端开发与联调契约 |
| 产品需求 | [`docs/产品需求书.md`](产品需求书.md) |
| 一期范围 | [`docs/一期开发产品需求书.md`](一期开发产品需求书.md) |
| 技术架构 | [`docs/技术架构.md`](技术架构.md) |
| 日期 | 2026-10-03 |

## 1. 文档约定和生效规则

本文件是前端页面调用和后端接口实现的共同契约。完整产品范围包含 `admin`（学校管理员）、`grade_quality_admin`（年级质量管理员）、`teacher`（教师）、`parent`（家长）、`student`（学生）五类角色；每个接口均标注阶段：

- `P1`：一期 MVP，限单学校及 `admin`、`teacher`、`student` 三类角色。
- `P2`：二期能力，不得作为一期上线或验收条件。
- `P3`：后续规划；只保留接口方向，不视为已批准开发范围。

产品需求、接口文档或实现发生变更时，产品、前端、后端和测试应同步更新文档、OpenAPI 定义与测试用例。接口如未明确为 `P1`，一期不得据此扩展实现。若接口细节与一期 PRD 有冲突，以一期 PRD 的范围边界为准；若与完整产品功能细节有冲突，应评审后同时修订两个文档。

业务功能编号对应 `docs/产品需求书.md` 第 6 节功能编号。各 API 编号全局唯一，可用于前端 API 方法、后端控制器、测试用例和缺陷追踪。

## 2. 通用协议

### 2.1 基础约定

| 项目 | 约定 |
|---|---|
| 基础路径 | `/api/v1` |
| 数据格式 | UTF-8 JSON；请求和响应媒体类型为 `application/json` |
| 认证 | 除登录外均需登录；请求头 `Authorization: <token>`，令牌由 Sa-Token 管理 |
| 令牌获取 | 登录响应返回 `data.token`；请求头名称与仓库现有 `Authorization` 配置一致 |
| 方法语义 | GET 查询；POST 创建或提交动作；PUT 全量更新或状态变更；PATCH 仅在明确列出的部分更新接口使用；DELETE 仅用于未产生历史记录的草稿或可安全删除关系 |
| 标识符 | API 中 ID 使用字符串，避免 JavaScript 对 64 位整数精度丢失 |
| 时间 | ISO 8601，携带时区偏移；服务端存储 UTC，展示时按学校时区转换。示例：`2026-10-03T09:30:00+08:00` |
| 日期 | `YYYY-MM-DD` |
| 枚举 | 小写 `snake_case` 字符串；未知枚举前端应安全降级显示 |
| 布尔值 | JSON `true` / `false` |
| 排序 | `sortBy` 使用接口明确允许的字段，`sortOrder` 为 `asc` 或 `desc`；禁止直接传入 SQL 字段表达式 |
| 幂等 | 学习完成、测验提交、积分入账、通知已读等接口按本文约定处理重复请求 |

现有脚手架包含 `/login` 等系统示例接口；本文件规定英语学习业务统一使用 `/api/v1` 契约。实现时应将脚手架登录能力适配到本文路径，或通过明确的网关映射兼容旧路径，不得让前后端各自采用不同路径和响应结构。

前端不得传入或依赖客户端自行指定的 `schoolId` 作为授权依据。`studentId`、`classId`、`gradeId` 等资源标识只用于定位资源；服务端仍须根据当前用户角色和关联关系重新校验数据权限。
### 2.2 统一成功和失败响应

响应体遵循仓库现有 `ResponseDTO<T>` 结构：`code`、`level`、`msg`、`ok`、`data`、`dataType`。成功时 `code=0`、`ok=true`；失败时 `ok=false`，`data` 为 `null` 或错误详情。HTTP 状态码应与结果类别一致。

成功示例：

```json
{
  "code": 0,
  "level": null,
  "msg": "操作成功",
  "ok": true,
  "data": {
    "id": "10001",
    "title": "Unit 1 Vocabulary"
  },
  "dataType": 0
}
```

失败示例：

```json
{
  "code": 40301,
  "level": "ERROR",
  "msg": "无权访问该班级数据",
  "ok": false,
  "data": null,
  "dataType": 0
}
```

HTTP 状态码约定：`200` 查询/更新成功，`201` 创建成功，`202` 异步任务已接受，`204` 成功且无响应体仅用于明确声明的删除接口，`400` 参数或业务状态错误，`401` 未登录/令牌失效，`403` 角色或数据范围不足，`404` 资源不存在或对当前用户不可见，`409` 状态冲突/重复提交，`429` 请求频率受限，`500` 未预期服务错误。为避免枚举资源是否存在泄露个人信息，对越权的学生/家长详情可统一返回 `404`。

### 2.3 分页和筛选

分页请求参数：

| 字段 | 类型 | 默认值 | 约束 |
|---|---|---:|---|
| `pageNum` | integer | 1 | 从 1 开始，最小 1 |
| `pageSize` | integer | 20 | 1–100；超出范围返回 `400` |
| `sortBy` | string | 资源默认字段 | 只能使用接口声明的白名单字段 |
| `sortOrder` | string | `desc` | `asc` 或 `desc` |

分页响应 `data` 使用 `PageResult<T>` 字段：`pageNum`、`pageSize`、`total`、`pages`、`list`、`emptyFlag`。筛选参数均为可选，除非接口字段说明标为必填；多值筛选以重复 query 参数或 JSON 数组表示，具体以接口定义为准。

### 2.4 通用错误码

| 错误码 | HTTP | 含义 | 前端处理 |
|---:|---:|---|---|
| `0` | 200/201 | 成功 | 展示结果 |
| `40001` | 400 | 参数校验失败 | 在对应字段显示校验提示 |
| `40002` | 400 | 业务规则不满足 | 展示 `msg`，保留已填表单 |
| `40101` | 401 | 未登录或令牌过期 | 清理会话并跳转登录页 |
| `40301` | 403 | 角色无权执行该操作 | 展示无权限状态 |
| `40302` | 403/404 | 超出当前用户的数据范围 | 不展示数据；按隐私要求可映射为 404 |
| `40401` | 404 | 资源不存在或不可见 | 展示不存在/不可访问状态 |
| `40901` | 409 | 状态冲突，例如非草稿不能修改 | 刷新详情并提示当前状态 |
| `40902` | 409 | 重复提交或重复关系 | 幂等接口按原结果返回；非幂等接口提示冲突 |
| `40903` | 409 | 资源已被历史记录引用，禁止物理删除或修改关键字段 | 提示改为停用/下架 |
| `42901` | 429 | 登录或高风险操作触发频率限制 | 提示稍后重试 |
| `50001` | 500 | 服务端未预期错误 | 展示通用错误和追踪编号，不向用户暴露堆栈 |

字段校验失败可在 `data.fieldErrors` 返回字段错误列表：`[{"field":"title","reason":"不能为空"}]`。响应不得返回密码、密码摘要、令牌密钥或其他学生的敏感信息。

## 3. 身份、通用数据模型与权限

### 3.1 角色标识

| 角色 | 标识 | 默认数据范围 |
|---|---|---|
| 学校管理员 | `admin` | 当前学校全部业务数据 |
| 年级质量管理员 | `grade_quality_admin` | 被授权的一个或多个年级，只读分析 |
| 教师 | `teacher` | 自己授课或负责的班级及其学生 |
| 家长 | `parent` | 经学校确认绑定的一个或多个孩子，只读学习情况 |
| 学生 | `student` | 本人详细学习、任务、测验数据；榜单仅本人和附近名次 |

### 3.2 通用字段对象

以下为接口对外字段；不要求数据库列名与 API 字段名一致。

| 对象 | 字段及类型 | 说明 |
|---|---|---|
| `UserSummary` | `id:string`、`account:string`、`displayName:string`、`role:string`、`status:enabled\|disabled`、`grade?:GradeSummary`、`class?:ClassSummary` | 不包含密码和认证密钥 |
| `GradeSummary` | `id:string`、`name:string`、`sort:number` | 年级展示信息 |
| `ClassSummary` | `id:string`、`name:string`、`grade:GradeSummary`、`studentCount?:number` | 班级展示信息 |
| `ContentSummary` | `id:string`、`title:string`、`type:vocabulary\|grammar\|reading\|listening`、`gradeIds:string[]`、`status:draft\|published\|offline`、`updatedAt:datetime` | 已发布内容面向学生/教师提供 |
| `QuestionSummary` | `id:string`、`type:single_choice\|true_false`、`stem:string`、`options?:QuestionOption[]`、`score:number`、`knowledgeTag?:string` | 公开答题接口不得提前返回正确答案 |
| `TaskSummary` | `id:string`、`title:string`、`description:string`、`content:ContentSummary`、`classIds:string[]`、`status:draft\|published\|expired\|closed`、`dueAt?:datetime` | 状态依据任务规则和截止时间计算 |
| `QuizSummary` | `id:string`、`title:string`、`description?:string`、`questionCount:number`、`totalScore:number`、`status:draft\|published\|expired\|closed`、`dueAt?:datetime` | 学生视图不可包含答案 |
| `LearningRecord` | `id:string`、`studentId:string`、`contentId:string`、`taskId?:string`、`status:in_progress\|completed`、`startedAt?:datetime`、`completedAt?:datetime`、`updatedAt:datetime` | 详细对象只对授权角色返回 |
| `QuizAttempt` | `id:string`、`quizId:string`、`studentId:string`、`classId:string`、`status:submitted`、`score:number`、`totalScore:number`、`submittedAt:datetime`、`answers:AnswerResult[]` | 仅本人或授权教师/管理员可读；家长按绑定关系只读 |
| `NotificationSummary` | `id:string`、`title:string`、`body:string`、`targetType:string`、`createdAt:datetime`、`readAt?:datetime` | 站内通知，`P2` |
| `PageResult<T>` | `pageNum:number`、`pageSize:number`、`total:number`、`pages:number`、`list:T[]`、`emptyFlag:boolean` | 所有分页接口统一使用 |

### 3.3 数据权限检查

1. 管理员访问必须限制在当前学校；一期为单学校部署但仍保留学校归属校验。
2. 教师每次读写班级、任务、学生、测验时，必须验证该班级处于教师授权关系内；创建任务/测验时逐个校验所有目标班级。
3. 学生每次访问任务、测验、答题、记录时，必须验证本人是目标班级成员且资源已分配给该班级。
4. 家长每次读取孩子详情时，必须验证有效且已确认的家长—学生关系，不接受客户端传入关系 ID 作为唯一校验。
5. 年级质量管理员读取班级或学生概要时，必须验证其授权年级覆盖该班级。
6. 角色权限、数据权限和资源状态在后端检查；前端菜单/按钮隐藏不构成安全控制。

## 4. API 目录

| 领域 | 阶段 | 接口编号范围 | 主要角色 |
|---|---|---|---|
| 认证和个人中心 | `P1/P2` | `API-AUTH-*` | 全部角色 |
| 首页和仪表盘 | `P1/P2` | `API-DASH-*` | 全部角色 |
| 学校、年级、班级 | `P1/P2` | `API-ORG-*` | 管理员、教师只读、质量管理员只读 |
| 账号与家庭/授权关系 | `P1/P2` | `API-USER-*`、`API-REL-*` | 管理员、家长 |
| 学习内容和审核 | `P1/P2` | `API-CONTENT-*` | 管理员、教师、学生只读 |
| 题库 | `P1/P2` | `API-QUESTION-*` | 管理员、教师只读选择 |
| 学习任务 | `P1/P2` | `API-TASK-*` | 教师、学生、管理角色只读 |
| 测验和答题 | `P1/P2` | `API-QUIZ-*` | 教师、学生、家长只读、管理员 |
| 学习记录和报告 | `P1/P2` | `API-RECORD-*` | 全部按范围授权 |
| 积分与榜单 | `P2` | `API-POINT-*`、`API-RANK-*` | 管理员、教师、家长、学生、质量管理员 |
| 通知 | `P2` | `API-NOTICE-*` | 管理员、教师、家长、学生、质量管理员 |
| 导出与日志 | `P1/P2` | `API-EXPORT-*`、`API-LOG-*` | 管理员及授权报表角色 |

## 5. 认证和个人中心接口

所有 `/auth` 以外受保护的 API 需携带登录令牌。除明确标记外，响应中的 `data` 均为下述表格列出的结构。

| ID / 阶段 | 方法与路径 | 角色/数据范围 | 请求 | 响应与规则 |
|---|---|---|---|---|
| `API-AUTH-001` `P1` | `POST /auth/login` | 公开 | `{account:string,password:string}` | `data:{token:string,expiresAt:datetime,firstLogin:boolean,user:UserSummary}`。账号停用不能登录；失败提示不得暴露账号是否存在；写登录成功/失败日志 |
| `API-AUTH-002` `P1` | `POST /auth/logout` | 全部已登录角色，本人会话 | 无 | `data:null`。撤销当前令牌；重复退出可按成功处理 |
| `API-AUTH-003` `P1` | `GET /auth/me` | 全部角色，本人 | 无 | `data:{user:UserSummary,permissions:string[],firstLogin:boolean}`。不包含密码字段 |
| `API-AUTH-004` `P1` | `PUT /auth/password` | 全部角色，本人 | `{oldPassword:string,newPassword:string,confirmPassword:string}` | `data:null`。验证原密码和密码策略；成功后使其他会话失效，并写安全日志 |
| `API-AUTH-005` `P1` | `POST /auth/first-password` | 首次登录用户本人 | `{newPassword:string,confirmPassword:string}` | `data:{firstLogin:false}`。首次改密前仅允许访问该接口和退出接口 |
| `API-AUTH-006` `P2` | `GET /profile/relationships` | `parent`，本人 | 无 | `data:RelationshipSummary[]`，只列学校已确认绑定关系 |

## 6. 首页和仪表盘接口

首页接口按角色返回不同聚合对象；只返回首页所需汇总，不默认携带全量个人明细。首页空状态以零值和空数组表示。

| ID / 阶段 | 方法与路径 | 角色/数据范围 | 请求 | 响应字段 |
|---|---|---|---|---|
| `API-DASH-001` `P1` | `GET /dashboard/admin` | `admin`，本校 | `from?:date,to?:date` | `{organization:{gradeCount,classCount,teacherCount,studentCount},content:{publishedCount},learning:{taskCount,quizCount,activeStudentCount,completionRate},recentOperations:OperationSummary[]}`。不含导出文件 |
| `API-DASH-002` `P1` | `GET /dashboard/teacher` | `teacher`，授权班级 | `classId?:string,from?:date,to?:date` | `{classes:ClassSummary[],pendingTaskCount,upcomingQuizCount,completionRate,needsAttention:StudentAttentionSummary[]}`。学生摘要仅授权班级 |
| `API-DASH-003` `P1` | `GET /dashboard/student` | `student`，本人 | `from?:date,to?:date` | `{today:{completedCount,pendingTaskCount},recentContents:ContentSummary[],recentRecords:LearningRecordSummary[],recentScores:ScoreSummary[]}` |
| `API-DASH-004` `P2` | `GET /dashboard/quality` | `grade_quality_admin`，授权年级 | `gradeId?:string,from?:date,to?:date` | `{gradeSummaries:GradeMetric[],classComparisons:ClassMetric[],attentionItems:AttentionSummary[]}`。只返回授权年级汇总 |
| `API-DASH-005` `P2` | `GET /dashboard/parent` | `parent`，已确认绑定孩子 | `studentId?:string,from?:date,to?:date` | `{children:ChildSummary[],selectedChild:ChildLearningSummary,upcomingTasks:TaskSummary[],recentScores:ScoreSummary[],notices:NotificationSummary[]}`。`studentId` 必须属于本人有效绑定 |

## 7. 组织、账号与关联关系接口

### 7.1 学校、年级和班级

| ID / 阶段 | 方法与路径 | 角色/数据范围 | 请求字段 | 响应/规则 |
|---|---|---|---|---|
| `API-ORG-001` `P1` | `GET /admin/school` | `admin`，本校 | 无 | `{id,name,status,timezone}`；一期单学校只返回当前学校 |
| `API-ORG-002` `P1` | `PUT /admin/school` | `admin`，本校 | `{name:string,timezone:string}` | 返回更新后的学校摘要；不支持创建其他学校 |
| `API-ORG-003` `P1` | `GET /organization/grades` | `admin`；教师/质量管理员只能读授权范围 | `status?:string,pageNum?,pageSize?` | `PageResult<GradeSummary>`；质量管理员按授权年级过滤 |
| `API-ORG-004` `P1` | `POST /admin/grades` | `admin` | `{name:string,sort:number}` | `201 data:GradeSummary`；年级名称在学校内唯一 |
| `API-ORG-005` `P1` | `PUT /admin/grades/{gradeId}` | `admin` | `{name:string,sort:number,status:enabled\|disabled}` | `data:GradeSummary`；存在关联班级时停用不删除历史数据 |
| `API-ORG-006` `P1` | `GET /organization/classes` | `admin`；教师/质量管理员限定授权范围 | `gradeId?:string,status?:string,keyword?:string,pageNum?,pageSize?` | `PageResult<ClassSummary>` |
| `API-ORG-007` `P1` | `POST /admin/classes` | `admin` | `{gradeId:string,name:string}` | `201 data:ClassSummary`；班级名称在年级内唯一 |
| `API-ORG-008` `P1` | `PUT /admin/classes/{classId}` | `admin` | `{gradeId:string,name:string,status:enabled\|disabled}` | 更新班级摘要；有学习历史时不得物理删除或清除归属 |
| `API-ORG-009` `P1` | `GET /organization/classes/{classId}/students` | `admin`；教师仅该班授权 | `keyword?:string,status?:string,pageNum?,pageSize?` | `PageResult<UserSummary>`；后端校验班级范围 |
| `API-ORG-010` `P1` | `PUT /admin/students/{studentId}/class` | `admin` | `{classId:string}` | `{student:UserSummary,class:ClassSummary}`；一期更新当前归属，不实现转班工作流；保留历史记录快照 |
| `API-ORG-011` `P1` | `GET /teacher/classes` | `teacher`，本人授课班级 | `pageNum?,pageSize?` | `PageResult<ClassSummary>` |
| `API-ORG-012` `P2` | `GET /quality/grades/{gradeId}/classes` | `grade_quality_admin`，指定授权年级 | `pageNum?,pageSize?` | `PageResult<ClassSummary>`；未授权年级返回 `403/404` |
| `API-ORG-013` `P2` | `GET /admin/quality-admins/{userId}/grades` | `admin`；管理员维护授权 | 无 | `data:GradeSummary[]` |
| `API-ORG-014` `P2` | `PUT /admin/quality-admins/{userId}/grades` | `admin` | `{gradeIds:string[]}` | `data:GradeSummary[]`；整体事务更新授权关系 |

### 7.2 用户账号和家长关系

| ID / 阶段 | 方法与路径 | 角色/数据范围 | 请求字段 | 响应/规则 |
|---|---|---|---|---|
| `API-USER-001` `P1` | `GET /admin/users` | `admin`，本校 | `role?:string,gradeId?:string,classId?:string,status?:string,keyword?:string,pageNum?,pageSize?` | `PageResult<UserSummary>`；不可返回密码或口令摘要 |
| `API-USER-002` `P1` | `POST /admin/users` | `admin` | `{account:string,displayName:string,role:teacher\|student,gradeId?:string,classId?:string,initialPassword:string}` | `201 data:UserSummary`；账号本校唯一；密码安全哈希；首次登录强制改密；学生须关联班级 |
| `API-USER-003` `P1` | `GET /admin/users/{userId}` | `admin`，本校；本人可经 `/auth/me` 查询 | 无 | `data:UserSummary`；如含组织关系需按本校校验 |
| `API-USER-004` `P1` | `PUT /admin/users/{userId}` | `admin` | `{displayName:string,role:teacher\|student,gradeId?:string,classId?:string}` | `data:UserSummary`；角色变更需保留操作日志并验证关系一致性 |
| `API-USER-005` `P1` | `PUT /admin/users/{userId}/status` | `admin` | `{status:enabled\|disabled}` | `data:UserSummary`；停用即时阻断登录，历史记录保留 |
| `API-USER-006` `P1` | `POST /admin/users/{userId}/reset-password` | `admin` | 无 | `{temporaryPassword:string,requireChange:true}`；仅一次性返回临时密码或按部署安全策略重置，禁止写入日志明文 |
| `API-USER-007` `P1` | `PUT /admin/teachers/{teacherId}/classes` | `admin` | `{classIds:string[]}` | `data:ClassSummary[]`；验证所有班级属于本校，事务替换授课关系 |
| `API-USER-008` `P2` | `POST /admin/users/import` | `admin` | `multipart/form-data` 文件或已确认的 JSON 批次格式 | `ImportResult:{createdCount,skippedCount,errors[]}`；写入导入审计；每行错误可定位 |
| `API-USER-009` `P2` | `GET /admin/users/imports/{importId}` | `admin` | 无 | 导入批次状态、汇总和错误列表 |
| `API-REL-001` `P2` | `GET /admin/parents/{parentId}/children` | `admin`；家长只能查询本人关系 | `status?:string,pageNum?,pageSize?` | `PageResult<RelationshipSummary>` |
| `API-REL-002` `P2` | `POST /admin/parents/{parentId}/children` | `admin` | `{studentId:string,relationshipType?:string}` | `201 data:RelationshipSummary`；防止重复关系；执行学校归属检查 |
| `API-REL-003` `P2` | `PUT /admin/relationships/{relationshipId}/status` | `admin` | `{status:pending\|confirmed\|rejected\|revoked}` | 更新关系；撤销后立即停止家长访问该孩子数据 |
| `API-REL-004` `P2` | `POST /parent/relationships/requests` | `parent`，本人 | `{studentAccount:string,relationshipType:string}` | `201 data:RelationshipSummary`；请求待学校确认，不立即授予学习数据访问权 |
| `API-REL-005` `P2` | `GET /parent/children` | `parent`，本人有效绑定 | `pageNum?,pageSize?` | `PageResult<ChildSummary>` |

## 8. 学习内容、审核和题库接口

### 8.1 学习内容

| ID / 阶段 | 方法与路径 | 角色/数据范围 | 请求字段 | 响应/规则 |
|---|---|---|---|---|
| `API-CONTENT-001` `P1` | `GET /contents` | 登录用户；学生仅已发布且适用本人年级，教师可浏览已发布内容，管理员本校 | `type?:vocabulary\|grammar\|reading\|listening,gradeId?:string,keyword?:string,pageNum?,pageSize?` | `PageResult<ContentSummary>`；学生请求草稿/下架内容不能获得 |
| `API-CONTENT-002` `P1` | `GET /contents/{contentId}` | 管理员本校；教师可见已发布内容；学生仅适用且已发布内容 | 无 | `ContentDetail:{...ContentSummary,body:string,resources:ResourceSummary[],questions:QuestionSummary[]}`；返回学生视图时不得含正确答案 |
| `API-CONTENT-003` `P1` | `GET /admin/contents` | `admin`，本校 | `type?,status?,gradeId?,keyword?,pageNum?,pageSize?` | `PageResult<ContentSummary>`，包含草稿和下架内容 |
| `API-CONTENT-004` `P1` | `POST /admin/contents` | `admin` | `{title,type,gradeIds:string[],body,resources?:ResourceInput[],sourceNote?:string,questionIds?:string[]}` | `201 data:ContentDetail`；新建为 `draft`；资源使用权由提交者确认 |
| `API-CONTENT-005` `P1` | `PUT /admin/contents/{contentId}` | `admin` | 与创建内容字段相同 | 更新草稿；已发布内容如果有关联历史记录，关键字段变更须保留审计或要求下架后新建版本 |
| `API-CONTENT-006` `P1` | `PUT /admin/contents/{contentId}/status` | `admin` | `{status:published\|offline}` | 返回 `ContentSummary`；只有资料完整的草稿可发布；下架禁止新建任务但不删历史记录 |
| `API-CONTENT-007` `P2` | `POST /teacher/contents` | `teacher`，内容适用授权班级 | `{title,type,gradeIds:string[],body,resources?:ResourceInput[],sourceNote?:string}` | `201 data:ContentDetail`，状态为 `pending_review`；审核通过前不可用于学生学习或正式任务 |
| `API-CONTENT-008` `P2` | `GET /teacher/contents/submissions` | `teacher`，本人提交 | `status?:string,pageNum?,pageSize?` | `PageResult<ContentSummary>`，返回审核意见 |
| `API-CONTENT-009` `P2` | `GET /admin/content-submissions` | `admin`，本校 | `status?:pending_review\|returned\|approved&pageNum?,pageSize?` | `PageResult<ContentSummary>` |
| `API-CONTENT-010` `P2` | `PUT /admin/content-submissions/{contentId}/review` | `admin` | `{decision:approved\|returned,comment:string}` | 更新审核状态、审核人和时间；`returned` 必须说明修改意见；通过后可显式发布 |

内容类型：`vocabulary`、`grammar`、`reading`、`listening`。内容状态：`draft`、`pending_review`、`returned`、`approved`、`published`、`offline`。资源 `ResourceSummary` 至少包含 `id`、`type:text\|image\|audio`、`url`、`title?`、`licenseNote?`；一期可使用已准备的合法资源 URL，不提供任意网络抓取。

### 8.2 客观题库

| ID / 阶段 | 方法与路径 | 角色/数据范围 | 请求字段 | 响应/规则 |
|---|---|---|---|---|
| `API-QUESTION-001` `P1` | `GET /admin/questions` | `admin`，本校 | `type?:single_choice\|true_false,status?:string,contentId?:string,keyword?,pageNum?,pageSize?` | `PageResult<QuestionDetail>`；管理员可查看答案 |
| `API-QUESTION-002` `P1` | `POST /admin/questions` | `admin` | `{type,stem,options?:QuestionOptionInput[],correctAnswer:string,score:number,contentId?:string,knowledgeTag?:string}` | `201 data:QuestionDetail`；单选题须有至少两个选项且唯一答案；判断题答案为 `true`/`false` |
| `API-QUESTION-003` `P1` | `PUT /admin/questions/{questionId}` | `admin` | 与创建题目相同 | 已用于已提交测验的题目不得改写历史判分依据；可停用或创建新题版本 |
| `API-QUESTION-004` `P1` | `PUT /admin/questions/{questionId}/status` | `admin` | `{status:enabled\|disabled}` | 更新状态；被历史引用不允许物理删除 |
| `API-QUESTION-005` `P1` | `GET /teacher/questions` | `teacher`，仅查询可用于授权教学的题目 | `contentId?:string,type?,keyword?,pageNum?,pageSize?` | `PageResult<QuestionSummary>`；不向未授权教师暴露不适用题库 |
| `API-QUESTION-006` `P2` | `POST /admin/questions/import` | `admin` | 经后续确定的导入格式 | 返回导入批次结果；不属于一期 |

`QuestionOption` 字段为 `{key:string,text:string}`。学生测验接口只返回选项，不返回 `correctAnswer`；正确答案只能在提交后按结果展示规则返回。

## 9. 学习任务接口

任务创建及发布只允许教师对授权班级操作。包含多目标班级的操作必须逐个校验权限，并在一个事务中保存任务和班级关系。

| ID / 阶段 | 方法与路径 | 角色/数据范围 | 请求字段 | 响应/规则 |
|---|---|---|---|---|
| `API-TASK-001` `P1` | `GET /teacher/tasks` | `teacher`，本人任务和授权班级 | `classId?:string,status?:string,from?:date,to?:date,pageNum?,pageSize?` | `PageResult<TaskSummary>` |
| `API-TASK-002` `P1` | `POST /teacher/tasks` | `teacher` | `{title:string,description:string,contentId:string,classIds:string[],dueAt?:datetime}` | `201 data:TaskDetail`，初始 `draft`；内容必须已发布，班级必须全部授权 |
| `API-TASK-003` `P1` | `PUT /teacher/tasks/{taskId}` | `teacher`，本人创建且草稿 | 与创建任务字段相同 | 更新草稿；已发布任务不允许无审计地修改核心字段 |
| `API-TASK-004` `P1` | `PUT /teacher/tasks/{taskId}/publish` | `teacher`，本人创建 | `{}` | 状态变为 `published`；需有目标班级、有效已发布内容；记录发布时间 |
| `API-TASK-005` `P1` | `PUT /teacher/tasks/{taskId}/close` | `teacher`，本人创建且授权班级 | `{reason?:string}` | 状态变为 `closed`；历史完成记录保留 |
| `API-TASK-006` `P1` | `GET /teacher/tasks/{taskId}` | `teacher`，本人任务且目标班级授权 | 无 | `TaskDetail`，包含目标班级和状态 |
| `API-TASK-007` `P1` | `GET /teacher/tasks/{taskId}/progress` | `teacher`，该任务目标班级 | `classId?:string,pageNum?,pageSize?` | `{summary:{assignedCount,completedCount,completionRate},students:PageResult<TaskStudentProgress>}`；可见目标班级学生 |
| `API-TASK-008` `P1` | `GET /student/tasks` | `student`，本人所属班级 | `status?:pending\|completed\|expired,classId?:string,pageNum?,pageSize?` | `PageResult<StudentTaskSummary>`；只返回已发布且分配给本人的任务 |
| `API-TASK-009` `P1` | `GET /student/tasks/{taskId}` | `student`，本人目标班级 | 无 | `StudentTaskDetail`，包括关联内容和本人完成状态，不含其他学生数据 |
| `API-TASK-010` `P1` | `POST /student/tasks/{taskId}/complete` | `student`，本人目标班级 | `{completedAt?:datetime}` | `LearningRecord`；重复请求返回原完成记录（幂等）；截止后是否允许提交按一期 PRD 确认规则执行 |
| `API-TASK-011` `P2` | `GET /admin/tasks` | `admin`，本校 | `classId?,teacherId?,status?,from?,to?,pageNum?,pageSize?` | `PageResult<TaskSummary>`；管理汇总和查询 |
| `API-TASK-012` `P2` | `POST /teacher/tasks/{taskId}/notice` | `teacher`，目标班级授权 | `{title:string,body:string}` | 创建站内提醒；通知写入审计，不能发给非目标班级 |

任务状态：`draft`、`published`、`expired`、`closed`。是否禁止逾期后首次完成必须在开发前由一期规则冻结；接口不得自行假设逾期策略。

## 10. 固定客观题测验和答题接口

测验每次提交须在同一事务中校验状态、保存提交、逐题保存答题结果、计算总分并生成错题。唯一约束应保证一名学生对同一测验只有一次有效提交；相同幂等键的网络重试返回原提交结果，不重复计分。

| ID / 阶段 | 方法与路径 | 角色/数据范围 | 请求字段 | 响应/规则 |
|---|---|---|---|---|
| `API-QUIZ-001` `P1` | `GET /teacher/quizzes` | `teacher`，本人创建及授权班级 | `classId?:string,status?:string,pageNum?,pageSize?` | `PageResult<QuizSummary>` |
| `API-QUIZ-002` `P1` | `POST /teacher/quizzes` | `teacher` | `{title:string,description?:string,questionItems:[{questionId:string,score:number,sort:number}],classIds:string[],dueAt?:datetime}` | `201 data:QuizDetail`，初始 `draft`；只允许单选/判断题；计算并返回总分 |
| `API-QUIZ-003` `P1` | `PUT /teacher/quizzes/{quizId}` | `teacher`，本人创建且草稿 | 与创建字段相同 | 更新草稿；发布后题目、答案和分值快照锁定 |
| `API-QUIZ-004` `P1` | `PUT /teacher/quizzes/{quizId}/publish` | `teacher`，本人创建 | `{}` | 发布测验；校验目标班级、题目状态、总分和截止时间 |
| `API-QUIZ-005` `P1` | `PUT /teacher/quizzes/{quizId}/close` | `teacher`，本人创建并授权 | `{reason?:string}` | 关闭后不可新提交；保留既有答题和成绩 |
| `API-QUIZ-006` `P1` | `GET /teacher/quizzes/{quizId}` | `teacher`，授权范围 | 无 | `QuizDetail`，教师可见题目和正确答案 |
| `API-QUIZ-007` `P1` | `GET /teacher/quizzes/{quizId}/results` | `teacher`，目标班级 | `classId?:string,pageNum?,pageSize?` | `{summary:{participantCount,averageScore,scoreDistribution},attempts:PageResult<AttemptSummary>}` |
| `API-QUIZ-008` `P1` | `GET /teacher/quizzes/{quizId}/results/{attemptId}` | `teacher`，学生属于目标班级 | 无 | `QuizAttempt` 含逐题作答、标准答案、对错和得分 |
| `API-QUIZ-009` `P1` | `GET /student/quizzes` | `student`，本人班级 | `status?:available\|submitted\|expired,pageNum?,pageSize?` | `PageResult<StudentQuizSummary>`；不含正确答案 |
| `API-QUIZ-010` `P1` | `GET /student/quizzes/{quizId}` | `student`，目标班级 | 无 | `StudentQuizDetail` 含题干选项、题目分值，不含正确答案；已提交时返回本人结果状态 |
| `API-QUIZ-011` `P1` | `POST /student/quizzes/{quizId}/submit` | `student`，目标班级且可提交 | Header `Idempotency-Key:string`；Body `{answers:[{questionId:string,answer:string}]}` | `QuizAttempt`；服务端重算正确答案和得分；重复有效提交返回原结果或 `40902`；保存提交时间 |
| `API-QUIZ-012` `P1` | `GET /student/quiz-attempts/{attemptId}` | `student`，仅本人 | 无 | `QuizAttempt`，含得分、逐题结果和错题；不可访问他人 attempt |
| `API-QUIZ-013` `P2` | `GET /parent/children/{studentId}/quiz-results` | `parent`，已确认绑定 | `from?:date,to?:date,pageNum?,pageSize?` | `PageResult<AttemptSummary>`；仅绑定孩子的结果 |
| `API-QUIZ-014` `P2` | `GET /admin/quiz-statistics` | `admin`，本校汇总 | `gradeId?,classId?,from?,to?` | `QuizStatistics`；一期不要求管理端复杂统计 |

`AnswerResult` 至少包括 `questionId`、`studentAnswer`、`correct`、`earnedScore`、`totalScore`、`correctAnswer`（仅提交后按角色策略返回）。不得通过题目详情、缓存或其他列表提前泄露标准答案。

## 11. 学习记录、错题、分析和报表

### 11.1 学生记录和教师/管理分析

| ID / 阶段 | 方法与路径 | 角色/数据范围 | 请求字段 | 响应/规则 |
|---|---|---|---|---|
| `API-RECORD-001` `P1` | `GET /student/learning-records` | `student`，本人 | `type?,status?,from?,to?,pageNum?,pageSize?` | `PageResult<LearningRecord>` |
| `API-RECORD-002` `P1` | `POST /student/learning-records` | `student`，本人可用内容 | `{contentId:string,taskId?:string,event:start\|complete,occurredAt?:datetime}` | `LearningRecord`；`start/complete` 重放不得重复生成完成记录；服务器记录可信完成时间 |
| `API-RECORD-003` `P1` | `GET /student/wrong-answers` | `student`，本人 | `contentType?,from?,to?,pageNum?,pageSize?` | `PageResult<WrongAnswerSummary>`；来源为已自动判错的客观题 |
| `API-RECORD-004` `P1` | `GET /student/learning-summary` | `student`，本人 | `from?:date,to?:date` | `{byContentType:[{type,completedCount,wrongCount}],totalCompleted,totalWrong}`；不推导未定义的能力等级 |
| `API-RECORD-005` `P1` | `GET /teacher/students/{studentId}/learning-report` | `teacher`，学生属于本人授权班级 | `from?:date,to?:date` | `{student:UserSummary,learningSummary,taskSummary,quizSummary,wrongAnswerSummary}` |
| `API-RECORD-006` `P1` | `GET /teacher/classes/{classId}/learning-summary` | `teacher`，该班已授权 | `from?:date,to?:date` | `{class:ClassSummary,participation,taskCompletion,quizScores,wrongAnswersByContent}`；只返回可解释的基础统计 |
| `API-RECORD-007` `P1` | `GET /admin/learning-summary` | `admin`，本校 | `gradeId?,classId?,from?,to?` | 学校基础汇总；不含跨校统计 |
| `API-RECORD-008` `P2` | `GET /quality/grades/{gradeId}/learning-analysis` | `grade_quality_admin`，授权年级 | `from?,to?,contentType?` | `{grade,classes:ClassMetric[],contentTypeMetrics,teacherActivityMetrics}`；指标仅用于观察系统记录，不直接形成教师绩效结论 |
| `API-RECORD-009` `P2` | `GET /quality/students/{studentId}/summary` | `grade_quality_admin`，学生所在授权年级 | `from?,to?` | 学生概要及学习统计；不得返回非必要家庭联系方式 |
| `API-RECORD-010` `P2` | `GET /parent/children/{studentId}/learning-report` | `parent`，已确认绑定 | `from?,to?` | 学习、任务、测验和错题摘要；仅当前绑定孩子 |
| `API-RECORD-011` `P2` | `GET /parent/children/{studentId}/wrong-answers` | `parent`，已确认绑定 | `contentType?,pageNum?,pageSize?` | `PageResult<WrongAnswerSummary>` |

### 11.2 报表导出

| ID / 阶段 | 方法与路径 | 角色/数据范围 | 请求字段 | 响应/规则 |
|---|---|---|---|---|
| `API-EXPORT-001` `P2` | `POST /reports/exports` | `admin`、授权 `teacher`、授权 `grade_quality_admin` | `{reportType:string,filters:object,format:csv\|xlsx}` | `202 data:{exportId,status:queued}`；服务端重新校验范围和字段 |
| `API-EXPORT-002` `P2` | `GET /reports/exports/{exportId}` | 创建者本人；管理员按管理策略查询 | 无 | `{exportId,status:queued\|processing\|completed\|failed,downloadUrl?,expiresAt?,error?}` |
| `API-EXPORT-003` `P2` | `GET /reports/exports/{exportId}/download` | 创建者本人且仍有数据权限 | 无 | 文件流；过期或越权拒绝；下载写入审计日志 |

报表导出不得扩大调用人的数据权限。导出字段应最小化，不默认导出学生联系方式、家庭关系或不必要的逐题明细。导出任务状态、筛选条件、操作者和下载时间须可追溯。

## 12. 积分和榜单接口

所有积分和排名接口为 `P2`，一期不实现。积分变更应有来源事件、规则版本和幂等键；不得接受客户端直接提交积分值。

| ID / 阶段 | 方法与路径 | 角色/数据范围 | 请求字段 | 响应/规则 |
|---|---|---|---|---|
| `API-POINT-001` `P2` | `GET /admin/point-rules` | `admin`，本校 | `status?:string,pageNum?,pageSize?` | `PageResult<PointRule>` |
| `API-POINT-002` `P2` | `POST /admin/point-rules` | `admin` | `{name,behavior,points,repeatPolicy,dailyLimit,status}` | 新建积分规则；参数范围和重复条件由产品规则另行冻结 |
| `API-POINT-003` `P2` | `PUT /admin/point-rules/{ruleId}` | `admin` | 规则字段 | 变更只影响生效时间后的事件，保留规则历史版本 |
| `API-POINT-004` `P2` | `GET /admin/points/ledger` | `admin`，本校 | `studentId?,classId?,from?,to?,pageNum?,pageSize?` | `PageResult<PointLedgerEntry>` |
| `API-POINT-005` `P2` | `POST /admin/points/adjustments` | `admin`，本校 | `{studentId,delta,reason,idempotencyKey}` | 创建调整流水；必须记录操作人和原因，严禁覆盖历史流水 |
| `API-POINT-006` `P2` | `GET /student/points/summary` | `student`，本人 | `period?:week\|month\|term` | `{totalPoints,period,rank?,nearbyRanks?}`；只返回本人及有限邻近榜单信息 |
| `API-POINT-007` `P2` | `GET /teacher/classes/{classId}/points` | `teacher`，授权班级 | `period, pageNum?,pageSize?` | 班级范围积分汇总；学生隐私按榜单规则处理 |
| `API-POINT-008` `P2` | `GET /parent/children/{studentId}/points` | `parent`，已确认绑定 | `period` | 绑定孩子积分摘要及允许展示的排名 |
| `API-POINT-009` `P2` | `GET /quality/grades/{gradeId}/points` | `grade_quality_admin`，授权年级 | `period` | 授权年级汇总，不提供越级学校明细 |
| `API-RANK-001` `P2` | `GET /rankings` | 按角色返回允许的范围 | `scope:class\|grade\|school,period:week\|month\|term,classId?,gradeId?,pageNum?,pageSize?` | `{period,scope,generatedAt,entries:RankEntry[],currentUserRank?}`；学生仅本人及附近名次；家长仅绑定孩子；服务端按角色裁剪明细 |

## 13. 通知接口

所有通知接口为 `P2`，限站内 Web 通知；一期不提供短信、微信或站外推送。

| ID / 阶段 | 方法与路径 | 角色/数据范围 | 请求字段 | 响应/规则 |
|---|---|---|---|---|
| `API-NOTICE-001` `P2` | `GET /notifications` | 当前用户可见通知 | `unreadOnly?:boolean,from?,to?,pageNum?,pageSize?` | `PageResult<NotificationSummary>` |
| `API-NOTICE-002` `P2` | `GET /notifications/{notificationId}` | 当前接收人 | 无 | 通知详情；非接收人返回 `404` |
| `API-NOTICE-003` `P2` | `PUT /notifications/{notificationId}/read` | 当前接收人 | `{}` | `{id,readAt}`；重复标记已读返回相同结果 |
| `API-NOTICE-004` `P2` | `PUT /notifications/read-all` | 当前用户 | `{}` | `{updatedCount:number}`；只更新当前用户收到的通知 |
| `API-NOTICE-005` `P2` | `POST /teacher/classes/{classId}/notifications` | `teacher`，授权班级 | `{title:string,body:string,taskId?:string}` | `201 data:NotificationSummary`；记录目标班级和发布人 |
| `API-NOTICE-006` `P2` | `PUT /teacher/notifications/{notificationId}` | `teacher`，本人发布且仍可编辑 | `{title:string,body:string}` | 更新通知；发布后能否编辑按通知状态规则控制并审计 |
| `API-NOTICE-007` `P2` | `PUT /teacher/notifications/{notificationId}/withdraw` | `teacher`，本人发布 | `{reason?:string}` | 撤回新接收；已读历史保留 |
| `API-NOTICE-008` `P2` | `GET /admin/notifications` | `admin`，本校 | `status?,pageNum?,pageSize?` | `PageResult<NotificationSummary>` |

## 14. 操作日志接口

| ID / 阶段 | 方法与路径 | 角色/数据范围 | 请求字段 | 响应/规则 |
|---|---|---|---|---|
| `API-LOG-001` `P1` | `GET /admin/operation-logs` | `admin`，本校 | `operatorId?,action?,targetType?,result?,from?,to?,pageNum?,pageSize?` | `PageResult<OperationLogSummary>`；不得返回密码、令牌或敏感请求体 |
| `API-LOG-002` `P1` | `GET /admin/operation-logs/{logId}` | `admin`，本校 | 无 | `OperationLogDetail`，敏感字段脱敏；日志不可编辑和删除 |
| `API-LOG-003` `P2` | `GET /admin/security-events` | `admin`，本校 | `eventType?,from?,to?,pageNum?,pageSize?` | 登录失败、越权拒绝、账号安全和导出等事件摘要 |

日志至少包含 `id`、`operatorId?`、`operatorRole?`、`action`、`targetType`、`targetId?`、`occurredAt`、`result`、`traceId?`。登录失败等未认证事件可无操作人 ID，但不得记录密码。

## 15. 核心写操作约束和状态迁移

### 15.1 状态约束

| 资源 | 状态 | 允许迁移 | 限制 |
|---|---|---|---|
| 用户 | `enabled`、`disabled` | 管理员启用/停用 | 停用不删除历史记录；停用后令牌失效 |
| 内容 | `draft`、`pending_review`、`returned`、`approved`、`published`、`offline` | 按角色和审核步骤迁移 | 只有已发布内容可被学生学习和被教师用于新任务 |
| 任务 | `draft`、`published`、`expired`、`closed` | 创建草稿、发布、按截止时间过期或手动关闭 | 已发布任务核心字段不可无记录修改 |
| 测验 | `draft`、`published`、`expired`、`closed` | 创建草稿、发布、到期或关闭 | 发布后题目和分值锁定；默认单次提交 |
| 家长关系 | `pending`、`confirmed`、`rejected`、`revoked` | 学校确认或撤销 | 只有 `confirmed` 才能访问孩子数据 |
| 通知 | `active`、`withdrawn` | 发布后可按规则撤回 | 已产生的阅读状态保留 |

### 15.2 幂等和事务

- 测验提交：必须使用 `Idempotency-Key`；重复键且请求内容相同返回首次结果，不同内容返回 `40902`。数据库须有学生与测验的唯一约束。
- 任务完成：基于学生、任务唯一记录；重复完成请求返回既有记录。
- 学习记录开始/完成：重复事件不能增加完成次数；采用服务端时间作为权威时间，客户端时间仅供诊断。
- 创建任务：任务主体及目标班级关系单事务；任一班级越权或数据校验失败时整体回滚。
- 创建测验：测验、题目快照、班级关系单事务；任一项失败时整体回滚。
- 提交测验：提交记录、答案、计分和错题单事务；失败不能留下部分成绩或部分答题明细。
- 批量账号、导入和导出：记录批次状态、操作者和结果；部分成功/全量回滚策略由导入确认规则决定，不得静默丢行。
- 积分：只有 `P2`；从可信业务事件生成不可覆盖的流水，修正只能新增反向/调整流水。

## 16. 与页面及阶段的追踪关系

一期必需接口域：认证、个人中心、管理员首页、组织账号、内容和题库管理、内容浏览、教师班级/学生概况、任务、测验、学习记录、错题、基础统计和操作日志。具体接口为本文件中标注 `P1` 的项目。

二期接口域：家长和年级质量管理员首页/报告、绑定关系、教师内容审核、积分和榜单、通知、报表导出、批量导入及高级分析。上述能力均标注 `P2`，未完成配套产品规则和隐私审核前不得仅凭接口草案直接交付。

每个产品页面功能编号应至少关联一个 API 编号；反向地，每个 API 必须有关联页面或明确标记为内部/系统接口。新增接口不能绕过对应页面功能、数据权限和验收用例的评审。

## 17. 前后端联调检查清单

- [ ] API 路径、HTTP 方法、阶段和功能编号已在需求、前端 API 模块及后端 Controller 中一致。
- [ ] 每个接口声明登录状态、角色列表和具体数据权限，不以角色判断替代班级/家庭/年级范围检查。
- [ ] 每个写接口声明状态前置条件、并发/重复提交行为和审计要求。
- [ ] 分页接口统一使用 `pageNum`、`pageSize`、`total`、`pages`、`list`、`emptyFlag`。
- [ ] ID 前后端均按字符串处理；时间字段包含时区信息。
- [ ] 客观题答案仅在允许的结果接口返回；学生提交时由服务器重新判分。
- [ ] 家长关系未确认、教师班级未授权、质量管理员年级未授权、学生非本人资源都无法通过改 URL 或请求体绕过。
- [ ] 错误码和 HTTP 状态码一致；页面处理未登录、无权限、资源不可见、状态冲突、重复提交和空数据。
- [ ] 导出和日志接口仅向有权角色开放，导出字段做未成年人数据最小化处理。
- [ ] 一期功能只依赖 `P1` 接口；`P2/P3` 不成为一期上线阻断项。

## 18. 变更记录

| 日期 | 版本 | 变更内容 |
|---|---|---|
| 2026-10-03 | V1.0 | 建立覆盖五类角色的完整产品 API 契约，区分 P1/P2/P3 阶段，定义统一响应、权限、分页、错误、数据模型、业务接口和联调约束。 |
