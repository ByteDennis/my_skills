---
id: 6a031f9fe54b5
name: Collaborative Unlearn
tags: [unlearn, relabel]
updated_at: 2026-05-16T19:55:41.438803Z
---

# COVA: Collaborative Vector Arithmetic for Recommendation Unlearning — 论文速读

> ICLR 2026 投稿（under review）

## 问题背景

### 推荐系统的`遗忘`为什么特别难

- 通用 unlearning 目标三件套：**完整性**（删干净）、**效用**（别砍模型性能）、**效率**（比从头训练快）。
- 但**协同过滤（CF）**有个独特的麻烦：用户和物品的表征是**纠缠在一起**联合训练的。删一个 `user-item` 交互，它的影响**已经扩散到整个表征空间**——不像 NLP/CV 那样样本独立，CF 里"删数据"和"删影响"是两件事。

### 现有两派方法的局限

| 方法 | 思路 | 致命问题 |
|---|---|---|
| **Partition-based**（SISA, RecEraser） | 把数据切成不相交的 shard，每个 shard 训子模型，删数据时只重训受影响的 shard | 切片本身就**切断了协同信号**；多 shard 跨删时退化成全量重训 |
| **Influence Function-based**（SCIF, IFRU） | 用 Hessian 逆近似估计某条交互对参数的影响，再反向修正 | 影响估计**本质是局部的**，要多次迭代且无收敛保证；大数据集上 Hessian 开销爆炸 |

> Partition-based, 把数据切成不相交的子集
- 假设有 10000 个用户的购物数据，要训推荐模型, 切成10份，每份训练一个独立的子模型
- 假设用户`#5234`要删除自己的购物记录, 查表发现该用户在第三个子集，则只重新训练M3, 可是CF模型最值钱的就是==跨用户==的信号, 另一个例子是假设一个用户要求删除50个商品的交互，这50个分散在5个子集里，则我们要重新训练5个子模型

> Influence Function-based 数学估计当初如果没有这个样本，参数会差多少
- $\theta - z^* \approx \theta^* + \frac{1}{n} H^{-1} \nabla_\theta \ell(z, \theta^*)$ , 其中$H$是 loss 的二阶导数矩阵。梯度告诉我们 loss 怎么变化，Hessian 告诉我们梯度怎么变化——综合起来就能估计删除一个样本后参数往哪移。假设`user #5234 喜欢电影 "流浪地球"`要求被删除，那么我们应该反推如果没有这条 (user=5234, item=流浪地球) 交互，参数应该往哪个方向微调
s- 但 A 的相似用户 `B, C, D...` 的 embedding 也都受过这条交互(user, item)的间接影响，因此后续IFRU的文章也考虑neighbor embedding也改, 令我


### Task Vector 范式（来自 NLP/CV）

定义 $\tau = \theta_{\text{ft}} - \theta_{\text{pt}}$（微调后参数 − 预训练参数）。Unlearning 时直接 $\theta^* = \theta_{\text{ft}} - \lambda \tau$，一步搞定，简单高效。

**为什么不能直接搬到 CF**：CF 模型（MF、LightGCN、GNN-based）学的是 **user/item 的 embedding**，不是层间权重。Task vector 操作权重的逻辑对"学 embedding"的模型根本不适用。

## COVA 的核心思路

> **把 task vector 的"做减法"思想从参数空间搬到 embedding 空间——但要先建一个所有 embedding 都能对齐的统一坐标系。**

### 三个关键矩阵

| 矩阵 | 含义 |
|---|---|
| $\mathbf{Y}_{\text{original}} \in \{0,1\}^{|U| \times |I|}$ | 原始交互矩阵（含要删的） |
| $\mathbf{Y}_{\text{ideal}} \in \{0,1\}^{|U| \times |I|}$ | 删除目标后的"理想"交互矩阵 |
| $\mathbf{R}_{\text{pred}} \in \mathbb{R}^{|U| \times |I|}$ | CF 模型在 $\mathbf{Y}_{\text{original}}$ 上训练后的预测分数矩阵 |

- $\mathbf{Y}_{\text{original}}$ vs $\mathbf{Y}_{\text{ideal}}$ 的差 = **该删的"结构性"部分**
- $\mathbf{R}_{\text{pred}}$ = **模型已经学到的协同模式**（影响已扩散）

### Step 1：构造统一矩阵 + 尺度对齐

垂直拼接：

$$
\mathbf{A} = \begin{bmatrix} \mathbf{Y}_{\text{original}} \\ \mathbf{Y}_{\text{ideal}} \\ \mathbf{R}_{\text{pred}} \end{bmatrix} \in \mathbb{R}^{3|U| \times |I|}
$$

