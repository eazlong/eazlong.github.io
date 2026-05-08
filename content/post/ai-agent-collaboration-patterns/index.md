---
title: "AI Agents 协作模式：Claude 官方的 5 种多 Agent 协调方案"
description: "深入解析 Anthropic 官方定义的 5 种多 Agent 协作模式，包括 Generator-Verifier、Orchestrator-Subagent、Agent Teams、Message Bus 和 Shared State，以及它们的适用场景与局限性。"
date: 2026-05-08
lastmod: 2026-05-08
categories:
    - 技术文章
tags:
    - AI
    - Agents
    - 架构设计
    - 多智能体
draft: false
---

> 本文基于 Anthropic 官方博客文章 [Multi-agent coordination patterns: Five approaches and when to use them](https://claude.com/blog/multi-agent-coordination-patterns)（Cara Phillips, Eugene Yan 等，2026-04-10）整理。

随着多 Agent 系统在生产环境中的应用越来越广泛，团队面临一个关键问题：**应该选择哪种协调模式？**

Anthropic 在其官方博客中定义了 5 种核心的多 Agent 协调模式。核心原则是：**从最简单的模式开始，观察哪里遇到困难，再逐步演进。**

## 模式 1: Generator-Verifier（生成-验证）

**适用场景:** 质量关键的输出，有明确的评估标准。

### 工作原理

Generator 接收任务并产生初始输出，传给 Verifier 评估。Verifier 检查输出是否符合标准，要么接受为完成，要么带着具体反馈拒绝。被拒绝的反馈传回 Generator 进行修订。循环持续直到 Verifier 接受或达到最大迭代次数。

```
Task → Generator → Output → Verifier
                              ↕ (feedback loop, until accept or max iterations)
```

### 典型案例

客服邮件回复系统：Generator 基于产品文档和工单上下文生成回复，Verifier 检查准确性（对照知识库）、语气（品牌指南）、完整性（是否覆盖所有问题）。失败时返回具体问题，如“功能归属到错误的定价层级”。

其他场景：代码生成（一个 Agent 写代码，另一个写并运行测试）、事实核查、基于量规的评分、合规验证。

### 关键设计要点

- **验证标准必须明确** — 只说“检查好不好”的 Verifier 会橡皮图章式通过
- **生成和验证必须是可分离的技能** — 如果评估创意和生成创意一样难，Verifier 不可靠
- **防止无限循环** — 设置最大迭代次数，配合降级策略（升级给人类、返回最佳尝试并标注 caveat）

### 局限性

- Verifier 的好坏完全取决于评估标准的质量
- 迭代循环可能卡住（Generator 无法解决 Verifier 的反馈时振荡不收敛）

---

## 模式 2: Orchestrator-Subagent（编排-子代理）

**适用场景:** 任务分解清晰，子任务有边界。

### 工作原理

一个 Lead Agent 作为团队领导，负责规划工作、分配任务、综合结果。Subagents 处理具体职责并回报结果。

```
Task → Orchestrator → [Subagent A, Subagent B, Subagent C] → Synthesis → Result
```

### 典型案例

自动化代码审查系统：PR 到达后，Orchestrator 将安全检查、测试覆盖率、代码风格、架构一致性分别分发给专门的 subagent，收集结果后合成统一的审查报告。

Claude Code 也使用此模式：主 agent 自己写代码、编辑文件、运行命令，在需要搜索大型代码库或调查独立问题时后台分发 subagent。每个 subagent 在自己的 context window 中操作，返回精炼的发现。

### 关键设计要点

- 子任务之间**独立性高**时效果最好
- Orchestrator 保持对整体目标的连贯视图，subagents 专注具体职责
- Subagent 是**一次性的** — 完成任务即终止

### 局限性

- **信息瓶颈** — Subagent 发现的重要信息必须经过 Orchestrator 中转，多轮传递后关键细节容易丢失
- **顺序执行限制吞吐** — 除非显式并行化，subagents 一个接一个运行，有多 agent 的 token 成本但没有速度优势

---

## 模式 3: Agent Teams（Agent 团队）

**适用场景:** 并行的、独立的、长期运行的子任务。

### 工作原理

Coordinator 生成多个 worker agents 作为独立进程。Teammates 从共享队列认领任务，自主多步工作，完成后发出信号。

与 Orchestrator-Subagent 的关键区别在于 **worker 的持久性**：
- Orchestrator-subagent: subagent 完成一个有界子任务后就终止
- Agent teams: teammates 在多个任务间存活，积累上下文和领域专长，持续提升性能

### 典型案例

大型代码库框架迁移：每个 teammate 负责一个服务的独立迁移 — 依赖更新、代码修改、测试修复、验证。Coordinator 分配服务给 teammates，收集完成的迁移并在整个系统上运行集成测试。

### 关键设计要点

- **独立性是核心要求** — teammates 之间不能共享中间发现
- Coordinator 只分配工作和收集结果，不在任务间重置 worker
- Teammates 积累领域上下文，性能随时间提升

### 局限性

- **独立性被打破时会冲突** — 如果 teammate A 的工作影响 teammate B，双方都不知道，输出可能冲突
- **完成检测更难** — teammates 自主工作不同时长，coordinator 必须处理部分完成
- **共享资源问题** — 多个 teammate 操作同一代码库/数据库/文件系统时可能编辑同一文件
- 需要仔细的任务分区和冲突解决机制

---

## 模式 4: Message Bus（消息总线）

**适用场景:** 事件驱动的流水线，agent 生态系统会持续增长。

### 工作原理

Agent 通过两个原语交互：**publish** 和 **subscribe**。Agent 订阅关心的主题，router 投递匹配的消息。新的 agent 可以开始接收相关工作而无需重新布线现有连接。

```
Event Source → [Router] → Subscribed Agents → Publish Results → [Router] → ...
```

### 典型案例

安全运营自动化系统：警报从多个来源到达，Triage Agent 按严重性和类型分类，将高危网络警报路由给网络调查 agent，凭证相关警报路由给身份分析 agent。每个调查 agent 可发布丰富请求由上下文收集 agent 完成。发现流向响应协调 agent 决定适当的行动。

### 关键设计要点

- 事件从一个阶段流到下一个阶段
- 可以随威胁类别演变添加新的 agent 类型
- Agent 可以独立开发和部署

### 局限性

- **追踪困难** — 一个警报触发跨 5 个 agent 的事件级联时，理解发生了什么需要仔细的日志记录和关联
- **调试困难** — 比跟踪 orchestrator 的顺序决策更难
- **路由准确性至关重要** — router 误分类或丢弃事件时系统静默失败（不处理但不崩溃）
- LLM-based router 提供语义灵活性但引入自己的失败模式

---

## 模式 5: Shared State（共享状态）

**适用场景:** 协作性工作，agents 基于彼此的发现继续构建。

### 工作原理

Agent 自主操作，直接读写共享数据库、文件系统或文档。**没有中心协调器**。Agent 检查存储中的相关信息，据此行动，将发现写回。

工作开始于初始化步骤向存储种子数据（问题或数据集），结束于终止条件满足：时间限制、收敛阈值、或指定 agent 判定存储已包含足够答案。

```
[Agent A] ↕
[Agent B] ←→ [Shared Store] ←→ [Agent C]
[Agent D] ↕
```

### 典型案例

研究综合系统：多个 agent 调查复杂问题的不同方面 — 一个探索学术文献，一个分析行业报告，一个检查专利文件，一个监控新闻报道。学术 agent 发现的关键研究者，其公司立即成为行业 agent 应该更密切调查的对象。

通过共享状态，发现直接进入存储。行业 agent 可以立即看到学术 agent 的发现，无需等待协调器路由信息。

### 关键设计要点

- **无单点故障** — 任一 agent 停止，其他 agent 继续读写
- Agent 的发现直接对所有人可见
- 共享存储成为不断演进的知识库

### 局限性

- **重复工作** — 没有显式协调，两个 agent 可能独立调查同一线索
- **结果不可预测** — agent 交互产生系统行为而非自上而下设计
- **反应式循环** — Agent A 写发现，Agent B 读并写跟进，Agent A 看到并回应，系统不断消耗 token 而不收敛
- **必须有终止条件** — 时间预算、收敛阈值（N 周期无新发现）、或指定 agent 判定完成

---

## 模式选择决策树

### Orchestrator-Subagent vs Agent Teams

**问题:** worker 需要维持上下文多久？

- 子任务短、聚焦、输出明确 → **Orchestrator-Subagent**
- 子任务受益于持续的、多步工作 → **Agent Teams**
- 需要跨调用保留状态 → **Agent Teams**

### Orchestrator-Subagent vs Message Bus

**问题:** 工作流结构有多可预测？

- 步骤序列预先已知 → **Orchestrator-Subagent**
- 工作流从事件中涌现，可能变化 → **Message Bus**
- Orchestrator 中条件逻辑越来越多 → 演进到 **Message Bus**

### Agent Teams vs Shared State

**问题:** agent 是否需要彼此的发现？

- agent 在不交互的独立分区上工作 → **Agent Teams**
- 工作是协作的，发现应实时流通 → **Shared State**
- teammate 需要互相通信而非只共享最终结果 → **Shared State**

### Message Bus vs Shared State

**问题:** 工作以离散事件流动还是累积成共享知识库？

- agent 对事件做出反应，流水线式处理 → **Message Bus**
- agent 随时间累积发现，反复回到存储 → **Shared State**
- 消除单点故障是优先事项 → **Shared State**（去中心化）
- agent 发布事件是为了分享发现而非触发动作 → **Shared State**

---

## 快速选择表

| 场景 | 推荐模式 |
|------|---------|
| 质量关键输出，明确评估标准 | Generator-Verifier |
| 清晰任务分解，有界子任务 | Orchestrator-Subagent |
| 并行工作负载，独立长期子任务 | Agent Teams |
| 事件驱动流水线，agent 生态增长 | Message Bus |
| 协作研究，agent 共享发现 | Shared State |
| 需要无单点故障 | Shared State |

## 起步建议

大多数用例推荐从 **Orchestrator-Subagent** 开始。它以最少的协调开销处理最广泛的问题。观察它在哪里遇到困难，然后朝着其他模式演进。

生产系统通常**组合使用多种模式**。常见混合：整体工作流用 Orchestrator-Subagent，协作密集的子任务用 Shared State；或事件路由用 Message Bus，每种事件的处理用 Agent Teams 风格的 worker。这些模式是积木，不是互斥选择。
