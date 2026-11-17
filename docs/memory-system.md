# Memory System：用户记忆、持久执行状态与整理服务

状态：设计草案 v0.6；2026-11-17。对外函数与共享类型见[交互协议](integration-protocol.md)。

## 1. 记忆对象与视图

Memory 以 userId 为唯一归属域，面向普通问答、学习、生活事务和工作等场景。项目、主题集合和固定画像字段都不是使用前提。

### 1.1 三类持久对象

| 对象 | 保存内容 | 主要写入方 |
| --- | --- | --- |
| 原始证据 | 用户消息、模型输出、工具结果、外部材料与执行事件 | Harness Core |
| 记忆条目 | 带来源的事实、偏好、目标、方法、经历和 notes | 提案者 + Memory Validator |
| 持久执行状态 | session、context 版本、task/run 记录、租约与生命周期 | Core 发起，Memory 条件提交 |

证据证明某次陈述或观察发生，不自动证明其内容正确。记忆条目保留适用条件、时间与修订；session/context 则保存尚在使用的工作状态。Worker process 不是存储分区。

### 1.2 画像、短期、长期与 notes

| 视图或维度 | 数据来源 | 解释 |
| --- | --- | --- |
| 用户画像 | 与用户有关的事实、偏好、目标及其证据 | 派生视图；区分已支持、推断和冲突，允许为空 |
| 短期记忆 | 近期事件、当前目标、未决事项、活跃/空闲 session，以及短期条目 | 当前工作需要的信息，仍可持久化 |
| 长期记忆 | 经验证适合跨会话复用的记忆条目 | 可被纠正或失效，长期不等于永久正确 |
| Note | MemoryKind 为 note 的整理文本 | 表达形式，可以服务短期工作或长期复用 |
| Collection / 标签 | 对记忆和相关来源的可选组织关系 | 可以不存在；旅行、学习、项目等均可作为名称 |

这些维度可重叠。例如“本次旅行预算”可以是短期 goal/constraint 的来源，而“用户明确要求优先采用公共交通”可以作为有适用条件的 preference；二者均可能出现在某份 note 中。它们不会因被写成 note 就成为永久用户画像。

工作 context 只是某个 session 的选定工作视图，不包含该用户全部短期或长期记忆。记忆从短期转为长期需经过适用性验证；进程退出、task 完成和 session 退役都不触发自动提升。

## 2. Session 与进程分离

SessionDescriptor、stream 和 context 属于稳定的逻辑工作线程。RuntimeBinding 只帮助将 dsh/pi 原生身份映射到该线程；同一逻辑 session 可以被不同进程、不同挂载先后加载。

Memory 不分配 Worker 进程，也不执行工具。服务通过 startRun/lease 保证每个 session 一个 writer；Core 在外部运行时加载状态。所有 Host 离线时，Memory 仍可整理记忆和维护目录。

Session 的生命周期为 open/retired，忙闲由有效 lease 和未决动作决定。正常 task 结束只 endRun；退役需额外条件提交。退役保留历史用于查询和分叉，与删除策略分开。

## 3. 写入、发现与整理流程

### 3.1 执行状态写入

appendEvents 原子保存事件、stream 进度、相关控制索引、user revision 与幂等结果。context 候选由 Core 单独提交，Memory 不因事件追加而自动改写模型输入。

新进程 startRun 后，原样加载当前 context 使用 confirmContextLoaded 记录本 run 的应用证明。context 改变时走 putContext/activateContext。旧进程回执不能证明新进程已经加载。

所有执行写入检查 session 开放、run 有效和 epoch 匹配。endRun 把任务结果与未决动作保留下来，释放占用；过期 lease 不代表外部动作成功。

### 3.2 Router 发现与检索

getUserOverview 枚举用户记忆、可选集合、session 和任务。getProfile 返回带来源的用户信息视图。discoverSessions 返回候选摘要及其 ContextRef，Router 精确核对当前 head 和执行状态后再继续。

recall 从原始事件和记忆条目中检索。关键词基线先按 user、run 授权、可选标签/集合和时间条件筛选，再读权威内容校验。后续可加入向量与重排；集合目录为空不会导致检索不可用。

会话摘要可能落后于当前 head，返回时同时保留摘要来源版本。摘要用于发现，执行权必须由 startRun 的条件检查取得。readEvidence 精确读取来源，恢复不依赖搜索相似文本。

### 3.3 后台整理

