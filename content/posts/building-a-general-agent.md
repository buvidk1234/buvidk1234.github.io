+++
date = '2026-09-04T01:24:15+08:00'
draft = false
title = '构建 Agent：从一个循环到通用架构'
tags = ['Agent']
+++

# 构建 Agent：从一个循环到通用架构

一次模型调用接收输入并生成输出。完成“修复一个测试失败并验证结果”需要多次调用：模型检查代码、执行操作、读取结果，再决定下一步。把这些调用连接起来，就得到一个最小的 Agent Loop。

本文沿着这段程序的数据和调用路径逐项扩展。每节从一个具体限制出发，加入解决它所需的机制。章节按工程实现依赖推进。

## 一、一个最小的 Agent

最小 Agent 只做三件事：让模型决定下一步，执行模型选择的工具，再把执行结果交还模型。模型给出最终回答时，循环结束。

```typescript
async function runAgent(userInput: string): Promise<string> {
  const messages: ModelMessage[] = [
    { role: 'user', content: userInput },
  ]

  while (true) {
    const response = await model.generate({
      messages,
      tools: toolRegistry.schemas(),
    })
    messages.push(response.message)

    if (response.toolCalls.length === 0) {
      return response.text
    }

    for (const call of response.toolCalls) {
      const result = await toolRegistry.execute(call.name, call.arguments)
      messages.push({
        role: 'tool',
        toolCallId: call.id,
        content: result,
      })
    }
  }
}
```

这个循环已经具备 Agent 的核心反馈结构：

```text
用户目标 -> 模型决策 -> 工具行动 -> 环境观察 -> 模型决策 -> ... -> 最终回答
```

本文先把这段程序称为最小 Agent：模型选择下一步行动，运行时执行行动，并把环境观察送回模型。模型停止调用 Tool 表示本轮循环结束，测试结果与任务要求共同决定工作是否完成。

模型既可以直接回答，也可以通过搜索、文件、Shell、浏览器或业务 API 观察或改变外部环境。工具结果成为下一轮观察，模型据此修正行动。

第一处问题就在 `toolRegistry.execute`：模型生成的工具名和参数必须先经过运行时校验。

## 二、校验并执行 Tool Call

要修复测试失败，模型需要读取源码、搜索符号、修改文件并运行测试。我们先把这些操作注册成 Tool：

```typescript
interface ToolDefinition<Input, Output> {
  name: string
  description: string
  inputSchema: JsonSchema
  execute(input: Input, context: ToolContext): Promise<ToolResult<Output>>
}

type ToolResult<Output> =
  | { ok: true; value: Output }
  | { ok: false; error: { code: string; message: string } }
```

运行时先解析 Tool Call 的外层结构和名称，再从当前可见的注册表中解析 Tool，最后使用该 Tool 的 Schema 校验参数：

```text
Model Tool Call
  -> Parse call envelope and name
  -> Resolve visible tool
  -> Validate arguments with its schema
  -> Execute
  -> Convert value or error into an observation
```

未知 Tool、参数错误和业务失败都返回结构化 `ToolResult`，成为下一次模型调用的观察。运行时错误保留独立类别，供后续恢复逻辑处理。

模型可以通过 Function Calling Schema 或代码式 SDK 表达调用。无论使用哪种表示，注册表都把名称、参数和执行结果收敛到同一个 `execute` 接口。

现在模型已经能调用 Shell。下一次运行中，它既可能调用测试命令，也可能访问工作区之外的文件。模型获得调用外部操作的能力后，紧接着需要执行约束。

## 三、在 Tool 执行前检查 Policy

同一个 Shell Tool 既能运行测试，也能读取凭证、启动网络进程或删除文件。模型继续负责选择行动，Policy 决定这次行动能否落实：

```typescript
type PolicyDecision =
  | { kind: 'allow' }
  | { kind: 'ask'; approval: ApprovalRequest }
  | { kind: 'deny'; reason: string }
```

本例中的决策输入包括调用者、当前 Agent、Tool、规范化参数、目标路径和副作用等级。`ask` 会生成一次性审批请求；批准结果精确绑定 Tool 名称、参数摘要和资源范围。参数发生变化时，运行时重新请求批准。

Approval 记录用户是否同意这次调用，基础权限仍以执行时的配置为准。每次真正进入执行环境前，运行时重新计算当前 Policy：新的 `deny` 直接终止调用；结果仍为 `ask` 时，只有 Tool、参数、资源范围和有效期完全匹配的 Approval 才能满足询问。

Tool 执行管线由此展开为：

```text
Resolve and validate
  -> Evaluate policy
  -> Obtain approval when required
  -> Re-evaluate current policy
  -> Enter sandboxed execution environment
  -> Execute Tool
  -> Return observation
```

Sandbox 在操作系统或远程执行环境中限制文件、进程和网络能力。Policy 决策和审批结果随 Tool Call 一起记录，恢复后仍能说明一次操作为何获准。

超时会使运行时无法判断副作用是否已经发生，因此 Tool 定义还要增加 `executionSemantics(input, context)`：

```typescript
interface ToolDefinition<Input, Output> {
  executionSemantics(input: Input, context: ToolContext): ExecutionSemantics
}

type ExecutionSemantics =
  | { effects: 'none'; retry: 'allowed' | 'forbidden' }
  | {
      effects: 'possible'
      recovery:
        | { kind: 'idempotent'; key: string }
        | { kind: 'reconcile'; operation: ReconcileOperation }
        | { kind: 'manual'; onUnknown: 'ask' | 'block' }
    }
```

这个返回值根据规范化参数和执行环境区分两类调用。无副作用的调用可以按 `retry` 自动重试；可能产生副作用的调用必须选择一种恢复方式：携带有效幂等键重试，先核对外部结果，或者交给用户处理。类型本身排除了“可能有副作用、缺少幂等键、仍然自动重试”的组合。

