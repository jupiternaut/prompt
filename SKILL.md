---
name: prompt-forge
description: >
  Expert System Prompt generation engine with adversarial hardening.
  Takes vague, fuzzy, or incomplete descriptions of what an AI should do
  and produces complete, production-grade System Prompts with structured
  formatting, multimodal guardrails, few-shot examples with reasoning
  traces, and negative constraints. Uses a build-attack-fix cycle:
  Architect drafts the prompt, Red Teamer finds exploit vectors and
  failure modes, Refiner hardens against them.
  Trigger when: user asks to "write a system prompt", "create a prompt",
  "design a prompt", "make a prompt for GPT/Claude/Gemini", "prompt
  engineering", "build an AI persona", "create an AI assistant prompt",
  "write instructions for an LLM", "optimize my prompt", or any variant
  of "help me prompt". Also trigger for implicit needs like describing
  how they want an AI to behave without using the word "prompt".
  Chinese triggers: "帮我写提示词", "设计 system prompt", "做个 prompt",
  "写个系统提示", "AI 角色设定", "prompt 优化", "提示词工程",
  "写个角色扮演提示", "优化我的 prompt".
  Do NOT trigger for: creating Claude Code skills (use skill-architect),
  completing software requirement specs (use requirement-refiner),
  stress-testing decisions (use decision skill), or task decomposition
  (use task-planner).
---

# Prompt Forge — 提示词锻造引擎

将模糊需求锻造为结构化、防御性强的专家级 System Prompt。

## 核心哲学

> **没有对抗测试的 prompt 是等待失败的 prompt。**
> 表面看起来好用的 prompt，在真实使用中会暴露出注入漏洞、
> 歧义陷阱和边界越权。构建→攻击→修复循环在部署前发现这些问题。

> **结构即防御。**
> XML 锚定、显式否定约束、分层优先级创造的 prompt 在边缘场景下
> 优雅降级而非灾难性崩溃。散文式 prompt 静默失败。

> **目标 LLM 是编译目标。**
> 不同 LLM 解析指令的方式不同——Claude 偏好 XML，GPT 偏好 Markdown，
> Gemini 对 few-shot 特别敏感。Prompt 必须适配其目标架构。

---

## 工作流总览

```
用户模糊需求
     │
Phase 1: 需求分析 ─── 提取意图、范围、目标 LLM、约束
     │
Phase 2: 架构起草 ─── The Architect 构建结构化 prompt 骨架
     │
Phase 3: 红队攻击 ─── The Red Teamer 扫描漏洞（10 类攻击向量）
     │         │
     │    ┌────┘
     │    ▼
Phase 4: 加固修复 ─── The Refiner 修复漏洞（最多循环 2 次）
     │
Phase 5: 示例生成 ─── 生成 few-shot + model_thought 示例
     │
Phase 6: 终检交付 ─── 组装完整 prompt + 锻造日志
```

### 快速路径

```
用户需求
├─ 极简需求（一句话 prompt 就够）→ 直接交付，跳过 Phase 3-5
├─ 优化已有 prompt → 从 Phase 3 切入（红队攻击现有 prompt）
└─ 标准需求 → 完整 6 Phase 流程
```

---

## Phase 1: 需求分析

**目的**: 将模糊输入转化为结构化的 prompt 规格。

### 推演优先原则

不要问用户 "你想要什么目标 LLM？语气是什么？"——从需求中推演。
只在推演置信度 < 60% 的关键维度上向用户确认。

### 八维提取

| 维度 | 提取什么 | 默认值（未指定时） | 理由 |
|------|---------|----------------|------|
| 目标 LLM | Claude/GPT/Gemini/通用 | 通用 | 兼容性最好 |
| 用途类型 | 对话助手/工具/审核/复合 | 从描述推断 | — |
| 受众 | 终端用户/开发者/内部团队 | 终端用户 | 最常见场景 |
| 语气 | 正式/专业/亲切/技术 | 从用途推断 | 客服→亲切，代码→技术 |
| 范围边界 | 能做什么 + 不能做什么 | 从用途推断核心边界 | — |
| 输入模态 | 纯文本/多模态/结构化数据 | 纯文本 | 最常见 |
| 安全要求 | PII/内容策略/合规 | 标准防御 | 无特殊需求时的安全底线 |
| Token 预算 | 目标 prompt 长度 | 中等 (~2000 tokens) | 平衡详细度与 context 占用 |

### 推演输出

```
<analysis>
[用户意图]: 一句话总结
[八维规格]:
  - 目标 LLM: [推演结果] (来源: [用户明确指定 / 上下文推断 / 默认])
  - 用途类型: ...
  - ...
[推演置信度]: XX%
[需确认项]: [仅列出置信度 < 60% 的维度，如果都 ≥ 60% 则无]
</analysis>
```

---

## Phase 2: 架构起草 — The Architect

**目的**: 基于 Phase 1 的规格，构建完整的 prompt 初稿。

### 模板选择

