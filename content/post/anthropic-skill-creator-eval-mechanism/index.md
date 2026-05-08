---
title: "Anthropic Skill-Creator 检验机制研究总结"
description: "深入解析 Anthropic 如何为 Claude Code Skills 建立自动化检验/评估体系，包括四路并行子 Agent 架构和三个核心检验维度。"
date: 2026-05-08
lastmod: 2026-05-08
categories:
    - 技术文章
tags:
    - AI
    - Agents
    - 架构设计
    - Anthropic
draft: false
cover: ./cover.jpg
---

# Anthropic Skill-Creator 检验机制研究总结

> 研究日期：2026-04-27
> 核心主题：Anthropic 如何为 Claude Code Skills 建立自动化检验/评估体系

---

## 一、背景与动机

### 1.1 为什么需要检验机制

ETH Zurich 研究表明：
- 开发者手写的 context 文件：任务完成率仅 +4%
- LLM 自动生成的 context 文件：性能反而 -3%
- 两种情况下计算成本均增加 >20%

**核心结论：未经检验的 context 文件会积累冗余甚至误导性的指令，验证是必须的。**

> "The answer isn't less context. It's tested context."

### 1.2 Skill 的本质

Anthropic 工程师明确指出：
- Skill **不是**单纯的 Markdown 文件
- Skill 是**文件夹**——包含脚本、资产、数据、动态 hooks
- Claude Code 生产环境中运行着**数百个** skill
- 最有效的 skill 清晰对齐单一类别，模糊分类会降低效果

---

## 二、Skill-Creator 检验机制架构

### 2.1 新的工作流程

Anthropic 更新了 skill-creator，引入了完整的评估流水线：

```
Create（创建） → Eval（评估） → Improve（改进） → Benchmark（基准测试）
```

### 2.2 四路并行子 Agent 架构

评估过程由 4 个并行子 Agent 协同完成：

| Agent | 职责 |
|-------|------|
| **Executor（执行器）** | 将 skill 应用于评估 prompt，生成实际输出 |
| **Grader（评分器）** | 对照预定义的期望标准评估输出质量 |
| **Comparator（比较器）** | 在 skill 版本间执行盲测 A/B 对比 |
| **Analyzer（分析器）** | 发现聚合统计数据遗漏的隐藏模式 |

### 2.3 评估用例结构

每个测试用例包含一个真实场景 prompt 和一组可验证的断言：

```json
{
  "eval_id": 2,
  "eval_name": "api-handler",
  "prompt": "Review this Express handler for me — it processes orders. Any issues?",
  "assertions": [
    {"id": "no-input-validation", "text": "标记出 req.body 未经验证直接使用", "type": "quality"},
    {"id": "foreach-async-inventory", "text": "标记 forEach 中使用 async 回调（未 await）", "type": "quality"},
    {"id": "structured-output", "text": "评审使用 severity 分级（critical/warning/suggestion）", "type": "format"}
  ]
}
```

### 2.4 关键基准洞察

- 如果 **有 skill** 和 **无 skill** 都得 100%，说明测试用例太简单
- **应对**：加大断言难度，或将 skill 聚焦于 base model 真正薄弱的领域

---

## 三、检验的三个核心维度

### 3.1 触发检验（Triggering）

检验 skill 是否在正确的场景被激活：

| 测试类型 | 目标 |
|---------|------|
| 显式请求 | 确保直接提到 skill 功能时触发 |
| 改写请求 | 确保用户用不同表述时也能触发 |
| 无关话题 | 确保不会在不相关场景误触发 |

**问题信号与修复：**
- **触发不足（Undertriggering）**：在 description 中补充关键词和技术术语
- **过度触发（Overtriggering）**：添加负向触发条件（"Do NOT use for..."），收窄范围

### 3.2 功能检验（Functional）

检验 skill 执行结果的质量：

- 输出是否符合预期格式
- API 调用是否成功
- 错误处理是否正确
- 边界情况是否覆盖

### 3.3 性能检验（Performance）

对比有无 skill 的基线差异：

