# 接口实现参考

> 已归档（v0.16 及以前），当前设计以 [概念模型](../concepts.md) 为准。
>
> 历史实现候选，非当前实现契约。当前设计见 [Harness 架构](../agent-harness.md)、[Memory 架构](../memory-system.md)与[最小接口协议](../integration-protocol.md)。本文中的 Session 级锁、History、Driver 和旧函数签名仅供历史查阅。

状态：实现候选，随概念模型 v0.9 整理；2026-11-17。未实现、未冻结。

本文件保留两系统的载荷和 HTTP 方案，以及 Harness SDK 与 Host 适配候选，供概念模型明确后评审。领域对象及能力边界以[主文档](../integration-protocol.md)为准；不能把这里的传输结构直接当成领域模型。

例如 SessionState 是便于客户端读取的聚合响应，其中包含 session、checkpoint 和有关执行状态，并不表示 lease、task 都是 Session 记忆内容。RecordInput 是统一记录封装，业务经历与控制记录在查询、整理时仍须分开处理。

## 1. 调用边界与候选映射

| 能力与提供方 | 抽象对象 | 实现候选 |
| --- | --- | --- |
| Memory 领域接口：会话管理 | Session | createSession、getSession、discoverSessions；分叉通过固定来源创建；requestSessionReview 提交维护意图 |
| Memory 领域接口：会话记录 | History / 来源 | appendEvents、readEvents、readEvidence |
| Memory 领域接口：工作状态 | Context | saveCheckpoint、readCheckpoint |
| Memory 领域接口：记忆使用与维护入口 | 记忆条目、视图、候选与纠正 | getUserOverview、getProfile、recall、proposeMemory、submitFeedback |
| Memory 执行协作接口 | task 状态、输入、通知、临时写入权 | recordTask、getTask、listTasks、postTaskInput、readTaskInputs、readTaskNotifications、claimSession/renewSessionClaim/releaseSession |
| Memory 提供的可选适配元数据接口 | Host 原生会话定位 | linkHostSession、resolveHostSession |
| Memory 内部维护实现 | session 生命周期、条目生命周期 | retireSession、compactMemories、archiveMemory、deleteMemory |
| Harness 内部执行端口，由 Driver/Host 提供 | 进程、调用和外部结果 | 见本文第 6 节及 [Harness 接口边界](../agent-harness.md)，不属于 Memory 领域 API |

后台维护接口与用户发起维护意图不同：Harness 可以请求检查或纠正，不需要直接调用 Memory 内部的 job 和事务函数。

以下数据结构按实现用途保留。共有身份和引用是基础类型；session/history/context 对应领域对象；任务、租约和 Host 映射属于支撑数据；SessionExitRecord 与 RetireRequest 属于维护实现。

## 2. 载荷候选

### 2.1 归属、引用与 session

```typescript
type ID = string;
type Timestamp = string;
type JsonValue = null | boolean | number | string
  | JsonValue[] | { [key: string]: JsonValue };
interface UserDomain { userId: ID; }

type MemoryKind = "fact" | "preference" | "goal" | "procedure" | "episode" | "note";
type MemoryHorizon = "short_term" | "long_term";
type MemoryStatus = "active" | "superseded" | "archived" | "deleted";
interface BlobRef { blobId: ID; sha256: string; mediaType: string; byteSize: number; }
type Content = { type: "inline"; value: JsonValue } | { type: "blob"; ref: BlobRef };
interface ContextRef { contextId: ID; versionId: ID; }
type EvidenceRef = { kind: "event"; sessionId: ID; eventId: ID }
  | { kind: "memory"; memoryId: ID; revision: number };
type ReadGrant = { kind: "user" }
  | { kind: "task"; taskId: ID; sessionIds: ID[]; refs: EvidenceRef[]; collectionIds: ID[] };

interface SessionCreate {
  sessionId: ID;
  role: "router" | "task";
  purpose: string;
  retention: "reusable" | "one_shot";
  parentContext?: ContextRef;
}
interface SessionDescriptor extends SessionCreate {
  domain: UserDomain;
  streamId: ID;
  contextId: ID;
  lifecycle: "open" | "retired";
  revision: number;
  createdAt: Timestamp;
  retiredAt?: Timestamp;
}
interface SessionState {
  session: SessionDescriptor;
  acceptedThrough: number;
  head: CheckpointHead;
  lease?: SessionLease;
  pendingTaskIds: ID[];
  queuedInputRefs: EvidenceRef[];
  unresolvedActionIds: ID[];
}
interface SessionQuery { query?: string; lifecycle?: "open" | "retired"; cursor?: string; limit: number; }
interface SessionPage {
  items: { session: SessionDescriptor; head: CheckpointHead; summary: string; summarySource?: ContextRef; busy: boolean }[];
  nextCursor?: string;
}
```

