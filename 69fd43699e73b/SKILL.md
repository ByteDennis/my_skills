---
id: 69fd43699e73b
name: AUC Measure
tags: [auc, imbalanced, loss-function, ranking, classification]
updated_at: 2026-05-22T19:09:12.817034Z
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

## 诊断：Loss 选对了但 AUC 还是不高

Loss 只保证梯度方向对；AUC 卡住往往是**模型在错的地方出错**。以下三个切片快速定位。

### 按自信度切片（所有错例，$\text{correct}=0$）

| 切片 | 条件 | 含义 | 该改什么 |
|------|------|------|----------|
| **Confident wrong** | $\lvert\hat{p}-0.5\rvert > 0.4$ | 模型很笃定但错了 → 过拟合到误导性特征 | 砍该特征 / 加 regularization / dropout |
| **Ambiguous wrong** | $\lvert\hat{p}-0.5\rvert < 0.1$ | 模型本来就不知道 → 欠拟合，缺关键特征 | 加新特征 / 加深模型 / 加 attention |
| **Medium wrong** | 其他 | 一般错例 | 常规 boosting 加权 |

对**正确样本**也做同样切片：Ambiguous right（猜对的）占比高 → AUC 虚高，泛化差的早期信号。

### 按方向切片（F+ vs F−）

| 类型 | 定义 | 常见原因 | 处方 |
|------|------|----------|------|
| **F+** | $y=0,\;\hat{p}>0.5$ | item 太热门 / user 活跃但本次无行为 → 负样本被拉高 | 降热度信号权重 |
| **F−** | $y=1,\;\hat{p}<0.5$ | cold-start user / 长尾 item → hard positive 被漏掉 | 提高 $\gamma$（focal prior） |

F+ 多 vs F− 多，调的方向完全相反，不切片就容易用错策略。

### AUC 补充信号

| 指标 | 公式 | 解读 |
|------|------|------|
| Separation | $\bar{\hat{p}}^+ - \bar{\hat{p}}^-$ | 越大越好；低 = 正负分布重叠严重 |
| Loss ratio | $\mathcal{L}_\text{pos} / \mathcal{L}_\text{neg}$ | 比值高 = 正样本欠学 |
| Overlap area | pos 与 neg $\hat{p}$ 直方图重叠面积 | 0 = 完美分开；比 AUC 更直观 |

### 特征分布对比（过拟合 vs 欠拟合定位）

对每个特征维度 $X$，比较三个分布：$D_\text{wrong}(X)$、$D_\text{all}(X)$、$D_\text{pos}(X)$。

| 现象 | 判定 | 修法 |
|------|------|------|
| $D_\text{wrong} \approx D_\text{all}$，但错例集中在某 $\hat{p}$ 区间 | **欠拟合**：模型对 $X$ 无感，但 $X$ 与 label 实际相关 | 加 $X$ 相关特征 / 加深 / attention |
| $D_\text{wrong} \not\approx D_\text{all}$（$\text{KL} > 0.1$） | **过拟合**：在 $X$ 的某个 bucket 上 over/underpredict | 降该段 embedding / 加 dropout |
| $D_\text{wrong} \approx D_\text{pos}$ 且 wrong 是 F+ | **伪相关**：$X$ 与正样本相关但与行为不相关 | 砍 $X$ 或重新构造 |

实现：pyarrow `group_by + histogram`，加几行 KL/JSD 计算即可。

## Focal Loss 在广告推荐中为什么好用

广告点击率的典型正样本率 ~1–10%，Focal Loss 在这个场景几乎是标配。原因分两层。

### α：损失再平衡，不是正则

$$\mathcal{L}_\text{focal} = \begin{cases} -\alpha\,(1-p)^\gamma \log p & y=1 \\ -(1-\alpha)\,p^\gamma \log(1-p) & y=0 \end{cases}$$

$\alpha$ 控制正负样本的**全局损失权重**。直觉上正样本稀有应该 $\alpha$ 大（如 0.75）——但实际训练里正样本 loss 已经远高于负样本（例：$\mathcal{L}^+=1.49$，$\mathcal{L}^-=0.13$，比值 ×16），梯度已被正样本过度主导。

此时 $\alpha=0.1$ 是**反向操作**：把每条正样本损失乘 0.1，让两侧贡献回到同量级：

$$0.1 \times 1.49 \approx 0.9 \times 0.13$$

这是**损失再平衡**，目的是消除梯度方差不对等，不是 L2/dropout 意义上的正则。

### γ：抑制 easy 样本，把梯度留给 hard 样本

$(1-p)^\gamma$ 这一项：
- 正样本中，已经学对的（$p \to 1$）：$(1-p)^2 \approx 0$，损失近似归零
- 正样本中，还没学对的（$p \to 0$）：$(1-p)^2 \approx 1$，损失完整保留

负样本侧同理。**easy 样本的梯度贡献被动态压扁，hard 样本的贡献被保留。**

### 为什么 ep4–6 的收益大于 ep1

$\gamma$ 的作用随训练进度自适应变化：

| 阶段 | 模型状态 | $\gamma=2$ 的作用 |
|------|----------|------------------|
| ep1 | 所有样本都是 hard | 几乎不抑制谁，等价于 BCE + $\alpha$ 缩放 |
| ep2–3 | easy 样本开始学会，hard 样本未掌握 | 梯度逐渐从 easy 转向 hard |
| ep4–6 | easy 样本已记牢，再看会过拟；hard 样本仍未掌握 | easy 贡献压近零，模型只对 hard 继续学 |

实测对照（相同架构，仅换 loss）：

| epoch | BCE | Focal $\alpha{=}0.1,\,\gamma{=}2$ |
| :-------------|:---------------------------|-----------------------------------|
| ep1 | 0.8508 | 0.8561 |
| ep2 | **0.8546** ← peak | 0.8580 |
| ep3 | 0.8521 (−0.0025) | 0.8599 |
| ep4 | 0.8509 (−0.0037) | **0.8612** ← 仍在涨 |

BCE 在 ep2 到顶，ep3–4 回退 0.004（easy 样本过拟）；Focal 让 ep4 仍有 +0.005 涨幅。