一个模型输出还可能包含多个 Tool Call。运行时可以并行执行互不冲突的调用，对访问同一资源或具有顺序依赖的调用串行执行。每项结果始终通过 Tool Call ID 关联原调用，组装下一次请求时再恢复模型协议要求的顺序。

执行器跨越可能产生外部副作用的位置之前，Session 必须先持久写入 `tool/start`，其中包含稳定的 Tool Call ID 和幂等键。`tool/start` 之后收到取消信号，但执行结果尚未确认的调用进入 `outcome_unknown`；执行器确认停止后才能记录 `cancelled`。

至此，一次运行已经能够在既定权限和隔离范围内修改代码并启动测试。它的全部历史仍保存在 `messages` 数组里，进程中断会让调查进度消失。

## 四、把 messages 写入 Session

当前程序把全部历史放在内存中的 `messages`。假设 Agent 修改了一个文件，测试运行到一半时进程退出；重新启动后，它需要知道用户要求、模型行动、Tool 结果以及中断发生的位置。

先给循环中的两个范围命名：

- **Turn** 从领取一项工作开始，到这次连续执行完成、失败或取消为止。
- **Step** 表示一个逻辑决策周期。它可以多次请求模型，最终至多有一个输出成为该 Step 的 Assistant Message，并触发一批 Tool Call。

Session 使用追加式事件日志记录这些事实。一次“先读文件，再给出答案”的 Turn 可以写成：

```text
turn/start
user/message
step/start
assistant/accepted       # contains a validated Tool Call
tool/call
tool/start
tool/result
step/end
step/start
assistant/accepted       # contains the validated final answer
step/end
turn/end
```

模型历史和 UI 都从日志投影出来。`assistant/accepted` 是模型输出进入历史的唯一事件；原始输出和校验失败用于审计与错误分类。模型历史保留后续推理所需的 user、assistant 和 tool 内容，UI 还可以显示流式增量、审批与执行阶段。

崩溃可能让日志停在任意一个已经提交的事件后面，因此“合法的日志前缀”和“可发送给模型的闭合历史”需要分开判断。恢复程序允许一个未闭合尾部，并按以下顺序处理：

1. 找出开放的 Tool Call、Step 和 Turn。
2. 将未完成的模型请求记为 `interrupted`。
3. 将结果未知的 Tool Call 记为 `outcome_unknown`，再依据 Tool 的执行属性核对外部状态。
4. 追加缺失的结束事件，使新的模型历史重新满足消息配对要求。

事件序号保持单调，每个 ID 只属于一个父范围。写入成功的事实保持不可变；状态变化通过新事件表达。现在运行时能够重建闭合的模型历史并识别中断位置，完整任务恢复将在后文加入。Agent 此时仍要重新摸索项目的测试命令和操作顺序。

## 五、按需加载 Skill

测试入口、代码生成步骤和验证范围属于项目知识。把这些内容全部写进系统 Prompt 会让每次模型请求携带大量无关指令。Skill 将一类任务的说明、脚本和参考资料放在一起：

```text
fix-test-failure/
  SKILL.md
  scripts/
    run-focused-tests.sh
  references/
    testing-policy.md
```

运行时先向模型提供简短目录，例如 Skill 的名称和用途。模型选择 `fix-test-failure` 后，`load_skill` 作为普通 Tool Call 进入刚才建立的解析、Policy 与日志管线；它从 Skill Registry 读取内容，再把 `SKILL.md` 作为 Tool Observation 送入下一个 Step。

这形成了渐进披露：

1. 名称和简介常驻模型输入。
2. `SKILL.md` 在选中后加载。
3. 明确引用的脚本、资料和资源按需读取。

Session 记录被加载内容的不可变快照，或者记录内容哈希及其对应的不可变存储引用，使恢复后的模型看到同一版说明。Skill 提供操作方法，Tool 与 Policy 继续决定每个动作如何执行。

按照 Skill 的说明，Agent 已经知道应先运行局部测试，再运行完整回归。这个任务包含多个阶段，下一步需要把当前进度显式保存下来。

## 六、用 Plan 保存任务进度

模型可以在对话里写一段计划，运行时更需要一份能够更新、投影和恢复的结构化记录。对于当前修复任务，它可以是：

```json
{
  "items": [
    { "content": "复现失败", "status": "completed" },
    { "content": "定位根因", "status": "in_progress" },
    { "content": "修改实现", "status": "pending" },
    { "content": "运行局部测试", "status": "pending" },
    { "content": "运行完整回归", "status": "pending" }
  ]
}
```

`update_plan` Tool 每次提交完整快照，Session 追加一条 `plan/replaced` 事件。重放时取最后一份快照即可得到当前 Plan；短计划采用全量替换也能避免局部补丁与旧版本错配。较大的任务可以为 Plan 增加版本号：提交更新时携带当前版本，版本仍匹配才写入，否则重新读取 Plan。

如果用户要求“先给出修改方案，再编辑代码”，运行时可以进入 Plan Mode，让 Agent 先探索和形成方案。只读要求同时下沉到 Tool Policy，由执行层落实。Plan 负责当前 Turn 的工作路径，模型内部推理仍由模型接口管理。

Agent 正在“定位根因”时，用户补充了一条约束：“保持现有数据库结构”。当前模型请求已经发出，运行时需要决定这条消息在何时生效。

## 七、在 Step 边界接收新输入

已经发出的模型请求应保持不变。用户补充的数据库约束要进入下一个 Step，CI 监控器可以静默附加最新测试状态，用户还可以安排“修复后补一条回归测试”。新消息先进入持久 Inbox，再由运行时选择一个安全的交付点：

```typescript
interface InboxMessage {
  id: string
  content: ModelContent
  target: 'next_step' | 'next_turn'
  wakeup: boolean
  status: 'pending' | 'reserved' | 'included' | 'cancelled'
}
```

