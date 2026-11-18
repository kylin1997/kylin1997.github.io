---
title: 构建常驻 agent
date: 2026-11-18 13:35:00
description: 常驻 agent 的外围架构由事件收件箱、定时器、记忆、权限和发件箱五个核心构件组成，它们共同让短暂的 run 无限期循环，形成「常驻」的体验。
tags:
  - AI Agent
  - 翻译
categories:
  - [harness, 01-architecture]
---

> 原文标题：Building always-on agents
> 作者：William Hu
> 发布日期：2026 年 11 月 13 日
> 原文链接：<https://claude.com/resources/articles/building-always-on-agents>

常驻 agent 会无限期运行，在众多 session 中决定如何行动、何时行动。本指南梳理它们的核心构件，以及这些构件如何协同工作。

过去几个月里，我们看到了常驻 agent 的再度兴起。

常驻 agent 是有状态且主动的：它们跨 session 管理 context，并在没有直接监督的情况下采取行动。在底层，它们和其他 agent 运行在同一套[核心循环](https://www.anthropic.com/research/building-effective-agents)之上。区别在于外围架构，它由一些构件组成，这些构件控制 agent 何时被唤醒、两次 run 之间有哪些内容得以延续，以及 agent 的工作如何送达用户。

这一再度兴起在很大程度上源于近期的模型进步，这些进步让常驻 agent 的运行更加切实可行。首先，最新的模型判断力更强：在理解用户意图、判断何时该主动联系用户方面更加可靠。其次，智能正变得更便宜，长周期任务从中受益最大。例如，在[一项长周期编码基准测试](https://platform.claude.com/docs/en/about-claude/models/optimizing-for-cost-and-intelligence)中，Opus 5.5 每解决一个任务的成本比 Opus 5 低 78%，而在典型工作负载下这一降幅为 40%。

客户已开始询问我们，如何把这种体验集成到他们自己的产品中。一些公司希望让现有的 agent 更具主动性，另一些则希望从零开始构建。

本文将拆解常驻 agent 背后的构件，并梳理它们如何协同工作。

## 常驻 agent 背后的循环

常驻 agent 并非真正“常驻”。这只是对用户体验的一种描述。

实际上，agent 大部分时间都在休眠，周期性地醒来工作。每一段工作称为一次 **run**：agent 收到一个事件，完成任务，然后保存关于所发生情况的信息。run 在一个 **session** 内部进行，session 就是消息与工具调用的记录。run 一旦结束，agent 便回到休眠状态，如此循环往复、无休无止。

![Animated diagram of an always-on agent’s timeline: long stretches asleep with no tokens spent, broken by brief runs. Each run has four steps: an event arrives, a session starts, the agent loop runs, and the run writes back to stored state that carries over between runs.](assets/always-on-agents/timeline.gif)

## 核心构件

常驻 agent 的外围架构包含五个核心构件：事件收件箱、定时器、记忆、权限和发件箱。下面逐一来看。

### 事件收件箱

大多数 AI agents 只有在用户发送消息时才会运行。相比之下，常驻 agent 可以由多种类型的事件触发，包括 webhooks、定时任务和后台作业。

一个看似诱人的做法是让每个事件来源直接调用 agent，但这有几种失败模式。例如，两个同时到达的事件可能启动相互竞争的 run，而一次崩溃可能让某个事件彻底丢失。

这正是事件收件箱的用武之地。每个来源不是调用 agent，而是把一个事件发送到收件箱，由 worker 从中取走。

最简单的版本包含三个组件：

- 一个持久化的事件列表，通常是一张数据库表；
- 一个 handler（通常是 HTTP 端点），负责把事件写入该列表；以及
- 一个 worker（通常是后台服务），负责从该列表读取。

收件箱应当跟踪每个事件的状态，并可根据来源提供的唯一标识符对事件去重。关系型数据库很适合做收件箱，每个事件都作为新的一行追加进去。我们建议从每个用户一个收件箱开始，但如果需要更高的并行度，可以进一步拆分。

![Diagram of the event inbox. A handler (an HTTP endpoint) writes each event to the inbox, a database table that tracks status and drops duplicates, and wakes the worker. The worker reads what’s pending, starts the run, waits for it to finish, and marks the events done.](assets/always-on-agents/event-inbox.png)

当新事件到来时，handler 将其写入收件箱并唤醒 worker。worker 读取所有待处理内容并启动一次 run。延续此前工作的事件（例如用户的回复）会恢复已有的 session，其他一切事件则启动新的 session。

随后，worker 把事件交给 agent 循环，等待 run 结束，并把相应事件标记为完成。实现细节可能相当棘手，以下是几条最佳实践：

- **handler 应尽快确认事件**。某些来源（例如 webhook 提供方）会把慢响应视为失败。handler 仍可运行轻量级检查，但任何更深入的验证（例如检查 prompt injection）都应在 run 期间完成。
- **收件箱应是唯一事实来源**。worker 被唤醒时，应首先从收件箱拉取最新事件。这样，如果 worker 在 run 中途挂掉，另一个 worker 可以接手同一事件并重试。
- **一个收件箱同一时间只应有一个活跃的 worker**。这可以避免两次 run 重复劳动或相互覆盖。当新事件在 run 中途到达时，你可以把它推给下一次 run、并入当前 run，或者中断并重启。
- **worker 应把待处理事件批量合并**。例如，如果用户连续发送多条消息，worker 可以把它们合并成一次 run，这样 agent 只需回复一次。

### 定时器

定时器与事件收件箱一起，让常驻 agent 能够主动作为。定时器通常分为两类：follow-up 或 schedule。follow-up 是 agent 为自己设定的截止时间，例如稍后确认一条回复。schedule 则是按已知节奏重复发生的任务，例如每日简报。

定时器存放在自己的持久化存储中，通常是另一张数据库表，与收件箱分开。收件箱只应保存已经可以处理的事件。follow-up 和 schedule 可以采用相似的结构，但 schedule 还需要跟踪它的重复规律。

![Diagram of timers. A run sets a timer in a database table, either a follow-up set by the agent or a schedule set by the user. A scheduler finds what’s due and adds an event to the event inbox, which starts the next run.](assets/always-on-agents/timers.png)

定时器本身不会启动 run。相反，scheduler 通常是一个后台服务，会以固定间隔轮询该存储，触发所有到期的定时器。每一轮，scheduler 会：

- 扫描存储，找出到期的定时器。
- 根据类别更新每个定时器。follow-up 应被标记为完成。schedule 则应把到期时间更新到下一次发生的时间。
- 向收件箱添加一个事件并唤醒 worker。

对于 scheduler，你可以自建服务，也可以使用持久化执行引擎。

### 记忆

常驻 agent 要在许多彼此不相连的 run 中工作，并依靠记忆来保持连续性。

与事件收件箱和定时器类似，记忆也应有自己的存储。在我们的产品中，例如 Claude Code 的 [auto memory](https://code.claude.com/docs/en/memory) 和 Messages API 的 [memory tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)，它是以文件目录的形式实现的。对常驻 agent 而言，这些目录应映射到持久化存储，通常是数据库或对象存储。这样，记忆就能比 worker 存活得更久。

一条经验法则是：记忆只应保存 agent 无法再次查到的内容。如果某样东西已经存在于其他系统中（例如问题追踪器），你应当保存一个指向它的指针。以下是常驻 agent 应存储的记忆类型细分：

- **用户指令：**用户关于 agent 行为的任何指令都应保存在单独的文件中，并在每次 run 开始时加载。用户不会总在旁边指导 agent，因此这些指令必须在 run 之间延续。
- **用户笔记：**agent 应保存它了解到的关于用户的信息，例如用户的角色、偏好或用户给出的纠正意见。你可以先用一个文件，随着笔记增多再拆成多个小文件。在后一种情况下，每次 run 开始时只加载一个索引，但当 agent 需要更多细节时，可以把各个文件拉入 context。
- **过往消息：**agent 应记录它已经告诉过用户的内容，避免重复。它可以把这些内容存放在单独的文件中并在每次 run 开始时加载，也可以让 worker 加载 agent 最近与用户的几条消息。
- **任务进度：**对于每个进行中的任务，agent 应保存一份进度文件，写明已完成和未完成的部分。这样，任何新的 run 都可以读取该文件，从上一次 run 停下的地方继续。与用户笔记类似，每次 run 开始时只加载一个索引，但当 agent 需要更多细节时，可以把各个文件拉入 context。

下图展示了一个示例配置。

![Diagram of an example memory setup as a directory of files. instructions.md, index.md, and reported.md are loaded at the start of every run, while the notes/ and progress/ folders are read when needed. The agent writes to memory during the run.](assets/always-on-agents/memory-setup.png)

以上描述的都是在 run 期间保存的记忆。你还可以加入 dreaming，它回顾过往 session 来整理记忆。dreaming 可以替换过时的条目，并发现那些只有在多次 run 中才会显现的规律。如果你想了解更多，我们的[关于 agent memory 的文章](https://www.anthropic.com/research/components-of-agent-memory)有更详细的介绍。

#### 在 context window 内部

在做任何工作之前，agent 会加载系统提示词、工具定义和记忆文件。这部分初始的 context“开销”会被计入每一次 run，因此我们建议保持精简，并使用 [prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching) 来降低成本。

新的 session 只带着这部分 context 开销和触发 run 的事件开始。相比之下，已有的 session 会带回它此前各次 run 中的消息和工具调用。

如果 session 变得过长，你可以[压缩较早的消息](https://platform.claude.com/docs/en/build-with-claude/compaction)，或者开启一个新的 session。两种做法都会损失细节，因此任何需要长期保留的内容都应在过程中保存到记忆里。

![Animated chart of context used over time. A fixed band for the system prompt, tools, and memory is present on every run. An existing session brings back its whole transcript and grows until compaction turns the older transcript into a summary.](assets/always-on-agents/context-window.gif)

### 权限

权限是常驻 agent 风险最大的地方。一次 run 持有用户的访问权限，并且按设计是在无人值守的情况下运行的。对于每一次工具调用，agent 都需要决定是自行继续、阻止它，还是等待批准。

权限所需的组件与其他 agent 使用的组件相同：

- **凭证存储：**常驻 agent 通常需要敏感的用户凭证（如密码或访问令牌）才能发挥作用。[推荐的做法](https://code.claude.com/docs/en/agent-sdk/secure-deployment#credential-management)是在 agent 环境之外设置一个代理，由它向对外请求添加凭证。agent 发出不带凭证的请求，代理在转发前注入真实凭证。
- **auto mode：**常驻 agent 应当能够在没有用户持续输入的情况下工作。我们已经用 Claude Code 中的 [auto mode](https://www.anthropic.com/research/claude-code-auto-mode) 解决过这个问题的一个版本。auto mode 使用分类器审查风险较高的操作，并阻止分类器判定为危险的操作，以便 Claude 找到更安全的做法。不过，对最重要的决策，用户复核仍然是必要的。
- **deferred tool calls：**在 Claude Code 中，[hook 可以延迟一次工具调用](https://code.claude.com/docs/en/hooks#defer-a-tool-call-for-later)，并等待用户的额外输入。同样的模式也适用于常驻 agent——在采取某些行动之前，它可能需要澄清或批准。当一次调用被延迟时，run 结束，session 在调用中途被保存。请求会展示给用户（例如通过用 [MCP Apps](https://modelcontextprotocol.io/extensions/apps/overview) 构建的交互式卡片），用户的回答会作为收件箱中的一个事件返回。该事件启动一次新的 run，用用户的输入恢复保存的 session。在延迟调用等待回复期间，worker 可以处理其他事件，因此 agent 不会被阻塞。

### 发件箱

对大多数 AI agents 来说，响应是在生成过程中流式传给用户的，这样用户可以跟随 Claude 推理问题、调用工具并写出答案的全过程。

常驻 agent 则相反，它们的大部分工作在后台完成。用户得到的是结果和少量细节，而不是完整的逐步分解。

![Diagram of the outbox. A run ends with nothing to say, a send tool call, or a deferred tool call that asks the user. Messages go to the outbox, a database table, and a dispatcher delivers them to the user. A deferred request can go through the outbox or be shown in your own interface.](assets/always-on-agents/outbox.png)

一次 run 可以通过三种方式结束：

- agent 向发件箱发送一条消息，
- agent 判定无需发送任何内容，或者
- agent 使用 deferred tool call 请求许可或额外输入。

大多数常驻 agent 用一个“send”工具把消息直接送达用户就够了。只有当你想控制 agent 消息的发送节奏，或让 agent 能够从错误中恢复时，才需要一个正式的发件箱。

发件箱是一张独立的外发消息表，类似于事件收件箱。还有一个 dispatcher（通常是后台服务），角色类似于 worker。dispatcher 读取待发送消息，决定发送哪些，并把它们送给用户。来自 deferred tool call 的请求既可以通过发件箱发送，也可以直接显示在界面上。

除了发送之外，调节 agent 发送多少消息也很重要。发得太多的 agent 会变得吵闹，而发得太少的 agent 会漏掉用户需要知道的事情。我们建议构建评估（evals）来找到合适的平衡点。在实践中，更好的做法是让 agent 一开始保持安静，然后随着时间推移逐渐赢得用户更多的注意力。

## 一切如何协同

以下是五个构件如何协同工作：

![Animated diagram of how the five primitives fit together. Timers and the event inbox start a run, the run works with memory and permissions inside a session, and the outbox delivers messages to the user, whose answer comes back as an event.](assets/always-on-agents/primitives.gif)

每次 run 都从收件箱中的一个事件开始，无论是用户消息、webhook 还是定时器。当 worker 取走一个事件时，它会启动一次 run，并在新的或已有的 session 中唤醒 agent。agent 读取记忆以获得基线，在自己的权限范围内采取行动，然后用学到的任何内容更新记忆。run 结束后，agent 向发件箱写入一行，dispatcher 把消息送达用户。

这些共同形成一个循环。run 可以设定一个定时器，在稍后再次唤醒 agent；而当用户回复一条消息时，其回答会作为收件箱中的一个事件返回。每次 run 都很短暂，但这个循环会无限期持续下去，正是这一点让 agent 感觉像是常驻。

## 致谢

*本文由 William Hu 撰写。特别感谢 Rodrigo Olivares 和 Michael Segner 的贡献。*
