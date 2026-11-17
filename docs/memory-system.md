# Memory System：用户记忆、持久执行状态与整理服务

状态：设计草案 v0.8；2026-11-17。先读[概念模型与能力边界](integration-protocol.md)；函数与载荷的实现候选见[接口实现参考](integration-reference.md)。本文描述 Memory 内部管理与存储设计。

## 1. 记忆对象与视图

Memory 以 userId 为唯一归属域，面向普通问答、学习、生活事务和工作等场景。项目、主题集合和固定画像字段都不是使用前提。

### 1.1 领域对象及其归属

| 对象 | 保存内容 | 主要写入方 |
| --- | --- | --- |
| Session | 持续工作的身份、目的、生命周期及相关引用；归属于用户 | Harness 选择与使用，Memory 管理持久生命周期 |
| History | Session 的经历与控制记录；Event 是其中的记录单元 | Harness 报告输入、观察与变化，Memory 保存 |
| Context | Session 当前选定的工作状态及历史版本 | Harness 构建，Memory 保存 |
| 记忆条目 | 用户域内带来源的事实、偏好、目标、方法、经历和 notes | 提案者 + Memory Agent / Validator |

Session 组织 History 与 Context；记忆条目可以引用其来源，但不受来源 Session 的存续限制。外部材料可作为证据被记录引用，不要求把材料正文复制进每个 Session。

证据证明某次陈述或观察发生，不自动证明其内容正确。记忆条目保留适用条件、时间与修订。Worker process 不是存储分区。

### 1.2 画像、短期、长期与 notes

| 视图或维度 | 数据来源 | 解释 |
| --- | --- | --- |
| 用户画像 | 与用户有关的事实、偏好、目标及其证据 | 派生视图；区分已支持、推断和冲突，允许为空 |
| 工作记忆 | 某个 Session 当前选定的目标、材料与进度 | 由其 Context 表达 |
| 短期记忆 | 近期有关经历、当前目标、未决事项、相关 Session 的工作状态及短期条目 | 当前工作需要的信息，仍可持久化 |
| 长期记忆 | 经验证适合跨会话复用的记忆条目 | 可被纠正或失效，长期不等于永久正确 |
| 历史存档 | 旧 History 与 Context 版本 | 可追溯的资料，不自动属于长期知识 |
| Note | MemoryKind 为 note 的整理文本 | 表达形式，可以服务短期工作或长期复用 |
| Collection / 标签 | 对记忆和相关来源的可选组织关系 | 可以不存在；旅行、学习、项目等均可作为名称 |

这些维度可重叠。例如“本次旅行预算”可以是短期 goal/constraint 的来源，而“用户明确要求优先采用公共交通”可以作为有适用条件的 preference；二者均可能出现在某份 note 中。它们不会因被写成 note 就成为永久用户画像。

工作 context 只是某个 session 的选定工作视图，不包含该用户全部短期或长期记忆。记忆从短期转为长期需经过适用性验证；进程退出、task 完成和 session 退役都不触发自动提升。

### 1.3 Session 内部如何关联

History 保存“发生过什么”，Context 保存“当前如何继续”。Context 引用自己的历史，也可读取该用户其他 Session 的资料及记忆条目。压缩 Context 不改写 History；向 History 追加记录，也不表示该记录已经被当前 Context 处理。

Memory 分别管理历史追加、工作状态版本和两者的对应关系。恢复时提供当前状态与尚未处理的记录；Harness 决定如何继续执行。Session 退役停止在线写入，历史保留与知识沉淀另行管理。

## 2. 领域服务与执行协作存储

Memory 的领域服务管理 Session、History、Context 和记忆条目。当前方案在同一服务内额外提供执行协作存储，保存 task/run 状态、输入队列、通知与临时写入所有权。两类能力可以共用数据库，但协作记录不自动成为用户知识。

Harness 决定任务语义与调度；模型和工具调用在 Harness 的执行环境中完成。以下函数是内部设计与实现候选，不改变这条能力边界。

claimSession 只登记 run 与写入权。saveCheckpoint 只保存工作快照并推进最新持久版本。Host session link 只保存原生会话定位关系。这三个动作都不调用模型，也不要求 Memory 知道进程已经加载 context。

执行状态由 ModelInputRecord、动作事件和结果记录分别证明。checkpoint 可用于恢复，但不能证明模型或工具已经执行成功。