这组字段可以表达几种常用操作：

| 操作 | `target` | `wakeup` | 含义 |
| --- | --- | --- | --- |
| Steer | `next_step` | `true` | 在当前 Turn 的下一个 Step 修正方向 |
| Inject | `next_step` | `false` | 附加背景信息，等待其他工作驱动 |
| Follow-up | `next_turn` | `true` | 当前 Turn 结束后开始下一项工作 |

用户的“保持现有数据库结构”属于 Steer。运行时让当前模型请求结束，在创建下一个 Step 时领取这条消息，并和 `step/start` 在同一个持久事务中提交。消息先进入 `reserved`；包含它的第一份不可变模型请求快照提交时再进入 `included`。`included` 只表示消息已经进入请求快照；请求是否实际发送以及如何结束由独立调用事件记录。进程在两者之间退出时，恢复程序可以继续处理同一条消息。

Agent 也可以主动发起 Ask User。问题包含稳定 ID、选项、自由输入规则和敏感信息标记，当前 Turn 进入等待；回答通过同一个 Inbox 关联到原问题。审批仍由 Policy 处理，因为它授权的是一个已经确定的副作用。

Interrupt 取消当前 Turn，并保留 Session 供后续继续。Turn 的取消信号传给模型和正在运行的 Tool；运行时停止派发新调用，等待已开始操作收敛，再为开放的 Step 与 Turn 追加取消事件。待处理 Follow-up 依据产品策略保留或取消。

现在用户可以介入运行过程。完整回归测试仍可能持续二十分钟，直接等待 Tool 返回会占住当前执行位置。

## 八、把长操作交给 Job

完整回归适合在后台运行。启动测试的 Tool 立即返回 Job ID，Agent 可以处理其他工作，随后查询或等待结果：

```typescript
interface JobBase {
  id: string
  owner: AgentId
}

type Job<Result> =
  | JobBase & { phase: 'pending' | 'running' }
  | JobBase & { phase: 'reconciling'; reason: 'outcome_unknown' }
  | JobBase & { phase: 'completed'; result: Result }
  | JobBase & { phase: 'failed'; reason: 'executor_error' }
  | JobBase & { phase: 'cancelled'; reason: 'user_cancelled' }
  | JobBase & { phase: 'interrupted'; reason: 'runtime_restarted' }
```

Job 提供启动、状态查询、等待和取消语义。外部结果尚未确认时进入 `reconciling`，核对完成后才转入明确终态。终态事件进入拥有者的 Inbox：空闲 Agent 可以被唤醒，正在执行的 Agent 在下一个 Step 或 Turn 接收通知。Session 保存 Job ID、所有者和阶段事件；进程句柄、线程与网络连接留在内存中，恢复时依据持久记录重新创建、重新连接或转为 `interrupted`。

调度器对前台 Tool 和后台 Job 统一施加并发、队列和预算限制。取消请求先阻止新动作，再通知执行器收敛；已经产生外部副作用的 Job 依据 Tool 的 `reconcile` 语义确认结果。

### 把固定步骤写成 Workflow

代码生成、局部测试和完整回归经常按同一拓扑重复执行，这时可以把拓扑写成 Workflow：

```text
run focused tests
  -> generate derived files when needed
  -> run full regression
  -> collect report
```

Workflow 的节点和分支由程序预先声明，节点本身仍可以调用模型、失败或重试。Agent 决定是否启动这个 Workflow；Workflow 决定节点按什么顺序运行。

后台测试可能跨越当前 Turn，外部服务也可能要求数小时后再次检查。运行时还需要一个能够跨 Turn 保存完成条件的对象。

## 九、用 Goal 保存跨 Turn 的完成条件

Plan 描述当前路径，Follow-up 描述下一项输入，Job 描述后台执行。Goal 保存更长时间的完成条件，例如“修复这个测试失败，并让完整回归通过”。一种实现可以写成：

```typescript
interface Goal {
  id: string
  objective: string
  phase: 'active' | 'paused' | 'blocked' | 'complete' | 'failed' | 'cancelled'
  reason?: string
  revision: number
  budget?: { tokens?: number; seconds?: number; cost?: number }
  usage: { tokens: number; seconds: number; cost: number }
}
```

`phase` 表示生命周期，`reason` 解释暂停或阻塞原因。用户和 Agent 更新 Goal 时携带版本号 `revision`，避免并发写入覆盖新状态。每次模型请求的 token、时间和费用都计入用量，包括失败与取消的请求。

Goal 记录事实，Continuation Policy 决定是否自动开启下一 Turn。它检查完成状态、剩余预算、正在等待的 Job、连续无进展次数以及外部依赖。代码发生变化、测试获得新结果或验证通过都可以作为进展证据。

Completion Policy 定义一类工作何时完成。本例修改了代码，因此要求保存测试结果等验证证据；普通问答可以在最终输出校验通过后完成。这项策略依据已接受的输出与持久证据更新 Goal。检查未通过时，Session 追加模型可见的 `completion/rejected`，写明缺少的条件和当前 Goal 或工作版本；同一工作连续被拒绝达到上限后，调度器停止自动创建 Step，并转为失败、阻塞或询问用户。

用户输入、Follow-up、Job 完成通知和 Goal continuation 最终都会产生可运行工作。调度器应在同一个持久事务中选择工作、创建 Turn，并把来源绑定到该 Turn：

```typescript
const turn = await scheduler.claimAndStartTurn({
  inbox,
  goals,
  jobs,
})
```

这一步统一确定用户消息与自动续跑的优先级，并消除“工作已经领取、Turn 尚未创建”之间的崩溃窗口。

