---
title: harness 工程：如何构建稳定可靠的 AI agent
date: 2026-08-29 13:00:03
tags:
  - AI Agent
  - harness
  - 翻译
categories:
  - [harness, 01-architecture]
---

![Harness Engineering 封面](assets/cover.zh.png)

agent 出问题时，大多数人的第一反应是修改提示词、然后换模型、然后加大 context window，

agent 还是会忘记之前的决策、还是会用错工具、还是会跳过验证、还是卡在同一个循环里。

问题并不总是出在智能本身，问题出在它周围的环境，这个环境就是 harness，而设计它，就是 harness 工程。

---

## 模型只是推理引擎

模型可以建议下一步动作

但它无法独自造出一个可靠的运行环境

harness 决定模型能看到什么、能碰到什么、哪些东西能跨会话留存、什么算证据，以及这次运行何时必须停下

```text
MODEL
推理并提出动作

HARNESS
选择 context
暴露工具
存储状态
强制执行权限
检查结果
记录轨迹
从失败中恢复
```

提示词是这个系统里的一个组件

模型也是其中一个组件

而产品，是周围每个组件协同运转后的结果

> **提示词工程改进的是指令**
>
> **harness 工程改进的是指令被执行的条件**

---

## 同一个模型，可以构建出截然不同的 agent

把同一个模型接入聊天界面，它会回答问题

让它在代码仓库中工作，并为它提供终端访问、测试、浏览器工具、项目记忆、相互隔离的 worktree 和评审循环，它能够交付软件

权重没有变

变的是 harness

OpenAI 在用 Codex 构建 agent 优先的代码库时，也描述过同一种转变

他们早期进展缓慢，是因为环境定义得不够充分，而不是模型缺少底层能力

他们的应对不是叫 agent 再努力一点

而是去追问缺的是哪种能力，把这项能力描述清楚，并使其可由系统机制强制落实