## 3. 在线存储与任务可见性

### 3.1 Session、lease 与 checkpoint

| 函数 | Memory 中的操作 |
| --- | --- |
| createSession / getSession / discoverSessions | 保存稳定身份、stream/context 关联，返回精确状态或版本化摘要 |
| linkHostSession / resolveHostSession | 保存或读取原生定位索引，与 lease 独立 |
| claimSession | 锁定 session，校验开放状态、当前 head、任务资格和现有 writer，登记 run/lease |
| renewSessionClaim / releaseSession | 更新活动时间，或保存结束记录并释放占用 |
| appendEvents | 事件、stream 进度、控制索引与任务投影同事务提交 |
| saveCheckpoint | 校验 lease、来源与消费位置，保存不可变正文并条件推进 head |
| readCheckpoint | 读取已保存版本，检查当前授权与删除状态 |

claimSession 还检查 task 是否允许启动或恢复。已完成任务不能因旧 Dispatcher 重试而再启动；一个 task 的当前 run 与 session lease 一起约束重复执行。

releaseSession 不把未知工具结果清为成功，也不自动把业务 task 置为 completed。实际状态须由已提交事件支持。

### 3.2 TaskSnapshot、输入与通知

recordTask 保存目标 session、originTurnId 和 queued 状态，加入 session 的 pendingTask 索引；与退役检查使用同一事务约束。Memory 仅提供持久队列和状态，Dispatcher 在 Harness 中安排执行。

Worker 的 TaskUpdate 来自 appendEvents。服务验证当前 run 与 task 的关联，更新 TaskSnapshot 的 revision、进度、checkpoint 和结果引用。心跳通过租约更新，不能伪装成业务进展。

| 数据 | 写入方 | 使用方 |
| --- | --- | --- |
| TaskDescriptor | Router/Core | Dispatcher、Worker、维护任务 |
| TaskSnapshot | 事件事务更新 | Router 查询、恢复与维护 |
| TaskInput | Router/Core 或用户授权入口 | Worker 在安全边界读取并确认 |
| TaskNotification | 重要状态提交时创建 | Router inbox，独立于原 delegate 工具调用 |

postTaskInput 持久保存纠正或取消，不能直接写 Worker context。Worker 实际处理时追加 task_input_applied；输入确认与效果记录一起保存，防止崩溃后丢失输入。

通知 ID 由 task 与状态修订确定，重试不生成新通知。Router 追加对应 task_notification 事件时，同事务保存 deliveredEventId。通知只证明已进入 Router 持久 inbox，不证明用户已经读到回复。

Router 离线时通知保留。轮询是基础能力，订阅可降低延迟；普通进度允许合并，waiting_input、completed、failed、canceled 等重要事件不可被心跳覆盖。

### 3.3 记忆查询

getUserOverview、getProfile、recall 和 readEvidence 保留统一用户域与来源规则。getTask 负责工作状态，recall 负责相关知识，两者不互相替代。

知识召回与 Memory Agent 整理默认使用业务经历及其证据，不把请求准备、心跳或写入权变化当作知识材料。控制记录仍可按需供执行恢复和诊断读取；共享事件封装不意味着共享检索规则。

关键词检索为第一版基线，可选标签/集合为空时仍可使用。索引落后于 minUserRevision 时补读权威数据或返回 NOT_READY；session 摘要保留其来源 checkpoint，不代替执行前的版本检查。

## 4. Session Maintenance：谁负责退役

### 4.1 独立后台职责

Memory 服务内部增加 **Session Maintenance** job 类型，与 Memory Agent 共用后台任务进程和数据库队列。前者执行生命周期规则，后者负责知识提取、修订和 notes 整理。Router 不执行维护循环。

触发条件：

- releaseSession 或 task 终态提交后，产生一次维护检查。
- one_shot 工作完成后优先检查。
- 用户/Core 的 requestSessionReview 产生显式请求。
- 定时扫描超过策略闲置阈值或被明确替代的 session。

可复用 session 不因一次完成事件立即退役。定时阈值只是候选筛选，最终提交仍要检查当前工作状态。

### 4.2 退役 job 的步骤

