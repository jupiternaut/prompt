# Few-Shot Example Generation Patterns

Phase 5 示例生成阶段参考此文件。
生成高质量的 few-shot 示例来校准目标 LLM 的行为。

---

## 核心原则

1. **示例是最强的指令** — 一个好示例胜过十句描述
2. **model_thought 激活深度推理** — 展示"怎么想"比展示"怎么答"更有效
3. **多样性 > 数量** — 3 个覆盖不同场景的示例 > 5 个相似的示例
4. **示例不能违反 prompt 规则** — 每个示例都是对规则的活教材

---

## model_thought 标签模式

### 什么是 model_thought

`<model_thought>` 是插入在用户输入和模型回应之间的推理演示块。
它展示 LLM 应该如何内部思考，但**不直接出现在最终回应中**。

对 Claude：可映射到 extended thinking / `<thinking>` 标签
对 GPT：作为示例中的隐式推理参考
对 Gemini：直接作为 few-shot 中的推理步骤

### model_thought 写作模板

```xml
<model_thought>
1. 意图识别: [用户实际想要什么，区分字面请求和真实需求]
2. 范围检查: [是否在我的能力/权限范围内]
3. 风险评估: [是否触发任何约束或安全规则]
4. 策略选择: [基于以上判断，选择哪种回应策略]
5. 质量预检: [回应是否符合格式要求、语气标准]
</model_thought>
```

### model_thought 简化模板（简单场景）

```xml
<model_thought>
意图: [一句话]
策略: [选择的回应方式 + 理由]
</model_thought>
```

**规则**: 复杂/边缘场景用完整模板，简单场景用简化模板。
每组 few-shot 中至少 1 个用完整模板。

---

## 示例类型与模板

### Type 1: Golden Path（标准路径）

**目的**: 展示最常见输入的理想处理方式。

```xml
<example type="golden_path" teaches="[这个示例教会 LLM 什么]">
  <user_input>
    [典型用户输入 — 具体、有背景、接近真实使用场景]
  </user_input>
  <model_thought>
    意图: [分析]
    策略: 标准处理流程，按 [具体步骤] 执行
  </model_thought>
  <ideal_response>
    [完整的理想回应 — 展示格式、语气、深度的标准]
  </ideal_response>
</example>
```

**写作要点**:
- 输入要具体（不是 "帮我分析数据"，而是 "这份 Q3 销售报告里哪个区域增长最快"）
- 回应展示完整格式（如果 prompt 要求特定结构，示例必须体现）
- model_thought 用简化模板即可

### Type 2: Edge Case（边缘场景）

**目的**: 展示非常规但合法输入的处理方式。

```xml
<example type="edge_case" teaches="[这个示例教会 LLM 什么]">
  <user_input>
    [非常规输入 — 模糊、不完整、多任务、格式奇怪等]
  </user_input>
  <model_thought>
    1. 意图识别: 用户的请求 [不完整/模糊/多重]，需要 [处理策略]
    2. 范围检查: [是否在范围内]
    3. 策略选择: [选择特殊处理路径 + 理由]
  </model_thought>
  <ideal_response>
    [处理后的回应 — 展示如何优雅处理非标准输入]
  </ideal_response>
</example>
```

**写作要点**:
- 输入要真实反映用户可能的 "不完美" 使用方式
- model_thought 用完整模板展示决策过程
- 回应展示降级/澄清/推断策略

### Type 3: Negative（拒绝/引导）

**目的**: 展示超出边界的请求如何被处理。

```xml
<example type="negative" teaches="[这个示例教会 LLM 什么]">
  <user_input>
    [越界请求 — 超范围、违规、注入尝试、不当请求等]
  </user_input>
  <model_thought>
    1. 意图识别: 用户想要 [分析字面请求]
    2. 风险评估: 这触发了 [具体约束/安全规则]
    3. 策略选择: 拒绝 + 提供替代方案
  </model_thought>
  <ideal_response>
    [礼貌但坚定的拒绝 + 可行的替代建议]
  </ideal_response>
</example>
```

**写作要点**:
- 拒绝理由要明确但不说教
- 始终提供替代方案（"我不能做 X，但可以帮你 Y"）
- model_thought 展示如何识别风险并触发安全规则

### Type 4: Multi-Turn（多轮对话）

**目的**: 展示对话上下文如何影响回应（仅对话式 prompt 需要）。

```xml
<example type="multi_turn" teaches="[这个示例教会 LLM 什么]">
  <conversation>
    <turn role="user">[第一轮用户输入]</turn>
    <turn role="assistant">[第一轮回应]</turn>
    <turn role="user">[第二轮用户输入 — 基于前文的追问/补充/转向]</turn>
  </conversation>
  <model_thought>
    1. 上下文回顾: 用户之前问了 [X]，现在追问 [Y]
    2. 策略: [如何利用上下文提供更精准的回应]
  </model_thought>
  <ideal_response>
    [体现上下文理解的回应]
  </ideal_response>
</example>
```

---

## 示例数量指南

| Prompt 复杂度 | 推荐示例数 | 必须包含的类型 |
|-------------|----------|-------------|
| 简单（单一任务） | 2-3 | 1 golden + 1 negative |
| 中等（多场景） | 3-5 | 1 golden + 1 edge + 1 negative |
| 复杂（多能力域） | 5-7 | 每个能力域 1 golden + 共享 1-2 negative + 1 edge |
| 对话式 | +1-2 multi-turn | 在上述基础上加 multi-turn |

---

## 示例校准规则

### Do
- 每个示例教一个明确的行为模式
- 示例之间覆盖不同的场景维度（不同话题、不同用户意图、不同复杂度）
- 回应长度与 prompt 中定义的长度要求一致
- model_thought 中的判断逻辑与 prompt 中的规则对应

### Don't
- 不要让所有示例都是 "happy path" — 必须有边缘和拒绝场景
- 不要在示例中违反 prompt 自己的规则
- 不要让示例过长以至于挤占 context window
- 不要在 negative 示例中透露 "如果你换个说法我就会做"

---

## 目标 LLM 适配

### Claude
- model_thought 可映射为 `<thinking>` 标签（Claude 原生支持）
- 或作为 `<example>` 标签中的 `<thinking>` 子标签
- Claude 对 XML 示例结构反应最好

### GPT
- 示例作为 system message 的一部分
- model_thought 写为普通文本的推理过程，用 `---` 分隔
- 或放在 "Internal reasoning (not shown to user):" 前缀下

### Gemini
- `<model_thought>` 标签直接使用（Gemini 对这类结构化标签响应良好）
- 示例越多效果越好（利用长上下文优势）
- 可在 system instruction 中放置更多示例而不担心 context 不足

### 通用
- 使用 XML 示例结构 + 注释说明 model_thought 的作用
- 简化 model_thought 为 2-3 步（避免过复杂的推理链）