> **“环境定义不足”**
>
> **OpenAI，[Harness engineering: leveraging Codex in an agent-first world](https://openai.com/index/harness-engineering/)**

这就是核心思路

当 agent 反复失败时，别再在提示词里换形容词

去检查模型周围的系统

---

## 一个生产级 harness 有七项职责

![七项职责](assets/seven-jobs.zh.png)

---

### 1. 把请求变成一份契约

在 agent 动手之前，先把请求转换成一个有边界的对象

```json
{
  "goal": "ship the feature",
  "inputs": ["issue", "repository", "design"],
  "output": "reviewable pull request",
  "constraints": ["no schema changes", "preserve public API"],
  "done_when": ["tests pass", "visual check passes", "review passes"]
}
```

这份契约能防止任务被悄悄重新定义

没有它，agent 可能做完另一件事，却仍然宣布成功

---

### 2. 为 agent 提供项目知识导航

agent 需要项目知识

它们不需要把每一份文档都装进每个 context window

用一份简短的根目录指南，告诉 agent 该去哪里找

```text
AGENTS.md
  -> 架构导航图
  -> 测试导航图
  -> 产品规则
  -> 安全规则
  -> 任务专属指南
```

一张导航图能节省 context

一份巨型手册会消耗 context

把详细知识放在它所约束的代码、工具或工作流附近

只在当前任务需要时加载

---

### 3. 在正确的环境里暴露正确的工具

工具访问不是一排按钮

它是模型与真实世界之间的接口

每个工具都需要明确的目的、可预测的输出、显式的失败状态，以及一条权限边界

```text
读取文件       默认允许
运行测试        沙箱内允许
写入文件       工作区内允许
访问网络       按任务限定范围
部署           需要审批
删除数据       需要审批
```

好的工具能在模型推理前减少歧义，降低推理出错的可能

坏工具会逼着模型去猜发生了什么

---

### 4. 把记忆外化成持久状态

对话不是权威记录系统

把决策、产物、失败和未决风险存到 context window 之外

```json
{
  "task_id": "task_042",
  "current_step": "verify_ui",
  "artifacts": ["build.zip", "report.md", "screenshot.png"],
  "decisions": ["keep existing schema"],
  "failures": ["mobile overflow at 390px"],
  "pending": ["human approval"]
}
```

下一次会话应继承工作的状态，而不是对话的有损复述

这正是 agent 能扛过 context 重置、崩溃和交接的方式

---

### 5. 先加观测机制，再加自主性

agent 修正不了它观察不到的东西

测试、lint 工具、截图、日志、指标和 schema 校验器，为原本模糊的质量判断提供可观察的证据

```text
代码       -> 测试 + 类型检查 + lint
UI         -> 渲染 + 截图 + 目视检查
研究       -> 来源核查 + 矛盾核查
数据       -> schema + 范围 + 时效检查
```

模型生成产物

环境针对这个产物给出证据

harness 决定这些证据是否足以支持继续执行

---

### 6. 在模型之外强制执行权限

模型可以建议动作

harness 必须负责授权

```text
模型建议 -> 策略检查 -> 工具执行
```

当动作代价高昂、不可逆，或会影响到另一个人时，这种分离最重要

别让同一个概率系统既设计方案，又批准风险，还执行有副作用的操作

---

### 7. 记录执行轨迹，并在局部环节恢复

每次运行都应留下一条可读的轨迹

```text
请求
选中的 context
工具调用
状态变更
验证结果
重试
成本
最终产物
回滚点
```

> 没有轨迹，失败就成了谜，有了轨迹，失败就成为下一次 harness 改进的输入

---

## 指令应落实到基础设施中

大多数团队把重要规则写成散文式说明

agent 会读取这些规则

然后总有一天忽略其中一条

更强的做法，是把重要规则编码两次

先写成 agent 能理解的指引

再写成 agent 无法绕过的自动检查

```text
GUIDE
“UI 代码不能直接查询数据库”

CHECK
当 UI 导入 repository 层时，lint 失败
```

指引解释原因

检查守住边界

这样就把一次过去的失败变成了系统的永久改进

下一个 agent 不需要记得这次事故

harness 会替它记住

---

## 执行循环由 harness 管理

长时间运行的工作需要迭代

但“一直试到能跑为止”不是一个控制系统

一个有用的循环要有证据、有界重试、预算和升级路径

```javascript
for (let attempt = 1; attempt <= 3; attempt += 1) {
  const artifact = await build(state)
  const evidence = await verify(artifact)

  if (evidence.pass) return artifact

  state.failures.push(evidence.gap)
  state.repair = evidence.repair
}

return requestHumanReview(state)
```

模型应决定如何修补局部缺口

harness 应决定是否允许再试一次

Anthropic 在长运行 agent 方面的工作也得出过类似结论

结构化产物让工作能跨会话延续，而独立的评估者会给构建者具体反馈，而不是让构建者批准自己的工作

> **“找到尽可能简单的解决方案，只在需要时才增加复杂度”**
>
> **Anthropic，[Harness design for long-running application development](https://www.anthropic.com/engineering/harness-design-long-running-apps)**

---

## 失败应推动系统升级

![把失败经验落实到基础设施中](assets/failure-upgrades.zh.png)

多数人修补的是当前输出

harness 工程师修复的是这一类失败

```text
缺失 context   -> 加一张导航图或一条检索规则
工具错误        -> 改进工具描述或路由
输出糟糕        -> 加一个校验器或更强的契约
重复循环        -> 加一个重试上限和升级机制
不安全动作      -> 加一道权限闸门
决策丢失        -> 存入持久状态
未知失败        -> 加轨迹记录和证据采集
```

眼前的补丁修复一次运行

harness 的改动会改进之后的每一次运行

这就是复利优势

> **一个优秀的 harness 会把 agent 的错误转化为基础设施**

---

## 把大脑、双手和历史分开

当三个组件彼此分开时，可靠的 agent 更容易分析

```text
BRAIN
负责推理的模型

HANDS
负责行动的沙箱和工具

HISTORY
记录发生过什么，且只追加
```

沙箱挂掉时，历史仍然保留

模型更换时，工具和策略仍然可供检查

任务恢复时，新会话可以根据产物和轨迹重建状态

**Anthropic 的 Managed Agents 架构通过 session、harness 和沙箱，把这种分离显式化了**

[@claudeai 的嵌入帖](https://x.com/claudeai/status/2041927687460024721)

重要的不是供应商

而是架构

推理引擎不应该同时兼任文件系统、权限系统、记忆数据库和审计日志

---

## 给每次运行一份变更回执

agent 完成后，不要只保留最终输出

保存一份简明回执，说明输出是如何产生的

```json
{
  "context_sources": ["issue", "repo_map", "design_spec"],
  "policy_version": "v12",
  "model_route": "complex_coding",
  "tools_used": ["shell", "browser", "tests"],
  "tests": { "passed": 42, "failed": 0 },
  "human_corrections": 1,
  "retries": 2,
  "cost_usd": 3.84,
  "accepted_artifact": "pr_1842",
  "rollback_point": "commit_7f3a"
}

```

这样就能比较模型升级后的表现

让回归可以归因

让审计成为可能

也防止最终答案掩盖一个坏掉的过程

---

## 从能闭环的最小 harness 开始

harness 工程不是指在第一个任务之前先搭建平台

从能观察、验证、恢复的最小系统开始

```text
LEVEL 0
提示词 + 模型

LEVEL 1
项目指南 + 工具

LEVEL 2
结构化状态 + 测试 + 有界循环

LEVEL 3
权限 + 轨迹 + 恢复 + 人工闸门
```

只有任务确实需要这份复杂度时，才向上升级

一个短小的低风险任务，也许只需要一个提示词和一次评审

一个能编辑文件、访问网络、打开 pull request 的六小时编码任务，需要一个真正的 harness

harness 应该比它控制的故障面更小

---

## harness 工程检查清单

在把真实工作交给 agent 之前，先问：

```text
[ ] 是否在执行开始前定义了成功标准
[ ] agent 能否不加载所有内容就找到正确的项目知识
[ ] 每个工具是否都有清晰的契约和失败状态
[ ] 执行是否与生产系统隔离
[ ] 重要决策是否存储在对话之外
[ ] 每次有风险的转换是否有证据
[ ] 不可逆操作是否受审批保护
[ ] 每个循环是否都有重试上限和预算
[ ] 运行中断后能否恢复
[ ] 你能否解释每一次工具调用和状态变更
[ ] 失败是否会更新指引、测试、工具或策略
[ ] 最终产物能否回滚

```

如果好几个答案都是否，更强的模型不会让系统变得可靠

它只会让失败变得更昂贵

---

## 真正的转变

提示词工程告诉模型要做什么

context 工程决定模型看到什么

harness 工程构建模型身处其中并采取行动的世界

```text
PROMPT      -> 指令
CONTEXT     -> 工作视图
HARNESS     -> 操作系统
LOOP        -> 局部改进
GRAPH       -> 协调
```

下个月模型可能就变了

工具、测试、状态、策略和轨迹可以继续改进

这就是为什么持久优势正从提示词转向模型周围的系统

最好的构建者不会只问哪个模型最聪明

他们还会问，哪个环境能让这种智能变得可靠

这就是 harness 工程

---

> 原题：Harness Engineering: How to Build AI Agents That Don't Fall Apart
> 作者：rari (@0xwhrrari)
> 来源：https://x.com/0xwhrrari/status/2093685107534000560
> 发布时间：2026-08-29T13:00:03.000Z
