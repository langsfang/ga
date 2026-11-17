# Agent Harness 插件设计

状态：设计草案 v0.4；2026-11-17。公共数据与网络接口见[对接协议](integration-protocol.md)。

## 1. 定位与边界

Host 是加载插件的完整 agent 应用，例如 dsh、pi；它们本身也属于广义的 agent harness。本文的 **Genvagent Harness** 专指加载其中的插件层：**Harness Core + Host Adapter + Memory Client**，Core 内的 Router 和 Context Policy 可替换。dsh、pi 分别通过原生插件或 extension 加载对应 Adapter，共用 Core 和协议。

Host 保留模型执行循环、原生 session、工具执行、授权、取消和界面。Genvagent 不再为这些能力建立第二套通用任务状态机；Core 负责记忆同步、上下文管理、Router 策略和对接协议。

原始记录、长期记忆和工作上下文版本存储在 Memory Service，Core 维护当前工作视图与同步状态。原生会话记录和本地同步 spool 用于可靠投递及故障核对，其存在不把长期记忆绑定到某个进程。

## 2. 模块与关键对象

| 模块 | 职责 | 关键结构 |
| --- | --- | --- |
| Host Adapter | 生命周期、事件、上下文和原生工具接入 | AdapterCapabilities、HostFrame、HostApplicationReceipt |
| Harness Core | 组织同步与请求边界，管理一次上下文变更 | SessionBinding、SyncState、ContextPlan |
| Router Strategy | 根据目标决定上下文需求和会话分派 | RoutePlan |
| Context Controller | 构建工作视图，接受模型编辑并校验 | WorkingContext、WorkingBlock、ContextPatch |
| Memory Client | 协议调用、认证、幂等、重试与变更拉取 | 协议请求/响应 |

Router Strategy 和 Context Policy 可替换。适配器只翻译 Host 能力，不复制事实提取、检索排序和 dreaming 逻辑。

## 3. Host 适配契约

类型 ID、JsonValue、Content、RecordInput、ContextRef、ContextHead、SessionBinding、WriterLease、HostApplicationReceipt、RecallRequest 和 Timestamp 均来自对接协议。

```typescript
interface AdapterCapabilities {
  durableEventExport: boolean;
  durableMarkers: boolean;
  safeRequestGate: boolean;
  contextRewrite: "none" | "regions" | "full_working_view";
  requestCapture: "working_view" | "model_messages" | "provider_payload";
  branchLifecycle: boolean;
  nativeResume: boolean;
  spawnChildren: boolean;
}

type HostEvent = Omit<RecordInput, "eventId" | "sequence">;

interface HostFrame {
  bindingId: ID;
  nativeRevision: string;         // 当前可见分支、消息范围及相关配置的版本摘要
  nativePosition: JsonValue;
  protectedContent: Content;
  workingContext: WorkingContext;
  inputTokenBudget: number;
  pendingToolCallIds: string[];
}

interface HarnessCorePort {
  open(binding: SessionBinding, capabilities: AdapterCapabilities): Promise<void>;
  ingest(events: HostEvent[]): Promise<void>;
  prepareRequest(frame: HostFrame): Promise<ContextPlan>;
  confirmApplied(planId: ID, receipt: HostApplicationReceipt): Promise<ContextHead>;
  close(): Promise<void>;
}

interface HostAdapter {
  name: string;
  capabilities: AdapterCapabilities;
  attach(core: HarnessCorePort): Promise<() => Promise<void>>;
}
```

以上是 Genvagent 内部拟定接口，不是 dsh 或 pi 的现有 SDK 名称。adapter.attach 注册原生 hooks、工具与必要的命令，并返回卸载函数。

一次请求分两次回调完成：prepareRequest 只返回候选；adapter 在原生 hook 中应用后，于允许模型请求前调用 confirmApplied。禁止在早期 hook 中等待一个必须等该 hook 返回后才发生的回调，避免生命周期死锁。

Core 对版本冲突返回明确错误，由 adapter 阻止放行并重新读取 HostFrame。nativeRevision 在候选准备后改变，尤其是用户插入消息、取消或切换分支时，原候选作废。

