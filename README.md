# Genvagent

Genvagent 包含 **Agent Harness** 与 **Memory System**。用户与 Router 对话，Router 使用该用户跨项目的统一 Memory 分派 Worker 并汇总结果。project notes 是用户 Memory 内的专题资料。

Harness Core 管理 turn、agent loop、任务与上下文；既可由参考 Runtime Driver 独立驱动，也可通过 Adapter 接入 dsh、pi。Memory 独立部署于单机或集群。

当前阶段：架构设计，尚未实现运行时代码。

- [整体架构与系统边界](docs/architecture.md)
- [交互逻辑、接口函数与数据结构](docs/integration-protocol.md)：一轮对话的产生、消息追加、上下文提交、Worker 回传与恢复。
- [Agent Harness](docs/agent-harness.md)：Core 内部模块、执行循环、运行时接口及 Host 适配。
- [Memory System](docs/memory-system.md)：用户域、服务接口、项目 notes、后台整理与部署。

首个联合验证目标：Router 在同一用户域跨项目检索，派发 Worker 完成任务并返回答复；所有完成消息与 context 可持久恢复，重复投递不产生重复事件或任务。
