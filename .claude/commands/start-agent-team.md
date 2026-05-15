# Start Agent Team

请根据 `CLAUDE.md`、`.claude/rules/agent-teams.md`、`.claude/agents`、相关技术规范和 `docs/project-plan.md` 启动一次多 Agent 协作规划。

执行顺序：

1. 阅读 `CLAUDE.md`，理解通用规则、模型适配原则、Git 规则和项目计划书优先级。
2. 阅读 `.claude/rules/agent-teams.md`，判断本次需求是否适合启用 Agent Teams。
3. 阅读 `docs/project-plan.md`，如果存在，确认需求是否在计划范围内。
4. 阅读与本次技术栈相关的 `.claude/rules/*.md`。
5. 阅读需要启用的 `.claude/agents/*.md`。
6. 输出团队组合、串行阶段、并行阶段、文件所有权、验证计划和风险。
7. 在用户确认前，不要直接大规模改代码。

输出格式：

```txt
是否启用 Agent Teams：
- 结论：
- 原因：

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

需要确认的问题：
-

风险与取舍：
-
```

要求：

- 如果单 Agent 更合适，必须明确说明并给出单 Agent 执行计划。
- 如果启用多个 Agent，必须明确每个 Agent 的职责、输入、输出和不可修改文件。
- 只有职责边界清晰、文件所有权明确后，才允许并行实现。
- 不得把 `docs/project-plan.md` 中暂不实现的功能纳入当前任务。
- 对 MiniMax-M2.7 或其他非 Claude 官方模型，必须使用明确步骤、固定输出和可验证检查项。

