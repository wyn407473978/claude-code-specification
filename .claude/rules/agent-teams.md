# Agent Teams 协作规范

本文档用于约束 Claude Code 在复杂需求中使用多个 Agent 协作开发。目标是让多个 Agent 可以分工明确、并行高效、交接清楚，并避免重复修改、上下文分裂和文件冲突。

## 1. 启用原则

默认不启用多 Agent。只有当需求具备明确拆分价值时，才使用 Agent Teams。

适合启用 Agent Teams 的场景：

- 跨前端、后端、数据库、测试或文档的中大型功能。
- 涉及权限、支付、数据迁移、文件上传、AI 调用、隐私数据等高风险功能。
- 需要先设计方案，再由不同 Agent 分别实现，再统一审查的任务。
- 一次需求中包含多个相对独立的模块，且能明确文件边界。

不建议启用 Agent Teams 的场景：

- 单文件修改、小 Bug 修复、文案调整、样式微调。
- 需求范围不清，无法拆分职责。
- 多个 Agent 必然同时修改同一批文件。
- 项目还没有 README、计划书、启动命令或基础目录，无法形成共同上下文。

## 2. 总体协作模型

多 Agent 协作必须采用“一个主 Agent 负责统筹，多个专业 Agent 负责子任务”的模式。

默认角色：

- `Lead Agent`：需求理解、阶段拆分、任务分派、冲突处理、最终验收。
- `Product Planner Agent`：需求澄清、范围边界、验收标准、阶段计划。
- `Go Backend Agent`：Go 后端接口、业务逻辑、测试、错误处理。
- `Frontend Agent`：Web 用户端、Web 后台管理端、H5 移动端和通用 React 前端开发。
- `Database Agent`：数据建模、PostgreSQL 迁移、索引、约束和数据风险。
- `QA Review Agent`：代码审查、测试建议、风险检查、交付验收。
- `Docs Agent`：README、接口说明、使用说明、变更记录。

每次任务不必启用全部角色。Lead Agent 必须根据需求选择最小可用团队。

## 3. 角色文件

角色提示词存放在：

```text
.claude/agents/
```

预设角色：

- `.claude/agents/lead-agent.md`
- `.claude/agents/product-planner-agent.md`
- `.claude/agents/go-backend-agent.md`
- `.claude/agents/frontend-agent.md`
- `.claude/agents/database-agent.md`
- `.claude/agents/qa-review-agent.md`
- `.claude/agents/docs-agent.md`

启用对应 Agent 前，必须先阅读其角色文件。

## 4. 协作顺序

### 4.1 默认串行阶段

多 Agent 开发必须按以下阶段推进：

```text
阶段 0：Lead Agent 建立共同上下文
阶段 1：Product Planner Agent 明确范围和验收
阶段 2：Database Agent 设计数据模型和迁移边界
阶段 3：Go Backend Agent 与 Frontend Agent 按文件边界并行实现
阶段 4：Docs Agent 补充文档和使用说明
阶段 5：QA Review Agent 审查风险、测试和交付完整性
阶段 6：Lead Agent 汇总 diff、运行验证、给出最终交付说明
```

### 4.2 可并行阶段

只有在职责和文件边界明确后，才允许并行。

允许并行：

- 后端接口实现与前端静态页面搭建。
- 数据库迁移评审与前端 UI 状态补齐。
- 文档更新与测试补充。
- QA Review Agent 在实现完成后独立审查。

不允许并行：

- 多个 Agent 同时修改同一个文件。
- 前端 Agent 在接口契约未确定前对接真实接口。
- 后端 Agent 在数据模型未确定前写入复杂业务逻辑。
- Docs Agent 在功能范围未稳定前写最终用户文档。

### 4.3 自动执行与阶段 Gate

当用户提供详细 `docs/project-plan.md` 并要求自动开发时，Agent Teams 必须遵守两阶段暂停策略：

```text
MVP 阶段：Lead -> Planner（如需要）-> Database（如需要）-> Backend + Frontend 并行 -> Docs -> QA -> Lead -> MVP Gate
完整版本阶段：用户确认 MVP 后 -> Lead -> 各专业 Agent 继续剩余范围 -> QA -> Lead -> Final Gate
```

Lead Agent 负责判断当前处于 MVP 阶段还是完整版本阶段，并把任务拆分限制在当前阶段范围内。

在到达 `MVP Gate` 或 `Final Gate` 前，专业 Agent 不得因为非阻塞信息缺口停止等待用户。以下问题应记录后继续：

- 第三方服务密钥、域名、Bucket、回调地址、部署参数暂未提供。
- 正式文案、品牌素材、运营配置、测试数据暂未确定。
- 可以通过配置占位、默认实现、Mock 数据、README 待补说明继续推进。

