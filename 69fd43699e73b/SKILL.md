---
id: 69fd43699e73b
name: AUC 优化
tags: [auc, imbalanced, loss-function, ranking, classification]
updated_at: 2026-05-08T02:05:51.715738Z
---

## 目标与背景

$$\text{AUC} = P(f(x^+) > f(x^-))$$

衡量正负样本**排序质量**，与阈值无关。1:9 不平衡时 Accuracy 失效，AUC 仍有意义——问题变成：如何让模型给正样本打更高分？

## 四种方法数学分析

| 方法 | 优化目标 | 为什么 work | 为什么可能不够 |
|------|----------|------------|----------------|
| **Weighted BCE** | $-w^+y\log p - w^-(1-y)\log(1-p)$ | 提高正样本梯度权重，校正梯度期望使其趋近平衡分布下的方向 | 仍优化逐样本交叉熵，与 AUC（pairwise 排序）无直接数学联系 |
| **Focal Loss** | $-\alpha_t(1-p_t)^\gamma \log p_t$ | 动态降低易分样本权重，迫使模型聚焦难样本；$\gamma$ 控制 focus 程度 | 同 Weighted BCE，是 pointwise loss，不保证排序 |
| **AUC-ML Loss** | $\min_w \max_{a,b}\; \mathbb{E}[(\text{margin} - b(f-a))^2]$ | **直接近似 AUC**，saddle-point 形式绕开非凸离散排序，有收敛理论 | 需维护 $(a,b)$ 动态参数，mini-batch 估计有偏，训练不稳定 |
| **Pairwise Ranking** | $\sum_{i \in \mathcal{P}, j \in \mathcal{N}} \ell(f(x_i) - f(x_j))$ | AUC 的**精确定义**就是正负对排序正确率，直接优化即直接优化 AUC | $O(n^+ \cdot n^-)$ 对数量巨大；需负采样/近似，否则显存不可行 |

## 数学本质对比

$$\text{AUC} = \frac{1}{|\mathcal{P}||\mathcal{N}|} \sum_{i \in \mathcal{P},\, j \in \mathcal{N}} \mathbf{1}[f(x_i) > f(x_j)]$$

- **Pairwise**：用 sigmoid/hinge 替换 $\mathbf{1}[\cdot]$ → 可微近似，数学最对齐
- **AUC-ML**：saddle-point 等价形式，把 pairwise 期望转化为可并行的 min-max
- **Focal/wBCE**：优化 pointwise likelihood → 隐式改善排序（假设：校准好的概率 → 好排序）

## 假设链与断链

**Weighted BCE / Focal Loss 的隐含假设：**
> 若模型输出概率校准良好，则 $\hat{p}(x^+) > \hat{p}(x^-)$ $\Leftrightarrow$ AUC 高

- 1:9 时校准本身就难（输出天然偏向多数类）
- Focal Loss 抑制易分负样本梯度，但不保证正样本排在负样本之上
- **断链**：pointwise → ranking 的转化依赖校准，不平衡时校准差

**Pairwise Ranking 的假设：**
> 直接让 $f(x^+) - f(x^-) > \text{margin}$，无需校准中介

- **无断链**，数学上与 AUC 定义完全一致
- 代价：1:9 时负样本多，需控制 batch 内正负对采样比例

**AUC-ML Loss 的假设：**
> $\text{AUC} \approx 1 - \mathbb{E}_{i \in \mathcal{P}, j \in \mathcal{N}}[(f(x_i) - f(x_j) - 1)^2]$ 的 surrogate 可用 min-max 优化

- 理论严格，但 $a, b$ 是全局统计量，mini-batch 估计有偏
- **断链**：batch size 小时 $a/b$ 估计噪声大 → 不稳定

## 实践建议

```python
# 快速基线：Weighted BCE（最稳定）
pos_weight = torch.tensor([9.0])
loss = F.binary_cross_entropy_with_logits(logits, labels, pos_weight=pos_weight)

# 更好：Pairwise Ranking（最对齐 AUC）
# 每个 batch 采样等量正负对，计算 margin ranking loss
loss = F.margin_ranking_loss(pos_scores, neg_scores, target=ones, margin=1.0)

# 激进：LibAUC（直接优化 AUC，调好超参数收益最大）
from libauc.losses import AUCMLoss
from libauc.optimizers import PESG
loss_fn = AUCMLoss()
optimizer = PESG(model.parameters(), loss_fn=loss_fn, lr=0.1, margin=1.0)
```

## 选择建议

- 资源有限 / 快速验证 → **Weighted BCE**（稳定，效果够用）
- 想直接优化 AUC，工程可控 → **Pairwise Ranking**（采样控制 batch 大小）
- 数据量大，愿意调参 → **AUC-ML Loss**（理论上限最高）
- Focal Loss 在极度不平衡目标检测中设计，分类 AUC 任务优势不明显
