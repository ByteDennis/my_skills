---
id: 6a031ff242ca3
name: Adversarial Unlearn
tags: [unlearn]
updated_at: 2026-05-16T20:00:43.913423Z
---

# AMUN: Adversarial Machine Unlearning — 论文速读
> Ebrahimpour-Boroojeny et al., ICML 2025 <br>
> *"Not All Wrong is Bad: Using Adversarial Examples for Unlearning"*

## 问题背景

**Machine Unlearning**：模型 $F$ 已经在数据集 $D$ 上训练好了，现在用户根据 GDPR / CCPA 要求删除某个子集 $D_F \subset D$ 的影响。理想做法是用 $D_R = D - D_F$ 从头重训（**retrain from scratch**），得到 $F_R$ 作为 gold standard。但重训太贵，所以需要**近似 unlearning**：从已训练的参数 $\theta_o$ 出发，改一改变成 $\theta_o'$，使其分布等价于 $\theta_u \sim \Theta_{D_F}$（在 $D_R$ 上训练得到的参数分布）。

## 核心洞察

### Key Observation 1：retrain 模型行为的特征

作者先**观察 gold standard** ($F_R$) 在三个集合上的预测置信度：

- $D_T$（测试集，没见过）
- $D_F$（被遗忘的样本，没见过）
- $D_R$（保留的样本，训练过）

发现：$F_R$ 对 $D_T$ 和 $D_F$ 的预测**置信度都比对 $D_R$ 低**——因为 $D_F$ 和 $D_T$ 同分布，且 $F_R$ 对两者都是"unseen"。

**所以 unlearning 的本质不是让模型"答错" $D_F$，而是让模型对 $D_F$ 的置信度降到测试集水平。** -- ==真正意义上的泛化==

之前的方法（GA, RL, SalUn 等）粗暴地做"反向训练"或"打错标签"，会导致 catastrophic forgetting，破坏整体性能。

### Key Observation 2：对抗样本微调不会灾难性遗忘

这是论文最反直觉的发现。

设 $D_A$ 是 $D_F$ 对应的对抗样本集合（用 PGD 找的 $x_{\text{adv}} = x + \delta_x, ||\delta_x||_2 \leq \epsilon$ 被模型错判为 $y_{\text{adv}}$，其中 $y_{\text{adv}} \neq y$）。在 $D_A$ 上用**模型预测出的错误标签** $y_{\text{adv}}$ 微调时：

- **直觉**会以为：用错标签微调 → 模型崩
- **实际**：测试精度只小幅下降

**为什么？** 因为对抗样本虽然标签"错"（相对人类标准），但它**已经在模型自己学到的分布上**——模型本来就把 $x_{\text{adv}}$ 预测为 $y_{\text{adv}}$。微调只是让模型对这个"已有的错误预测"更自信一点，不与已学分布对抗，所以不破坏整体决策边界。==就是说用学好的模型本身去做对抗样本只是让他在已有的错误预测上更加自信==

## AMUN 方法

### 算法（一句话）

对每个 $(x, y) \in D_F$，找一个最近的对抗样本 $(x_{\text{adv}}, y_{\text{adv}})$；然后在 $D_F \cup D_A$（或加上 $D_R$）上用各自标签微调 $\theta_o$。

### 关键步骤

**Algorithm 1**：从小 $\epsilon$ 起步，跑 PGD 攻击，找不到对抗样本就 $\epsilon \leftarrow 2\epsilon$，直到找到——**保证 $\|x - x_{\text{adv}}\|_2$ 尽可能小**。

### 为什么 work（直觉版）

把 $(x_{\text{adv}}, y_{\text{adv}})$ 和 $(x, y)$ 同时塞进微调集——这两个样本**在输入空间几乎重合**（距离 $\leq \epsilon$），却有**不同标签**。模型必须在这个小邻域里调整决策边界以同时拟合两者，结果就是：

