# Agent Harness：Router 响应、Worker 循环与状态同步

状态：设计草案 v0.8；2026-11-17。第一版采用 append + compact。先读[概念模型与能力边界](integration-protocol.md)；共享载荷的实现候选见[接口实现参考](integration-reference.md)。本文描述 Harness 内部模块与执行流程。

## 1. Core 的模块与职责

| 模块 | 职责 |
| --- | --- |
| Router Controller | 串行处理用户输入和重要任务通知，及时回应、澄清和分派 |
| Router Policy | 根据目标与记忆选择回应、new/resume/fork 或排队；不直接执行事务 |
| Task Dispatcher / Controller | 持久化任务，安排 Worker，处理任务输入与取消，协调同 session 排队 |
| Worker Controller | 每个 run 独立推进模型/工具 loop，产生进度与结果 |
| Context Controller | 追加已提交消息，按阈值 compact，构建 checkpoint |
| Memory Coordinator | 可靠投递事件、保存快照、读取状态与通知，处理恢复 |
| Runtime Driver / Host Adapter | 进程与环境、实际模型/工具调用、原生接入及动作核对 |

Session 退役由 Memory 的后台维护任务负责。Harness 可以提交维护请求和工作结束事件，但不会占用 Router turn 做清理。

## 2. 两个循环

### 2.1 Router 的短对话循环

```text
用户输入 / 任务重要通知
  → 持久化 inbox 事件
  → 创建一个 Router turn
  → 读取必要记忆与任务状态
  → 回应、澄清或持久分派
  → 保存回复与 checkpoint
  → 结束本 turn，处理下一条输入
```

delegate 在 recordTask 成功后返回 TaskAccepted。Worker 的完成通过新通知触发后续 turn，不把原模型工具调用一直挂起。因此 Worker 的执行时长不决定 Router 何时能再次接收输入。

Router 可使用记忆查询、session 查询和任务控制工具；耗时分析、文件操作和业务工具委派给 Worker。普通进度和心跳只更新任务视图，不逐条调用 Router 模型。

同一 Router session 的写入与模型调用仍串行。新输入到达时持久排队；关键纠正或取消使尚未执行的旧计划失效，在安全边界处理，不并发改写正在使用的输入。

### 2.2 Worker 的独立执行循环

```text
取得 session 写入权，加载 checkpoint 与增量
  → 读取本 task 的待处理输入
  → 追加消息，必要时 compact，保存 checkpoint
  → 准备模型输入并保存请求记录
  → 调用模型
  → 保存输出，执行工具并保存结果
  → 更新 TaskSnapshot 所需的进度事件
  → 下一步，或保存任务终态并释放 session
```

Worker 不需要 Router 为每一步批准。Router 通过 TaskSnapshot 了解状态，必要时读取具体 checkpoint 或证据。所有影响恢复的消息与动作仍持久化，简短进度不能替代原始日志。

纠正和取消写入 TaskInput 队列。Worker 在请求边界处理输入，并随实际处理事件确认 inputId；若正在执行外部工具，先发取消信号并核对结果。waiting_input 任务可以释放进程，待新输入到达后由 Dispatcher 安排继续。

### 2.3 执行环境和接入方式

执行环境提供模型客户端、工具、进程、工作目录和权限；Runtime Driver 只是其接口。Run 是一次执行尝试，不是执行环境或持久 session 的别名。

| 方式 | 具体执行过程 |
| --- | --- |
| 独立运行 | Genvagent Runner 调用 Core.advance，再调用 Driver 的模型/工具接口；完成结果回到 Core |
| Host 插件接入 | dsh/pi 原有 loop 调用模型；Adapter 在请求前让 Core 准备并保存输入，完成后导出结果 |

这两种方式分别对应旧称 Core-owned 和 Host-driven。第一版配置使用 standalone / host_plugin。插件方式不再启动第二个模型循环；Host 不能提供必要请求边界时，只能开放同步/检索能力。

## 3. 逻辑接口函数

### 3.1 Core 入口

