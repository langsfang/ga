# Genvagent

Genvagent 包含 **Agent Harness** 与独立的 **Memory System**。Router 负责对话、路由和汇总，异步任务交给 Worker；Worker 独立循环并把进度、结果和 context 保存到统一用户 Memory。

Worker 进程可以替换，持久 session/context 可以继续或分叉。第一版采用 **消息追加 + compact**，使用后台 Session Maintenance 管理退役；CLM 与资源虚拟化延后。

当前阶段：架构设计，尚未实现运行时代码。

- [整体架构与职责](docs/architecture.md)
- [交互顺序、函数与数据结构](docs/integration-protocol.md)
- [Router/Worker 循环与 Host 接入](docs/agent-harness.md)
- [Memory 存储、退役与 notes 维护](docs/memory-system.md)

首个联合验证目标：Worker 工作时 Router 仍可响应；任务状态与通知可靠；新进程可从 checkpoint 继续；后台退役保留退出记录，notes 的整理和删除独立处理。
