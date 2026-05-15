# Claude Code Specification

这是一套面向个人开发者的 Claude Code 项目规范模板，用于让 Agent 在新项目或长期维护项目中保持一致的工作方式、代码质量和交付标准。

它主要解决四个问题：

1. 让 Claude Code 明确知道“怎么开发、怎么验证、怎么交付”。
2. 让不同项目可以复用同一套 Go、React、PostgreSQL 规范。
3. 让新项目可以先基于项目计划书拆阶段，再开始写代码，减少跑偏和过度工程化。
4. 让复杂需求可以按 Agent Teams 模式拆分、并行、审查和统一交付。

## 目录结构

```text
.
├── CLAUDE.md
├── README.md
├── templates
│   └── project-plan.md
└── .claude
    ├── settings.json
    ├── agents
    │   ├── database-agent.md
    │   ├── docs-agent.md
    │   ├── frontend-agent.md
    │   ├── go-backend-agent.md
    │   ├── lead-agent.md
    │   ├── product-planner-agent.md
    │   └── qa-review-agent.md
    ├── commands
    │   ├── review-spec.md
    │   ├── start-agent-team.md
    │   └── start-project.md
    └── rules
        ├── agent-teams.md
        ├── go-backend.md
        ├── react-frontend.md
        └── postgresql.md
```

## 文件说明

### `CLAUDE.md`

Claude Code 的项目入口文件，定义通用工作原则、个人开发者默认策略、模型适配原则、项目计划书优先级、Git 提交规则和规范冲突处理方式。

新项目中建议保留本文件内容，并补充该项目自己的启动方式、测试命令、目录说明和特殊约定。

### `.claude/rules`

公共技术规范：

- `go-backend.md`：Go 后端项目规范。
- `react-frontend.md`：React 前端项目规范。
- `postgresql.md`：PostgreSQL 数据库设计规范。
- `agent-teams.md`：多 Agent 协作规范，定义启用条件、角色边界、串并行顺序、文件所有权和验收要求。

这些文件定义“怎么做”，例如目录结构、命名、测试、错误处理、安全、数据库迁移等。

### `.claude/agents`

多 Agent 协作角色提示词：

- `lead-agent.md`：主协调者，负责需求理解、任务拆分、文件所有权、冲突处理和最终验收。
- `product-planner-agent.md`：产品规划角色，负责范围边界、阶段计划和验收标准。
- `go-backend-agent.md`：Go 后端开发角色，负责接口、业务逻辑、分层、测试和后端交付。
- `frontend-agent.md`：前端开发角色，负责 Web 用户端、Web 后台管理端、H5 移动端、组件、交互和前端测试。
- `database-agent.md`：数据库角色，负责 PostgreSQL 建模、迁移、约束、索引和数据风险。
- `qa-review-agent.md`：质量审查角色，负责代码审查、测试缺口、安全和回归风险。
- `docs-agent.md`：文档角色，负责 README、接口说明、使用说明和部署说明。

默认不要求每个需求都启用多 Agent。只有跨前端、后端、数据库、测试、文档，或涉及权限、支付、迁移、隐私、安全等高风险需求时，才建议启用 Agent Teams。

### `.claude/settings.json`

Claude Code 项目级配置。目前主要用于禁止读取 `.env`、`secrets/`、credentials、secret/private 类敏感文件。

本机私有配置请使用 `.claude/settings.local.json`，不要提交到仓库。

### `.claude/commands/review-spec.md`

规范审查命令。用于让 Claude Code 从个人开发者、长期维护、模型适配、安全和验证闭环等角度审查本规范。

### `.claude/commands/start-project.md`

新项目启动命令。用于让 Claude Code 先读取 `docs/project-plan.md`，再拆分阶段、识别风险、生成 Phase 1 任务清单。

### `.claude/commands/start-agent-team.md`

多 Agent 协作启动命令。用于让 Claude Code 先判断是否需要 Agent Teams，再输出推荐团队、任务拆分、串行顺序、并行顺序、文件所有权、验证计划和风险。

### `templates/project-plan.md`

项目计划书模板。新项目建议复制为：

```text
docs/project-plan.md
```

项目计划书定义“做什么”和“不做什么”，公共规则定义“怎么做”。

## 在新项目中使用

### 1. 先初始化 Claude Code

如果新项目还没有 `CLAUDE.md`，建议先在新项目根目录执行：

```text
/init
```

`/init` 会让 Claude Code 扫描当前项目，并生成项目专属的初始 `CLAUDE.md`。

如果项目已经有 `CLAUDE.md`，不要直接覆盖，优先合并规则引用和通用原则。

### 2. 复制规范文件

在新项目根目录执行：

```bash
cp -R /Users/risemini/project/claude-code-specification/.claude .
```

如果新项目没有 `CLAUDE.md`，可以复制本仓库的入口文件：

```bash
cp /Users/risemini/project/claude-code-specification/CLAUDE.md .
```

如果新项目已经有 `/init` 生成的 `CLAUDE.md`，建议只追加规范引用：

```md
## 项目开发规范

请遵守以下规范：

- Go 后端规范见：`.claude/rules/go-backend.md`
- React 前端规范见：`.claude/rules/react-frontend.md`
- PostgreSQL 规范见：`.claude/rules/postgresql.md`
- 多 Agent 协作规范见：`.claude/rules/agent-teams.md`
```

然后根据项目实际情况补充：

- 启动命令
- 测试命令
- 构建命令
- 环境变量说明
- 目录结构
- 项目特殊约定

