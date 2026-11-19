# Genvagent

Genvagent 由两个系统组成：

- **Agent Harness**：持久化执行。负责任务委派、调度、消息投递和故障恢复，保证面向用户的 Router 不被后台任务阻塞。
- **Memory System**：统一用户记忆。负责 Context 版本、证据、可追溯的记忆条目，以及检索。

当前阶段是概念设计，还没有运行时代码。

## 文档

1. [概念模型](docs/concepts.md)：顶层概念、状态归属、核心交互与不变量。**建议先读这一篇。**
2. [Agent Harness 架构](docs/agent-harness.md)
3. [Memory System 架构](docs/memory-system.md)
4. [Harness 与 Memory 交互与接口](docs/integration-protocol.md)：两系统的边界、Memory 接口与调用场景

v0.16 及以前的设计材料保存在 [docs/archive/](docs/archive/)，不作为当前依据。新旧术语对照见[概念模型 §7](docs/concepts.md#7-术语对照)。
