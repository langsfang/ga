# Harness 与 Memory 最小接口协议

版本：v0.16 · 状态：设计契约 · 日期：2026-11-18

## 1. 边界与原则

Harness 决定任务目标、调度顺序和执行者，负责调用模型与工具。Memory 保存 Context、任务及消息，保证版本唯一领取、持久提交和消息可恢复。Memory 不启动 Worker，也不承担业务调度。

本文是两系统接口的唯一规范。[Harness 架构](agent-harness.md)描述调用流程，[Memory 架构](memory-system.md)描述存储与维护。首版在线接口限定为下述 13 个操作；Trace 导入另有 2 个操作。画像、摘要刷新、压缩建议、生命周期检查等属于读取视图或后台维护，不要求 Harness 调用额外接口。

接口均在已认证用户域内操作。所有写入携带稳定请求身份，重试返回既有结果；版本及消息引用不能跨越未授权用户域。

## 2. 共享对象

| 对象 | 含义 |
| --- | --- |
| Session | 一项工作；执行 Session 具有单一元数据 Event 流，导入 Session 不创建 Event 流 |
| ContextRef | 明确的 contextId 与 versionId；内容不可变，可按来源派生版本 |
| Task | 可调度执行目标，包含角色、Session、父任务、续接版本、状态及返回地址 |
| Claim | Memory 授予具体 Context 版本的执行权，绑定 Task、执行者、有效期及执行代次 |
| Mailbox | 以稳定 Task ID 寻址的持久收件箱；消息正文引用 Context 待处理内容或结果版本 |
| Message | 稳定消息身份、目标、类型、来源及内容引用；类型包括输入、纠正、取消、恢复和任务结果 |

Task 的主要状态为 queued、running、waiting、paused、completed、failed、cancelled。Router 对话也对应一个长期 Task；没有输入时等待，新输入到达时再次可调度。

Event 用于审计和摘要，Mailbox 用于定向交付；二者不是同一条流。Event 不保存对话正文，Mailbox 不另存一套 Worker History。

<a id="5-对外接口"></a>
<a id="5-两系统的交互接口"></a>

## 3. 最小在线接口

### 3.1 工作资料：4 个操作

| 操作 | 输入 | 结果及语义 |
| --- | --- | --- |
| openSession | 工作目标、初始 Context 内容、请求身份 | 原子创建 Session 及初始 Context，返回固定引用 |
| read | 对象或内容引用 | 读取 Session 目录与摘要、Task 状态、明确 Context 版本或引用原文；只读，不授予执行权 |
| deriveContext | 基础 ContextRef、派生类型及明确内容变化 | 创建 fork、compact 或恢复候选版本；默认保持 Context 身份，可建立独立 Context；不移动任何 Task 的续接位置 |
| recall | 查询、用户内范围和数量限制 | 返回相关历史、摘要或知识及来源；用户概览和画像是查询视图 |

新增独立 Context 通过明确来源派生；openSession 提供初始根版本。deriveContext 只发布版本，是否供某个任务继续使用由版本领取或释放时的条件更新决定。只读操作采用统一引用模型，不提供通用数据库写入接口。

### 3.2 版本执行：5 个操作

| 操作 | 输入 | 结果及语义 |
| --- | --- | --- |
| claimContext | Task、预期续接引用、执行者；可指定显式派生的执行起点 | 原子核对 Task 状态并领取具体版本，Task 转为 running，返回内容与 Claim；同时受理对应待调度项 |
| renewClaim | 当前 Claim | 续期；已过期、失效或被替换的凭证不能续期 |
| startInteraction | Claim、本轮输入及来源消息 ID、调用身份 | 保存待完成交互和输入，返回交互引用；不得仅保存引用而丢失正文 |
| commitInteraction | 交互引用、Claim、模型输出及调用结果 | 成功时幂等发布后继版本、完成事件、消息消费确认和 Task 续接位置，并转移领取；失败时保存未完成状态，不确认输入消费 |
| releaseClaim | 当前 Claim、queued/waiting/paused、可选后续版本及等待条件 | 原子保存续接和等待条件、释放领取；采用派生版本时核对来源与任务修订；未决交互须保留 |

领取对象始终是 `(contextId, versionId)`，不是 Session。不同版本可同时被不同执行者领取。同一 Task 的续接状态也须校验，不能通过旧派发请求重复启动已推进的任务。

版本派生与采用分开：fork 可分配给新任务，compact 或恢复候选只在持有有效执行权或领取时的条件更新中成为指定 Task 的续接版本。旧版本及其他分支保持不变。

### 3.3 任务与消息：4 个操作

| 操作 | 输入 | 结果及语义 |
| --- | --- | --- |
| submitTask | 稳定 Task ID、角色、Session、来源版本、目标输入、parentTaskId、returnTo | 原子派生独立起点、保存 Task 和初始输入并登记待调度项，返回 TaskAccepted；无初始输入的 Router Task 进入 waiting |
| sendMessage | 目标 Task、稳定消息 ID、类型及内容/引用 | 持久保存输入；必要时将等待任务标记为可调度，返回持久受理结果，不代表已处理 |
| receive | 调度队列或 Task 收件箱、分页位置、等待时长 | 返回待调度任务或未处理消息；允许长轮询，读取和分页不确认消费 |
| finishTask | Task、有效 Claim 或经核对的取消依据、终态、结果引用 | 原子保存终态、释放领取、追加完成元数据，并向 returnTo 收件箱写入唯一结果通知 |