第二天继续修复时，Agent 需要重新取得测试规范，并延续用户保存的“这个仓库始终先跑局部测试”。与此同时，完整测试日志已经大到不适合放进消息。项目约定、跨 Session 的经验和大型结果具有不同的写入者、更新周期和读取方式。

## 十、用文件保存知识与产物

### 项目约定随仓库保存

测试命令、目录规则和验收方式通常由维护者写进 `AGENTS.md` 等仓库文档。运行时从项目根目录走到当前工作目录，按从宽到窄的顺序收集适用文件；进入更深的目录时，再加载作用域更小的说明。文档随代码接受版本控制和审查，模型实际看到的版本同时记录进 Session。

项目文档说明“在这个仓库工作要遵守什么”，Skill 说明“某一类任务怎样完成”。前者按工作目录生效，后者按任务选择。超出常驻指令预算的设计文档和测试资料仍保留为普通文件，由 Agent 使用 `read` 和 `search` Tool 按需取得。

### 跨会话经验写入 Memory

环境事实和反复验证过的工作方法适合写进 `MEMORY.md`，用户偏好和工作习惯可以单独写进 `USER.md`。最小实现提供一个 Memory Tool，以 `add`、`replace` 和 `remove` 等结构化操作更新这些文件。运行时用文件锁或版本号保护读改写，发现外部变化时重新读取，避免多个 Session 互相覆盖。

短小的 Memory 可以在 Session 开始时直接装入。内容增长后，运行时先装入一份紧凑摘要或目录，再让模型按关键词搜索 `MEMORY.md`，只读取相关段落。需要自动积累经验时，可以在 Session 结束后增加一条整理流水线：从历史中提取候选事实，合并重复项，淘汰过期内容，最后更新 Markdown 和它的摘要。原始对话仍由 Session 保存，Memory 只保留预计会再次使用的知识。

```text
Session history -> extract -> consolidate -> MEMORY.md
                                      +-> memory summary

next Session -> summary -> search/read -> selected excerpts
```

进入模型的 Memory 摘要和片段带上文件版本与位置，并作为本次请求的一部分记录进 Session。文件随后发生变化时，恢复程序仍能重建当时的输入。一条经验成为团队共同遵守的规则后，再由维护者把它提升到项目文档。

某次调查的完整过程继续保存在原 Session 中。用户引用旧会话，或者 Agent 通过会话搜索找到相关记录时，运行时读取一份有界快照并写入当前 Session；从中提炼出的可复用结论再进入 Memory。

### 资料增大后建立检索索引

Markdown 文件保存事实，检索机制负责找到相关片段。单个代码仓库通常从已知路径、文件名和全文搜索开始；语料继续增大时，可以依次加入 FTS、重排和语义索引：

```text
known path -> filename/grep -> full-text index -> semantic index
```

索引是源文件的派生数据，可以重新构建。检索先按用户、项目和权限缩小范围，再按 token 预算返回带有来源、版本和位置的片段。`RAG` 适合大型、远程或多租户知识库；通用 Agent 只需定义稳定的 Retriever 接口，本地全文搜索与向量服务都可以成为它的实现。

### 大型结果写成 Artifact

完整测试日志、补丁、报告和图片适合保存在文件系统或对象存储中。Tool Observation 只携带 Artifact ID、摘要和相关片段，Session 保存稳定引用。恢复后的 Agent、后台 Job 和其他 Agent 可以通过同一个 ID 读取结果。

现在单个 Agent 已经拥有完成调查所需的信息。根因分析还可以拆成两项互相独立的工作：一路检查最近改动，另一路分析失败日志。

## 十一、把独立调查交给 Subagent

父 Agent 可以把两项调查写成明确任务，并为每项任务创建一个 Subagent：

```typescript
const child = await agents.spawnSubagent({
  requestId: stableSpawnRequestId,
  parent: parentAgentId,
  task: '分析失败日志，返回最可能的根因和证据',
  context: forkContext(parentSession, { through: closedStepId }),
  tools: ['read', 'search'],
  budget: childBudget,
})
```

`spawnSubagent` 本身作为 Tool 经过参数校验与 Policy。启动执行器之前，Session 先保存稳定的请求 ID、父子关系、任务、权限快照和预算；子 Agent 执行器以请求 ID 幂等创建实例。进程在启动前后退出时，恢复程序可以查回同一个子 Agent ID，避免重复创建。

上下文可以从空白配置开始，也可以 Fork 父 Session 的一个闭合 Step。显式的子任务始终单独附加，父 Agent 当前 Turn 的相关信息因此不会因 Fork 边界而丢失。子 Agent 从创建后拥有自己的 Session 和后续事件。

独立 Session 隔离模型上下文，工作区隔离需要额外机制。并行写代码时，可以为每个子 Agent 创建 worktree 或 Sandbox、对共享资源加锁，或者要求子 Agent 只返回 Patch/Artifact，再由父 Agent 合并。

### 父子 Agent 怎样传递消息

父子 Agent 各自拥有 Session 和 Inbox，运行时通过稳定的 Child ID 和持久父子关系寻址。创建子 Agent 时，运行时先保存父子关系和初始上下文，再把任务写入子 Agent 的 Inbox，由它启动首个 Turn。Fork 只决定子 Agent 最初继承哪些历史，后续通信仍通过消息完成。

```text
父 Agent                                      子 Agent
spawn(task, context) -----------------------> 首个 Turn
send_message / Follow-up ------------------> Inbox -> Step / Turn
Inbox <---------------- report / completion notice
interrupt ---------------------------------> 当前 Turn 的取消信号
```

父 Agent 发送消息时只提交 Child ID 和内容。运行时从当前执行环境确定发送者，校验父子关系，再把带有 Message ID、发送者和来源的消息持久写入子 Agent 的 Inbox。`send_message` 可以只排队，Follow-up 则会唤醒空闲的子 Agent；子 Agent 正在运行时，消息在当前模型请求结束后的安全 Step 领取，或者按实现约定等待下一个 Turn。已经发出的模型请求始终保持不变。

