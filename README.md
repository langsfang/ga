# Genvagent

Genvagent 是按 **Super Agent** 架构实现的产品。Super Agent 是一种以记忆系统为中心的 agent 架构：Context、经验与知识长期保存在 Memory System 中，agent 进程以插件形式运行在 dsh、pi 等已有的 agent 程序里，可以随时替换。

Genvagent 由两个系统组成：

- **Agent Harness**：持久化执行。负责任务委派、调度、消息投递和故障恢复，保证面向用户的 Router 不被后台任务阻塞。
- **Memory System**：上下文、经验与知识的长期中心。负责 Context 版本、证据、可追溯的记忆条目与检索，并在后台由 Memory Agent 提炼和沉淀知识。

当前阶段是概念设计，还没有运行时代码。

## 文档

1. [Super Agent 架构蓝图](docs/vision.md)：定位、要解决的问题、总体架构与演进路线。**建议先读这一篇。**
2. [概念模型](docs/concepts.md)：顶层概念、状态归属、核心交互与不变量
3. [Agent Harness 架构](docs/agent-harness.md)
4. [Memory System 架构](docs/memory-system.md)
5. [Harness 与 Memory 交互与接口](docs/integration-protocol.md)：两系统的边界、Memory 接口与调用场景
6. [同类系统对比](docs/landscape.md)：与 dsh-superagent、DS-Bot、LangGraph、Temporal、Letta 等的对比和可借鉴之处

v0.16 及以前的设计材料保存在 [docs/archive/](docs/archive/)，不作为当前依据。新旧术语对照见[概念模型 §7](docs/concepts.md#7-术语对照)。