| 指标 | 目标 | 测量方式 |
|------|------|---------|
| 触发率 | ≥90% | 运行 10-20 个测试 prompt，追踪自动加载率 |
| 效率 | 更少 tool calls 和 tokens | 对比基线与启用 skill 的消耗 |
| 可靠性 | 0 次 API 调用失败 | 监控 MCP 日志中的重试/错误 |
| 用户体验 | 零追问式 next-step prompt | 运行 3-5 次检验结构一致性 |

---

## 四、Skill 结构的强制检验规则

### 4.1 开发前（Before Start）

- [ ] 已定义 2-3 个具体用例
- [ ] 已识别所需工具（内置或 MCP）
- [ ] 已阅读官方指南和示例 skill
- [ ] 已规划文件夹结构

### 4.2 开发中（During Development）

- [ ] 文件夹使用 kebab-case 命名
- [ ] SKILL.md 文件存在（精确拼写，区分大小写）
- [ ] YAML frontmatter 使用 `---` 分隔符
- [ ] name 字段：kebab-case，无空格，无大写
- [ ] description 包含 WHAT 和 WHEN（触发条件）
- [ ] **禁止**在 frontmatter 中使用 XML 尖括号 `< >`（防止 prompt 注入）
- [ ] description < 1024 字符
- [ ] 指令清晰可执行
- [ ] 包含错误处理
- [ ] 提供示例
- [ ] references 正确关联

### 4.3 发布前（Before Upload）

- [ ] 在显式任务上测试触发
- [ ] 在改写请求上测试触发
- [ ] 验证不触发于无关话题
- [ ] 功能测试通过
- [ ] 工具集成正常（如适用）
- [ ] 压缩为 .zip 文件

### 4.4 发布后（After Upload）

- [ ] 在真实对话中测试
- [ ] 监控触发不足/过度触发
- [ ] 收集用户反馈
- [ ] 迭代 description 和指令
- [ ] 更新 metadata 中的版本号

---

## 五、三级渐进式披露系统

Claude 加载 skill 的三层机制本身就是检验体系的一部分：

| 层级          | 内容                         | 加载时机                                 |
| ----------- | -------------------------- | ------------------------------------ |
| **Level 1** | YAML frontmatter           | 始终加载在 system prompt 中，告诉 Claude 何时使用 |
| **Level 2** | SKILL.md 正文                | 判断相关时加载，包含完整指令                       |
| **Level 3** | 链接文件（references/, assets/） | 按需发现加载，最小化 token 消耗                  |

**关键原则**：description 字段**不是给人看的摘要**，而是**给模型看的触发条件集合**。

---

## 六、9 类 Skill 分类体系

Anthropic 工程师将 skill 分为 9 类，每个 skill 应清晰对齐单一类别：

| 类别           | 用途                      | 示例                                    |
| ------------ | ----------------------- | ------------------------------------- |
| 1. 库与 API 参考 | 解释正确用法，含示例和已知陷阱         | billing-lib, frontend-design          |
| 2. 产品验证      | 使用 Playwright/tmux 测试行为 | signup-flow-driver, checkout-verifier |
| 3. 数据检索与分析   | 连接数据/监控栈                | funnel-query, cohort-compare          |
| 4. 业务流程与自动化  | 自动化重复工作流                | standup-post, create-ticket           |
| 5. 代码模板与脚手架  | 用自然语言需求生成样板代码           | new-workflow, new-migration           |
| 6. 代码质量与审查   | 强制执行标准，辅助审查             | adversarial-review, code-style        |
| 7. CI/CD 与部署 | 构建、发布、部署                | babysit-pr, deploy-service            |
| 8. Runbooks  | 跨工具调查症状并生成报告            | service-debugging, oncall-runner      |
| 9. 基础设施运维    | 日常维护，含安全保护              | resource-orphans, cost-investigation  |

---

## 七、生产级检验要求

本地检验只是起点，生产就绪需要：

1. **云端规模执行**：跨多个配置运行数十个场景，无本地 session 限制
2. **CI/CD 集成**：发布时自动运行检验，在模型更新破坏 skill 前捕获回归
3. **版本锁定结果**：将分数绑定到特定 skill 版本（v1.2.0 vs v1.1.0）
4. **跨模型/Agent 对比**：在 Claude、GPT、Gemini 等多模型上验证性能
5. **公开可见性**：在注册表上展示评估分数，用户可在安装前验证质量

