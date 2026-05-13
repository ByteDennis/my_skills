---
id: 6a03086232beb
name: Flash Attention
tags: []
updated_at: 2026-05-12T11:42:58.404095Z
---

# Flash Attention 分块算法原理

## 朴素 attention 的问题

$$
\text{Attention}(Q, K, V) = \text{softmax}\!\left(\frac{QK^\top}{\sqrt{d}}\right) V
$$

按字面照搬，要这么算：

1. 算 $S = QK^\top / \sqrt{d}$，得到 $[N, N]$ 矩阵（$N$ 是序列长度）
2. 算 $P = \text{softmax}(S)$，还是 $[N, N]$
3. 算 $O = PV$，得到 $[N, d]$

**问题**：$N = 4096$ 时，那个 $[N, N]$ 矩阵是 $1600$ 万个元素。fp16 下 32MB，要在显存（HBM）里来回搬，而 GPU 的 SRAM（片上高速缓存）只有几十 KB 到几 MB，根本塞不下整个矩阵——只能不停地从 HBM 读、写回 HBM。

GPU 算得很快，但**搬数据慢**。这种"算得起但搬不起"的算子叫 **memory-bound**——瓶颈在内存带宽（bandwidth），不在算力。

> **背景常识：GPU 的内存层级**
> - **HBM**（High Bandwidth Memory）：显存，几十 GB，带宽约 1–3 TB/s。看起来很快，但相对于 GPU 算力还是慢。
> - **SRAM**：每个 SM（Streaming Multiprocessor）上的片上内存，几十到几百 KB，带宽约 19 TB/s。比 HBM 快一个数量级。
> - 现代 GPU 的算力增长远快于内存带宽增长——这就是"算得起搬不起"的物理来源。

`Tile`之间完全不重叠——把 K/V 切成不相交的连续块（比如 `[0:64], [64:128], [128:192]...`），一块块过；softmax 用 online softmax 技巧，看完一块就用修正因子 $\exp(m_{old} - m_{new})$ 把已累积的结果重新加权，再把新块的贡献加进去，等所有块过完一遍，结果和"一次性算完整 softmax"数学上完全等价。

![Naive vs Flash Attention 对比图](/home/dalab2/.local/files/images/screenshots/clipboard_20260512_112129.png=90%)

## 分块的核心思路

> **别一次性算完整个 $[N, N]$ 矩阵。分块算，每块算完直接和 $V$ 相乘累加到输出里，大矩阵从来不需要完整存在过。**

具体地：把 $Q$ 切成行块（每块比如 128 行），$K$、$V$ 切成列块（不相交的连续块，比如 $[0{:}64], [64{:}128], [128{:}192], \dots$）。然后双层循环，先写一个**有问题的简化版**伪代码：

```
for 每个 Q 的行块 Q_i:
    初始化输出 O_i = 0
    for 每个 K/V 的列块 K_j, V_j:
        S_ij = Q_i @ K_j^T          # 小块，能塞进 SRAM
        P_ij = softmax(S_ij)        # ⚠️ 这一步是错的
        O_i += P_ij @ V_j
```

**注意**：这个伪代码是教学用的"稻草人"。如果真这么做——只对当前 tile 内的几个 score 做 softmax——结果会错，因为 softmax 的归一化需要看完所有 key。下一节讲为什么错，再下一节讲真实算法怎么修正。


## 为什么"块内 softmax"是错的

softmax 的定义：

$$
\text{softmax}(x_i) = \frac{\exp(x_i)}{\sum_j \exp(x_j)}
$$

**分母是对所有 $j$ 求和**——要算第 $i$ 个位置的 softmax，必须知道**所有其他位置**的 $\exp(x_j)$。

矛盾出现了：我们想分块算，但 softmax 要求"看完所有列才能 normalize"。如果只看了 $K$ 的前两块，分母还没算完，块内做的 softmax 就用错了归一化基数。

**这就是 Flash Attention 的核心技巧要解决的问题：online softmax。**

## Online Softmax：增量更新的数学技巧

直觉：**先记账，不归一化；每来一个新块，用一个修正因子把之前的累积结果重新加权，再把新块的贡献加进去；所有块走完，最后统一归一化一次**。

真实的 Flash Attention **不显式调用** `softmax(S_ij)`。它对每个 query 行（实际是 query 块的每一行）维护三个累积状态：

- $m$：到目前为止见过的**所有** $q_i \cdot k_j$ 中的最大值（用于数值稳定，防止 $\exp$ 溢出）
- $\ell$：到目前为止 $\sum_j \exp(q_i \cdot k_j - m)$ 的累积（softmax 的分母）
- $O$：到目前为止 $\sum_j \exp(q_i \cdot k_j - m) \cdot v_j$ 的累积（**未归一化**的输出）

每来一个新 K/V 块，更新逻辑是：