fork 的 parentContext 是固定初始化来源，不是新 session 已执行的证明。新流序号从 1 起步，首个 checkpoint 使用新 contextId；未决工具执行权不随快照复制。retired session 可读不可写，需要时从固定历史创建分叉。

### 2.2 写入租约与 Host 定位

```typescript
interface ClaimRequest {
  runId: ID;
  taskId?: ID;
  executorId: ID;
  expectedHeadRevision: number;
  readGrant: ReadGrant;
  ttlSeconds: number;
}
interface SessionLease {
  sessionId: ID;
  runId: ID;
  executorId: ID;
  epoch: number;
  expiresAt: Timestamp;
}
interface SessionClaim { state: SessionState; lease: SessionLease; }
interface ReleaseRequest {
  outcome: "completed" | "paused" | "failed";
  checkpoint?: ContextRef;
  resultRefs: EvidenceRef[];
}

interface HostSessionKey { hostInstanceId: ID; nativeSessionId: string; nativeBranchId: string; }
interface HostSessionLink {
  sessionId: ID;
  key: HostSessionKey;
  host: "dsh" | "pi" | "other";
  hostVersion: string;
}
```

HostSessionKey 在用户域内唯一映射到逻辑 session。同一 session 可先后关联多个原生容器。SessionLease 控制写入；租约过期只说明失去写入权，不说明外部工具已经停止。

### 2.3 事件、checkpoint 与模型输入

```typescript
interface ToolCall { toolCallId: ID; name: string; arguments: JsonValue; }
interface Message {
  messageId: ID;
  speaker: "user" | "assistant" | "tool";
  content: Content;
  toolCalls?: ToolCall[];
  inReplyToToolCallId?: ID;
  completeness: "complete" | "interrupted";
}
interface RecordInput {
  eventId: ID;
  sequence: number;
  originId: string;
  producerRunId: ID;
  actor: { kind: "user" | "router" | "worker" | "runtime"; id: ID };
  kind: "message" | "turn_state" | "action_state" | "model_request_prepared"
    | "task_update" | "task_input_applied" | "task_notification" | "observation" | "feedback";
  turnId?: ID;                    // Worker 可以在原 Router turn 结束后继续
  taskId?: ID;
  stepId?: ID;
  occurredAt: Timestamp;
  payload: Content;
  sourceRefs: EvidenceRef[];
}
interface AppendRequest { afterSequence: number; records: RecordInput[]; }
interface AppendReceipt { acceptedThrough: number; userRevision: number; }
interface EventPage { records: RecordInput[]; nextSequence: number; hasMore: boolean; }

interface WorkingBlock {
  id: ID;
  category: "goal" | "constraint" | "memory" | "evidence" | "progress" | "message" | "tool_exchange";
  content: Content;
  sourceRefs: EvidenceRef[];
  protected: boolean;
}
interface WorkingContext { format: "genvagent.context.v1"; goal: string; blocks: WorkingBlock[]; }

interface ContextCheckpoint {
  ref: ContextRef;
  body: Content;
  format: string;
  bodyHash: string;
  coveredEventSequence: number;
  sourceRefs: EvidenceRef[];
  recalledAtUserRevision: number;
}
interface CheckpointWrite { expectedHeadRevision: number; checkpoint: ContextCheckpoint; }
interface CheckpointHead { contextId: ID; revision: number; latest?: ContextRef; }

interface ModelInputRecord {
  actionId: ID;
  stepId: ID;
  runId: ID;
  checkpoint: ContextRef;
  input: Content;
  inputHash: string;
  coverage: "working_view" | "model_messages" | "provider_payload";
}
```

eventId 在用户域唯一，originId 在 session 内唯一；sequence 跨 run 连续。新执行者补传旧批次时沿用事件及 producerRunId，使用当前有效 lease 写入；旧结果不能冒充新执行。

checkpoint 的 coveredEventSequence 表示工作投影已处理的位置，不代表全部排队输入、task 或工具都已完成。控制状态独立从记录和任务存储恢复。准备请求后发生新用户纠正或取消时，Core 校验输入位置，必要时重新准备。

