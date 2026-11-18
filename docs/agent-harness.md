# Agent Harness：执行对象、Core 与插件边界

状态：设计草案 v0.9；2026-11-17。本文先定义 Harness 管理的对象，再说明 Core、执行循环和接入能力。两系统的逻辑接口及完整例子见[交互协议](integration-protocol.md)；SDK、载荷及 Host 扩展点候选见[实现参考](integration-reference.md)。

## 1. Harness 管理什么

Harness 把用户意图推进为对话答复和任务结果：理解输入、选择工作 Session、安排执行、维护工作 Context，并把需要恢复的状态交给 Memory 保存。它可以作为独立应用运行，也可以嵌入 dsh、pi 等 Host。

### 1.1 Router 与 Worker：两种执行角色

| 角色 | 负责什么 | 读取和维护哪些状态 |
| --- | --- | --- |
| Router | 接收用户输入，理解意图，回答、澄清、分派及汇总 | 自己的对话 Session；按需读取该用户的记忆、其他 Session 摘要和任务状态 |
| Worker | 推进具体任务，调用工具，产生进度、结果或待澄清问题 | 被分配的工作 Session；读取与任务相关且获准使用的记忆 |

用户只与 Router 对话。Router 面向统一用户记忆域，但每次模型调用只选取必要材料。Worker 的结果经 Router 汇总；Worker 不需要 Router 为每一步重新批准。

Router 和 Worker 描述职责，不规定操作系统进程数量。Worker 进程是可替换的执行资源；其工作身份与持续状态由 Session 承载。

### 1.2 Turn、Task、Run 与 Action：不同的执行单位

| 对象 | 含义与边界 | 持久依据 |
| --- | --- | --- |
| Turn | Router 处理一次用户输入或重要任务通知的对话轮次；可含若干模型与工具步骤 | Router Session 的输入、输出和轮次记录 |
| Task | 一项有目标、约束和完成条件的工作；可以长于触发它的 Turn | Memory 协作存储中的任务状态、来源与结果引用 |
| Run | 某执行者推进某 Session 的一次执行尝试；暂停或换进程后可产生新 Run | 执行占用与运行记录 |
| Action | 一次待执行或已执行的动作，例如模型请求、工具调用或任务分派 | 动作意图、调用记录与结果；用于核对和恢复 |

这些对象的执行语义由 Harness 定义，跨进程的记录由 Memory 保存。本地仅持有当前执行所需的副本、连接和句柄。

一次 Turn 可以分派多个 Task；Task 完成时可以触发新的 Turn。一个 Task 可因暂停或重试经历多个 Run。Turn 与 Run 不是互相替代的身份：前者划分对话，后者划分执行尝试。

### 1.3 与 Session、Context 和 History 的关系

Session 是 Memory 管理的持续工作边界。Harness 选择在哪个 Session 上执行，使用其 History 和 Context，不另外建立一套只存在于进程中的权威会话。

- Router 的多个 Turn 通常属于同一个对话 Session。
- 每个 Task 有明确的目标工作 Session；同一 Session 可以先后处理多个相关 Task。
- 一个 Run 只推进一个 Session；同一 Session 的写入串行。
- 新建独立工作或分叉才产生新 Session。换 Worker、暂停恢复和 Context 压缩都可以继续使用原 Session。

Task 状态回答“工作进行到哪里”；Context 回答“接下来如何继续”；History 回答“已经发生过什么”。Router 可以先查询任务摘要，再按需读取 Context 或原始证据，无须持续接收 Worker 的完整模型输入。

### 1.4 输入与通知：协作信息

任务输入把用户的补充、纠正或取消交给相应 Worker；任务通知把等待输入、完成或失败等重要变化交回 Router。它们在 Memory 的协作存储中持久保存。

输入已送达不等于 Worker 已处理；通知已保存不等于用户已收到答复。Worker 处理输入后记录实际效果，Router 处理通知后产生自己的对话记录。普通进度和心跳可以用于查询，不逐条触发新对话。

## 2. Harness Core 包含什么

### 2.1 Core 的模块与责任

Core 是执行规则和状态推进的核心。模块可以是同一进程中的组件，不要求拆成多个服务。

| Core 模块 | 核心责任 | 使用的对象 |
| --- | --- | --- |
| Router Controller | 接收并串行处理输入、运行短对话循环、组织答复 | Router Session、Turn、任务通知 |
| Router Policy | 根据用户意图、记忆和任务状态，选择回应、澄清、新建、继续或分叉 | 目标、相关记忆、Session 候选 |
| Task Controller / Dispatcher | 登记任务、安排 Worker、处理排队、补充输入、取消和恢复 | Task、Run、执行资源 |
| Worker Controller | 推进任务的模型/工具循环，判断继续、等待、完成或失败 | Task、Run、Action |
| Context Controller | 从已保存记录及召回材料组装工作状态，追加消息、执行 compact | History、Context、来源引用 |
| Memory Coordinator | 调用 Memory 接口，保存执行记录和工作状态，读取恢复资料与协作信息 | Session、History、Context、任务与记忆 |

