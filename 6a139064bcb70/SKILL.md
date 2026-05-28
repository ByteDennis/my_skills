---
id: 6a139064bcb70
name: Mental-LLM Analysis
tags: []
updated_at: 2026-05-25T01:11:03.423963Z
---

# MentaLLaMA: Interpretable Mental Health Analysis on Social Media -  论文速读
> Yang, Kailai, et al., ACM Web Conference 2024 <br>
> "MentaLLaMA: interpretable mental health analysis on social media with large language models"

## 问题背景

| 维度 | 说明 |
|------|-------------------|
| 需求 | 社交媒体（Reddit/Twitter）文本海量，需自动化心理健康分析 |
| 判别模型瓶颈 | BERT/RoBERTa 泛化差、可解释性低，只输出标签无理由 |
| LLM 零样本瓶颈 | ChatGPT 零/少样本分类精度不及监督基线，错误预测会污染解释质量 |
| 资源缺口 | 缺乏带详细解释的公开训练数据；无开源可解释心理健康 LLM |

---

## 核心洞察

> 把心理健康分析从**分类任务**重构为**文本生成任务**：
> 模型同时输出 `[标签] Reasoning: [解释]`，解释质量与预测正确性相互约束。

- ChatGPT 能生成"人类水准"的解释，但分类精度弱 → 用 ChatGPT 生成**解释标注数据**，再微调开源 LLM
- 专家少样本示例 + 标签监督提示（with-label prompt）显著优于零样本生成质量
- RLHF 预训练的 LLaMA2-chat 底座比纯 LLaMA2 更易被指令调优激活

---

## 方法

```
原始数据（10 数据集 / 8 任务）
    ↓  专家写 35 条示例 × 10 数据集 → 金标准集 G（350条）
ChatGPT with-label 提示（2条/类 few-shot + 已知标签）→ 生成解释（105K 样本）
    ↓  三重质检：正确性 / 一致性 / 质量
IMHI 指令数据集（train 72K / val 14K / test 10 子集）
    ↓  LLaMA2 / LLaMA2-chat 指令微调
MentaLLaMA-7B / chat-7B / chat-13B
```

**8 任务覆盖**：二分类检测（抑郁/压力/孤独）、多分类检测（焦虑/PTSD/双相）、原因检测（SAD/CAMS）、风险/健康维度检测（IRF/MultiWD）

### 数据生成细节

**专家标注**
- 每个数据集：1条任务指令 + 35条解释示例 → 共 350 条
- GPT 生成时**不是把 35 条全塞进 prompt**：：随机抽 **2条/类** 作为 few-shot。所以实际 prompt = 任务指令 + 每个类别随机抽 2 条示例 + **目标帖子的已知标签**

**GPT 生成：with-label 是关键细节**
- GPT 不是自己预测标签再解释，而是**直接被告知正确答案（原始数据集的标注）**，只需生成与该标签一致的解释
- 有时 GPT 会在回复中表示"不同意"这个标签，那些样本由人工复核/重写

**三重质检**

| 维度 | 判断方法 | 结果 |
|---|-----------------|----------|
| **正确性** | 统计 ChatGPT 生成标签与原始标注一致率；不一致的人工复核 | 7/10 数据集 >90% 一致 |
| **一致性** | 用 MentalBERT 训练一个分类器：输入 `[explanation]` → 预测 `[label]`；在测试集和专家标注集 G 上测 F1 | 全部 >93.5%（即解释内容确实在"支持"对应标签）；SAD 最低 86.6%（标签语义相近） |
| **质量** |  用 BART-score 对比三种 prompt（零样本 / 少样本 / with-label）生成结果与 G 的相似度 | with-label > few-shot > zero-shot，证明质量足够高  |

另补充 **200条人工评估**（心理学博士生）：一致性/可靠性/专业性/总体，均分 >2.0/3.0。

**IMHI 指令数据集**
- 把 `(帖子 + ChatGPT解释)` 转为指令问答格式：
  ```
  Q: [任务描述] Post: ... Question: ...
  A: Yes. Reasoning: [解释]

- Train：72,095 条
- Val：14,346 条
- Test：10 个子集（来自各原始数据集的 test split）
  ```
- 另有 **IMHI-completion**（非指令格式），用于公平对比 BART/T5 等不支持指令跟随的模型

**训练的三个模型**

| 模型 | 底座 |
|------|------|
| MentaLLaMA-7B | LLaMA2-7B |
| MentaLLaMA-chat-7B | LLaMA2-chat-7B（含 RLHF） |
| MentaLLaMA-chat-13B | LLaMA2-chat-13B（含 RLHF） |

另外还单独训了一个 **LLaMA2-7B on IMHI-completion**，用作 baseline 对比

---

## 数学目标

$$\max_\phi \sum_{(q,r)\in\mathcal{D}} \sum_{j=1}^{|r|} \log P_\phi(r_j \mid q, r_{<j})$$

- **标准自回归语言模型目标**，在多任务混合指令数据上联合优化
- 一致性评估：用 MentalBERT 分类器验证 `[explanation] → [label]` 映射精度（F1 > 93.5%）
- 质量评估：BART-score（与人工评估相关性最强的自动指标）对比零/少/含标签三种提示输出

---

## 实验亮点

### ✅ Why it works

| 发现 | 原因 |
|------|-----------|
| MentaLLaMA-chat-13B 在 7/10 测试集逼近 MentalRoBERTa | 指令微调 + RLHF 底座激活领域能力 |
| chat 系列 > 纯 LLaMA2 微调 | LLaMA2-chat 已内化高质量指令跟随，更易对齐心理健康问答 |
| 解释质量 ≈ ChatGPT 水准 | with-label 提示 + 专家 few-shot 使训练数据解释质量高，微调后继承 |
| 泛化到未见任务优于 ChatGPT ZS | 多任务联合训练带来跨任务迁移能力 |
| 指令微调 > completion 微调 | IMHI-completion 格式"不自然"，无法充分激活 LLaMA2 能力 |

### ❌ Why it doesn't work

| 局限 | 原因 |
|------|---------|
| 整体分类仍弱于 MentalBERT/RoBERTa | 小判别模型在有监督数据上更高效，生成范式有精度代价 |
| T-SID / loneliness / IRF 数据一致性偏低（<80%） | 弱监督标注 or 主观性强的任务，ChatGPT 与标注不一致 |
| SAD 一致性分类器 F1 仅 86.6% | 标签语义相近（如 School vs Work），难以从解释区分 |
| Completion 微调 LLaMA2 < T5/BART（仅 15% 参数） | 大模型接受非指令格式数据时能力激活不充分 |
| GPT-4 未显著优于 ChatGPT | 心理健康任务主观性强，规模提升边际收益有限 |