Harness 或 Memory Agent 通过 MemoryProposal 提交候选，验证后发布 MemoryItemRevision。kind、horizon、标签和集合相互独立；同一条目不因放入两个集合就保存两份事实。

Memory Agent 可提炼经历、维护画像引用、修订方法和生成预备 notes，但不能擅自推进活动 ContextHead。session 退役后的资料仍可成为整理来源，当前任务约束不能因为退役就变成长期偏好。

## 4. 模块与接口函数

| 模块 | 责任 |
| --- | --- |
| API / Session Registry | 用户授权、逻辑 session、原生映射、run/lease、生命周期 |
| Record / Context Store | 事件追加、控制索引、context 保存、激活与加载证明 |
| Memory Store / Views / Retrieval | 条目、画像、短期/长期视图、集合、session 发现与来源读取 |
| Memory Agent / Validator | 提取、校正、压缩、预备 notes 与发布验证 |

### 4.1 Session 与执行状态

| 函数 | 语义与事务边界 |
| --- | --- |
| createSession / forkSession | 创建稳定 session、stream/context；fork 来源为固定版本，不能继承未决动作的执行权 |
| bindRuntime / resolveRuntime | user + RuntimeKey 唯一映射；不会改变 session 的逻辑归属 |
| startRun / renewRun | 原子校验 lifecycle、head、授权与 lease；epoch 按 session 单调增加 |
| appendBatch | 记录与控制索引一起提交，避免退休检查使用落后状态 |
| saveContext / activateContext | 候选不可变；激活检查 current run、head、hash 和来源 |
| confirmContextLoaded | 为当前 run 保存当前版本的加载回执，不改变 head |
| endRun | 保存终态、结果引用与未决动作后释放占用 |
| retireSession | session revision、head 与接收位置一致，无有效 run、排队输入、未完成任务和未决动作时冻结 |

未决动作、排队输入和未完成任务来自已提交的控制记录与核对结果，不能仅依据 endRun 的空列表清除。等待用户的 task 不因释放进程就消失，必须完成、明确取消或转交后才能退役。startRun 与 retireSession 竞争同一条件事务；只允许一种结果成功。

### 4.2 查询、视图与记忆发布

| 函数 | 结果 |
| --- | --- |
| getOverview / getProfile | 可分页目录或画像视图，保留来源与读取 revision |
| getShortTermView / getLongTermView | 按用途组合近期状态或可复用条目；可由 overview/recall 参数表达，不强制独立网络接口 |
| discoverSessions / readSession | 候选摘要或精确状态 |
| recall / readEvidence | 相关材料或精确原文 |
| acceptProposal / publishMemories | 持久提案，或经验证后原子发布修订 |
| recordFeedback / invalidateSource | 保存纠正、标记来源与依赖失效 |
| listChanges | 用户资料、session 生命周期及来源变化，至少一次交付 |

所有视图检查当前授权。Worker 的 run grant 不因 profile、集合或来源展开而扩大。派生视图不成为新的权威事实副本。

### 4.3 Memory Agent

scheduleDream 按 user 和可选输入范围创建 job；claimJob 取得后台 worker lease；buildChangeSet 生成候选；validateChangeSet 验证来源、适用性和并发修订；commitChangeSet 原子发布并完成 job。

未发布候选不进入普通 recall。预算耗尽停止后台任务，不阻塞在线消息存储或读取。后台 worker 与 Harness Worker process 是不同角色。

## 5. 服务内部结构

共享 ID、UserDomain、MemoryKind、MemoryHorizon、Content 和 EvidenceRef 由协议定义。

### 5.1 条目、视图来源与集合

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
}

interface MemoryReadState {
  memoryId: ID;
  revision: number;
  stale: boolean;
  invalidatedAt?: Timestamp;
}

interface MemoryCollection {
  collectionId: ID;
  domain: UserDomain;
  label: string;
  description?: string;
}

