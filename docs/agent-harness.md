# Agent Harness：持久状态路由与可替换执行进程

状态：设计草案 v0.6；2026-11-17。公共函数与类型见[交互协议](integration-protocol.md)。核心对象关系见[架构](architecture.md)。

## 1. Core 的责任

Harness Core 管理逻辑执行循环；Router 决定任务与上下文策略，Worker process 提供执行能力。业务状态由 Memory 中的 session/context、事件和任务记录承载，不归某个进程独占。

| Core 模块 | 责任 |
| --- | --- |
| Turn Controller | 用户输入去重、轮次排队、纠正、取消和最终答复 |
| Agent Controller | 推进某个 run 的 step，管理模型/工具动作与完成结果 |
| Router Policy | 理解用户目标，检查记忆与候选 session，选择直接答复或 new/resume/fork |
| Session / Task Controller | 持久化目标选择，创建/继续/分叉 session，管理 run、结果投递及退役 |
| Context Controller | 消息投影、检索材料组织、CLM 编辑、checkpoint 与加载核对 |
| Memory Coordinator | 稳定事件、发送日志、服务调用、重放与变更同步 |

Runtime Driver 管理进程分配、挂载、模型、工具和执行环境。Memory Client 实现公共存储协议。第一版同一 Router conversation 对应一个逻辑 Router session；task session 可以在后续对话中复用。

## 2. 执行与路由逻辑

### 2.1 先选状态，再选进程

```text
Router 接收 task
  → 读取 profile / 有关记忆 / session 目录
  → 判断 direct / new / resume / fork / wait
  → 保存 task 及目标 session 选择
  → 创建或读取持久 session
  → 分配具备能力的 Worker process
  → startRun 获取 session lease
  → 挂载、加载、核对并确认 context
  → 推进 agent loop
  → 保存结果与 checkpoint，endRun
  → 释放进程，把结果送回 Router
```

new 和 fork 的 sessionId 由 Core 分配并在首次创建前持久化。重试分派复用已保存的选择；resume 不创建 context，进程替换也不创建 context。

一次 Worker 释放后可以承接另一 session，但 Driver 必须清理上一任务的消息、工具关联和授权。Host 若不能安全重置，就新建原生运行容器。逻辑 session 在两种情况下都保持独立。

### 2.2 Core 与运行时的 loop

| 模式 | 谁推进逻辑状态 | 谁执行模型/工具 |
| --- | --- | --- |
| Core-owned | Runner 调用 advance(runId)，完成后继续 | 参考 Driver |
| Host-driven | Adapter 在原生安全边界调用 advance(runId) | Host 只执行 Core 放行的动作 |

一个 run 只选一种模式。Host-driven 的 execute(invoke_model) 表示放行原生请求，不再额外发起 SDK 调用。无法控制请求 gate 的 Host 只提供同步/检索能力。

Worker loop 为：提交 context → 模型调用 → 保存输出 → 处理工具/检索 → 保存结果 → 更新 context → 下一 step。Task 结束后产生结果，不因为进程结束就销毁 session。

advance 遇到待确认动作时返回等待或原动作，不产生第二个动作 ID。Driver 按 actionId 持久去重；未知外部结果先 reconcile，不自动重放。

### 2.3 Router 的 CLM 策略

