# 文章分类总表（Category Taxonomy）

本表是本站分类的**唯一来源**。写新文章时只从本表里选，不要临时发明新分类；
需要新增分类时，先在这里登记、再写进文章 front-matter，避免同一含义出现多个分类名。

## 分类树

```text
harness
  01-architecture   架构与职责：harness 由哪些部分组成、各自边界在哪
  02-build          构建与实现：工具、沙箱、状态、循环的具体做法
  03-governance     治理与约束：权限、审批、证据、审计
  04-optimization   优化与调优：成本、重试、失败复盘带来的系统改进
  05-case-study     案例与拆解：具体项目 / 产品的实现分析

model
  微调               在已有模型上继续训练，让它适配新任务
```

## 分类清单

| 一级分类 | 二级分类 | front-matter 写法 | 用途 |
| --- | --- | --- | --- |
| harness | 01-architecture | `- [harness, 01-architecture]` | 架构与职责划分 |
| harness | 02-build | `- [harness, 02-build]` | 构建、工具、状态、循环 |
| harness | 03-governance | `- [harness, 03-governance]` | 权限、审批、证据、审计 |
| harness | 04-optimization | `- [harness, 04-optimization]` | 成本、重试、改进复盘 |
| harness | 05-case-study | `- [harness, 05-case-study]` | 具体案例拆解 |
| model | 微调 | `- [model, 微调]` | 在已有模型上继续训练 |

### 层级写法（实测结论，别踩坑）

Hexo 的层级用**数组嵌套**表示，不是斜杠：

| 写法 | 实际生成 | 结论 |
| --- | --- | --- |
| `categories: [harness, harness/01-architecture]` | `/categories/harness/harness-01-architecture/` | ❌ 多出一层，子分类名被拼进 slug |
| `categories: [harness/01-architecture]` | `/categories/harness-01-architecture/` | ❌ 扁平成一个分类 |
| `categories: [[harness, 01-architecture]]` | `/categories/harness/` + `/categories/harness/01-architecture/` | ✅ 正确的父子结构 |

只有一个分类时用普通列表：

```yaml
categories:
  - 随笔
```

> 二级分类名里**不要带 `/`**（如 `训练/微调`），斜杠会被当成层级。
> `model` 的子分类定为单层叶子 **`微调`**（2026-11-06 确认），写 `- [model, 微调]`。
> 若以后出现从零训练的内容，再单独登记 `训练`，不要提前拆。

## 已归类文章

| 文章 | 分类 |
| --- | --- |
| harness 工程：如何构建稳定可靠的 AI agent | harness → 01-architecture |
| 教Laya玩贪吃蛇：决策模型的监督微调【原理篇】 | model → 微调 |
| 你的 AI 编码智能体，需要一个管理者 | harness → 02-build |

## 新文章 front-matter 模板

```yaml
---
title: 文章标题
date: 2026-11-06 10:00:00
categories:
  - [harness, 02-build]   # 一级 + 二级，父子结构
  - [model, 微调]          # 也可以挂多个分类
---
```

## 约定

- 分类名一旦有文章使用就不要再改名；确需改名时，同步改掉所有引用它的文章，否则会裂成两个分类。
- 分类用于**归档定位**（一篇一个主题），横向关键词放 `tags`。
- 新增 / 调整分类后，同步更新本文件的分类树和「已归类文章」表。