```text
读取一致性状态
  → 检查策略及无活动 lease、排队输入、未完成 task、未决动作
  → 固定 session revision / checkpoint head / acceptedThrough
  → 准备 SessionExitRecord，必要时读取 checkpoint 之后的控制事件
  → 再次比较版本并检查前提
  → 同事务保存退出记录、标记 retired、发布变化
  → 可选排入 Memory Agent 整理任务
```

ExitRecord 至少保存原因、最终进度、checkpoint/事件范围及证据。它可以由 TaskSnapshot 与结果引用形成结构化摘要，不要求调用模型生成长期 note。

准备期间若有新任务、输入、writer 或新 checkpoint，条件提交失败，job 重新检查或延期。pendingTask 与任务输入索引必须同步更新，不能依赖落后的检索索引判断空闲。

lease 到期但 task 仍 running，或者工具结果未知时，维护 job 延期并保留原因，等待 Harness 核对。后台维护不能把“失联”当成“已完成”。已退役 session 不再接受新工作，但历史仍可读。

### 4.3 退役和知识提取分开

退出记录是审计与发现资料，长期 note 是复用知识。维护 job 可以在退役事务中排入一个整理 job，但不在数据库事务内调用模型，也不等待知识提取完成才退役。

可复用信息经过 Memory Agent 验证后成为条目；没有知识产出也是正常结果。失败的整理 job 可重试，session 已有退出记录和原始来源，不因此丢失上下文。

## 5. Notes 的修订、compact、归档与删除

note 是 MemoryKind 的一种，沿用记忆条目的来源、时效和版本机制。

| 操作 | 执行规则 | 默认检索行为 |
| --- | --- | --- |
| 更新/纠正 | 创建新 revision，保留旧版本和纠正证据 | 使用当前有效修订 |
| compact/合并 | 验证重要约束、精确值和来源后发布新修订或替代条目 | 被 superseded 的旧条目不再默认返回 |
| 归档 | 按过期、低价值或明确请求标记 archived | 默认排除，可显式历史读取 |
| 来源失效 | 标记依赖 revision 为 stale | 不作为确定知识使用，等待校验 |
| 删除 | 按授权请求或保留策略先写 tombstone，再清理规定范围 | 历史读取也拒绝，索引与缓存失效 |

compact 不是物理删除。多个相似 notes 只有在适用条件一致时才能合并；某次旅行约束不能被合并成用户永久偏好。旧证据保留可取回引用，删除策略另行决定保留期限。

Session 退役既不强制生成 note，也不自动删除已有 notes。仅删除 note 不等于删除原始 session；若请求的语义是遗忘来源内容，则必须覆盖相应来源与依赖，并抑制后台重新提取。删除范围由明确请求或配置策略决定。

Memory Agent 提出变更，Validator 检查来源、状态和并发修订，再原子更新正文、heads、替代关系和 changes。物理清理由维护任务执行，不由模型随意决定。

## 6. 关键内部结构

SessionExitRecord、RetireRequest、TaskSnapshot、ModelInputRecord 等候选载荷见[接口实现参考](integration-reference.md)。它们服务于维护和执行协作，不是新增的记忆领域对象。

### 6.1 记忆与运行记录

```typescript
interface UserMemoryState {
  domain: UserDomain;
  revision: number;
  indexedThroughRevision: number;
}
interface MemoryItemRevision {
  memoryId: ID;
  revision: number;
  previousRevision?: number;
  domain: UserDomain;
  kind: MemoryKind;
  horizon: MemoryHorizon;
  key: string;
  content: Content;
  labels: string[];
  collectionIds: ID[];
  sourceRefs: EvidenceRef[];
  applicability: string[];
  epistemicStatus: "supported" | "inferred" | "unverified" | "disputed";
  conflictWith: EvidenceRef[];
  validFrom?: Timestamp;
  validUntil?: Timestamp;
  createdAt: Timestamp;
}
interface MemoryHead {
  memoryId: ID;
  currentRevision: number;
  status: MemoryStatus;
  supersededBy: ID[];
}
interface MemoryReadState {
  memoryId: ID;
  revision: number;
  stale: boolean;
  invalidatedAt?: Timestamp;
}
interface MemoryCollection { collectionId: ID; domain: UserDomain; label: string; description?: string; }
interface SessionSummary { sessionId: ID; sourceContext?: ContextRef; text: string; sourceRefs: EvidenceRef[]; }

interface RunRecord {
  runId: ID;
  sessionId: ID;
  taskId?: ID;
  executorId: ID;
  readGrant: ReadGrant;
  epoch: number;
  claimState: "claimed" | "released" | "expired";
  lastSeenAt: Timestamp;
  outcome?: "completed" | "paused" | "failed";
}
```