### 2.4 异步任务与进度

```typescript
type SessionTarget =
  | { mode: "new"; sessionId: ID; retention: "reusable" | "one_shot" }
  | { mode: "resume"; sessionId: ID; expectedHeadRevision: number }
  | { mode: "fork"; sessionId: ID; parent: ContextRef; retention: "reusable" | "one_shot" };

interface TaskDescriptor {
  taskId: ID;
  routerSessionId: ID;
  originTurnId: ID;
  goal: string;
  constraints: string[];
  target: SessionTarget;
  inputRefs: EvidenceRef[];
  readGrant: ReadGrant;
  maxSteps: number;
}
type TaskStatus = "queued" | "running" | "waiting_input" | "completed" | "failed" | "canceled";
interface TaskAccepted { taskId: ID; sessionId: ID; status: "queued"; }

interface TaskUpdate {
  taskId: ID;
  status: TaskStatus;
  phase: string;
  summary: string;
  completedSteps: number;
  checkpoint?: ContextRef;
  resultRefs: EvidenceRef[];
}
interface TaskSnapshot extends TaskUpdate {
  revision: number;
  sessionId: ID;
  runId?: ID;
  updatedAt: Timestamp;
  pendingInputIds: ID[];
  activity?: { lastSeenAt: Timestamp; leaseExpiresAt?: Timestamp; claimState: "claimed" | "released" | "expired" };
}
interface TaskInput {
  inputId: ID;
  taskId: ID;
  kind: "message" | "cancel";
  content: Content;
  sourceRefs: EvidenceRef[];
}
interface TaskNotification {
  notificationId: ID;
  routerSessionId: ID;
  taskId: ID;
  originTurnId: ID;
  taskRevision: number;
  kind: "waiting_input" | "completed" | "failed" | "canceled";
  sourceRefs: EvidenceRef[];
  deliveredEventId?: ID;
}
```

recordTask 检查目标开放，并与 pendingTask 索引一起提交。重复 taskId 返回原接受结果；不同内容不能静默覆盖。Worker 的 TaskUpdate 由其当前 task/run 授权约束；普通变化不直接改变 Router context。

task_input_applied 在保存实际输入处理事件时确认 inputId，不能读到就先确认。task_notification 追加到目标 Router 流时，在同一事务设置 deliveredEventId，重复通知不产生重复 turn。通知触发的 turn 使用稳定通知 ID 创建。

completed 和正常 waiting_input 需要已保存 checkpoint；异常失败可以只有最后已确认状态。TaskSnapshot 的状态与 lease 活性分别呈现，心跳丢失不直接推导业务失败。

### 2.5 查询、记忆与维护

```typescript
interface UserOverview {
  userRevision: number;
  entries: { kind: "memory" | "collection" | "session" | "task"; id: ID; label: string; summary?: string; sourceRefs: EvidenceRef[] }[];
  nextCursor?: string;
}
interface ProfileView {
  readUserRevision: number;
  claims: { key: string; text: string; refs: EvidenceRef[]; status: "supported" | "inferred" | "disputed"; validUntil?: Timestamp }[];
}
interface RecallRequest {
  runId: ID;
  query: string;
  corpora: ("memories" | "session_events" | "user_events")[];
  horizons?: MemoryHorizon[];
  labels?: string[];
  collectionIds?: ID[];
  requiredRefs: EvidenceRef[];
  minUserRevision?: number;
  maxItems: number;
  maxBytes: number;
}
interface RecallItem {
  ref: EvidenceRef;
  text: string;
  sourceRefs: EvidenceRef[];
  epistemicStatus: "supported" | "inferred" | "unverified" | "disputed";
  stale: boolean;
  memoryStatus?: MemoryStatus;
  validUntil?: Timestamp;
}
interface RecallResponse { readUserRevision: number; items: RecallItem[]; missingRequiredRefs: EvidenceRef[]; }
interface MemoryProposal {
  proposalId: ID;
  producerRunId: ID;
  memoryId?: ID;
  expectedRevision?: number;
  kind: MemoryKind;
  horizon: MemoryHorizon;
  content: Content;
  labels: string[];
  collectionIds: ID[];
  applicability: string[];
  sourceRefs: EvidenceRef[];
}
interface FeedbackInput { feedbackId: ID; sourceEvent: EvidenceRef; target: EvidenceRef; kind: "correction" | "stale" | "useful" | "rejected"; }

interface SessionExitRecord {
  sessionId: ID;
  reason: string;
  summary: string;
  checkpoint?: ContextRef;
  eventThrough: number;
  sourceRefs: EvidenceRef[];
  successorSessionId?: ID;
}
interface RetireRequest {
  expectedSessionRevision: number;
  expectedHeadRevision: number;
  expectedAcceptedThrough: number;
  exitRecord: SessionExitRecord;
}
interface ServerCapabilities {
  protocolMajor: number;
  protocolMinor: number;
  contextFormats: string[];
  maxInlineBytes: number;
  maxBatchEvents: number;
}
```