### 3. 配置 `.gitignore`

如果新项目已经有 `.gitignore`，建议追加：

```gitignore
.DS_Store
**/.DS_Store
CLAUDE.local.md
.claude/settings.local.json
```

如果新项目没有 `.gitignore`，可以复制本仓库的 `.gitignore` 后再按项目调整。

### 4. 准备项目计划书

复制模板：

```bash
mkdir -p docs
cp /Users/risemini/project/claude-code-specification/templates/project-plan.md docs/project-plan.md
```

然后填写：

- 项目目标
- 用户与场景
- 功能范围
- 暂不实现项
- 技术栈，模板已预设 React + TypeScript + Vite、Go + Gin、PostgreSQL、阿里云 OSS 等默认组合；如需偏离，必须写明原因
- 数据模型
- 页面与接口
- 开发阶段
- 验收标准
- 风险与注意事项

## 推荐工作流

### 新项目开发

1. 在新项目中执行 `/init`。
2. 复制 `.claude` 规范目录。
3. 合并或补充 `CLAUDE.md`。
4. 复制并填写 `docs/project-plan.md`。
5. 在 Claude Code 中执行或引用：

```text
/start-project
```

6. 让 Claude Code 输出项目理解、计划书缺口、Phase 1 任务清单、风险与取舍。
7. 确认 Phase 1 后，再开始实现。

如果是跨前端、后端、数据库或高风险功能，可以改用：

```text
/start-agent-team
```

它会先输出是否启用 Agent Teams、推荐团队组合、各 Agent 的职责、文件所有权、串行和并行顺序。确认前不要让多个 Agent 同时大规模改代码。

也可以直接输入：

```text
请先阅读 CLAUDE.md、docs/project-plan.md 和 .claude/rules 下的规范。
然后根据项目计划书拆分开发阶段，先输出 Phase 1 任务清单，不要立即大规模写代码。
```

### 已有项目接入

1. 不覆盖已有 `CLAUDE.md`。
2. 复制 `.claude/rules` 和 `.claude/settings.json`。
3. 在已有 `CLAUDE.md` 中追加规范引用。
4. 让 Claude Code 先阅读现有 README、package.json、go.mod、配置文件、目录结构。
5. 根据项目当前技术栈，只启用相关规范。

### 审查规范本身

在本仓库中使用：

```text
/review-spec
```

它会检查规范是否存在：

- 过度工程化
- 规则冲突
- 不适合个人开发者的硬约束
- 安全、验证、迁移、交付报告遗漏
- 对 MiniMax-M2.7 或其他非 Claude 官方模型不够显式的问题

## 规则优先级

建议在新项目中采用以下优先级：

1. 用户本次明确要求。
2. 安全、隐私、数据正确性和迁移安全底线。
3. `docs/project-plan.md` 中定义的范围和阶段。
4. 项目现有 README、脚本、目录和代码风格。
5. `.claude/rules` 中的公共技术规范。
6. Agent 的默认经验和建议。

如果规则冲突，Claude Code 必须显式说明取舍，不应静默选择。

## Agent Teams 推荐用法

后台管理类功能推荐团队：

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

用户端 Web 或 H5 功能推荐顺序：

```text
Lead -> Planner -> Backend 契约（如需要接口） -> Frontend + Backend 并行 -> QA -> Lead
```

纯后端接口功能推荐顺序：

```text
Lead -> Database（如涉及表结构） -> Go Backend -> QA -> Lead
```

纯前端页面推荐顺序：

```text
Lead -> Planner（如需求不清） -> Frontend -> QA -> Lead
```

多 Agent 协作必须遵守文件所有权：每个 Agent 开始前都要知道自己能改哪些文件、不能改哪些文件。最终由 Lead Agent 检查 diff、运行验证并统一交付。

## 提交策略

本规范默认不允许 Agent 自动提交 Git。

只有在用户明确要求提交，或项目文档明确授权自动提交时，Agent 才能执行：

```bash
git commit
```

提交前必须检查：

```bash
git status
```

并且只提交与当前需求相关的文件。

## 适配 MiniMax-M2.7

如果 Claude Code 使用 MiniMax-M2.7 或其他非 Claude 官方模型，建议坚持以下原则：

- 任务要拆小。
- 输出格式要固定。
- 修改前先读文件。
- 修改后检查 diff。
- 能运行验证就运行。
- 不能验证就说明原因。
- 不依赖隐式记忆。
- 不确定时先读项目文件，而不是凭经验猜。

这些原则已经写入 `CLAUDE.md` 的“模型适配原则”中。

## 建议复制到新项目的最小文件集

普通业务项目建议复制：

```text
CLAUDE.md
.claude/settings.json
.claude/rules/
templates/project-plan.md
```

如果使用计划书驱动开发，再复制：

```text
.claude/commands/start-project.md
```

如果使用 Agent Teams，再复制：

```text
.claude/agents/
.claude/commands/start-agent-team.md
```

如果需要审查规范本身，再复制：

```text
.claude/commands/review-spec.md
```

## 维护建议

- 公共规则不要为了单个项目频繁改动。
- 单个项目的特殊约定应写在该项目自己的 `CLAUDE.md` 或 `docs/project-plan.md`。
- 如果某条规则多次在不同项目中复用，再考虑沉淀回本仓库。
- 如果某条规则让小项目明显变慢，应改成按项目规模分级执行。
- 修改规范后，建议使用 `/review-spec` 复查一次。