**陷阱**：$\mathbf{Y}$ 是 0/1 二值，$\mathbf{R}_{\text{pred}}$ 是连续实数。SVD 会优先重建数值大的，导致二值矩阵被"忽略"。

**解决**：用 $\mathbf{R}_{\text{pred}}$ 的用户级统计量替换 0/1：

- 正交互（$y_{ui} = 1$，保留的）→ $\bar{r}_u^+ = \frac{1}{|Y_u^+|}\sum_{i \in Y_u^+} \hat{r}_{ui}$
- 无交互（$y_{ui} = 0$）→ $\bar{r}_u^0 = \frac{1}{|Y_u^0|}\sum_{i \in Y_u^0} \hat{r}_{ui}$
- **要删的交互**（在 $\mathbf{Y}_{\text{ideal}}$ 里）→ $\min_{i \in Y_u} \hat{r}_{ui}$，**最大化和保留交互的分离**

得到归一化后的 $\tilde{\mathbf{A}}$。

### Step 2：低秩 SVD（含可扩展性优化）

$$
\tilde{\mathbf{A}} \approx \mathbf{U}\boldsymbol{\Sigma}\mathbf{V}^\top, \quad \mathbf{U} \in \mathbb{R}^{3|U| \times k},\ \boldsymbol{\Sigma} \in \mathbb{R}^{k \times k},\ \mathbf{V} \in \mathbb{R}^{|I| \times k}
$$

- 用 **randomized low-rank SVD**（Halko et al., 2011），把 $O((3|U|)^2)$ 内存降到 $O(3|U| \cdot k)$，$k \ll 3|U|$
- **Chunk-based** 分块处理，让大数据集能塞下 GPU

把 $\mathbf{U}$ 按行切成三块：$\mathbf{U}_{\text{original}},\ \mathbf{U}_{\text{ideal}},\ \mathbf{U}_{\text{pred}} \in \mathbb{R}^{|U| \times k}$。

**关键点**：三个 user embedding **共享同一个 item basis $\mathbf{V}$**——这才让"做减法"在数学上有意义。

### Step 3：双任务向量做减法

把"该删的"拆成两个分量：

**结构性差异**（数据层面）：

$$
\Delta_1 = \mathbf{U}_{\text{original}} - \mathbf{U}_{\text{ideal}}
$$

**协同扩散差异**（模型层面）：

$$
\Delta_2 = \mathbf{U}_{\text{pred}} - \mathbf{U}_{\text{original}}
$$

最终的 unlearned embedding 和预测矩阵：

$$
\mathbf{U}_{\text{unlearned}} = \mathbf{U}_{\text{pred}} - \alpha \cdot \Delta_1 - \beta \cdot \Delta_2
$$

$$
\hat{\mathbf{R}}_{\text{unlearned}} = \mathbf{U}_{\text{unlearned}}\,\boldsymbol{\Sigma}\,\mathbf{V}^\top
$$

$\alpha, \beta$ 是超参，分别控制"结构遗忘"和"协同遗忘"的强度。

## 为什么 work

### 1. SVD 提供"统一坐标系"——这是技术核心

Task vector 在 NLP/CV 里能用，是因为 $\theta_{\text{ft}}$ 和 $\theta_{\text{pt}}$ 在同一个参数空间里，**减法天然有意义**。但 CF 里每次训练的 user embedding 都在**不同的随机基**下（embedding 初始化不同 → 旋转不同），直接相减是无意义的。

COVA 用 SVD 强制三个状态映射到**同一个 item basis $\mathbf{V}$ 张成的子空间**——这时 $\mathbf{U}_{\text{original}} - \mathbf{U}_{\text{ideal}}$ 才在数学上合法。**这一步是把 task vector 从参数搬到 embedding 的关键桥梁。**

### 2. 双 $\Delta$ 分解对应两类 influence

| 分量 | 抓的是什么 | 来自哪里 |
|---|---|---|
| $\Delta_1$ | "如果当初就没有这条交互，embedding 结构上的差异" | 数据本身 |
| $\Delta_2$ | "训练把交互的影响扩散到了哪些方向" | 模型优化过程 |

只减 $\Delta_1$ → 数据删了但模型记忆还在（IF 方法的通病）。只减 $\Delta_2$ → 抹掉训练痕迹但数据结构没改。**两个都减才是完整 unlearning**——论文 4.3.2 节实验验证了这个必要性。

### 3. Output-level 操作的实践优势

COVA **不碰模型内部参数**，只操作交互矩阵和预测矩阵。这有几个好处：

- 任何 CF 模型都能套（MF / LightGCN / GNN 都行），不需要为每个模型写专属 unlearning 代码
- 不需要训练过程的中间状态（梯度、Hessian），部署友好
- 单步操作，无迭代——比 IF 类方法快很多

### 4. 计算复杂度上的胜利