画像、集合和检索都受 user/run 授权过滤。分类可为空；requiredRefs 精确读取。记忆提案持久化不等于知识已验证。SessionExitRecord 与可选的长期 note 分开，退役不等待知识提取成功。

## 3. 序列化示例

Worker P2 继续 session-trip，当前 checkpoint 为 v8，已覆盖事件 40：

1. Harness 分配 P2，调用 claimSession，Memory 返回新 epoch 的 SessionLease。
2. Harness 读取 v8 和 40 之后的事件；Driver 恢复本地工作副本。
3. 新输入 e-41 通过 appendEvents 保存；appendMessages 合入工作视图。
4. Harness 调用 saveCheckpoint，提交覆盖至 41 的 v9。Memory 保存 v9 并把最新指针推进到 v9。
5. Driver 准备请求，Core 保存引用 v9 的 ModelInputRecord，然后才调用模型。
6. 模型输出和工具结果继续追加；任务阶段更新形成 TaskSnapshot，重要状态产生通知。

步骤 4 成功、步骤 5 尚未执行时，Memory 只确认“v9 已保存”；模型是否调用由请求与结果记录另行核对。保存快照与执行请求分别提交。

```json
{
  "expectedHeadRevision": 8,
  "checkpoint": {
    "ref": {
      "contextId": "context-trip",
      "versionId": "v9"
    },
    "body": {
      "type": "inline",
      "value": {
        "format": "genvagent.context.v1",
        "goal": "继续行程规划",
        "blocks": [
          {
            "id": "progress-40",
            "category": "progress",
            "content": {
              "type": "inline",
              "value": "此前已收集候选行程，尚待确认最终方案。"
            },
            "sourceRefs": [
              {
                "kind": "event",
                "sessionId": "session-trip",
                "eventId": "e-40"
              }
            ],
            "protected": false
          },
          {
            "id": "message-41",
            "category": "message",
            "content": {
              "type": "inline",
              "value": {
                "messageId": "m-41",
                "speaker": "user",
                "content": {
                  "type": "inline",
                  "value": "继续之前的行程，尽量减少换乘。"
                },
                "completeness": "complete"
              }
            },
            "sourceRefs": [
              {
                "kind": "event",
                "sessionId": "session-trip",
                "eventId": "e-41"
              }
            ],
            "protected": true
          }
        ]
      }
    },
    "format": "genvagent.context.v1",
    "bodyHash": "sha256:<64-hex-digits>",
    "coveredEventSequence": 41,
    "sourceRefs": [
      {
        "kind": "event",
        "sessionId": "session-trip",
        "eventId": "e-40"
      },
      {
        "kind": "event",
        "sessionId": "session-trip",
        "eventId": "e-41"
      }
    ],
    "recalledAtUserRevision": 208
  }
}
```

示例假设 e-40 保存此前的工作摘要，e-41 是新的用户请求；正文同时保留两者及来源。bodyHash 为占位值，实际请求使用正文的真实规范化摘要。

## 4. HTTP 与一致性机制候选

域前缀 P 为 `/v1/users/{userId}`。服务端核对认证或委托身份。写操作带 Idempotency-Key；执行相关写入还携带 run ID 与 epoch。