interface SessionSummary {
  sessionId: ID;
  sourceContext: ContextRef;
  text: string;
  unresolvedQuestions: string[];
  sourceRefs: EvidenceRef[];
}
```

条目正文与修订不可变，当前失效状态单独维护。horizon 改变生成新修订并说明依据；用户归属不可改变。key 只辅助主题匹配，不构成自动合并依据。

用户画像基于有来源的条目组成 ProfileView，不设置强制的人口统计字段。近期事件和 session 进度可以直接服务短期视图，无须先转成 note。

### 5.2 Run 记录

```typescript
interface RunRecord {
  runId: ID;
  domain: UserDomain;
  sessionId: ID;
  taskId?: ID;
  executorId: ID;
  readGrant: ReadGrant;
  leaseEpoch: number;
  status: "active" | "completed" | "paused" | "failed" | "expired";
  startedAt: Timestamp;
  endedAt?: Timestamp;
  finalContext?: ContextRef;
  resultRefs: EvidenceRef[];
}
```

RunRecord 保存执行历史，进程本身不存入业务状态。expired 只表示写入权结束；未知外部动作仍保留在 session 控制索引中，直到核对。

### 5.3 Dreaming 与候选发布

```typescript
interface DreamJob {
  id: ID;
  domain: UserDomain;
  mode: "consolidate" | "repair" | "prepare_note";
  inputRefs: EvidenceRef[];
  collectionIds?: ID[];
  readUserRevision: number;
  status: "queued" | "running" | "validating" | "completed" | "failed" | "canceled";
  maxModelCalls: number;
  maxCostUsd: number;
  workerOwnerId?: ID;
  leaseEpoch: number;
  leaseExpiresAt?: Timestamp;
  changeSetId?: ID;
}

interface MemoryChangeSet {
  id: ID;
  dreamJobId: ID;
  domain: UserDomain;
  readUserRevision: number;
  expectedRevisions: { memoryId: ID; revision: number | null }[];
  candidates: MemoryItemRevision[];
  checks: {
    candidateMemoryId: ID;
    result: "supported" | "uncertain" | "rejected";
    evidence: EvidenceRef[];
    explanation: string;
  }[];
  status: "proposed" | "validated" | "committed" | "rejected";
}
```

首版每个 user 同时运行一个整理 job，用户之间并行；在线 session 可以并行写入。发布检查 expectedRevisions 和新增证据的影响，推理期间不持有事务锁，重验受预算约束。

发布、memory heads、user revision、changes 和 job 完成在同一事务提交。worker 丢失 lease 后不得发布，重试沿用 changeSet ID。内部 job 状态与未发布候选不推进资料版本。

## 6. 存储、一致性与退役

| 数据集合 | 关键关联与约束 |
| --- | --- |
| user_state / changes | 用户域资料版本与变化 |
| sessions / stream_heads | 稳定 sessionId，stream 连续序号，open/retired 生命周期 |
| runtime_bindings | 原生身份定位逻辑 session，一对多运行映射不改变逻辑身份 |
| runs / session_leases | 执行记录，每个 session 一个有效 writer |
| records / control_indexes | 原始事件与待处理 task/action 的权威提交索引 |
| context_versions / heads / application_receipts | 不可变快照，当前指针，以及各 run 的加载/应用证明 |
| memory_revisions / heads / read_states | 记忆修订、当前版本与失效 |
| collections / profile_index / session_summaries | 可选组织和派生视图 |
| proposals / jobs / change_sets | 候选、预算、验证与幂等发布 |
| blobs / source_links / retrieval_indexes | 大正文、来源反向关系与可重建索引 |

retireSession 只冻结该 session 的继续执行能力，保留流、快照和引用。one_shot 或 idle 策略必须经过同一服务检查，不能直接删除数据表；有未决动作或仍在执行时拒绝退役。

物理删除先阻止来源及依赖读取，再清理条目、索引、快照和 blob，发布失效变化。已退役内容的读取同样服从删除与授权规则。

索引落后于 minUserRevision 时补读权威存储或返回 NOT_READY。profile、短期/长期视图与 session 摘要均可重建；摘要来源版本与当前 head 不相同必须可见，不能充当执行前版本校验。

## 7. 独立部署与验收

单机使用 API、Memory Worker、PostgreSQL 与持久 blob 目录；集群增加无状态 API/后台副本并改用共享对象存储。所有 session、epoch、head、run 与幂等记录在共享数据库中，客户端只改变 endpoint。

后台整理生成经历摘要、修订方法及预备 notes；原始来源保留可取回引用。未来 query 的预备计算计入成本，适用性和时效必须验证。[Dreams](https://platform.claude.com/docs/en/managed-agents/dreams)、[Sleep-time Compute](https://arxiv.org/abs/2504.13171)

验收包括：无集合用户正常使用；画像可追溯且不把一次性约束泛化；不同进程继续同一 session；fork 只继承固定来源；retire 与 startRun 竞争正确；未知工具结果阻止错误退役；全部 Host 离线后仍可整理记忆；单机/集群通过相同持久化契约。
