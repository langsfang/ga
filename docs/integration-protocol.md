# Harness 与 Memory：交互顺序、函数与数据结构

状态：设计草案 v0.7；2026-11-17。接口尚未实现。第一版采用消息追加与 compact；Router 异步分派任务，Worker 独立执行。

## 1. 先区分执行与存储

**Harness 负责执行，Memory 负责持久化与条件更新。** Memory 不启动进程、不加载模型输入、不调用模型或工具。

| 名词 | 含义 |
| --- | --- |
| Session | Memory 中稳定的逻辑工作线程，关联事件流和 context |
| Context checkpoint | 某次已保存的工作快照；只表示状态已持久化 |
| Run | 某进程处理某 session 的一次执行尝试，具有独立 runId |
| 执行环境（Runtime） | 模型客户端、工具、进程、工作目录和权限的实现 |
| Runtime Driver | Harness 调用执行环境的接口，不是 Memory 服务 |
| Host session link | dsh/pi 原生会话到逻辑 session 的定位索引 |
| Task | 可跨 Router turn 运行的异步工作，保存状态、输入和结果 |
| Session lease | 同一 session 的临时写入权，防止两个执行者并发修改 |

一个 session 可以由不同进程先后执行，事件序号和 contextId 不重置。Host session link 只帮助找到 session；它不会获得 lease，也不会启动 run。

Memory 中“最新 checkpoint”与“某次模型调用使用的输入”分开记录。前者由 saveCheckpoint 更新，后者由 Driver 准备后作为请求事件保存；快照保存成功不证明模型已运行。

## 2. Router 与 Worker 的异步协作

### 2.1 接收、分派与响应

1. 用户输入在 Router session 中持久化，开启一个短 Router turn。
2. Router 根据对话、记忆和已有 session 选择回应、澄清，或 new/resume/fork。
3. Core 保存稳定的 taskId 和目标选择，创建或核对目标 session，再 recordTask。
4. recordTask 提交后，delegate 返回 TaskAccepted，表示工作已经持久接收。Core 的后台 Dispatcher 安排 Worker。
5. Router 可回复接收状态并结束本 turn，继续接收用户输入，不等待 Worker 完成。

Router 只做对话、路由、任务查询和控制。耗时研究、文件操作、复杂分析及业务工具调用交给 Worker。一个 session 只允许一个有效 writer；选定 session 忙碌时，任务排队。

new 创建独立工作状态；resume 继续目标一致的状态；fork 从固定版本创建独立分支。普通消息追加和 compact 都不创建新 session，只保存同一 contextId 下的新版本。

### 2.2 Worker 开始执行

```text
Harness Dispatcher
  1. allocateWorker：分配进程
  2. Memory.claimSession：登记本次 run 并取得 session 写入权
  3. readCheckpoint + readEvents：取得已保存工作状态及增量
  4. Driver.attachSession：建立本地执行容器，恢复工作副本
  5. 核对未完成动作，读取待处理任务输入
  6. Worker loop 开始
```

claimSession 的服务端行为只是数据库事务：校验 session 开放、没有其他有效 writer、版本和授权有效，然后保存租约与 run 记录。它不远程调用 Driver，不代表进程已经开始工作。

Host 插件需要时，可先 resolveHostSession 找到逻辑 session，或在建立原生容器后 linkHostSession。这个适配索引与 lease 独立；纯 Genvagent Runner 不需要 Host 索引。

### 2.3 Worker 每一步如何更新 context

```text
已完成的模型消息 / 工具结果 / 当前任务输入
  → appendEvents：先保存原始事件
  → appendMessages：在 Harness 内更新 context 工作副本
  → 按阈值 compact：仍由 Harness 处理，保留约束与来源
  → saveCheckpoint：原子保存快照并推进最新持久版本
  → Driver.prepareModelCall：按快照及运行配置准备实际输入
  → appendEvents(model_request_prepared)：记录 checkpoint、输入及覆盖范围
  → Driver.invokeModel：发起模型调用
  → 保存模型输出，处理工具，再回到本循环
```