$$
\text{confidence}_{\theta_o'}(y \mid x) \;<\; \text{confidence}_{\theta_o}(y \mid x)
$$

这正是 $F_R$ 在 $D_F$ 上表现出的行为——置信度降到 unseen 水平。

而且因为 $\|x - x_{\text{adv}}\|$ 远小于 $D_F$ 中样本到 $D_R$ 中样本的距离，决策边界的扰动**被局部化**，不影响其他区域。

## 数学保证：Theorem 4.1

在 $L$-Lipschitz 模型、$\hat{R}$ 是 $\beta$-smooth 凸函数等常见假设下，AMUN 一步梯度下降（学习率 $\frac{1}{\beta}$）得到的参数 $\theta'$ 满足：

$$
\|\theta' - \theta_u\|_2^2 \;\leq\; \|\theta_o - \theta_u\|_2^2 + \frac{2}{\beta}(L\delta - C)
$$

其中

$$
C = \ell(f_{\theta_o}(x_{\text{adv}}), y) + \ell(f_{\theta'}(x_{\text{adv}}), y_{\text{adv}}) - \ell(f_{\theta_u}(x), y) - \ell(f_{\theta_u}(x_{\text{adv}}), y_{\text{adv}})
$$

### 这个 bound 在说什么

参数到 gold standard 的距离上界由三个因素决定：

| 因素 | 想要的方向 | 原因 |
|---|---|---|
| Lipschitz 常数 $L$ | **越小越好** | 模型对输入变化越不敏感，参数邻域越平滑 |
| 对抗样本距离 $\delta$ | **越小越好** | 改动越局部，越不影响其他区域 |
| $C$ | **越大越好** | 见下表分解 |

### $C$ 越大需要什么

- $\ell(f_{\theta_o}(x_{\text{adv}}), y)$ **大** → 对抗样本要"真的对抗"，在原模型上正确标签的 loss 高
- $\ell(f_{\theta_u}(x_{\text{adv}}), y_{\text{adv}})$ **小** → 对抗样本可迁移到 retrained 模型（Lipschitz 小有助于此）
- $\ell(f_{\theta'}(x_{\text{adv}}), y_{\text{adv}})$ 不能太低 → 微调时**别过拟合**对抗样本（早停 + 适当学习率）
- $\ell(f_{\theta_u}(x), y)$ **小** → retrained 模型对 unseen 样本泛化要好（这与方法本身无关，是数据/模型固有的）

## 实验亮点

- **两种 setting 分开评测**：(1) 有 $D_R$ 访问权 (2) 只有 $D_F$（更符合隐私法规真实场景）
- 用 **RMIA**（SOTA membership inference attack）替代过时的 MIA 评测
- 在 ResNet-18 / CIFAR-10、VGG19 / Tiny ImageNet 等设置上击败 SalUn、SOTA baselines
- 评测指标：**Average Gap**（与 $F_R$ 的差距），AMUN 显著更接近 gold standard

## 为什么 work（总结）

1. **对抗样本不破坏分布**：$y_{\text{adv}}$ 虽是错标签，但**在模型已学到的分布上**，微调不会引发对抗
2. **局部化干扰**：$\|x - x_{\text{adv}}\|$ 极小，决策边界只在 $D_F$ 邻域被推动，$D_R$ 区域不受影响
3. **置信度降低正中目标**：$x$ 和 $x_{\text{adv}}$ 标签冲突 → 模型必须降低对 $x$ 的预测置信度 → 复现 $F_R$ 在 $D_F$ 上的表现
4. **不需要 $D_R$**：与前作不同，AMUN 在 forget-only 设置下也成立——更符合实际部署约束

## 为什么不一定 work / 局限

### 来自 Theorem 4.1 的暗示

1. **依赖好的对抗样本**：如果攻击太弱（FGSM 而非 PGD-50），$C$ 中 $\ell(f_{\theta_o}(x_{\text{adv}}), y)$ 上不去，bound 松；论文附录 F.5 确实显示 FGSM 下性能下降。
2. **依赖 Lipschitz 常数小**：$L$ 大的模型，bound 直接放大 $L\delta$ 项，且对抗样本可迁移性差（$\ell(f_{\theta_u}(x_{\text{adv}}), y_{\text{adv}})$ 高）。
3. **$|D_F|$ 太大时退化**：bound 第四项 $\ell(f_{\theta_u}(x), y)$ 依赖 retrained 模型的泛化——$|D_F|$ 大 → $|D_R|$ 小 → $F_R$ 本身就差 → AMUN 也就跟着差。这是**所有 unlearning 方法的本质局限**，不是 AMUN 独有。

### 方法层面的潜在问题

1. **PGD-50 攻击成本不低**：每个 $D_F$ 样本要跑多次攻击直到找到对抗样本，$|D_F|$ 大时计算成本可观（虽然仍远小于 retrain）
2. **凸性假设不成立**：Theorem 4.1 假设 $\hat{R}$ 凸，但深度网络的损失曲面非凸；bound 是 sanity check 而非严格保证
3. **需要早停**：第三个 $C$ 因素要求微调不能过头，超参（学习率、epochs）敏感
4. **对 catastrophic forgetting 的"免疫"是经验观察**：Key Observation 2 没有理论解释为什么在 $D_A$ 上微调不崩——只是实验上没崩
5. **没有形式化的隐私保证**：与 differential-privacy 风格的 certified unlearning 方法不同，AMUN 是"approximate + empirically validated by MIA"，不能给出 $(\epsilon, \delta)$-级别的可证明保证

## 一句话总结

**用模型自己制造的对抗样本（标签错但落在模型已学分布里）去微调，巧妙地把"答错"变成"降低自信"，从而模拟 retrained 模型在 $D_F$ 上的低置信度行为——不破坏整体性能，不需要保留集，比之前的强制遗忘方法稳健得多。**