### 注册表实际数据示例

| Skill | 总分 | 提升倍数 | 关键指标 |
|-------|------|---------|---------|
| Cisco 软件安全 | 84% | 1.78x | 覆盖 23 条安全代码生成规则 |
| ElevenLabs TTS | 93% | 1.32x | 评审分 94%；Agent 正确使用 API 概率 +32% |
| Hugging Face Tool-Builder | 81% | 1.63x | 测量正确 API 使用率 |

---

## 八、Hooks 检验机制

Claude Code 支持动态 hooks，可在 skill 被调用时激活：

### 8.1 PreToolUse Hook

在工具使用前拦截，用于安全检查：

- `/careful` 模式：拦截 `rm -rf`、`DROP TABLE`、`force-push`、`kubectl delete` 等危险操作
- `/freeze` 模式：禁止在指定目录外编辑（安全调试）

### 8.2 使用率追踪

通过 PreToolUse hook 内部记录 skill 使用日志：
- 发现高频热门 skill
- 识别触发率低于预期的 skill

---

## 九、核心最佳实践

### 9.1 内容编写

- **不要写显而易见的内容**：聚焦推动 Claude 超越默认知识的信息
- **收集 "Gotchas" 部分**：最有价值的部分，记录 Claude 实际犯过的错误
- **避免僵化规则**：过度具体的指令会适得其反，提供必要上下文但保留灵活性
- **使用文件系统做渐进式披露**：将详细文档放在 references/，输出模板放在 assets/

### 9.2 Skill 组合

- Skill 可以按名称引用其他 skill
- 模型会自动调用已安装的被引用 skill
- 示例：CSV 生成 skill 可调用文件上传 skill 完成完整工作流

### 9.3 MCP 与 Skill 的关系

> "MCP 提供专业厨房：工具、食材、设备。Skill 提供菜谱：一步步指导如何创造有价值的东西。"

- **MCP** = 连接性与工具访问
- **Skill** = 知识层与工作流指导

---

## 十、行业影响与趋势

### 10.1 "Skills as Software" 范式

Anthropic 将测试基础设施嵌入 skill-creator，确认 skill 需要：
- 版本管理
- 自动化测试
- 分发机制
- 生命周期管理

### 10.2 质量门槛提升

开发者将像筛选经过测试的 npm 包一样，筛选经过评估的 skill。

### 10.3 创建者自检清单

- [ ] 我的 skill 有自动化测试吗？
- [ ] 有明确的、真实的评估场景吗？
- [ ] 有检测 skill 破坏或回归的机制吗？

---

## 十一、关键资源

| 资源 | 链接 |
|------|------|
| Anthropic skill-creator GitHub | https://github.com/anthropics/skills/tree/main/skills/skill-creator |
| Anthropic 官方 Skill 构建指南 | https://resources.anthropic.com/hubfs/The-Complete-Guide-to-Building-Skill-for-Claude.pdf |
| Anthropic 工程师终极指南 | https://medium.com/@tort_mario/skills-for-claude-code-the-ultimate-guide-from-an-anthropic-engineer-bcd66faaa2d6 |
| Tessl Skill 注册表 | https://tessl.io/registry |
| Tessl 评估文档 | https://docs.tessl.io/evaluate/evaluate-skill-quality-using-scenarios |
| Skill Evaluator 工具 | https://mcpmarket.com/tools/skills/skill-evaluator-1 |

---

## 十二、对我方工作的启示

1. **建立 skill 检验流水线**：为每个自建 skill 定义 eval 用例，在发布前运行
2. **Gotchas 文档化**：持续记录 COO 和 quant-chief 团队执行中的典型错误
3. **渐进式披露优化**：将大型文档拆分到 references/，减少 system prompt 负担
4. **跨模型验证**：在 qwen3.6-plus 等不同模型上测试 skill 兼容性
5. **使用率监控**：记录哪些 skill 被频繁使用，哪些从未触发，持续迭代