模型可以参与路由判断、任务推进和压缩；Controller 负责落实执行顺序、持久化和状态检查。一个模型建议本身不构成已完成的状态变更。

### 2.2 loop 为什么属于 Core

**Router loop 与 Worker loop 的推进规则都在 Core 中。** Core 决定下一步需要什么输入、执行什么动作、何时等待或停止，以及哪些状态必须先保存。

Runner、Runtime Driver 与 Host Adapter 负责把这些决定接到实际环境：

| 边界组件 | 提供什么 | 与 Core 的关系 |
| --- | --- | --- |
| Runner | 独立运行时的启动、事件驱动、并发与进程生命周期 | 驱动 Core 并转交动作与完成结果 |
| Runtime Driver | 模型、工具、执行资源、取消与外部结果核对 | 接受 Core 决定的动作，报告实际执行情况 |
| Host Adapter | 将 dsh/pi 的生命周期、请求边界、消息和原生会话映射接入 Core | 把 Host 的事件与能力转换成统一执行接口 |

Core 可以按“推进一步 → 发出动作 → 接收结果 → 再推进”的方式实现；调用它的事件循环位于 Runner 或 Host，不改变执行规则的归属。

插件模式下，Host 继续发起底层模型/工具调用；Adapter 必须能在必要边界让 Core 准备输入、保存状态并决定放行或暂停。一个 Run 只使用一种调用驱动方式，不能同时由 Host 和 Runner 发出同一请求。

## 3. Harness 如何推进工作

### 3.1 Router 的短对话循环

1. 接收并持久记录用户输入或重要任务通知，建立对应 Turn。
2. 恢复 Router Context，读取必要的用户记忆、任务或 Session 状态。
3. 决定回应、澄清、分派工作，或向已有任务追加输入。
4. 分派时先让任务得到持久接收确认，再向用户说明已接收；无需等 Worker 完成。
5. 保存本轮回复和 Context，结束 Turn，继续处理下一条输入。

同一 Router Session 的轮次串行。Worker 在执行时 Router 仍可处理新输入；这不意味着并发改写同一份模型请求。当前模型调用期间到达的输入先持久排队，在下一安全边界处理。

### 3.2 Worker 的任务循环

1. Dispatcher 选择执行资源，取得目标 Session 的写入权，建立本次 Run。
2. 从 Memory 恢复 Context、尚未处理的历史和任务输入，核对未完成动作。
3. Context Controller 纳入输入，必要时 compact，并保存下一步使用的 Context。
4. Core 通过 Driver 准备并记录实际模型输入，再执行模型调用。
5. 将输出和工具结果保存到 History，更新 Context 与任务进度，再决定下一步。
6. 达到完成、等待或停止条件时保存工作状态、报告任务状态并释放执行占用。

Worker 可以独立执行多步；Router 通过任务状态与通知了解进展。任务完成只结束这项工作，不自动结束整个 Session。

### 3.3 补充、取消与继续

Router 将针对运行中任务的补充或取消写入任务输入队列。Worker 在安全边界读取，处理后将效果记入自己的 History 和 Context。Router 不直接改写 Worker 正在使用的工作状态。

取消当前 Router Turn 只取消这轮对话动作。取消已分派任务需要明确指定 Task；取消请求已接收也不表示外部操作已经停止，Driver 必须核对实际结果。

暂停可以释放 Worker。再次推进时 Dispatcher 创建新的 Run，读取原 Session 的持久状态；先确定未完成动作的结果，再决定是否需要重试。所有 Harness 离线时任务继续保留，Memory 不代替 Harness 调度执行。

## 4. Context 与执行状态如何管理

### 4.1 一条消息如何进入 Context

对 Router 与 Worker 使用同一条逻辑顺序：

消息或结果 → Memory 保存到所属 Session 的 History → Core 更新本地工作 Context → 按需 compact → Memory 保存新的 Context 版本。

下一次模型调用前、正常暂停前和任务完成前，保存相应工作状态。实际模型输入由 Driver 准备并单独记录；Context 已保存仅说明工作状态可读取。

新事件已落盘而 Context 尚未更新时，恢复过程根据已处理位置补齐缺失内容。涉及外部副作用时，恢复工作状态与重新执行动作是两件事，后者必须先核对执行记录。

### 4.2 四种管理动作的分工