子 Agent 可以用 Report 主动返回阶段性发现；运行时根据持久父子关系确定它的直接父 Agent，无须让模型填写接收者。子 Agent 结束时，运行时还会自动向父 Agent 投递 Completion Notice，至少包含 Child ID、终态以及最终回答或 Artifact 引用。Report 与完成通知进入父 Agent 的 Inbox：父 Agent 正在运行时可在下一个 Step 读取，空闲时可以被唤醒，也可以依部署策略静默排队。父 Agent 默认只接收这些明确消息，详细的推理过程和 Tool 结果继续保存在子 Agent 自己的 Session 中。

等待方式决定最终结果从哪条路径返回。前台委派保持原 Tool Call 打开，子 Agent 的最终输出直接成为 Tool Result；一次性后台委派返回 Job ID，由父 Agent 查询或等待；可继续的后台委派在初始任务被接受后返回 Child ID，后续工作通过消息发送。`wait` 只等待 Inbox 或子 Agent 状态发生变化，`Interrupt` 只请求取消目标当前 Turn。一次投递返回的 Message ID 表示接收方 Inbox 已经接受消息，不表示接收方已经执行完毕。

子 Agent 的 Tool、网络、凭证和预算来自父权限的子集。父 Agent 结束时，所有权规则决定继续等待、转交或取消子任务。最大并发数与委派深度由调度器配置。

能力增加到这里，下一次模型请求已经要汇集 Session、Skill、Plan、Inbox、Goal、适用的文档与 Memory，以及子任务通知。请求构造开始成为独立问题。

## 十二、组装下一次模型请求

最初的 Loop 直接传入 `messages` 和 Tool Schema。现在，一次请求需要汇集已经出现的全部信息：

```text
stable instructions and applicable workspace documents
  + visible Tool schemas and selected Skills
  + Session history
  + current Plan and Goal
  + claimed Inbox messages and Job notifications
  + Memory summary and selected file excerpts
  + optional Retriever results
  + Artifact and Subagent summaries
```

Context Assembly 接收结构化贡献。每项贡献包含内容、来源、作用域、信任级别、内容哈希和预计 token 数。装配器负责选择、排序和分配预算，模型调用层负责把结果转换为具体提供方的请求格式。

系统指令、项目规则、用户输入、Tool Observation 和外部文档具有不同来源。来源标记帮助运行时隔离外部文本、展示引用并决定何时审批。它属于 Prompt Injection 缓解与审计信息；Tool 的硬约束继续由参数 Schema、资源白名单、Policy 和 Sandbox 实施。

稳定指令与 Tool Schema 保持固定顺序，变化较频繁的 Session 尾部和 Inbox 消息靠近请求末尾。这既让优先级可预测，也便于复用模型提供方的 Prompt Cache。

模型实际看到的内容必须能够从 Session 重建。运行时可以保存完整的已准备请求；也可以把 Prompt、Tool Schema、项目文档、Skill、Memory 片段和 Retrieval 结果保存成受保留的内容寻址对象，再在 Session 中记录引用与投影版本。

当历史继续增长时，装配器会发现当前输入、固定开销、预计下一轮 Tool Observation 和最大输出预算即将超过模型窗口。接下来需要缩短模型可见历史，同时保留 Session 原始事实。

## 十三、用 Compaction 缩短历史

Compaction 保留原始 Session 日志，并在模型输入中用 Checkpoint 替换较早的完整 Step：

```text
Session log:   [old events........................][recent events]
Model input:   [checkpoint summary][recent verbatim tail]
```

直接总结全部旧消息会暴露三个问题。

第一，普通摘要容易丢失完成条件和精确引用。Checkpoint 因此要保留用户目标与修正、已确认事实、关键决策、已修改文件、验证结果、当前 Plan 与 Goal、未完成事项、权限限制以及仍在运行的 Job 和 Subagent ID。

第二，压缩边界可能切开 Assistant Tool Call 与 Tool Result。运行时只选择闭合的 Step，并完整保留刚发生的错误、用户原话和本轮已领取的 Inbox 消息。

第三，生成摘要期间仍可能出现新事件或进程退出。我们把一次实际发送给模型的请求称为 Attempt。推进当前 Step 和生成 Compaction 摘要分别创建 Attempt，并记录取消、错误、用量和预算。

生成摘要时，运行时先选择由闭合 Step 组成的候选区间。摘要请求使用预先限定的输入，不能递归触发 Compaction。

提交摘要前，运行时先用目标模型的 tokenizer 组装候选投影，计算它相对旧投影减少的 token。候选没有达到配置的最低收益时，保留旧投影并返回 `ContextCannotFit`。

通过收益检查的候选携带待替换历史的版本 `historyRevision`，只在版本未变时写入 Checkpoint。记录已准备请求、Inbox 的 `included` 状态等尾部事件不会改变这个历史版本。版本冲突时重新选择区间，提交成功前继续使用旧投影。生效的 Checkpoint 也可以成为后续 Subagent 的 Fork 点。

没有可压缩的闭合 Step、摘要内容未变，或者请求版本 `requestRevision` 达到上限时，运行时同样以 `ContextCannotFit` 结束当前 Step。固定指令本身已经超过窗口时，运行时会返回 `ContextCannotFit`。

触发条件由实际 token 预算计算：

```text
current input
  + fixed instructions and Tool schemas
  + reserved maximum output
  + expected next observations
  >= model context limit
```

循环每次最多提交一个新的 Checkpoint，然后重新组装请求：