```typescript
interface HarnessCorePort {
  openConversation(request: SessionCreate): Promise<SessionState>;
  submitUserMessage(routerSessionId: ID, inputId: ID, content: Content): Promise<TurnHandle>;
  acceptTaskNotification(notification: TaskNotification): Promise<TurnHandle>;
  advance(runId: ID): Promise<AgentAction>;
  acceptCompletion(actionId: ID, completion: ActionCompletion): Promise<void>;
  cancelTurn(turnId: ID): Promise<void>;
}
```

openConversation 只处理 Router session。cancelTurn 取消当前对话动作，不自动取消已分派的 task；任务取消必须通过 postTaskInput 明确指定 taskId。

acceptTaskNotification 按 notificationId 创建稳定 inbox 事件和 turn。Core 将事件提交到 Router 流后，Memory 同事务确认通知投递；恢复时从已提交事件继续，不重复创建 turn。

### 3.2 Dispatcher 与 Memory

dispatchTask 先持久化 taskId 和目标选择，创建或核对目标 session，然后 recordTask。返回 TaskAccepted 后即可结束 delegate；本地 Dispatcher 从已持久化的 queued 任务继续安排执行。

```text
allocateWorker
  → claimSession（Memory 只登记 run 和 lease）
  → readCheckpoint + readEvents
  → attachSession
  → Worker Controller.advance
```

claimSession 前重新检查目标 head；与原路由判断不一致时核对进度，不能覆盖。任务已完成或被取消时不再启动第二次执行；恢复或重试必须有明确任务状态依据。

多个 Dispatcher 竞争时，session lease 和任务的当前 run 共同排除重复执行。失败者释放空进程或等待，不改选新的业务 session。所有 Harness 离线时，未启动任务保留 queued，Memory 不代替 Harness 执行。

### 3.3 Driver 接口

```typescript
interface RuntimeDriver {
  capabilities: AdapterCapabilities;
  allocateWorker(requirements: WorkerRequirements): Promise<WorkerHandle>;
  attachSession(worker: WorkerHandle, session: SessionDescriptor, lease: SessionLease): Promise<HostSessionLink | undefined>;
  prepareModelCall(actionId: ID, stepId: ID, lease: SessionLease, checkpoint: ContextCheckpoint): Promise<ModelInputRecord>;
  invokeModel(actionId: ID): Promise<void>;
  executeTool(actionId: ID, call: ToolCall): Promise<void>;
  reconcile(actionId: ID): Promise<RuntimeActionState>;
  cancel(actionId: ID): Promise<void>;
  detachSession(runId: ID): Promise<void>;
  releaseWorker(processId: ID): Promise<void>;
}
interface HostAdapter {
  attach(core: HarnessCorePort): Promise<RuntimeDriver>;
  detach(): Promise<void>;
}
```

prepareModelCall 生成并冻结本次输入；Core 记录它后再 invokeModel。准备成功不等于调用已发生。实际覆盖 working_view、model_messages 或 provider_payload，必须如实记录；请求记录排除凭据。

Driver 按 actionId 记录动作接收和执行状态，重复调用不得无声启动第二个动作。崩溃后结果未知时先 reconcile，不假设任意外部工具支持 exactly-once。

attachSession 返回的 HostSessionLink 可供 Adapter 保存定位关系。独立运行返回 undefined。这个索引既不建立执行权，也不需要每次 context 更新重写。

## 4. 第一版的 context 管理

Context Controller 按已确认事件顺序追加模型消息。原始记录先经 appendEvents 保存，工作副本随后更新；候选快照完成后调用 saveCheckpoint。

compact 由 Harness 策略触发，首版至少保留：当前目标、用户明确约束、重要精确值、未完成工具交换和来源引用。旧正文可移出工作视图，但原始事件可精确读取。

compact 的输出必须验证预算、来源、保护内容和工具格式。验证失败时保留原状态并限制继续执行或缩小可删历史，不发布丢失必要约束的摘要。模型辅助压缩的成本同样计入预算。

| 操作 | 是否创建新 session | 是否产生新 checkpoint |
| --- | --- | --- |
| 追加消息、处理任务输入 | 否 | 在下次请求前或结束前保存 |
| compact | 否 | 是 |
| 继续原工作 | 否 | 合并新增输入后保存 |
| 独立目标 | 是 | 建立首个快照 |
| 平行方案 | fork 新 session | 建立新 context 的首个快照 |

