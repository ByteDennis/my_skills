---
id: 6a030872f09e3
name: RMSNorm
tags: []
updated_at: 2026-05-12T11:13:50.621889Z
---

# RMSNorm 公式直觉理解

公式 

$$
y = x \cdot \left(\text{mean}(x^2) + \varepsilon\right)^{-1/2} \cdot w
$$


**把一个向量的"整体大小"标准化到 1 附近，再让模型学一个缩放因子。**

## 拆解每一步

假设 $x$ 是一个向量，比如 $x = [3, 4, 0]$。

> Step 1: $x^2 \rightarrow [9, 16, 0]$

- 每个元素平方。这一步消除符号，只关心"大小"（magnitude），不关心方向。

> Step 2: $\text{mean}(x^2) = \frac{9 + 16 + 0}{3} \approx 8.33$

- 平均一下。这就是这个向量的**"平均能量"**——一个标量，描述整个向量"有多大"。

$$
\text{RMS}(x) = \sqrt{\text{mean}(x^2)}
$$

> Step 3: $\text{rsqrt}(\cdot + \varepsilon) = \frac{1}{\sqrt{8.33 + \varepsilon}} \approx 0.347$

- $\frac{x}{|||x||}$ 就是 $x \times \text{rsqrt}(\sum(x²)/n)$，几何上等价于把向量投影到单位球面上，只保留方向、丢弃长度

- $\text{rsqrt}$ 是 reciprocal square root，"平方根的倒数"。$\varepsilon$（一个很小的数，比如 $10^{-6}$）是防止$0/0$发生。

> Step 4: $x \cdot \text{rsqrt}(\cdot) \approx [1.04,\ 1.39,\ 0]$

把这个因子乘回 $x$。**效果**：不管原来 $x$ 多大多小，乘完之后它的 RMS 永远是 1。
>
> $$
> \text{RMS}(x_{\text{new}}) = \sqrt{\frac{1.04^2 + 1.39^2 + 0^2}{3}} = \sqrt{1.0} = 1 \;\checkmark
> $$

> Step 5: $w$

- $w$ 是一个可学习参数（和 $x$ 同维度），每个位置一个数。它让模型有机会说："虽然你把我标准化到 1 了，但我希望第 0 维放大 2 倍，第 1 维缩小一半..."

- 这一步给了模型**重新调节每个维度尺度的自由度**。如果模型觉得标准化做过头了，它可以学个 $w$ 把尺度调回来。

## 为什么要做这件事

神经网络深的时候，不同层的激活值大小可能差很多——有的层输出几百，有的输出 0.001。这种**尺度不稳定**会让训练崩掉（梯度爆炸或消失）。

RMSNorm 做的事：**每层算完，强制把激活值的"整体大小"统一到 1 量级，再让模型自己学要不要调回去**。

## 和 LayerNorm 的关系

LayerNorm 的公式是：

$$
\text{LayerNorm}(x) = \frac{x - \text{mean}(x)}{\sqrt{\text{var}(x) + \varepsilon}} \cdot w + b
$$

比 RMSNorm 多了"减均值"和"加 bias"两步。

RMSNorm 论文发现：**减均值这一步其实没啥用，砍掉也不影响效果，还能省计算**。所以 LLaMA、Mistral、Qwen 这些现代大模型基本全用 RMSNorm 了。

公式上的差别就是：LayerNorm 标准化"分布"（均值 0 方差 1），RMSNorm 只标准化"大小"（RMS 为 1, 只需要稳定不同参数的激活长度即可，不需要做平移让他们都在 $0$ 附近徘徊）。