CLM 的核心是让模型编辑下一步实际使用的上下文，而不仅是产出一段外部摘要。[论文](https://arxiv.org/html/2609.37725v1)

本项目将这一机制用于 Router 的工作状态管理；以下是设计选择，不是已验证的性能结论：

| 工具/能力 | Router 可做的操作 | Core 校验 |
| --- | --- | --- |
| context.inspect | 查看本 session 的目标、约束、消息及证据引用 | 当前 run 的读取权限 |
| context.edit | 重排、改写可编辑块，维护候选 session 和未决问题 | baseVersion、保护块、来源、工具配对、预算 |
| session.search / inspect | 查找可继续工作，读取摘要或固定快照 | user、授权、生命周期、摘要来源版本 |
| memory.recall / read | 补充或重新读取材料 | 可见范围、修订与时效 |
| 路由工具 | 提议 new/resume/fork 并组织输入 | 稳定目标、lease、执行环境与待处理动作 |

Router 自己的可编辑 context 可记录当前用户意图、路由依据和多任务进展。对繁忙 session 的修改必须等待或分叉；跨 session 读取不等于获得写入权。

编辑保留原始证据，并经 putContext → applyContext → activateContext 后影响下一调用。可选文件入口只是对同一版本化对象的映射。Memory Agent 的后台整理不直接修改 Router 当前工作视图。

## 3. 接口函数

### 3.1 Core 的入口

```typescript
interface HarnessCorePort {
  openConversation(request: SessionCreate): Promise<SessionState>;
  submitUserMessage(routerSessionId: ID, inputId: ID, content: Content): Promise<TurnHandle>;
  cancelTurn(turnId: ID): Promise<void>;
  advance(runId: ID): Promise<AgentAction>;
  acceptCompletion(actionId: ID, completion: ActionCompletion): Promise<void>;
}
```

openConversation 仅接受 router 角色，按稳定 sessionId 创建或读取；Host 恢复时先 resolveRuntime 取得逻辑身份。Task session 由 Session/Task Controller 操作，不能由 allocateWorker 隐式创建。

用户输入先持久化；排队消息在 startTurn 时才按原始引用进入 context。完成回调同样先保存事件，再更新工作视图；原生消息与工具通知按稳定来源身份归并，避免重复记录。

### 3.2 Core 调用 Runtime

```typescript
interface RuntimeDriver {
  capabilities: AdapterCapabilities;
  allocateWorker(request: WorkerRequirements): Promise<WorkerHandle>;
  attachSession(worker: WorkerHandle, session: SessionDescriptor, lease: RunLease): Promise<RuntimeBindingInput>;
  applyContext(plan: ContextPlan): Promise<RuntimeApplicationReceipt>;
  execute(action: AgentAction & { kind: "invoke_model" | "execute_tool" }): Promise<void>;
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

allocateWorker 只分配运行资源；attachSession 建立暂停的运行挂载。实际模型调用必须等待本 run 的 context 应用确认。现有 Router 的 Host 进程可作为已分配资源交给同一 Driver 管理。

原样加载当前版本后调用 Memory.confirmContextLoaded；context 发生变化才走新版本激活。新 run 必须生成自己的 marker，不能复用旧 run 的应用 receipt 作为加载完成证明。

execute 接收动作不代表成功，结果经 acceptCompletion 回流。detach/release 不能使未决外部动作被视为已完成。正常 endRun 后仍需清理运行容器；清理失败时该进程不可复用，但已保存 session 不受影响。

### 3.3 Core 内部函数

| 模块 | 函数 | 语义 |
| --- | --- | --- |
| Router Policy | decideRoute | 形成路由提案与理由，Core 为新目标分配稳定标识 |
| Session Controller | resolveTarget | 按已提交提案 create/get/fork session |
| Session Controller | prepareRun / retireSession | 取得写入权、校验兼容性；在终止条件满足时退役 |
| Task Controller | dispatchTask / deliverResult | 记录目标、启动 run、幂等投递结果 |
| Context Controller | appendMessages / buildContext / editContext | 已提交事件投影与 CLM 编辑 |
| Context Controller | load / checkpoint | 恢复当前视图，或保存并确认新版本 |
| Memory Coordinator | commitEvents / replay / pollChanges | 可靠追加、恢复与失效同步 |

dispatch_task、finish_task、finish_turn 和 wait 由 Core 处理；只有 invoke_model 与 execute_tool 交给 Driver.execute。第一版的进程池只需按能力分配空闲进程，不扩展为独立通用调度平台。

## 4. 关键数据结构

### 4.1 路由、任务与结果

```typescript
type RouteTarget =
  | { mode: "new"; sessionId: ID; retention: "reusable" | "one_shot" }
  | { mode: "resume"; sessionId: ID; expectedHeadRevision: number }
  | { mode: "fork"; sessionId: ID; parent: ContextRef; retention: "reusable" | "one_shot" };

interface RouteDecision {
  action: "answer" | "dispatch" | "wait" | "clarify";
  target?: RouteTarget;           // dispatch 时必填
  reason: string;
  evidence: EvidenceRef[];
}

interface TaskSpec {
  taskId: ID;
  userId: ID;
  routerSessionId: ID;
  parentTurnId: ID;
  dispatchActionId: ID;
  delegateToolCallId: ID;
  target: RouteTarget;
  goal: string;
  constraints: string[];
  inputRefs: EvidenceRef[];
  readGrant: ReadGrant;
  maxSteps: number;
}

type TaskResult = {
  taskId: ID;
  resultId: ID;
  runId: ID;
  sessionId: ID;
  output: Content;
  sourceRefs: EvidenceRef[];
  memoryProposalIds: ID[];
} & (
  | { status: "completed" | "needs_input"; finalContext: ContextRef }
  | { status: "failed" | "canceled"; finalContext?: ContextRef }
);

interface TaskState {
  spec: TaskSpec;
  status: "created" | "running" | "waiting_input" | "completed" | "failed" | "canceled";
  resolvedSessionId?: ID;
  currentRunId?: ID;
  result?: TaskResult;
  deliveredToRouterEventId?: ID;
}
```

目标 session 选择与实际进程分开保存。继续已有任务可以沿用 taskId 并创建新 run；新的用户工作也可以创建新 taskId 并选择同一 session。第一次 resume 的 head 冲突需重新检查，不能覆盖他人已追加的进度。

TaskResult 正常完成或等待用户补充时引用最终 checkpoint；异常发生在首个快照前时，失败结果省略 finalContext 并保留诊断来源。结果投递引用原 dispatchActionId，先确认输出，再追加 Router 的工具结果。迟到结果保留原 task/run 关联，不无条件进入新的 turn。

### 4.2 Turn、Step 与动作

```typescript
interface TurnHandle {
  turnId: ID;
  routerSessionId: ID;
  inputEvent: EvidenceRef;
  status: "queued" | "running" | "completed" | "awaiting_user" | "failed" | "canceled";
}

interface StepState {
  stepId: ID;
  runId: ID;
  sessionId: ID;
  turnId: ID;
  status: "preparing" | "model_pending" | "action_pending" | "completed" | "failed";
  inputContext?: ContextRef;
  pendingActionIds: ID[];
  resultRefs: EvidenceRef[];
}

type AgentAction = {
  actionId: ID;
  runId: ID;
  sessionId: ID;
  turnId: ID;
  stepId?: ID;
} & (
  | { kind: "invoke_model"; context: ContextRef }
  | { kind: "execute_tool"; call: ToolCall }
  | { kind: "dispatch_task"; task: TaskSpec }
  | { kind: "wait"; pendingActionIds: ID[] }
  | { kind: "finish_task"; result: TaskResult }
  | { kind: "finish_turn"; status: "completed" | "awaiting_user" | "failed" | "canceled"; output?: EvidenceRef }
);

type ActionCompletion =
  | { kind: "model"; message: Message }
  | { kind: "tool"; message: Message }
  | { kind: "task"; result: TaskResult }
  | { kind: "error"; message: string; outcome: "failed" | "unknown" };

interface RuntimeActionState {
  actionId: ID;
  status: "not_started" | "running" | "completed" | "failed" | "unknown";
  completion?: ActionCompletion;
}
```

逻辑控制状态由事件恢复，不能仅重放 context 消费位置之后的消息。finish_turn 先确认包含最终答复的 checkpoint，再追加引用它的终态事件。finish_task 同样保存结果和执行终态后才回传。

### 4.3 进程、context 与同步

```typescript
interface WorkerRequirements {
  modelProfile: string;
  tools: string[];
  contextFormat: string;
}

interface WorkerHandle {
  processId: ID;
  driverId: string;
  capabilities: AdapterCapabilities;
}

interface ContextPlan {
  planId: ID;
  sessionId: ID;
  lease: RunLease;
  baseNativeRevision: string;
  version: ContextVersion;
  desired: WorkingContext;
}

interface ContextPatch {
  baseVersion: ContextRef;
  replacementEditableBlocks: WorkingBlock[];
  explanation: string;
}

interface SyncState {
  sessionId: ID;
  lease?: RunLease;
  runtimeBinding?: RuntimeBinding;
  acceptedThrough: number;
  projectedThrough: number;
  nextSequence: number;
  activeContext?: ContextRef;
  loadedContextForRun?: ContextRef;
  pendingPlanId?: ID;
  pendingActionIds: ID[];
  changesCursor?: string;
}

interface AdapterCapabilities {
  driveMode: "core_owned" | "host_driven";
  durableEventExport: boolean;
  durableMarkers: boolean;
  safeRequestGate: boolean;
  contextRewrite: "none" | "regions" | "full_working_view";
  requestCapture: "working_view" | "model_messages" | "provider_payload";
  attachExistingSession: boolean;
  resetOrIsolateProcess: boolean;
  actionReconciliation: boolean;
}
```

SyncState 是传输与当前挂载状态；session 的权威状态在 Memory。projectedThrough 不能代替远端版本的 coveredEventSequence；loadedContextForRun 在进程替换后必须重新确认。

## 5. 生命周期与恢复

| 触发 | Core 行为 |
| --- | --- |
| 独立工作或无合适状态 | new，选择最少必要证据 |
| 明确继续既有工作 | resume，检查目标、head、授权、未决动作与运行环境 |
| 平行探索或原 session 需要保持不变 | fork 固定版本，或 new 加选定来源 |
| run 正常结束 | 保存 checkpoint/result、endRun、释放进程；session 默认保持 open |
| one_shot 结束或已确认替代 | 满足退役前提后 retireSession |
| 进程崩溃 | 新 run 重建状态、核对外部动作，再确认加载 |
| session retired | 不继续写入，需要时创建分叉或从来源新建 |

退役前保存最终进度和来源，不等待生成完美的长期摘要；Memory 可以随后整理。排队输入、待澄清或待继续的 task 须先完成、明确取消或转交；工具未知结果必须核对。没有未决工作、没有活动 lease、版本与接收位置仍匹配，才允许冻结。自动策略只处理满足条件的 session。

本地 stateDir 保存原生身份映射、事件编号、未确认批次、待激活计划及 marker。所有影响任务恢复的记录最终写入 Memory。正在运行的外部进程、文件状态和供应商隐状态不属于可直接移植的 context。

## 6. dsh 与 pi 接入

下列扩展点沿用 2026-11-16 的候选资料映射。Host 原生 session 不再作为逻辑 session 的身份；Adapter 需显式实现 RuntimeBinding、挂载兼容性和 run 内请求控制。

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

插件依赖公开 Agent 接口而非具体 agent-loop 实现。请求 gate、上下文应用确认和 capture 覆盖范围要在固定版本上验证；只有检索注入时按 Recall 等级发布。[核心扩展点](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/core.md)、[插件 cookbook](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cookbook/extension-cookbook.md)

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

context handler 之间会链式变换，不能假设本插件一定最后执行。可在后续 provider hook 核对最终请求并确认激活；若该传输不支持所需 hook，则降低 capture 能力或不启用 strict 模式。hook 内必须传递取消信号。[官方 Extensions 文档](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/extensions.md)

pi 会话通过 id/parentId 表达树状历史。只导出当前分支对应的可见路径，并保存原生位置；导航到旧节点后另起执行分支时创建新的逻辑 session，并记录 RuntimeBinding。已经通过 parentContext 继承的历史保留来源引用，不作为新用户输入再次导出。持久 marker 是插件元数据，不应被当成重复的用户消息注入模型。[Session 格式](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/session-format.md)



## 7. 配置与验收

```typescript
interface HarnessConfig {
  memoryEndpoint: string;
  credentialRef: string;
  stateDir: string;
  driveMode: "core_owned" | "host_driven";
  syncMode: "strict" | "buffered";
  contextMode: "recall" | "rewrite";
  routerStrategy: string;
  maxContextTokens: number;
  maxWorkerSteps: number;
}
```

配置不要求 project、collection 或固定用户画像字段。受控模式启动前检查 gate、marker、格式与恢复能力；能力不足时明确采用 Sync/Recall 等级。

验收重点是：同一 session 由不同进程继续；同一进程清理后执行不同 session；new/resume/fork 和退役符合协议；CLM 修改真正进入下一次输入；同模型与预算下比较路由正确率、约束保留、无关历史干扰、延迟与 token 成本。