### 3.1 能力等级

| 等级 | 要求与行为 |
| --- | --- |
| Recall | 同步记录、读取相关记忆并注入 Host；不宣称具备完整 CLM |
| Context rewrite | Host 允许改变下一轮工作消息；模型编辑经过保护和工具配对校验 |
| Strict recoverable | 在上述基础上有持久 marker、安全请求 gate 和可重放事件，遵守准备/激活协议 |

能力不足时配置检查失败或显式选择较低模式。不能以“记录到本地文件”代替实际改写下一轮输入，也不能以自定义 UI 展示证明已影响模型。

## 4. Core 同步状态与上下文计划

```typescript
interface SyncState {
  binding: SessionBinding;
  nativeSessionLocator: string;   // Adapter 可解析的 Host 会话存档位置
  writer?: WriterLease;           // 启动后重新获取，不以本地旧值证明有效
  lastExportedNativePosition: JsonValue;
  acceptedThrough: number;
  nextSequence: number;
  activeContext?: ContextRef;
  headRevision: number;
  pendingPlanId?: ID;
  lastApplicationMarkerId?: string;
  changesCursor?: string;
}

interface ContextPlan {
  planId: ID;
  bindingId: ID;
  baseNativeRevision: string;
  version: ContextRef;
  expectedHeadRevision: number;
  desired: WorkingContext;
  bodyHash: string;
}
```

SyncState 是可持久化的同步检查点；本地目录由 HarnessConfig.stateDir 指定。Adapter 也可以在 Host 的持久扩展记录中保存关联标记，但必须能在插件重新加载后定位。

| 本地对象 | 保存内容 | 恢复用途 |
| --- | --- | --- |
| 安装身份 | 稳定的 hostInstanceId | 区分 Host 安装实例；进程重启不重新分配 |
| 会话映射 | scope、ownerUserId、Host session/branch → bindingId | 确定应读取哪条记录流和上下文 |
| 同步检查点 | SyncState、Host 会话存档定位信息 | 对比本地进度和 Memory 已确认进度 |
| 待发送日志 | 原始事件 ID 映射、序号、批次正文或持久来源、Idempotency-Key | 原样补传未确认批次 |
| 待确认上下文计划 | ContextPlan、应用 marker、激活请求正文与 Idempotency-Key | 核对已应用但未确认的版本 |

会话映射的 ownerUserId 来自认证身份。用户切换账号或项目时重新选择映射，不复用其他用户的 binding。安装标识 hostInstanceId 与 WriterLease.ownerId 不同：后者每次插件实例启动重新分配，只用于排除并发写入。