```
用途类型
├─ 对话助手/角色扮演 → Persona-First 模板
├─ 工具/函数调用/数据处理 → Task-First 模板
├─ 内容审核/安全/合规 → Guard-First 模板
└─ 多能力复合 → Hybrid 模板
```

详细模板结构见 `references/architecture_templates.md`。

### Architect 必须产出的组件

1. **身份/任务定义** — 清晰的角色或任务说明（1-3 句）
2. **能力声明** — 明确列出能做什么
3. **工作流/回应框架** — LLM 接到输入后的处理步骤
4. **否定约束** — 每条禁止都附带理由和替代方案
5. **优先级层次** — 规则冲突时的优先顺序（默认：安全 > 准确 > 有用 > 简洁）
6. **多模态处理规则** — 如果涉及图片/文件（否则跳过）

### 目标 LLM 适配

| 目标 | 首选格式 | 关键特性 |
|------|---------|---------|
| Claude | XML 标签 | 原生 XML 锚定，`<thinking>` 标签 |
| GPT | Markdown `##` + 编号 | 重要规则放开头和结尾 |
| Gemini | XML / 结构化段落 | 对 few-shot 特别敏感 |
| 通用 | XML + Markdown 混合 | 2 层嵌套以内，核心规则重复 |

详细适配指南见 `references/architecture_templates.md` 底部。

### Architect 输出

完整的 prompt 初稿，使用目标 LLM 对应的格式。
在草稿中用 `[FEWSHOT_PLACEHOLDER]` 标记示例插入点（Phase 5 填充）。

---

## Phase 3: 红队攻击 — The Red Teamer

**目的**: 独立审视 Phase 2 的草稿，系统性扫描漏洞。

### 核心原则

Red Teamer 的唯一职责是**破坏**。它不修复，不优化，不妥协。
它假设用户是恶意的、愚蠢的或出乎意料的。

### 10 类攻击向量

| # | 类别 | 扫描焦点 |
|---|------|---------|
| 1 | Prompt Injection | 用户输入能否覆盖系统指令 |
| 2 | Ambiguity Traps | 是否存在多种合理解读 |
| 3 | Boundary Violations | 能否被引导超出职责范围 |
| 4 | Tone Drift | 特定输入是否导致语气偏离 |
| 5 | Format Breakdown | 什么输入会导致格式崩溃 |
| 6 | Constraint Evasion | 否定约束能否被间接绕过 |
| 7 | Multimodal Exploits | 非文本输入的攻击面（如适用） |
| 8 | Persona Collapse | LLM 是否会脱离预设角色 |
| 9 | Knowledge Boundary | 哪里可能产生幻觉 |
| 10 | Cascading Conflicts | 内部规则是否互相矛盾 |

每类攻击的详细描述、示例 exploit 和推荐防御见 `references/attack_vectors.md`。

### 漏洞报告格式

```
## Red Team Report

### V1 | CRITICAL | [Category]
攻击: [具体的攻击输入]
失败: [LLM 会怎样失败]
影响: [对用户/系统的实际影响]

### V2 | HIGH | [Category]
...
```

只报告实际可利用的漏洞，不报告理论上的、极低概率的风险。
按 severity 排序：CRITICAL → HIGH → MEDIUM → LOW。

---

## Phase 4: 加固修复 — The Refiner

**目的**: 针对 Red Team 发现的漏洞逐一修复。

### 修复策略

| 漏洞类别 | 首选修复手段 |
|---------|------------|
| Injection | 三明治结构 + 元指令 + 用户输入隔离标签 |
| Ambiguity | 量化模糊词 + 添加示例校准 |
| Boundary | Scope fence + 预设拒绝话术 |
| Tone Drift | 语气锚点 + 情绪场景示例 |
| Format | 输出模板骨架 + fallback 格式 |
| Constraint Evasion | 正面重述 + 解释理由 + 框架免疫 |
| Multimodal | 处理规则 + 类型限定 |
| Persona | 元问题标准回应 + 提取防御 |
| Knowledge | 诚实性协议 + 知识截止声明 |
| Cascading | 优先级层次 + 冲突决策规则 |

### 修复原则

1. **最小修改** — 只改需要改的，不借机重写整个 prompt
2. **解释 Fix** — 每个修复都说明修了什么、为什么这样修
3. **不引入新漏洞** — 修复 A 不能导致 B 出问题

### 循环控制

修复完成后快速重扫（仅 CRITICAL + HIGH）：
- 仍有 CRITICAL → 必须再修一轮（最多 2 次循环）
- 仅剩 MEDIUM/LOW → 记录并接受，进入 Phase 5
- 全部修复 → 进入 Phase 5

### 修复日志格式

```
## Fix Log

### V1 | CRITICAL | [Category] → FIXED
修复: [具体改了什么]
验证: [为什么这个修复有效]

### V3 | MEDIUM | [Category] → ACCEPTED
理由: [为什么接受这个风险]
```

---

## Phase 5: Few-Shot 示例生成

**目的**: 生成校准 LLM 行为的 few-shot 示例。