submitTask 的来源版本必须属于目标工作范围。子任务默认在同一 Session 内派生独立版本，不因并行创建新 Session。顶层任务可以将另一个 Session 内的 Router Task 作为返回地址，但必须属于同一授权用户。

finishTask 的结果必须引用已持久可读的内容。相同终态请求重复提交不重复产生通知；正在运行的任务必须校验执行代次。取消请求已接收不能作为进程已停止的证明。

消息入箱不直接启动实例：waiting 任务满足唤醒条件后转为 queued；running 任务保留消息供下一边界处理；paused 任务仅由有效恢复指令解除暂停；终态任务不自动重启。结果到达、恢复与取消均须按 Task 当前修订更新，不能覆盖并发推进的状态。

## 4. 队列与交付契约

### 4.1 两类逻辑队列

| 队列 | 内容与消费者 | 持久依据 |
| --- | --- | --- |
| 调度队列 | 可执行 Task 引用，由 Harness 调度器消费 | Task 状态及可调度标记；成功 claimContext 后受理 |
| Task 收件箱 | 用户输入、补充、取消、子结果，由目标任务执行者消费 | 稳定 Message 及处理状态；随实际处理提交确认 |

首版由 Memory 的事务存储实现，不要求额外消息中间件。进程内通知和长轮询返回仅用于唤醒；服务重启或唤醒丢失后仍能读取持久队列。普通进度可供 read 查询，不必唤醒 Router 调用模型。

### 4.2 原子边界

1. submitTask：任务、独立 Context 起点、目标输入、待调度标记一致发布。
2. claimContext：Task 可执行性、具体版本领取和待调度项受理一致更新；竞争者只能有一个成功。
3. commitInteraction：版本、执行事件、Task 续接及本次输入消费一致提交。
4. finishTask：任务终态、结果引用、领取释放、返回消息及目标任务的可调度标记一致提交。
5. releaseClaim：注册等待条件时同时检查结果是否已到达，避免结果早于等待注册导致永久休眠。

LLM 输出提交和 Task 结束可以分两步。若二者之间中断，恢复者读取已提交版本及任务状态，再幂等完成 finishTask；不得提前宣告任务完成，也不得因重复通知再执行已提交交互。

### 4.3 投递、恢复与取消

消息采用至少一次交付。目标按稳定 Message ID 去重；只有处理效果已持久化才确认消费。模型输入的消费由 commitInteraction 确认；纯控制消息可由对应领取、释放或任务终态提交确认。两类情况均不得“读到即删除”。

同一 Router Task 的输入按收件箱顺序处理，完成通知建立新的对话轮次，不作为旧分派调用的第二次返回。多执行者竞争时，Task 状态和具体版本领取共同限制有效处理者。

领取到期后，Memory 回收占用并恢复待核对标记；Harness 决定是否重试并重新安排执行者。未完成模型或工具调用先核对结果。旧凭证不得提交版本、任务终态或消费确认。

取消通过 sendMessage 投递。对运行任务，Harness 的控制监听器立即调用宿主取消能力；对排队任务，可在核对修订后直接结束。取消确认通过 finishTask 统一返回；待处理输入、子任务传播范围和外部动作结果必须有明确处置。

## 5. Trace 导入

| 操作 | 输入 | 结果 |
| --- | --- | --- |
| importTrace | 稳定来源身份或 sourceKey，以及内容快照 | 固定 Session/Context 映射、版本、checkpoint、created/appended/unchanged/conflict |
| getTraceImportState | sourceKey | 导入进度、摘要及知识提取覆盖范围、缺失或冲突 |

一个 Trace 对应一个导入 Session 和一个 Context，无调度项、收件箱或 Event 流。重复导入按稳定身份定位，以记录身份和内容指纹对齐。普通追加增量发布，旧快照不回退，历史改写返回冲突。导入、摘要和知识提取分别记录进度。

## 6. 典型交互

<a id="6-完整例子规划行程中途纠正换-worker-后继续"></a>

```text
用户输入
  → sendMessage(Router Task)
  → receive 调度队列 → claimContext → Router 模型交互
  → submitTask(工作任务，returnTo = Router Task)
  → TaskAccepted → Router 保存本轮答复并等待

receive 调度队列 → claimContext → Worker 模型/工具交互
  → commitInteraction(最终结果)
  → finishTask(结果引用)
  → Router 收件箱新增结果消息并变为可调度
  → Router 从自身续接版本处理新轮次 → 保存用户答复
```

例如行程规划的父任务可分派交通、活动两个子任务。三个任务共用工作 Session 的事件流，使用独立 Context 版本；父任务释放执行名额后等待子结果，按自己的续接版本恢复。任务与返回地址稳定，执行进程可以替换。
