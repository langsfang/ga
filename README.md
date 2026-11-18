# Genvagent

Genvagent 由 Agent Harness 与独立部署的 Memory System 组成。Router 接收用户输入并分派任务；Harness 调度 Worker；Worker 的结果经持久收件箱返回 Router。

当前阶段为架构设计，尚未实现运行时代码。

## 当前设计

- [Agent Harness 架构](docs/agent-harness.md)：任务分派、队列调度、信号与结果回传，以及 dsh 插件接入。
- [Memory System 架构](docs/memory-system.md)：用户记忆、Context 版本、持久任务与消息、Trace 导入和后台维护。
- [两系统最小接口协议](docs/integration-protocol.md)：唯一接口规范；13 个在线操作及 2 个 Trace 导入操作。

首版使用 Memory 的事务存储实现调度队列和 Task 收件箱，不依赖额外消息中间件。执行权按具体 Context 版本分配；原始消息、任务结果及恢复状态独立于执行进程。

联合验证重点：Router 在 Worker 执行时仍可响应；同一任务不重复启动；等待父任务释放执行名额；结果先于等待注册也能唤醒；重启和重复投递不丢任务、不重复产生答复；dsh 的实际模型请求与 Memory 版本一致。

## 历史材料

[早期整体设计](docs/architecture.md)和[早期实现参考](docs/integration-reference.md)保留供查阅，其中旧 Session 级锁、History 及函数签名不作为当前实现依据。