```
S_ij = Q_i @ K_j^T              # 当前块的原始 score，没做 softmax
m_new = max(m_old, rowmax(S_ij))

# 用新最大值修正之前的累积
α = exp(m_old - m_new)          # 修正因子
O = O * α
ℓ = ℓ * α

# 把当前块的贡献加进去（用 m_new 做数值稳定）
P_ij = exp(S_ij - m_new)        # 未归一化的指数
O += P_ij @ V_j                 # 累积分子
ℓ += rowsum(P_ij)               # 累积分母

m_old = m_new

# 所有块跑完后，最后归一化一次
O_final = O / ℓ
```

### 为什么需要那个修正因子 $\exp(m_{\text{old}} - m_{\text{new}})$

数值稳定的 softmax 公式是 $\frac{\exp(x_i - m)}{\sum_j \exp(x_j - m)}$，其中 $m$ 是所有 $x_j$ 的最大值。减最大值不改变结果（分子分母同除 $\exp(m)$），但避免了 $\exp$ 爆炸。

问题是：流式处理时，"全局最大值"会随着看到新块而更新。假设之前用 $m_{\text{old}}$ 累积了：

$$
O_{\text{old}} = \sum_{j \in \text{已见}} \exp(s_j - m_{\text{old}}) \cdot v_j
$$

现在最大值变成了 $m_{\text{new}}$，我希望累积里用的是 $m_{\text{new}}$：

$$
O_{\text{correct}} = \sum_{j \in \text{已见}} \exp(s_j - m_{\text{new}}) \cdot v_j
$$

两者的关系是：

$$
\exp(s_j - m_{\text{new}}) = \exp(s_j - m_{\text{old}}) \cdot \exp(m_{\text{old}} - m_{\text{new}})
$$

所以只要把 $O_{\text{old}}$ 整体乘上标量 $\exp(m_{\text{old}} - m_{\text{new}})$，就把"基准最大值"切换好了。$\ell$ 也是同样的修正逻辑。

注意 $m_{\text{new}} \geq m_{\text{old}}$，所以 $\exp(m_{\text{old}} - m_{\text{new}}) \leq 1$，旧累积只会被"缩小"，不会爆炸。

### 关键事实

- $q_i \cdot k_j$ 这些 dot product 是**逐块算的**
- 但 softmax 的归一化（除以 $\sum_j \exp(\cdot)$）是**跨所有块的全局操作**，只是被拆成了"边算边累加"的增量形式
- 等所有块跑完一遍，结果**和"一次性算完整 softmax"数学上完全等价**——不是近似

## 一个生活化的类比

想象你在统计班级考试**加权平均分**，学生一个个走进教室告诉你成绩，你不能等所有人都来：

- 来第一个：60 分。当前累积总分 60，人数 1。
- 来第二个：80 分。当前累积总分 140，人数 2。
- 来第三个：90 分。当前累积总分 230，人数 3。
- 最后算平均：$230 / 3 = 76.67$。

你不需要记住每个人的具体分数，只要维护"累积总分"和"人数"两个状态，每来一个新人增量更新即可。最后再除一次。

**Online softmax 做的是结构类似的事**——只是因为涉及 $\exp$ 和数值稳定，状态变量从 2 个变成 3 个（$m, \ell, O$），更新时要额外做一次"修正旧累积"的操作。

## 为什么这样就快了

1. **省显存**：$[N, N]$ 大矩阵从来不需要物化。$N = 4096$ 时省下 32MB，$N = 32768$ 时省下 2GB——这是长序列训练能跑起来的关键。
2. **省带宽**：所有中间结果都在 SRAM 里完成，HBM 上只有"读 Q/K/V 块"和"写 O 块"两件事，不存在反复读写 $[N, N]$ 矩阵的成本。
3. **GPU 利用率高**：算力没浪费在等数据上，从 memory-bound 转向 compute-bound。

## 为什么短序列上 FA 不一定快

回头看那份 Flash Attention benchmark：seq_len 只有 128。

- 朴素 attention 的 $[N, N]$ 矩阵只有 $128 \times 128 = 16\text{K}$ 元素，本来就塞得进 SRAM，**根本不需要省**
- 但 Flash Attention 的代价（双层循环开销、online softmax 的修正算术、维护 $m / \ell / O$ 三个状态、kernel launch overhead）一分不少
- 净结果：在小形状上 FA 可能**比朴素实现还慢**

这就是为什么 benchmark 文档强调"**Flash Attention 的收益需要按真实模型形状重测**"——FA 是为长序列设计的，短序列拿它当通用方案反而吃亏。

## 共同主题

RMSNorm 和 Flash Attention，两个看起来毫不相关的东西，**优化思路其实是同一个**——

> **减少"算了又算"和"读了又读"。把多步骤计算融合成一个 kernel，让数据只在 HBM 里走一趟。**

RMSNorm 的融合 kernel 让 $x^2$、求均值、$\text{rsqrt}$、乘 $w$ 这一串只读一次 $x$ 写一次结果。Flash Attention 的分块让 $[N, N]$ 矩阵从不物化。

这是现代 AI kernel 优化的**核心母题**：算力够用，瓶颈在内存。**谁更省搬运，谁更快**。
