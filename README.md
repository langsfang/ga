# Genvagent

Genvagent 由两个系统组成：以插件接入 dsh、pi 等 **Host**（完整 agent 应用）的 **Genvagent Harness**，以及可独立部署在单机或集群上的 **Memory System**。两个系统通过版本化协议协作，使执行进程与持久上下文分离。

当前阶段：架构设计，尚未实现运行时代码。

- [整体架构与边界](docs/architecture.md)
- [交互模型与对接协议](docs/integration-protocol.md)：角色与数据模型、用户数据组织、同步与检索、执行与恢复，再到类型和接口。
- [Agent Harness](docs/agent-harness.md)：核心模块、插件契约、dsh/pi 适配及关键结构。
- [Memory System](docs/memory-system.md)：记忆管理、后台维护、关键结构和单机/集群部署。

首个联合验证目标：同一用户在两个 Host 上复用同项目记忆，纠正一条事实后两侧都能获取修订；重启插件或 Memory 节点后，同步不重复、上下文版本可核对。执行环境和工具状态通过 Host 自身的恢复机制处理。
