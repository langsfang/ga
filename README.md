# Genvagent

Genvagent 由两个系统组成：以插件接入 dsh、pi 等宿主的 **Agent Harness**，以及可独立部署在单机或集群上的 **Memory System**。两个系统通过稳定协议协作，使执行进程与持久上下文分离。

当前阶段：架构设计，尚未实现运行时代码。

- [整体架构与边界](docs/architecture.md)
- [交互模型与对接协议](docs/integration-protocol.md)：先理解两侧交换哪些信息、一次任务如何流转，再查看类型与接口。
- [Agent Harness](docs/agent-harness.md)：核心模块、插件契约、dsh/pi 适配及关键结构。
- [Memory System](docs/memory-system.md)：记忆管理、后台维护、关键结构和单机/集群部署。

首个联合验证目标：让两个宿主共享同一项目记忆，纠正一条事实后两侧都能获取修订；重启插件或 Memory 节点后，同步不重复、上下文版本可核对，宿主执行状态仍由宿主正确恢复。