| 函数组 | HTTP |
| --- | --- |
| create/get/discover session | POST P/sessions；GET P/sessions/{id}；POST P/sessions:search |
| claim/renew/release | POST P/sessions/{id}:claim；POST P/sessions/{id}:renew；POST P/sessions/{id}:release |
| append/read events | POST P/sessions/{id}/events:append；GET P/sessions/{id}/events |
| save/read checkpoint | POST P/sessions/{id}/checkpoints；GET P/contexts/{contextId}/versions/{versionId} |
| record/get/list task | PUT P/tasks/{taskId}；GET P/tasks/{taskId}；GET P/tasks?status=... |
| task input | POST / GET P/tasks/{taskId}/inputs |
| notifications | GET P/sessions/{routerSessionId}/task-notifications |
| Host 定位 | POST P/host-session-links；POST P/host-session-links:resolve |
| overview/profile/recall | GET P/overview；GET P/profile；POST P/recall |
| 原文与记忆 | GET P/sessions/{sessionId}/events/{eventId}；GET P/memories/{memoryId}?revision=... |
| 提案/反馈/变更 | POST P/memory-proposals；POST P/feedback；GET P/changes |
| 维护请求 | POST P/sessions/{id}/maintenance-requests |
| 大正文 | POST P/blobs；GET P/blobs/{blobId} |

retireSession 是 Memory 内部维护提交操作，不作为 Harness 在线执行所必需的 RPC。查询、定位和 search 不改变执行状态。

保存 checkpoint 与执行动作不跨服务做原子事务。原始事件先持久化、checkpoint 条件提交、请求输入单独记录，恢复时分别核对。请求记录仅证明已准备或请求过，不证明外部动作成功；未知结果通过 reconcile 处理。

JSON 正文使用 RFC 8785 JCS 规范化后计算 SHA-256；blob 对原始字节计算。序号、revision、epoch 使用非负安全整数，拒绝重复属性和非有限数值。[JCS 规范](https://www.rfc-editor.org/rfc/rfc8785)

| 故障或竞争 | 处理 |
| --- | --- |
| 消息已存，checkpoint 未保存 | 从旧快照重放增量 |
| checkpoint 已存，请求未发出 | 准备输入，先核对 action 记录再执行 |
| 请求可能发出，结果未知 | 核对运行环境，不盲目重放工具 |
| Router 离线或通知重复 | 保留通知，按稳定 ID 幂等追加到 Router 流 |
| 旧 Worker 恢复写入 | 拒绝旧 epoch，现任执行者核对其未确认记录 |
| 新任务/写入权与退役竞争 | 事务锁与版本检查只允许合法的一方提交 |
| 索引未达 minUserRevision | 补读权威数据或返回 NOT_READY |

strict 模式在持久化或 lease 不可用时暂停新步骤。第一版以此为默认保证；本地离线执行作为后续显式选项，不静默改变保证。删除、授权撤回对历史快照和派生视图同样生效。

错误返回 code、message、retryable、requestId，区分版本/序号冲突、session 忙碌、已退役、未决动作、lease 丢失、来源删除与服务不可用。通知和 changes 至少一次交付，不把网络超时视为服务端撤销。

## 5. 实现验收候选

1. Worker 运行期间，Router 可以完成新的用户输入、任务状态查询和取消请求。
2. delegate 立即返回持久接收结果；Worker 最终输出通过新的通知 turn 汇总。
3. TaskSnapshot 可追溯到 Worker 事件与 checkpoint，心跳不触发大量 Router 调用。
4. 新进程继续旧 session，原始消息与 context 增量不丢失；保存状态不被误报为已执行。
5. 第一版只需 append + compact，无 CLM 编辑依赖。
6. 后台维护主动处理 one_shot 和闲置候选；有未决工作时不能退役。
7. 退役必须保存 exit record，长期 note 可选；notes compact、归档和删除各自遵循独立语义。

## 6. Harness SDK 候选

以下签名保留 Core 与 Driver 的实现候选。应用层对话、任务控制和回复读取的完整逻辑能力以 [Harness 接口边界](../agent-harness.md)为准，SDK 尚未冻结。

### 6.1 Core 入口

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

### 6.2 Dispatcher 与 Memory

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

### 6.3 Driver 接口

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

## 7. Harness 执行载荷候选

TaskDescriptor、TaskSnapshot、TaskInput、TaskNotification、SessionLease 和 checkpoint 的候选载荷见本文第 2 节。任务、通知和租约服务于执行协作；它们不是与用户知识并列的记忆类型。

### 7.1 Router turn 与执行动作

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

### 7.2 进程、同步与配置

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

## 8. dsh 与 pi 适配候选

以下为 2026-11-16 核对的候选扩展点。接入要求变为：导出最终事件、准备并记录实际输入、放行原生请求；不调用 Memory 的上下文激活接口。

### 8.1 dsh 适配

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

### 8.2 pi 适配

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