| 方法 | 多 shard 跨删 | 大数据集 |
|---|---|---|
| P-based | 退化成重训 | 内存吃紧 |
| IF-based | 多次迭代 + Hessian 逆 | 计算爆炸 |
| **COVA** | 一次 SVD + 一次减法 | low-rank + chunk 可扩展 |

## 为什么不一定 work / 局限

### 1. SVD 的低秩假设是个赌注

随机低秩 SVD 假设 $\tilde{\mathbf{A}}$ 的主要结构能被 top-$k$ 奇异值捕捉。但：

- 如果删除目标对应的协同模式落在**尾部奇异向量**里，低秩近似会丢失这部分信息
- $k$ 的选取没有原则性方法，是个超参——选小了漏信息，选大了内存崩
- 对**稀疏极端**的数据集（长尾用户/物品），低秩假设可能严重失真

### 2. 双重启发式：归一化策略 + $\alpha, \beta$

**归一化部分**：用 $\bar{r}_u^+$、$\bar{r}_u^0$、$\min \hat{r}_{ui}$ 替换 0/1 是经验设计。论文 4.3.1 节做了消融，但这种选择本质上是**为了让 SVD 行为符合预期**的工程 trick，不是从第一性原理推出来的。换数据集可能要重调。

**$\alpha, \beta$ 调参**：两个超参没有理论指导怎么选。如果 $\alpha \gg \beta$ 偏向结构遗忘，$\beta \gg \alpha$ 偏向协同遗忘——实际部署中需要根据 MIA 或下游指标反复试，**和宣称的"non-iterative"是有 tension 的**（你 unlearning 操作本身一步完成，但找超参不是）。

### 3. 没有理论保证

- **没有 differential privacy 风格的形式化 guarantee**——这是 approximate unlearning 整个流派的通病，但 COVA 也没解决
- **没有证明 $\hat{\mathbf{R}}_{\text{unlearned}}$ 在某种意义下等价于 retrain 的结果**——只有实验上的"接近"

### 4. $\mathbf{Y}_{\text{ideal}}$ 的依赖

COVA 需要构造 $\mathbf{Y}_{\text{ideal}}$（删除后的理想交互矩阵）作为参考。对**大批量删除请求**或**频繁删除场景**：

- 每次删除都要重新构造矩阵 + 跑 SVD
- 虽然单次比 retrain 快，但**频次高时总成本未必赢**
- 论文实验设计的是"单批删除"，**流式删除（streaming unlearning）**场景没覆盖

### 5. Output-level 的另一面

"不碰模型内部"既是优点也是局限：

- 优点：通用、易部署
- 局限：**模型本身的参数没动**——下次模型继续训练或在线学习时，原始数据的影响可能"复活"。COVA 修改的是预测输出层面，不是真正"让模型忘了"
- 严格意义上，这是**"输出层 unlearning"**而非"模型层 unlearning"

### 6. 评估上的盲点（推测）

论文摘要说在三个维度（完整性、效用、效率）上击败 baseline。但需要看具体实验：

- 是否用了 SOTA MIA（如 RMIA）？还是用过时的 Yeom/Song 那套？
- 是否测了 user-side / item-side / 交互-side 三种删除？
- 是否测了**冷启动用户**的删除（这是 CF unlearning 最难的情形）？

## 与前一篇 AMUN 的对比

| 维度 | AMUN（分类） | COVA（推荐） |
|---|---|---|
| **领域** | 图像分类 | 协同过滤 |
| **数据假设** | 样本独立 | 样本纠缠（user-item） |
| **核心机制** | 对抗样本 + 微调 | SVD + 任务向量减法 |
| **是否需要 retain set** | 可选（forget-only 也行） | 不需要（只要 $\mathbf{Y}_{\text{original}}$ 和 $\mathbf{Y}_{\text{ideal}}$） |
| **理论分析** | 凸假设下的距离上界 | 无 |
| **碰模型内部吗** | 改参数 $\theta$ | 不改，只改 output |
| **迭代次数** | 多步微调 + 早停 | 一步 SVD + 一步减法 |

**有意思的共同点**：两者都在用"**借助某种数学结构做局部、可控的扰动**"来近似 retrain——AMUN 用对抗样本制造局部决策边界扰动，COVA 用 SVD 创造统一空间让向量减法合法。**本质都是把"删除"这件事翻译成一个数学上有意义的几何操作**。

## 一句话总结

**COVA 用 SVD 把原始交互、理想交互、模型预测三个矩阵映射到同一个正交基下，让推荐模型的 embedding 也能像 NLP 的 task vector 那样"做减法"——一步同时减掉数据结构差异和协同扩散差异，绕开 partition 切断协同信号和 IF 局部化的两大老问题。代价是依赖低秩假设和两个超参 $\alpha, \beta$，且本质上是输出层修正而非真正改动模型。**
