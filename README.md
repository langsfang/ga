# Genvagent

Genvagent 包含 **Agent Harness** 与独立的 **Memory System**。用户与 Router 对话；Router 使用统一用户记忆，决定直接处理、新建工作状态、继续既有 session 或从旧状态分叉。

Worker 是可替换的执行进程。业务状态保存在 Memory 的 session/context 与记录中；任务结束可以释放进程，之后由另一个进程继续。用户画像、短期/长期记忆和 notes 属于用户记忆的不同视图或表达方式，集合与项目分类均可选。

当前阶段：架构设计，尚未实现运行时代码。

- [整体架构与概念模型](docs/architecture.md)
- [交互逻辑、接口函数与数据结构](docs/integration-protocol.md)
- [Harness Core、Router CLM 与运行时接入](docs/agent-harness.md)
- [Memory 对象、视图、整理与部署](docs/memory-system.md)

首个联合验证目标：无项目/集合也能使用；P1 执行后释放，P2 能继续同一 session；新建、继续、分叉和退役分别保持明确语义；Router 的 context 编辑实际影响下一模型输入。