```typescript
async function compactForRetry(
  agent: AgentRuntime,
  turn: Turn,
  step: Step,
  prepared: PreparedRequest,
  requestRevision: number,
): Promise<number> {
  if (requestRevision >= agent.compaction.maxRequestRevisions) {
    throw new ContextCannotFit('request revision limit reached')
  }

  // 选择区间并用一个可计量的 Attempt 生成候选摘要。
  const candidate = await agent.compaction.buildCandidate({
    historyRevision: prepared.historyRevision,
    preserve: step.claimedMessageIds,
    tokenizer: prepared.tokenizer,
    signal: turn.signal,
  })

  if (candidate.kind !== 'ready') {
    throw new ContextCannotFit(candidate.reason)
  }

  const tokenGain = candidate.inputTokensBefore - candidate.inputTokensAfter
  if (tokenGain < agent.compaction.minimumTokenGain) {
    throw new ContextCannotFit('compaction made no progress')
  }

  const result = await agent.compaction.commitCandidate(candidate, {
    expectedHistoryRevision: prepared.historyRevision,
  })
  if (result.kind === 'history_changed') {
    return requestRevision + 1
  }
  if (result.kind !== 'committed') {
    throw new ContextCannotFit(result.reason)
  }

  invariant(result.checkpointDigest === candidate.checkpointDigest)
  return requestRevision + 1
}
```

Compaction 处理历史投影。单个巨大 Tool Result 应先转成 Artifact，过多 Tool Schema 应按任务选择，图片等资源则采用各自的缩放或裁剪策略。

至此，Context 已能适应长任务。接入第二个模型提供方时，窗口大小、Tool Call 事件、结束原因和错误类型又会发生变化。

## 十四、用 Model Adapter 隔离模型差异

当前模型在调查中遇到限流时，运行时可能需要在下一个 Step 选择备用提供方。多个提供方带来不同的上下文容量、图片支持、并行 Tool Call、Structured Output、流式事件和错误格式。Model Adapter 把这些差异转换成运行时接口：

```typescript
interface ModelAdapter {
  capabilities(model: string): Promise<ModelCapabilities>
  prepare(request: AgentRequest): Promise<PreparedRequest>
  stream(request: PreparedRequest, signal: AbortSignal): AsyncIterable<ModelEvent>
  classifyError(error: unknown): ModelError
}
```

一次 Step 开始时冻结提供方、模型、推理强度、输出上限和能力快照。Adapter 把文本增量、Tool Call 参数增量、用量和提供方元数据转换成统一事件，并把结束原因归一为 `stop`、`tool_calls`、`length`、`refusal`、`content_filter` 或错误。只有经过校验的 `stop` 才能作为最终回答结束 Turn。

Step 记录一次决策；归属于它的 Attempt 记录为完成这次决策而实际发出的每个模型请求，Compaction 的摘要调用也使用同一种记录。临时网络错误或限流会创建新的 Attempt，并复用完全相同的已准备请求。上下文溢出先提交新的 Compaction，再递增 `requestRevision` 并重新准备请求；它仍属于当前 Step，却已经是一份不同请求。每个 Attempt 都记录请求哈希、流式事件、错误、退避和实际用量。

模型流结束后，运行时先检查结束原因和输出结构。合法 Tool Call 或通过最终输出校验的 `stop` 才进入模型历史；`length`、拒答、内容过滤和结构错误作为未接受的 Attempt 结果保留在审计事件中。`acceptModelOutcome` 是模型输出进入后续推理内容的唯一入口；它把 Assistant Message 与 `attempt/end` 在同一个事务中写入。

下面的 `generateStep` 在同一个 Step 内处理请求重建和重试，成功结果仍需经过接受步骤：

```typescript
async function generateStep(
  agent: AgentRuntime,
  turn: Turn,
  step: Step,
): Promise<Generated> {
  let requestRevision = 0

  requestLoop: while (true) {
    const request = await agent.context.assemble({
      step,
      requestRevision,
      modelConfig: step.frozenModelConfig,
    })
    const prepared = await agent.model.prepare(request)
    await agent.session.recordPreparedRequest({
      step,
      snapshot: prepared.snapshot,
      include: step.claimedMessageIds,
    })

    if (prepared.exceedsInputBudget) {
      requestRevision = await compactForRetry(
        agent,
        turn,
        step,
        prepared,
        requestRevision,
      )
      continue
    }

    while (true) {
      const attempt = await agent.session.startAttempt({
        step,
        requestRevision,
        requestDigest: prepared.digest,
      })
      const meter = new UsageMeter()

      try {
        const outcome = await collectOutcome(
          agent.model.stream(prepared, turn.signal),
          {
            onEvent: (event) => attempt.recordEvent(event),
            usage: meter,
          },
        )
        return { attempt, outcome, usage: meter.usage }
      } catch (error) {
        if (isCancelled(error)) {
          await attempt.finish({ reason: aborted(error), usage: meter.usage })
          throw error
        }

        if (isModelFailure(error)) {
          await attempt.finish({ reason: failed(error), usage: meter.usage })

          if (error.kind === 'context_overflow') {
            requestRevision = await compactForRetry(
              agent,
              turn,
              step,
              prepared,
              requestRevision,
            )
            continue requestLoop
          }

          if (agent.retryPolicy.permits(error, attempt.number)) {
            await agent.retryPolicy.backoff(error, turn.signal)
            continue
          }
          throw error
        }

        await attempt.finish({
          reason: runtimeError(error),
          usage: meter.usage,
        })
        throw error
      }
    }
  }
}

async function acceptModelOutcome(
  agent: AgentRuntime,
  generated: Generated,
): Promise<AcceptedModelOutcome> {
  let accepted: AcceptedModelOutcome
  try {
    accepted = validateModelOutcome(generated.outcome)
  } catch (error) {
    await agent.session.rejectAttemptOutput({
      attempt: generated.attempt,
      rawOutcome: generated.outcome,
      reason: invalidOutput(error),
      usage: generated.usage,
    })
    throw error
  }

  // 原始结果、已接受的 Assistant Message 与 attempt/end 原子提交。
  await agent.session.acceptAttemptOutput({
    attempt: generated.attempt,
    rawOutcome: generated.outcome,
    message: accepted.message,
    usage: generated.usage,
  })
  return accepted
}
```