| 动作 | 决策与执行责任 | 改变的对象 |
| --- | --- | --- |
| 追加、组装与 compact | Harness Context Controller；Memory 保存版本 | 当前 Session 的 Context |
| 释放或替换执行者 | Harness Dispatcher / Driver | Run 与执行资源 |
| Session 退役 | Memory Session Maintenance；Harness 可提出检查请求 | Session 生命周期 |
| 知识提取、纠正与 notes 整理 | Memory Agent / Validator；Harness 可提交候选与反馈 | 用户记忆条目及其修订 |

第一版 Context 策略是 append + compact。压缩保留当前目标、明确约束、重要精确值、未完成工具交换和可取回来源；验证不通过时保留旧状态，不保存丢失约束的版本。

Memory 后台整理产生新条目或修订后，Harness 在后续检索或安全边界选用它们。后台任务不直接修改正在执行的 Context。短期工作状态与长期知识的关系见[概念模型](integration-protocol.md)。

## 5. Harness 暴露和依赖哪些接口

本节区分接口的提供方与调用方；具体的两系统接口清单见[交互协议第 5 节](integration-protocol.md#5-两系统的交互接口)。

| 接口边界 | 提供方 → 调用方 | 抽象对象与能力 |
| --- | --- | --- |
| 对话与任务入口 | Harness Core → 应用入口、Host Adapter | 开启或继续对话、提交用户输入、查询任务、补充或取消任务、取得回复 |
| 执行推进接口 | Harness Core → Runner、Host Adapter | 推进当前执行、接收完成结果、决定等待或结束 |
| 运行能力接口 | Runtime Driver / Host Adapter → Core | 分配执行资源、准备输入、模型/工具调用、取消和结果核对 |
| 记忆领域接口 | Memory → Core 的 Memory Coordinator | Session、History、Context、记忆检索与维护意图 |
| 执行协作存储接口 | Memory → Core 的 Memory Coordinator | 任务状态、输入、通知与写入所有权 |

应用入口不必知道 Memory 的字段或数据库；Memory 也不需要调用 Harness 的执行推进接口。Harness 主动读取任务通知，再交给自己的 Router Controller。提供这些接口不要求 Harness 独立部署为 HTTP 服务，它们也可以是嵌入式库接口。

## 6. 可插件化设计与 dsh/pi 接入

### 6.1 稳定 Core 与可替换实现

| 扩展位置 | 可替换内容 | 需要遵守的边界 |
| --- | --- | --- |
| Router Policy | 路由模型、选择策略与任务拆分方法 | 输出可由 Core 检查和落实的工作决定 |
| Context Policy | 材料选择、预算及 compact 策略 | 保留约束、来源与合法工具交换；首版仍为 append + compact |
| Runtime Driver | 模型、工具及 Worker 执行环境 | 如实报告动作与结果，支持能力声明和恢复核对 |
| Host Adapter | dsh、pi 或其他 Host 的生命周期与请求钩子映射 | 使用同一套 Core 对象；一个动作只有一条实际调用路径 |
| Memory 访问实现 | 服务连接、认证及传输 | 遵守同一领域和协作接口，独立于 Host 类型 |

Harness 的业务逻辑不依赖某个 Host 的会话文件格式。Host 原生会话与逻辑 Session 的对应关系只是适配元数据；Memory 中的 Session、History 与 Context 仍承担持久恢复职责。

### 6.2 按 Host 能力接入

| 接入能力 | 必要条件 | 可以承诺什么 |
| --- | --- | --- |
| 记忆检索 | 能在请求前注入召回材料 | 使用统一用户记忆 |
| 记录同步 | 能导出完整消息、结果和来源位置 | 持久保留可导出的经历；恢复能力还取决于 Context 与动作核对支持 |
| 完整执行接入 | 能控制请求边界、保存和恢复 Context、导出结果，并支持所需执行资源及动作核对 | 运行 Router/Worker 协作、Context 管理与可恢复执行 |

能力按 Adapter 实际支持范围声明。只有检索或记录钩子时，不宣称已经获得完整 Harness 的执行保证。Host 自带压缩与 Core compact 必须指定唯一执行责任，避免重复改变同一工作视图。

dsh/pi 的具体钩子与候选 SDK 已放到[实现参考](integration-reference.md)。接入时固定 Host 版本并验证这些能力，再决定启用范围。

## 7. 第一版范围与文档关系

第一版包含 Router 短对话循环、Worker 独立任务循环、任务输入与通知、append + compact、状态恢复及维护请求。重点验证：Worker 工作期间 Router 仍能响应；换执行者后工作可继续；用户纠正能够生效；结果通知不会重复产生回复；未确认的外部操作不会被当作成功。

CLM 和资源虚拟化保留扩展位置，后续单独实验。

- [Memory System](memory-system.md)：另一侧管理哪些持久对象及维护流程。
- [Harness 与 Memory 交互协议](integration-protocol.md)：接口提供方、调用方、操作对象及完整工作示例。
- [接口实现参考](integration-reference.md)：候选 SDK、载荷、传输及 Host 具体映射。