只有以下情况允许 Lead Agent 提前暂停并请求用户确认：

- 缺少核心业务规则，导致数据模型、权限边界或主流程无法设计。
- 存在安全、隐私、资金、生产数据迁移或破坏性命令风险。
- 多个规范或需求互相冲突，且无法通过保守实现规避。
- 必需验证命令无法运行，且没有可替代验证方式。

阶段 Gate 必须由 Lead Agent 统一汇总，不允许专业 Agent 直接向用户零散交付。

## 5. 文件所有权

Lead Agent 分派任务时必须明确每个 Agent 的文件所有权。

示例：

```text
Database Agent：
- 负责：migrations/**、docs/database.md
- 禁止修改：internal/**、src/**

Go Backend Agent：
- 负责：internal/module/user/**、cmd/** 中的路由注册
- 禁止修改：src/**、migrations/** 中已确认的迁移

Frontend Agent：
- 负责：src/features/**、src/pages/**、src/router/**、src/components/** 中与本次需求相关的文件
- 禁止修改：internal/**、migrations/**
```

如果必须跨所有权修改，Agent 必须在交接说明中写明原因和影响。

## 6. 共享上下文

所有 Agent 开始前必须读取：

1. `CLAUDE.md`
2. `docs/project-plan.md`，如果存在
3. 与任务相关的 `.claude/rules/*.md`
4. 自己对应的 `.claude/agents/*.md`
5. 项目 README、构建脚本、相邻实现和相关代码

Agent 不得依赖其他会话中的隐式记忆。

## 7. 任务分派格式

Lead Agent 分派任务时必须使用以下格式：

```txt
任务目标：
-

必须遵守：
- CLAUDE.md
- docs/project-plan.md
- .claude/rules/agent-teams.md
- .claude/agents/<role>.md

文件所有权：
- 可修改：
- 不可修改：

输入上下文：
-

输出要求：
- 修改范围
- 验证方式
- 风险与取舍
- 需要交接给其他 Agent 的事项
```

## 8. Agent 交付格式

每个专业 Agent 完成后必须输出：

```txt
完成内容：
-

修改文件：
-

验证结果：
-

未验证项：
-

风险与取舍：
-

交接事项：
-
```

## 9. 冲突处理

出现以下情况时，必须由 Lead Agent 决策；只有属于硬阻塞时才暂停询问用户：

- 两个 Agent 需要修改同一文件。
- 数据库模型和接口契约冲突。
- 用户需求与 `docs/project-plan.md` 范围冲突。
- 安全、权限、隐私、迁移风险无法由单个 Agent 判断。
- 测试结果与实现预期不一致。

冲突解决优先级：

1. 安全、隐私、数据正确性和迁移安全。
2. 用户本次明确要求。
3. `docs/project-plan.md`。
4. 项目现有代码和 README。
5. `.claude/rules`。
6. Agent 默认经验。

## 10. 验收要求

Lead Agent 最终交付前必须完成：

- 检查 `git status` 和关键 diff。
- 确认每个 Agent 的修改都在文件所有权范围内。
- 运行可用的格式化、类型检查、测试或构建命令。
- 说明无法运行的验证命令和原因。
- 汇总修改范围、验证结果、未验证项、风险和建议提交信息。

禁止把多个 Agent 的输出简单拼接后直接交付。

## 11. 推荐团队组合

### 后台管理功能

```text
Lead Agent
Product Planner Agent
Database Agent
Go Backend Agent
Frontend Agent（后台管理模式）
QA Review Agent
Docs Agent（按需）
```

推荐顺序：

```text
Lead -> Planner -> Database -> Backend + Frontend 并行 -> Docs -> QA -> Lead
```

### Web 用户端或 H5 功能

```text
Lead Agent
Product Planner Agent
Go Backend Agent（如需要接口）
Frontend Agent
QA Review Agent
Docs Agent（按需）
```

推荐顺序：

```text
Lead -> Planner -> Backend 契约 -> Frontend + Backend 并行 -> QA -> Lead
```

### 纯后端接口功能

```text
Lead Agent
Database Agent（如涉及表结构）
Go Backend Agent
QA Review Agent
Docs Agent（按需）
```

推荐顺序：

```text
Lead -> Database -> Backend -> QA -> Lead
```

### 纯前端页面

```text
Lead Agent
Product Planner Agent（如需求不清）
Frontend Agent
QA Review Agent
```

推荐顺序：

```text
Lead -> Planner -> Frontend -> QA -> Lead
```

### 数据迁移或权限高风险功能

```text
Lead Agent
Product Planner Agent
Database Agent
Go Backend Agent
QA Review Agent
Docs Agent
```

推荐顺序：

```text
Lead -> Planner -> Database -> Backend -> QA -> Docs -> Lead
```