下一模型调用前、正常暂停和完成前必须保存 checkpoint。流式 delta 可先展示，完整或中断消息按实际状态落盘。原始事件与工作快照是不同对象；compact 不删除原始消息。

prepareModelCall、invokeModel 是 Harness 的本地执行接口。Memory 不提供“加载 context”或“激活模型输入”的 RPC。Host 插件在实际请求边界准备输入并放行原生调用，不另起一个重复模型请求。

### 2.4 Worker 状态如何到达 Router

Worker 在状态变化和阶段完成时追加 TaskUpdate 事件。appendEvents 在同一事务更新 TaskSnapshot；completed、failed、waiting_input 等变化同时生成持久 TaskNotification。普通心跳只更新运行活动时间，不触发 Router 模型调用。

Router 可以 getTask 查询当前阶段、简短进度、最新 checkpoint、结果与证据。需要诊断时再读取具体 Worker 事件或快照，默认不复制整个 Worker context。

任务通知开启新的 Router turn。原 delegate 调用已经由 TaskAccepted 结束，因此任务完成通知不能成为它的第二次工具结果。通知包含原 taskId 与 originTurnId，保留对话关联。

用户的纠正或取消通过 postTaskInput 写入持久任务输入队列。Worker 在安全边界读取，保存处理结果并确认该输入；Router 不越过 Worker lease 直接追加其工作消息。取消是请求，不证明外部工具已停止。

### 2.5 完成与释放

Worker 保存最终消息和 checkpoint，再提交任务终态及结果。Memory 生成通知；Harness 通过 releaseSession 结束写入占用，然后释放或清理进程。任务完成默认不退役 session。

Router 不在线时通知仍保留。重新上线后读取通知，并将其追加到自己的事件流；该追加按 notificationId 去重，同时标记通知已投递。Router 关闭不会丢失 Worker 结果；若运行 Worker 的整个 Host 退出，则按恢复机制处理，不假设它继续执行。

## 3. 函数按责任分组

### 3.1 Harness 本地执行函数

| 函数 | 调用者 → 执行者 | 实际操作 |
| --- | --- | --- |
| submitUserMessage | UI/Host → Core | 保存输入、安排 Router turn |
| dispatchTask | Router/Core → Dispatcher | 保存目标与任务，返回 TaskAccepted |
| allocateWorker | Dispatcher → Driver | 分配有能力的进程 |
| attachSession | Core → Driver | 加载工作副本，建立暂停状态的执行容器 |
| appendMessages / compactContext | Core → Context Controller | 修改本地工作视图，不直接写 Memory |
| prepareModelCall | Core → Driver | 生成本次请求输入及可记录的证明 |
| invokeModel / executeTool | Core → Driver | 真正执行模型或工具 |
| acceptCompletion | Driver → Core | 保存结果，推进 Worker loop |
| reconcile / detachSession | Core → Driver | 核对外部动作，或释放挂载 |

### 3.2 持久 session 与写入权

| Memory 函数 | 输入 | 返回 / 服务端事务 |
| --- | --- | --- |
| createSession | SessionCreate | SessionState；按稳定 ID 创建记录流与 context |
| getSession / discoverSessions | sessionId / SessionQuery | 精确状态或候选摘要 |
| claimSession | sessionId、ClaimRequest | SessionClaim；登记 run，分配单调 epoch 的租约 |
| renewSessionClaim | SessionLease、请求时长 | 新 expiresAt；校验当前 owner/epoch |
| releaseSession | SessionLease、ReleaseRequest | 保存本次执行结束记录、释放占用；不启动或停止进程 |

若输入 request 中带有 runId，Memory 只是保存其关联。真正的模型请求仍须由 Harness 调用 invokeModel。创建、续租和释放是三种存储操作。

### 3.3 原始记录与 checkpoint

| Memory 函数 | 输入 | 返回 / 服务端事务 |
| --- | --- | --- |
| appendEvents | sessionId、AppendRequest、lease | AppendReceipt；事件、消费索引、任务投影和必要通知一起提交 |
| readEvents / readEvidence | 序号范围 / EvidenceRef | 原始记录或指定记忆修订 |
| saveCheckpoint | sessionId、CheckpointWrite、lease | CheckpointHead；快照与最新指针原子提交 |
| readCheckpoint | ContextRef | 精确读取不可变快照 |