本地映射缺失时，Core 以原 Host 身份调用 bindings:resolve 精确查询服务端。随后读取 binding 状态、取得新 lease，核对应用标记并补传原事件。读取当前上下文后，再取 coveredEventSequence 之后的事件作为增量。[协议恢复步骤](integration-protocol.md#53-确定性恢复步骤)

尚无激活版本的新会话从空工作视图和全部已提交记录开始。nativeSessionLocator 只定位 Host 的会话存档，不代表 Memory 保存了对应工作区文件或运行中的工具。恢复工具执行由 Host 负责。

未收到 receipt 的事件按原标识补传；上下文写入受到 Memory writer lease 与版本检查约束。断电后仍需恢复的映射、日志和计划必须先持久化，再发送对应请求。

事件同步以最终持久化的原生记录为准。早期可变 hook 的参数只用于运行时观察，不能抢先当成最终消息上传；工具生命周期通知与最终 tool-result 消息按同一原生结果身份归并，防止重复记忆。Host 需要显式 flush 才能保证 marker 持久时，adapter 必须等待 flush。

同一 binding 的请求边界串行处理。并发子 agent 使用独立 binding；Core 不接管 Host 本身的调度锁。Memory 不可用时执行 strict 或 buffered 策略，不能无提示地切换保证级别。

## 5. Router 与上下文策略

```typescript
interface RoutePlan {
  action: "continue" | "select_session" | "delegate";
  targetBindingId?: ID;
  goal: string;
  informationNeeds: string[];
  recallRequests: RecallRequest[];
  contextBudgetTokens: number;
  reason: string;
}

interface RouterStrategy {
  decide(frame: HostFrame, userGoal: string): Promise<RoutePlan>;
}

interface WorkingContext {
  format: "genvagent.context.v1";
  goal: string;
  blocks: WorkingBlock[];
}

interface WorkingBlock {
  id: ID;
  category: "goal" | "constraint" | "memory" | "evidence"
    | "progress" | "message" | "tool_exchange";
  content: Content;
  sourceRefs: EvidenceRef[];
  editable: boolean;
}

interface ContextPatch {
  baseVersion: ContextRef;
  replacementEditableBlocks: WorkingBlock[];
  explanation: string;
}
```

EvidenceRef 同样来自对接协议。Router 首先负责“当前目标需要哪些信息”；delegate 只有在 adapter 支持 spawnChildren 且当前授权允许时启用。第一版可用规则实现 Router，再替换为模型策略。

Context Controller 按实际模型 token 预算选择材料并保留输出余量。system prompt、工具定义、最新用户目标及必须遵守的约束从 Host 读取并保护；不允许记忆内容覆盖权限。

CLM 能力向模型开放 `context.inspect`、`context.edit` 等 Host 工具：模型可以自由重写可编辑区域、组织状态表、移出冗余内容及重新读取证据，修改经准备/应用/激活后进入下一轮输入。文件形式只是后续可选入口，语义与 ContextPatch 相同。

编辑校验包括：baseVersion 与 Host 位置未过期、保护块完整、来源可读、工具调用/结果成对、token 不超限，以及被移出内容已存于原始记录。不支持的 vendor 私有块作为不可编辑单元处理。

Memory Agent 的长期整理不直接触发活动上下文重写。Core 拉取失效信息，在下一请求边界重新读取材料；关键用户纠正立即进入受保护的本轮目标与约束。

## 6. dsh 适配

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

## 7. pi 适配

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

pi 会话通过 id/parentId 表达树状历史。只导出当前分支对应的可见路径，并保存原生位置；导航到旧节点后另起执行分支时创建新的 binding。持久 marker 是插件元数据，不应被当成重复的用户消息注入模型。[Session 格式](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/session-format.md)

## 8. 插件配置与扩展点

```typescript
interface HarnessConfig {
  memoryEndpoint: string;
  credentialRef: string;
  stateDir: string;               // 插件本地持久目录，保存身份与同步恢复数据
  scope: Scope;
  syncMode: "strict" | "buffered";
  contextMode: "recall" | "rewrite";
  routerStrategy: string;
  contextPolicy: string;
  maxContextTokens: number;
}
```

Scope 来自对接协议。配置先做能力协商，例如 strict + rewrite 不能在没有安全请求 gate 的 adapter 上静默启动。

扩展点限定为 Host Adapter、Router Strategy、Context Policy 和 Memory Client transport。第一版不额外建设通用插件依赖管理平台；Host 负责插件发现、安装和卸载。

卸载时取消本插件的辅助调用，停止注册的 hooks，保存未同步位置并释放 lease。失败退出依赖原生日志和 spool 恢复，不依赖 shutdown 必然执行。Memory notes 不会被解析为要加载的插件代码。

## 9. 最小交付与验收

1. Core 与模拟 Host/Memory：事件同步、候选保存、激活和故障窗口可验证。
2. pi adapter：先 Recall，再在固定版本上验证 rewrite、marker 与 gate。
3. dsh adapter：先公开事件与注入，再验证 surface/compaction 接入。
4. 同一用户在同一项目接入两个 Host，可读取同一条私有记忆，各自工作上下文与分支独立；另一用户只能读取已授权共享的记忆。

每个 adapter 必须检查：原始记录不丢、实际输入按声明变化、用户纠正不被压缩掉、工具交换有效、分支映射正确、重启不重复同步、其他插件不会无声覆盖本插件效果。

Core 不宣称替 Host 实现任意工具的 exactly-once 执行，也不在 Memory 恢复时重新发起 Host 工具。恢复时读取执行结果和上下文版本；未知外部结果仍由 Host 核对。