CLM 的自由 context 编辑工具、文件编辑入口和策略学习不进入第一版。后续用本基线对照测试，不与基础持久化同时引入。

## 5. 执行状态结构

TaskDescriptor、TaskSnapshot、TaskInput、TaskNotification、SessionLease 和 checkpoint 的候选载荷见[接口实现参考](integration-reference.md)。任务、通知和租约服务于执行协作；它们不是与用户知识并列的记忆类型。

### 5.1 Router turn 与执行动作

```typescript
interface TurnHandle {
  turnId: ID;
  routerSessionId: ID;
  trigger: "user_input" | "task_notification";
  inputEvent: EvidenceRef;
  status: "queued" | "running" | "completed" | "failed" | "canceled";
}
interface StepState {
  stepId: ID;
  runId: ID;
  sessionId: ID;
  status: "preparing" | "model_pending" | "tool_pending" | "completed" | "failed";
  checkpoint?: ContextRef;
  pendingActionIds: ID[];
}
type AgentAction = {
  actionId: ID;
  runId: ID;
  sessionId: ID;
  stepId?: ID;
} & (
  | { kind: "invoke_model"; checkpoint: ContextRef }
  | { kind: "execute_tool"; call: ToolCall }
  | { kind: "dispatch_task"; task: TaskDescriptor }
  | { kind: "wait"; reason: string }
  | { kind: "finish_task"; update: TaskUpdate }
  | { kind: "finish_turn"; output?: EvidenceRef }
);
type ActionCompletion =
  | { kind: "model"; message: Message }
  | { kind: "tool"; message: Message }
  | { kind: "error"; message: string; outcome: "failed" | "unknown" };

interface RuntimeActionState {
  actionId: ID;
  status: "not_started" | "running" | "completed" | "failed" | "unknown";
  completion?: ActionCompletion;
}
```

dispatch_task、wait 和 finish 由 Core 处理；invoke_model 与 execute_tool 才交给 Driver。Router 的 wait 只表示记录排队或结束本次路由尝试，不能同步等待 Worker 占住 turn。task 完成不是原 dispatch 动作的异步第二次 completion，通知有独立身份和 turn。

### 5.2 进程、同步与配置

```typescript
interface WorkerRequirements { modelProfile: string; tools: string[]; contextFormat: string; }
interface WorkerHandle { processId: ID; driverId: string; capabilities: AdapterCapabilities; }

interface SyncState {
  sessionId: ID;
  lease?: SessionLease;
  hostLink?: HostSessionLink;
  acceptedThrough: number;
  projectedThrough: number;
  nextSequence: number;
  latestCheckpoint?: ContextRef;
  pendingActionIds: ID[];
  changesCursor?: string;
}
interface AdapterCapabilities {
  executionMode: "standalone" | "host_plugin";
  durableEventExport: boolean;
  safeRequestGate: boolean;
  contextRewrite: "none" | "regions" | "full_working_view";
  requestCapture: "working_view" | "model_messages" | "provider_payload";
  attachExistingSession: boolean;
  resetOrIsolateProcess: boolean;
  actionReconciliation: boolean;
}
interface HarnessConfig {
  memoryEndpoint: string;
  credentialRef: string;
  stateDir: string;
  executionMode: "standalone" | "host_plugin";
  contextPolicy: "append_compact";
  maxContextTokens: number;
  compactAtTokens: number;
  maxWorkerSteps: number;
}
```

SyncState 只保存传输与当前工作副本进度；最新已确认 checkpoint 在 Memory。进程恢复不能仅凭本地 projectedThrough 跳过未存储的增量，也不能把请求记录当成外部动作完成。

## 6. dsh 与 pi 适配

以下为 2026-11-16 核对的候选扩展点。接入要求变为：导出最终事件、准备并记录实际输入、放行原生请求；不调用 Memory 的上下文激活接口。

### 6.1 dsh 适配