saveCheckpoint 检查 lease、基线 head revision、已存在的来源和 coveredEventSequence；成功只说明“从这里可以恢复工作状态”。Driver 之后如何加载、输入是否已发出，记录在 model_request_prepared 与后续结果事件中。

| 场景 | Harness 的操作 | Memory 的操作 |
| --- | --- | --- |
| 恢复旧 context | 读取快照和增量，构建本地工作副本 | 返回持久内容 |
| 保存新 context | 构建快照，调用 saveCheckpoint | 条件写入版本并推进 head |
| 准备模型调用 | prepareModelCall，记录准备结果 | 保存引用 checkpoint 的请求事件 |
| 发起模型调用 | invokeModel 或放行 Host 原生请求 | 不参与执行，随后保存结果 |

### 3.4 任务、通知与记忆查询

| Memory 函数 | 作用 |
| --- | --- |
| recordTask(TaskDescriptor) | 保存目标、来源 turn 和 queued 状态，关联目标 session；不调度进程 |
| getTask / listTasks | 返回 TaskSnapshot 或其分页列表，可按状态筛选，供 Router 查询和 Dispatcher 恢复队列 |
| postTaskInput / readTaskInputs | 保存或读取纠正、补充、取消等待处理输入 |
| readTaskNotifications(routerSessionId) | 读取尚未投递的任务通知；Router 追加对应事件后确认投递 |
| getUserOverview / getProfile | 用户资料目录及带来源画像 |
| recall(RecallRequest) | 按用户授权和可选条件检索 |
| proposeMemory / submitFeedback | 持久化知识候选或纠正，不保证后台整理已经结束 |
| readChanges | 用户资料和生命周期的变更流 |

TaskUpdate、任务输入确认和通知投递使用原始事件中的稳定 ID；没有另一条可随意覆盖 TaskSnapshot 的写接口。Router 创建任务及提交输入时服务端检查目标 session 仍可接收工作，并更新退役检查所需的待办索引。

### 3.5 适配与维护接口

| 接口组 | 函数 | 边界 |
| --- | --- | --- |
| Host Adapter 元数据 | linkHostSession / resolveHostSession | 保存原生会话定位关系，与执行权独立 |
| 维护请求 | requestSessionReview | 将 session 加入后台维护队列，返回 job ID |
| Memory 内部维护 | retireSession | 校验条件后原子保存退出记录并冻结 session；不在 Router 对话内执行 |
| 记忆维护 | compactMemories / archiveMemory / deleteMemory | 由 Memory 维护任务按策略执行，不由 session 退役隐式触发删除 |

v0.6 中的 bindRuntime 改为定位语义明确的 linkHostSession，startRun 改为 claimSession。原 putContext / confirmContextLoaded / activateContext 流程收敛为 saveCheckpoint；本地准备输入及实际执行分别记录。这是对“当前版本”含义的简化，不把“已保存”误报为“已执行”。

## 4. 数据结构

### 4.1 归属、引用与 session

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

### 4.2 写入租约与 Host 定位

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

### 4.3 事件、checkpoint 与模型输入

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

### 4.4 异步任务与进度

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

### 4.5 查询、记忆与维护

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

## 5. 一次具体调用示例

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

## 6. HTTP 映射与可靠性

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

## 7. 验收重点

1. Worker 运行期间，Router 可以完成新的用户输入、任务状态查询和取消请求。
2. delegate 立即返回持久接收结果；Worker 最终输出通过新的通知 turn 汇总。
3. TaskSnapshot 可追溯到 Worker 事件与 checkpoint，心跳不触发大量 Router 调用。
4. 新进程继续旧 session，原始消息与 context 增量不丢失；保存状态不被误报为已执行。
5. 第一版只需 append + compact，无 CLM 编辑依赖。
6. 后台维护主动处理 one_shot 和闲置候选；有未决工作时不能退役。
7. 退役必须保存 exit record，长期 note 可选；notes compact、归档和删除各自遵循独立语义。