Workflow 中负责分类测试失败的模型节点必须返回固定 JSON。Structured Output 因此也通过 Adapter 提交 JSON Schema。提供方可以使用原生约束或兼容实现，返回值继续经过 Schema 和业务规则校验。

完整回归准备从本地进程迁移到远程 CI，用户也要从 CLI 切换到 IDE。前一个变化要求替换能力实现，后一个变化要求客户端能够接续同一个 Session。

## 十五、注册扩展并连接客户端

### 替换能力实现

远程 CI Tool 继续使用原来的名称、参数和结果类型，只把执行部分换成新的能力实现（Provider）。Tool、Memory、Model 和 Subagent 都可以采用同一种组织方法：运行时依赖稳定接口，部署配置选择本地或远程 Provider。

Provider 通过 Registry 注册，注册操作返回注销函数；应用、Agent 或 Session 结束时注销其所属条目。准备模型请求、执行 Tool、提交 Compaction 和结束 Turn 等组合点也可以接受有序扩展；每个处理器明确选择继续、返回结果或抛出错误。MCP 是连接远程 Tool、资源和 Prompt Provider 的一种实现。

远程 CI 还需要认证。Credential Provider 根据 Agent、Tool 和目标服务的作用域，把短期凭证注入受控执行环境。模型输入和 Tool 参数只保存凭证引用，Session 记录使用事件，不记录凭证内容。

### 从 CLI 接续到 IDE

CLI 和 IDE 消费同一组 Session 事件。客户端提交工作后立即获得稳定的 Session ID、Turn ID 或 Inbox Message ID，再按事件序号接收文本增量、Tool Call、审批、Plan、Job 和完成状态。断线重连从最后确认的事件序号续传，历史分页与实时事件使用同一套标识。

JSON-RPC、WebSocket 和 HTTP 事件流都可以承载这组消息。内部接口、Session 事件格式和客户端协议分别版本化，因为三者的兼容周期不同。

接口已经能够替换，事件也能够传输。现在可以做一次完整故障演练：在模型流式输出、测试 Job 或写入 Tool 运行期间终止进程，然后恢复任务。

## 十六、恢复被中断的任务

假设进程在完整回归运行期间退出。新的运行时先重放 Session 的已提交前缀，再执行恢复：

1. 找到开放的 Turn、Step、Attempt、Tool Call 和审批。
2. 为中断的模型 Attempt 追加独立的 `interrupted` 终态并结算已有用量。
3. 按 Tool 的副作用、幂等键、结果核对能力和未知结果策略处理开放调用。
4. 恢复待审批交互。执行调用前始终按当前权限重新计算 Policy；旧 Approval 只有在结果仍为 `ask`，且 Tool 版本、规范化参数摘要、资源范围和有效期全部匹配时才能满足这次询问。
5. 检查 Inbox reservation：已经进入不可变请求快照的消息保持 `included`，其余消息重新变为 `pending`。
6. 查询外部 Job 和 Subagent；保存 ID 的任务重新连接，进程内任务转为 `interrupted`，结果不明的外部任务进入 `reconciling/outcome_unknown`。
7. 恢复最后生效的 Plan、Goal、Skill 快照和 Compaction 版本。
8. 检查所有未完成的已领取工作，包括尚未启动的 Tool Call、已返回结果但未结束的 Step，以及尚未结算的 Turn。需要继续推进时，在结束旧 Step 和 Turn 的同一个事务中追加 `recovery/continuation`。等待审批或核对外部结果的工作先保持等待，条件满足后再使 continuation 可运行。
9. 创建新的取消信号、锁、进程句柄与网络连接，再把可运行工作交给调度器。

`recovery/continuation` 和用户输入、Job 通知及 Goal continuation 一样，由调度器原子领取并创建新 Turn。恢复始终追加事件，不改写已经提交的历史。副作用状态无法确认时，对应 Tool Call 或 Job 记为 `outcome_unknown`，其工作来源保持等待；关联了 Goal 的工作再把 Goal 标为 `blocked`，等待查询结果或用户决定。

主动关闭使用同一套收敛逻辑：停止领取新 Turn，取消开放的模型请求，阻止新的 Tool Call，通知 Job 与 Subagent，等待带副作用的操作达到可记录终态，最后释放临时资源。

恢复演练验证了持久状态，下一步还要验证模型实际看到了什么、用户实际收到了什么，以及不同模型输出是否走过相同的运行路径。

## 十七、回放并验证运行行为

Session、Goal、Turn、Step、Attempt、Tool Call 和 Job 都有稳定 ID，因此一次任务可以投射成完整 Trace。事件记录开始与结束时间、模型版本、用量、Tool 参数摘要、Policy 决策、错误类别和取消原因；敏感内容在离开相应信任范围前脱敏。

测试证据可以来自不同范围：

- Tool Schema、Policy、Inbox 领取和日志投影使用组件测试。
- Turn/Step/Attempt 嵌套、Tool Call 终态、工作只领取一次和取消最终收敛使用不变量测试。
- “修复测试失败”使用固定模型事件进行完整回放，检查已准备请求、Session 日志和客户端事件快照。
- 真实环境评测覆盖模型选择、外部 API、Sandbox、延迟、成本和恢复行为，失败轨迹再进入回归集。

最终文本是其中一项结果。任务完成率、误操作率、人工接管率、未知副作用数量、每个 Goal 的 token 与耗时、Compaction 后的关键信息保留率，以及 Subagent 对成功率和延迟的影响，可以帮助定位运行质量变化的来源。