核对日期：2026-11-16，依据官方仓库 master 文档。这里给出候选映射；实现时需固定 Host 版本，并通过契约验证后才声明支持能力。

| Genvagent 需求 | dsh 接入点与约束 |
| --- | --- |
| 插件加载/卸载 | Cordis 插件的 apply(ctx)、依赖注入及受生命周期管理的注册 |
| 会话开启/恢复 | agent/created；读取原生 session 与恢复位置 |
| 同步最终记录 | session/event；工具最终结果使用 tools/result，避免把流式 delta 重复记为最终消息 |
| 请求前检索注入 | agent/pre-step；它处理进入该步骤的新消息，不等同于任意重写完整历史 |
| 工作历史替换 | 通过受支持的 session surface replacement 或 compaction backend，保留原始 log 和替换来源 |
| 上下文操作工具 | ctx.tools.register()，继续使用 Host 工具权限检查 |
| 子 agent / steering | Host 公开 agent 与 subagent 能力，按 adapter 实际支持程度开放 |

dsh 使用追加 session log 派生模型消息；surface replacement 可以改变后续工作视图而保留原记录。适配时必须遵守 replacement 范围、来源覆盖和工具配对规则，不能直接覆盖 JSONL。compaction backend 还要求其原生 bracket 与恢复标记。[Session 契约](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/session.md)、[Compaction 契约](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/compaction/compaction/README.md)

插件依赖公开 Agent 接口而非具体 agent-loop 实现。请求 gate、输入准备和 capture 覆盖范围要在固定版本上验证；只有检索注入时按 Recall 等级发布。[核心扩展点](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/core.md)、[插件 cookbook](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cookbook/extension-cookbook.md)

### 6.2 pi 适配

核对日期：2026-11-16，依据 earendil-works/pi 官方 main 文档。pi adapter 以原生 TypeScript extension 加载。

| Genvagent 需求 | pi 接入点与约束 |
| --- | --- |
| 生命周期 | session_start、session_shutdown；同时处理 session 替换和分支操作 |
| 请求视图变换 | context 可以改变对话消息，保留 Host system/tool 处理；更广变换需谨慎使用 context_with_system |
| 更靠近供应商的捕获 | before_provider_request；仅在当前 transport 暴露该阶段时可用 |
| 最终消息/工具结果 | message_end 及 tool_execution_end；对同一 toolCallId 去重并等待规范化消息记录 |
| Host 压缩 | session_before_compact；与 Genvagent 上下文改写确定唯一责任，避免双重压缩 |
| 持久插件状态 | pi.appendEntry() 等原生状态记录，用于恢复 marker 和同步位置 |
| 上下文工具 | pi.registerTool()；继续由 Host 执行工具和权限流程 |

context handler 之间会链式变换，不能假设本插件一定最后执行。可在后续 provider hook 核对并记录最终请求；若该传输不支持所需 hook，则降低 capture 能力或不启用 strict 模式。hook 内必须传递取消信号。[官方 Extensions 文档](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/extensions.md)

pi 会话通过 id/parentId 表达树状历史。只导出当前分支对应的可见路径，并保存原生位置；导航到旧节点后另起执行分支时创建新的逻辑 session，并记录 HostSessionLink。已经通过 parentContext 继承的历史保留来源引用，不作为新用户输入再次导出。持久 marker 是插件元数据，不应被当成重复的用户消息注入模型。[Session 格式](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/session-format.md)




## 7. 恢复与交付

恢复时定位逻辑 session，取得有效 lease，读取最新 checkpoint 与后续事件，恢复控制状态，再核对未完成动作。Host 原生会话映射用于定位和兼容性判断，不是业务状态的唯一来源。

第一版交付 Router 的短事件循环、Worker 独立 loop、TaskSnapshot/通知、append + compact、恢复和 Session Maintenance 触发。CLM 与资源虚拟化延后；资源管理扩展可以保留 Driver 边界，不预先实现专有句柄服务。

验证重点是 Worker 执行时 Router 仍能处理输入；同一任务不重复启动；纠正由 Worker 安全消费；通知不会重复生成对话；消息与 checkpoint 可恢复；外部结果未知时不被自动重放。