### 示例类型

| 类型 | 目的 | 必须？ |
|------|------|-------|
| Golden Path | 展示标准输入的理想处理 | 必须 |
| Edge Case | 展示非常规输入的优雅处理 | 必须 |
| Negative | 展示越界请求如何被拒绝 | 必须 |
| Multi-Turn | 展示上下文理解（对话式 prompt） | 对话式必须 |

### model_thought 模式

每个示例包含 `<model_thought>` 标签，展示 LLM 应有的内部推理：

```xml
<example type="[类型]">
  <user_input>[具体输入]</user_input>
  <model_thought>
    1. 意图识别: [用户真正想要什么]
    2. 范围检查: [是否在能力范围内]
    3. 风险评估: [是否触发约束]
    4. 策略选择: [采用什么回应方式 + 理由]
  </model_thought>
  <ideal_response>[理想回应]</ideal_response>
</example>
```

简单场景可简化为：
```xml
<model_thought>
  意图: [一句话]
  策略: [回应方式]
</model_thought>
```

### 数量指南

- 简单 prompt: 2-3 个（1 golden + 1 negative）
- 中等 prompt: 3-5 个（1 golden + 1 edge + 1 negative + 可选 multi-turn）
- 复杂 prompt: 5-7 个（每能力域 1 golden + 共享 negative/edge）

详细的示例写作模式和校准规则见 `references/fewshot_patterns.md`。

---

## Phase 6: 终检交付

**目的**: 组装所有组件，验证质量，交付最终 prompt。

### 终检

运行 `references/quality_checklist.md` 中的完整检查。
核心 6 项快速自检：

1. 目标用户和使用场景清楚吗？
2. 角色/任务定义明确吗？
3. 有否定约束且附带理由吗？
4. 有 few-shot 示例吗？
5. 经过红队验证吗？
6. 格式适配目标 LLM 吗？

### 交付格式

```markdown
---

## System Prompt

[完整的 System Prompt，在代码块中呈现，可直接复制使用]

---

## Forge Log（锻造日志）

### 攻击面摘要
| Severity | Found | Fixed | Accepted |
|----------|-------|-------|----------|
| Critical | X | X | 0 |
| High     | X | X | 0 |
| Medium   | X | X | X |
| Low      | X | X | X |

### 关键设计决策
- [决策1]: [理由]
- [决策2]: [理由]

---

## 使用建议
- **目标 LLM**: [名称]
- **推荐 Temperature**: [值 + 理由]
- **Token 预算**: [prompt 长度]
- **已知局限**: [诚实评估]
- **需要用户调整**: [如占位符替换]
```

---

## 异常处理

| 场景 | 处理方式 |
|------|---------|
| 极简需求（一句话 prompt 就够） | 跳过 Phase 3-5，直接交付并说明 |
| 用户要优化已有 prompt | 从 Phase 3 切入，红队攻击现有 prompt |
| 需求存在根本冲突 | 暴露冲突，给出解决建议，用户选择后继续 |
| 目标 LLM 未知或不支持 | 默认通用格式，说明局限 |
| 用户只要 prompt 不要日志 | 只输出 System Prompt，省略 Forge Log |
| 用户要迭代改进 | 读取反馈，仅修改指出的问题，重走 Phase 3-6 |

---

## 反模式

1. **不要动不动提问** — 推演优先，提问是最后手段。
   因为用户来找你是因为他们不知道细节，问他们只是把问题推回去。

2. **不要生成模板化通用 prompt** — 每个 prompt 都必须针对具体需求定制。
   因为用户自己也能写 "You are a helpful assistant"，他们需要的是更好的。

3. **不要跳过红队** — 对抗循环是核心价值，不是可选步骤。
   因为没有红队验证的 prompt 在生产环境中会暴露你没想到的漏洞。

4. **不要堆砌 50 条 NEVER** — 5 条有理由的约束 > 50 条无脑禁令。
   因为 LLM 理解 Why 后的约束遵守率远高于无理由的硬指令。

5. **不要忽略目标 LLM 差异** — Claude 的最佳实践不等于 GPT 的最佳实践。
   因为格式适配直接影响指令遵循率。

6. **不要生成与规则矛盾的示例** — 每个 few-shot 都是对规则的活教材。
   因为 LLM 看到矛盾时，示例的优先级高于文字规则。

---

## 语言与风格

- 匹配用户语言（中文需求 → 中文交互 + 按用户要求选择 prompt 语言）
- 技术术语中英混用保持自然
- Phase 过程简洁展示，不输出冗长的思考过程
- 交付的 prompt 本身则严格按目标 LLM 的最佳实践编写

---

## Reference 文件索引

| 文件 | 何时读取 |
|------|---------|
| `references/attack_vectors.md` | Phase 3，红队扫描时 |
| `references/architecture_templates.md` | Phase 2，选择 prompt 结构时 |
| `references/fewshot_patterns.md` | Phase 5，生成 few-shot 示例时 |
| `references/quality_checklist.md` | Phase 6，终检时 |