到这里，每项能力都已经从一个具体限制中产生。现在可以把它们装回最初的 Loop，并检查生命周期是否仍然清楚。

## 十八、把所有能力装回 Loop

完整运行时仍然围绕最初的反馈循环。每个影响恢复或后续模型输入的变化都写入 Session；创建 Turn、Step、Attempt 或 Tool Call 的组件也负责写入对应的结束事件：

```typescript
async function drive(agent: AgentRuntime): Promise<void> {
  while (true) {
    // 领取工作、写入 turn/start，并把工作来源绑定到 Turn。
    const turn = await agent.scheduler.claimAndStartTurn({
      agent,
      signal: agent.stopSignal,
    })
    if (turn === null) return

    try {
      turnLoop: while (true) {
        // 原子写入 step/start，并领取当前可交付的 Inbox 消息。
        const step = await agent.scheduler.claimAndStartStep(turn)

        try {
          const generated = await generateStep(agent, turn, step)
          const accepted = await acceptModelOutcome(agent, generated)

          if (accepted.kind === 'tool_calls') {
            await agent.tools.runPipeline({
              step,
              calls: accepted.calls,
              signal: turn.signal,
            })
            await agent.scheduler.finishStep(step, { terminal: 'tools' })
            continue
          }

          await agent.scheduler.finishStep(step, { terminal: 'answer' })

          // 原子检查 Steer、Completion Policy 和 Goal 版本。
          const decision = await agent.scheduler.settleTurnIfQuiescent({
            turn,
            acceptedAnswer: accepted.answer,
            expectedGoalRevision: turn.goalRevision,
          })
          switch (decision.kind) {
            case 'next_step':
              continue turnLoop
            case 'waiting_input':
              await agent.scheduler.waitUntilRunnable(turn, {
                signal: turn.signal,
              })
              continue turnLoop
            case 'terminal':
              break turnLoop // completed, failed or blocked
            default:
              assertNever(decision)
          }
        } catch (error) {
          const terminal = isCancelled(error) ? aborted(error) : failed(error)
          await agent.scheduler.finishStep(step, { terminal })
          throw error
        }
      }
    } catch (error) {
      const terminal = isCancelled(error) ? aborted(error) : failed(error)
      await agent.scheduler.finishTurn(turn, { terminal })
    } finally {
      await agent.accounting.settleTurnOnce(turn)
    }
  }
}
```

`validateModelOutcome` 只接受结构完整的 Tool Call 或通过最终输出校验的 `stop`。`finishStep` 和 `finishTurn` 都是幂等操作，并从内向外结束尚未闭合的 Attempt、Tool Call、Step 与 Turn。

`settleTurnIfQuiescent` 在同一个事务中检查待交付的 Steer 与 Job 通知，运行这类工作的 Completion Policy，以预期版本更新 Goal，再返回下一 Step、等待输入或 Turn 终态。新输入直接进入下一个 Step；Completion Policy 拒绝完成时，事务写入 `completion/rejected`，使模型在下一 Step 看到缺失条件。Goal 用量从已经结束的 Attempt 事件投影得到。

这里保留几条贯穿全文的不变量：任何被领取的工作都有 Turn 归属；一个 Turn 同时只有一个开放 Step，一个 Step 同时只有一个开放 Attempt；每个 Tool Call 最终都有 result、denied、cancelled 或 outcome_unknown；只有经过校验的 `stop` 和通过 Completion Policy 的工作可以成功结束 Turn；每个 Attempt 的用量都会进入结算。

## 十九、总结

回到开头，每项新增机制都对应一次运行中实际遇到的限制：

| 遇到的限制 | 加入的机制 | 对最小 Loop 的改变 |
| --- | --- | --- |
| 模型给出的名称和参数不能直接执行 | Tool、Policy、审批与 Sandbox | 行动先解析、校验和授权，再进入受控执行环境 |
| 进程退出会丢失 `messages` 与执行位置 | Session、Turn 与 Step | 内存数组变成可重放的追加式事件 |
| 每个项目有不同的操作方法 | Skill | 常驻简介，选中后再加载说明与资源 |
| 多阶段任务的当前进度只存在于对话中 | Plan | 工作路径成为可更新、可恢复的显式状态 |
| 新输入只能等待整次运行结束 | Inbox、Steer 与 Follow-up | 输入在 Step 或 Turn 边界持久交付 |
| 长操作占住前台，固定流程反复重写 | Job 与 Workflow | 后台工作可查询，重复拓扑由程序执行 |
| 完成条件跨越一次连续执行 | Goal、Completion Policy 与 Continuation Policy | 调度器依据持久条件结束或开始 Turn |
| 项目约定、跨会话经验和大型结果需要长期复用 | 作用域文档、Memory、可选 Retriever 与 Artifact | 请求只装入当前需要且可重建的内容 |
| 独立调查可以并行推进 | Subagent | 子任务获得独立 Session，并通过 Inbox 与父级交换任务和结果 |
| 请求来源增多并逼近上下文窗口 | Context Assembly 与 Compaction | 每次请求按预算组装，旧 Step 由 Checkpoint 投影 |
| 模型提供方的事件和失败语义不同 | Model Adapter 与 Attempt | 请求快照、重试、用量和结束原因得到统一记录 |
| 运行会中断，客户端也会断线重连 | 恢复、事件投影与回放测试 | 调度器从合法日志前缀继续同一项工作 |

最小 Agent 的核心反馈关系保持不变：模型选择行动，运行时执行行动，环境结果返回模型。通用 Agent 在这个循环外增加持久事实、执行约束、任务调度和恢复规则，使同一项工作能够跨越模型请求、用户介入、进程中断和上下文窗口继续推进。

因此，本文最终构建出的 Agent 可以定义为：

> Agent 是由模型选择行动、由运行时约束和执行行动，并用持久事件连接观察、任务进度与后续运行的反馈循环。
