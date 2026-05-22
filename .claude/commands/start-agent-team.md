# Start Agent Team

请根据 `CLAUDE.md`、`.claude/rules/agent-teams.md`、`.claude/agents`、相关技术规范和 `docs/project-plan.md` 启动一次多 Agent 协作，并默认执行到当前阶段 Gate。

本命令适用于复杂项目或跨前端、后端、数据库、测试、文档的任务。Lead Agent 必须自行选择最小可用团队、分配文件所有权并推进开发；除硬阻塞外，不要在 MVP Gate 或 Final Gate 前频繁停下来询问用户。

## 执行顺序

1. 阅读 `CLAUDE.md`，理解通用规则、自动执行与暂停策略、模型适配原则、Git 规则和项目计划书优先级。
2. 阅读 `.claude/rules/agent-teams.md`，判断本次需求是否适合启用 Agent Teams。
3. 阅读 `docs/project-plan.md`，如果存在，确认当前阶段是 MVP 版本还是完整版本。
4. 阅读与本次技术栈相关的 `.claude/rules/*.md`。
5. 阅读需要启用的 `.claude/agents/*.md`。
6. Lead Agent 自行输出并执行团队组合、串行阶段、并行阶段、文件所有权、验证计划和风险处理方式。
7. 将问题分为两类：
   - 硬阻塞：不确认就无法继续核心数据模型、权限边界、主流程，或涉及安全、隐私、资金、生产数据、破坏性命令风险。
   - 非阻塞信息缺口：可以用 Assumption、TODO、配置占位、README 待补项继续推进的问题。
8. 只有存在硬阻塞时，才暂停并向用户提出最小确认问题。
9. 如果没有硬阻塞，按当前阶段目标自动开发到 `MVP Gate` 或 `Final Gate`。
10. 到达阶段 Gate 后，由 Lead Agent 汇总所有 Agent 输出、检查 diff、运行验证，并停止等待用户确认或交付最终结果。

## Lead Agent 执行摘要格式

```txt
是否启用 Agent Teams：
- 结论：
- 原因：

当前阶段：
- MVP / 完整版本

推荐团队：
-

任务拆分：
-

串行顺序：
1.
2.
3.

并行顺序：
-

文件所有权：
-

每个 Agent 的任务提示词：
-

验证计划：
-

非阻塞信息缺口处理：
-

硬阻塞检查：
-
```

## 阶段 Gate 输出格式

```txt
阶段交付报告：
-

当前 Gate：
- MVP Gate / Final Gate

完成范围：
-

未包含范围：
-

各 Agent 交付摘要：
-

Assumptions：
-

预留 TODO / 配置占位：
-

验证结果：
-

未验证项：
-

风险与取舍：
-

建议提交信息：
-
```

## 要求

- 如果单 Agent 更合适，必须明确说明并按单 Agent 自动执行到当前阶段 Gate。
- 如果启用多个 Agent，必须明确每个 Agent 的职责、输入、输出和不可修改文件。
- 只有职责边界清晰、文件所有权明确后，才允许并行实现。
- 不得把 `docs/project-plan.md` 中暂不实现的功能纳入当前任务。
- 对不影响代码开发的信息缺口，必须记录并继续执行。
- 除硬阻塞外，不得在 MVP Gate 或 Final Gate 前频繁向用户确认。
- 对 MiniMax-M2.7 或其他非 Claude 官方模型，必须使用明确步骤、固定输出和可验证检查项。