RunRecord.claimState 表示占用状态，不推断模型是否调用或任务是否成功。原始请求和结果事件承担执行事实证明。用户画像仍是条目派生视图，不维护一份无来源的“用户真相”。

### 6.2 维护 job 与记忆变更

```typescript
interface SessionMaintenanceJob {
  jobId: ID;
  domain: UserDomain;
  sessionId: ID;
  trigger: "work_finished" | "one_shot_finished" | "idle_scan" | "explicit";
  status: "queued" | "checking" | "committing" | "completed" | "deferred" | "failed";
  policyVersion: string;
  reason?: string;
  nextCheckAt?: Timestamp;
  workerOwnerId?: ID;
  leaseEpoch: number;
}

interface DreamJob {
  id: ID;
  domain: UserDomain;
  mode: "consolidate" | "repair" | "prepare_note";
  inputRefs: EvidenceRef[];
  retiredSessionId?: ID;
  readUserRevision: number;
  status: "queued" | "running" | "validating" | "completed" | "failed" | "canceled";
  maxModelCalls: number;
  maxCostUsd: number;
  workerOwnerId?: ID;
  leaseEpoch: number;
  changeSetId?: ID;
}
interface MemoryChangeSet {
  id: ID;
  dreamJobId: ID;
  domain: UserDomain;
  readUserRevision: number;
  expectedRevisions: { memoryId: ID; revision: number | null }[];
  candidates: MemoryItemRevision[];
  headChanges: {
    memoryId: ID;
    status: "active" | "superseded" | "archived";
    supersededBy: ID[];
  }[];
  status: "proposed" | "validated" | "committed" | "rejected";
}
```

删除不作为模型任意 headChanges：授权删除请求由独立维护路径校验。合并需要同时校验被替代条目和新条目的 revision，并维护来源反向索引。

Session Maintenance 按 session 串行提交，模型整理首版按 user 串行；在线 session 仍可并行执行。后台 worker 的 lease 与在线 SessionLease 分开，不能拿维护 job 的身份修改活动 context。

## 7. 存储与事务

| 集合 | 保证 |
| --- | --- |
| sessions / session_leases / runs | 稳定逻辑身份，单 writer，epoch 单调增加 |
| host_session_links | 原生会话定位，不参与模型执行 |
| records / stream_heads / control_indexes | 连续事件、恢复位置与未决动作一致提交 |
| checkpoints / checkpoint_heads | 不可变快照与最新持久指针原子提交 |
| tasks / task_inputs / task_notifications | 状态投影、待处理输入与持久通知 |
| session_maintenance_jobs / session_exit_records | 退役责任、检查原因、原子退出记录 |
| memory_revisions / heads / read_states | 修订、归档、替代、失效与删除状态 |
| jobs / change_sets / changes | 有预算后台整理、幂等发布及变更通知 |
| blobs / source_links / retrieval_indexes | 大内容、来源关系与可重建索引 |

user revision 随可检索资料与目录变化推进；context head 和 task snapshot 各有 revision。纯心跳和 job 内部状态不产生知识更新。退役使用权威控制索引而不是搜索索引。

retireSession 事务同时保存 exit record、标记 retired、发布变化，并按策略排入知识整理；模型推理在事务之外。删除先阻止所有版本读取，再清理正文、依赖、索引与缓存。

## 8. 部署与第一版验收

单机使用 API、后台 Worker、PostgreSQL 与持久 blob 目录；集群增加无状态 API 与后台副本，使用共享数据库/对象存储。Session Maintenance 和 Memory Agent 是同一服务内的不同 job 类型，不要求独立平台。

第一版必须验证：Worker 进度可查询；通知和任务输入不丢失；Router 离线不丢结果；checkpoint 保存不被误报为执行；无活动 Host 时仍可检查已结束 session 的退役；未决工作阻止退役；exit record 必需而长期 note 可选；compact、归档和删除各自符合版本与来源规则。

CLM、模型策略学习和资源虚拟化不作为这些验收的依赖。后台 notes 的质量、收益和模型成本单独评估。
