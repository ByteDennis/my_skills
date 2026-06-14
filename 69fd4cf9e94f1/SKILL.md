---
id: 69fd4cf9e94f1
name: Ads Recommend
tags: [recommender, ads, conversion, auc, calibration, rss]
updated_at: 2026-06-14T00:38:05.316900Z
---

## 核心指标 (每 epoch 必出)

| 指标 | 直觉理解 |
|------|----------|
| `AUC=0.8541` | 随机抽一个转化用户和一个未转化用户，模型给前者打分更高的概率是 85.4% |
| `PR-AUC=0.4839` | 转化率只有 9.6%，AUC 会被大量易分负样本"稀释"；PR-AUC 只看正样本被召回的质量 |
| `LogLoss=0.2305` | 模型说"转化概率=0.9"但没转化，比说"0.6"没转化罚更重；对概率校准敏感 |
| `pos_loss=1.620` | 平均每个转化样本的 loss；高说明模型低估了转化用户 |
| `neg_loss=0.0817` | 平均每个非转化样本的 loss；低是正常的，负样本太多太容易学 |
| `gap=+0.020` | valid_loss − train_loss；正数=过拟合，gap 越大越需要正则 |

## 加权 & 时间切片 AUC

| 标签 | 直觉理解 |
|------|----------|
| `valid_weighting=0.8487` | 给近期样本更高权重后 AUC 略降 → 模型对旧数据拟合更好，新鲜度有损 |
| `night_filter=0.8501` | 只看 0–6 点行为；夜间用户行为稀疏，AUC 低说明夜间特征弱 |
| `gaussian_kernel=0.8587` | 最近几小时样本权重最大，AUC 最高 → 近期行为对排序最有帮助 |
| `time_range_last24h=0.8519` | 只切最近 24h；如果比全量低，说明模型对最新分布有漂移 |

## 预测质量快照 (`eval-extra`)

| 字段 | 直觉理解 |
|------|----------|
| `calib=1.022` | pred_rate/pos_rate=0.098/0.096；模型平均多预测了 2% 的转化，轻微高估 |
| `sep=0.4123` | 转化用户平均得分 0.452，非转化用户 0.040，差 0.41；越大正负越好分开 |
| `prob_hist=70/15/10/4/1%` | 70% 样本预测概率 <10%；分布极度左偏，符合低转化率场景 |

## 高损失正样本 (`pos-top-N`)

> Top-500 = loss 最大的转化样本，即"模型最不相信会转化但实际转化了的用户"

| 字段 | 直觉理解 |
|------|----------|
| `share=42.1%` | 500/9844 个正样本（5%）扛了 42% 的正样本 loss — 少数难例主导训练信号 |
| `pred_p=0.014 vs 0.452` | 这 500 人模型只给了 1.4% 的转化概率，全量正样本均值是 45% — 严重漏判 |
| `hour_dist(top)` | Top500 中夜间(0-4h)占比远高于全量 → 夜间转化用户特别难学 |

## 校准表 (`calib` decile)

把样本按预测概率从低到高分 10 桶，比较桶内均值 vs 实际正率：

```
decile=0  pred=0.001  actual=0.001  ✓ 低分段准
decile=9  pred=0.613  actual=0.534  ✗ 高分段高估 0.08（最大问题）
ece=0.0241   brier=0.1350
```

高置信度区间高估 → 上线前需 Platt Scaling 或 isotonic 校准。

## 特征重要性 (`per-domain-LOO`)

屏蔽某块特征后 AUC 的下降量（下降越多 = 该块越重要）：

| 特征块 | ΔAUC | 理解 |
|--------|------|------|
| `ns` 静态特征 | −0.0079 | 用户/商品基础属性，信息量最大 |
| `seq_d` | −0.0041 | 最近行为序列（freshness≈50min），时效性强 |
| `seq_a/b/c` | −0.003/0.002/0.001 | 其他行为域，贡献递减 |

## 用户级明细 (`user-breakdown`)

```
wrong_pos=4521   真实转化用户但模型排在后 90% → 这批人是重点优化对象
wrong_neg=11884  非转化用户但模型排在前 10% → 误召回，浪费曝光
阈值 p=0.097     业务上"认为会转化"的分界线
```

## Sequence Token 编码 (SemanticSeqEmbedder)

每个 seq token 携带多个 fid（如 seq_a = 9 个 fid），按语义分 3 类 role：

| role | 含义 | 例子 |
|------|------|------|
| `item` | token 指向的 item 本身 | `item_id`, `item_category_id`, `item_tag` |
| `action` | 用户对该 item 的动作 | `action_type`（click/like/share，vocab ≤ 16）|
| `stat` | 该时刻的统计上下文 | `recency`, `dwell_time`, `behavior_count` |

**Token 表示公式：**

```
token = Σ_fid (fid_token_emb + role_emb[role_of_fid]) + time_emb + domain_bias
```

`role_emb = nn.Embedding(4, D)`，让模型显式区分 fid 语义，无需从 fid id 自己猜。

**启发式 role 分类规则** (`scripts/classify_seq_roles.py`)：

| 条件 | 分配 role |
|------|-----------|
| fid ∈ `item_ns_groups.json` | `item` |
| `vocab_size ≤ 16` | `action` |
| 其他 | `stat` |

**实测分布** (5/18，`experiments/20260518/eda/seq_role_assignment.json`)：

| domain | fids | item | action | stat |
|--------|------|------|--------|------|
| seq_a | 9 | 0 | 1 | 8 |
| seq_b | 14 | 0 | 0 | 14 |
| seq_c | 12 | 0 | 2 | 10 |
| seq_d | 10 | 0 | 2 | 8 |

> item role 全为 0 — `item_ns_groups.json` 可能未覆盖 seq fid，需复查映射。

## TQF Block 架构（× 3 堆叠）

```
INPUT
Q ∈(B,Nq,D)    H_A,H_B,H_C,H_D ∈(B,L_d,D)    H_ns ∈(B,Nns,D)    z_t ∈(B,D)
│              │  │  │  │                     │                   │
│              ▼  ▼  ▼  ▼                     │                   │
│           ┌──────────────────┐               │                   │
│           │ ① per-domain     │  Q × H_d 各跑一份                   │
└─────────►│ cross-attn × 4   │                                    │
            └──────────────────┘                                    │
                │  │  │  │                                          │
                ▼  ▼  ▼  ▼   Q_A, Q_B, Q_C, Q_D ∈(B,Nq,D)            │
              ┌──────────────────────────────────┐                  │
              │ ② Reliability MLP per domain     │ 6D contrast feat │
              │   [ev_d; ev_neg; q; ns; z_t; q_d]│ ◄──────────────  │
              │   → score_d                      │                  │
              └──────────────────────────────────┘                  │
                            │                                       │
                            ▼   scores ∈(B, 4)                      │
              ┌──────────────────────────────────┐                  │
              │ ③ Sharpened Softmax              │                  │
              │   α = softmax(exp(scale)·s/0.45) │                  │
              │   + domain_bias + invalid mask   │                  │
              │   → α ∈(B, 4)                    │                  │
              │   last_alpha_entropy → 训练 loss │                  │
              └──────────────────────────────────┘                  │
                            │                                       │
                            ▼   weighted sum                        │
              Q_seq = Σ_d α_d · Q_d   ∈(B, Nq, D)                  │
                            │                                       │
                            ▼                                       │
              ┌──────────────────────────────────┐                  │
              │ ④ Self-Refinement                │                  │
              │   cat([Q, Q_seq]) → self-attn    │                  │
              │   → split → Q', Q_seq'           │                  │
              └──────────────────────────────────┘                  │
                            │                                       │
                            ▼                                       │
              Z = cat([Q', Q_seq', H_ns])  ◄────────────────────────┤
                            │                                       │
                            ▼                                       │
              ┌──────────────────────────────────┐                  │
              │ ⑤ Boost Mixer                    │                  │
              │   Z += boost_token(LN(Z))        │                  │
              │   Z += boost_chan (LN(Z))        │                  │
              └──────────────────────────────────┘                  │
                            │                                       │
                            ▼   切回 Z_q, Z_seq, Z_ns                │
              ┌──────────────────────────────────┐                  │
              │ ⑥ γ Write-Back (learnable scale) │                  │
              │   Q_new   = Q   + γ_q  · (Z_q + Z_seq)  γ_q =0.01   │
              │   H_ns_new= H_ns+ γ_ns ·  Z_ns          γ_ns=0.03   │
              └──────────────────────────────────┘                  │
                            │                                       │
                            ▼                                       │
OUTPUT: Q_new ∈(B,Nq,D),  per_domain_tokens (默认 unchanged),  H_ns_new ∈(B,Nns,D)
                            └──→ 进下一个 TQF block (× 3 总共)
```

**参数说明：**

| 符号 | 形状 | 含义 |
|------|------|------------|
| $Q$ | $(B, N_q, D)$ | 输入 query tokens |
| $H_d$，$d\in\{A,B,C,D\}$ | $(B, L_d, D)$ | 第 $d$ 域序列编码（各域长度可不同）|
| $H_{ns}$ | $(B, N_{ns}, D)$ | 静态特征 tokens（user/item sparse+dense）|
| $z_t$ | $(B, D)$ | target item embedding |
| $Q_d$ | $(B, N_q, D)$ | $Q$ cross-attend $H_d$ 后的 per-domain query |
| $\text{score}_d$ | $(B, 1)$ | Reliability MLP 输出的域可靠性得分 |
| $\alpha$ | $(B, 4)$ | sharpened softmax 后的域融合权重 |
| $Q_{seq}$ | $(B, N_q, D)$ | $\sum_d \alpha_d \cdot Q_d$，加权融合结果 |
| $Q',\,Q_{seq}'$ | $(B, N_q, D)$ | Self-Refinement 后的精炼 query |
| $Z$ | $(B,\,2N_q+N_{ns},\,D)$ | $\text{cat}[Q',\,Q_{seq}',\,H_{ns}]$，送入 Boost Mixer |
| $\gamma_q$ | scalar | Q write-back 可学习缩放（init $0.01$）|
| $\gamma_{ns}$ | scalar | $H_{ns}$ write-back 可学习缩放（init $0.03$）|
| $Q_{new}$ | $(B, N_q, D)$ | 输出 query，进下一 TQF block |
| $H_{ns,new}$ | $(B, N_{ns}, D)$ | 更新后的静态特征 tokens |

> Reliability MLP 的输入是 6 段拼接：$[\mathbf{ev}_d;\,\mathbf{ev}_{neg};\,\mathbf{q};\,\mathbf{ns};\,z_t;\,\mathbf{q}_d]$，显式引入 target item 做对比，让 score 反映"该域历史与当前 item 的相关度"。

## Reliability Score 计算细节

>   Reliability 直觉: MLP 同时看到"自己说什么" + "别人说什么" + "原始问题是什么" + "context (NS + 时间)", 输出一个标量表示 "在当前 (B, user, item, time) 上下文, 这个 domain 的证据有多可信"。

**特征构造**（`selective_query_blocks.py TQFBlock`，4 个 domain 共享同一个 MLP）：

```python
per_domain_q_mean = Q_stack.mean(dim=2)          # (B, 4, D)
ev      = per_domain_q_mean[:, d, :]             # (B, D) 当前域 Q_d 均值
ev_neg  = (其他 3 个域之和) / 3                   # (B, D) 其他域平均
q       = Q.mean(dim=1)                          # (B, D) 原始 query 均值
ns      = H_ns.mean(dim=1)                       # (B, D) NS token 全局均值
z_t     = z_t                                    # (B, D) 时间向量
ev_dup  = ev                                     # (B, D) ⚠️ stub bug，重复 ev

feat_d  = cat([ev, ev_neg, q, ns, z_t, ev_dup], dim=-1)  # (B, 6D)
score_d = reliability_mlp(feat_d).squeeze(-1)            # (B,)
```

**MLP 结构**（参数量 $\approx 6D^2$，$D{=}128$ 时约 98K，4 域共享）：

```python
nn.Sequential(
    nn.LayerNorm(6*D),
    nn.Linear(6*D, D),
    nn.SiLU(),
    nn.Linear(D, 1),
)
```

**6 个特征的直觉：**

| # | 特征 | 给 MLP 看什么 |
|---|------|--------------|
| 1 | $\mathbf{ev}$（当前域证据） | 当前域 $Q_d$ 长啥样 — "这个域给了我什么信号" |
| 2 | $\mathbf{ev}_{neg}$（其他域均值） | 其他域综合信号 — "别的域在说啥，跟我有冲突吗" |
| 3 | $\mathbf{q}$（原始 query） | query token 自身状态 — "我本来想问什么" |
| 4 | $\mathbf{ns}$（NS summary） | user+item NS tokens 全局均值 — "当前 user/item 长啥样" |
| 5 | $z_t$（时间） | 全局时间上下文 — "现在是什么时间窗" |
| 6 | $\mathbf{ev}_{dup}$ ⚠️ | stub bug，重复了 $\mathbf{ev}$，应替换为其他特征 |

## Sharpened Softmax & Self-Refinement

> 普通 softmax 容易出"均匀分布": 4 个 domain 各 ~25%。 但 v7.6 的核心假设是 "不同样本应该信不同 domain" — 比如这个用户行为序列 seq_b 信号强, 那 α_B 应该到 0.7+, 不是 0.3。

**③ Sharpened Softmax**

```
α = softmax(exp(scale) · score / 0.45)  + domain_bias + invalid_mask
```

| 参数 | 作用 |
|------|------|
| $\tau = 0.45$ | 固定温度，比默认 1 小 → logits 更尖，赢家通吃 |
| `exp(scale)`，init=0 | 可学习缩放；训练自动调"决断度"，分不开就加大，过拟就缩小 |
| `last_alpha_entropy` → loss | entropy penalty，防 $\alpha$ 退化成 100% 单域 |

> 直觉：$\tau=1$ 是民主投票（4 域各 ~25%）；$\tau=0.45$ 是主席投票（谁稍高就拿大头）。entropy penalty 是防独裁的少数派保险。

**④ Self-Refinement**

> [!NOTE]
> 刚才 sharpened softmax 出 Q_seq 是 "4 个 domain 加权融合的证据", 但是原始 Q 还是 query 自己的状态, 两者来自不同的语义空间，直接喂 mixer 会信号打架。Self-Refinement 让两路互相 attend 一次：
> - $Q$ 看 $Q_{seq}$："你抽到的证据跟我想问的 align 吗"
> - $Q_{seq}$ 看 $Q$："我漏了你关心的角度吗"


```
cat([Q, Q_seq], dim=1)   # (B, 2·Nq, D)
    → self-attn
    → split → Q',  Q_seq'
```

输出 $Q'$、$Q_{seq}'$ 均已互相校准，再进 Boost Mixer 不冲突。

## γ Write-Back（⑥）

```python
Q_new    = Q    + γ_q  * (Z_q + Z_seq)   # γ_q  = 0.01
H_ns_new = H_ns + γ_ns *  Z_ns           # γ_ns = 0.03
```

不直接 `Q = Z_q` 替换，原因：

| 原因 | 说明 |
|------|------|
| **3 层堆叠稳定性** | 每层只贡献 1–3% 修改，3 层累积 ~5%，防信号爆炸/塌缩（无 residual 的深 transformer 同病）|
| **训练初期 identity** | $\gamma$ init ≈ 0 → block 起步等价透传，下层先稳定，再慢慢"开始调味"（LayerScale / ReZero 套路）|
| **双路分速更新** | $\gamma_q=0.01$（Q 是演化探针，慢调）vs $\gamma_{ns}=0.03$（NS 静态画像，可稍激进灌入跨域信号）|

**v2 RankMixer vs v7.6 Write-Back：**

| | v2 RankMixer | v7.6 γ Write-Back |
|---|---|---|
| residual 强度 | $Q + Q_e$（$\beta=1$，全开）| $Q + \gamma\Delta Q$（$\beta\approx0$，近 identity）|
| 训练初期 | block 已开始工作 | block 是 identity |
| 控制变量 | 无 | $\gamma_q$ / $\gamma_{ns}$ 各自可学 |
| 适合层数 | 2–3 层 | 3+ 层（LayerScale 设计）|

> v2 用 $\beta=1$ 因为只有 2 层 + 一个全连接，信号平衡；v7.6 有 3 层 × 6 子步骤，需要 $\gamma$ 刹车。

## HyFormer Block 信息流

每个 block 内 5 个阶段，4 条序列流水线并行：

| 阶段 | 操作 | 粒度 |
|------|------|------|
| [1] seq self-attn | `seq_encoders[i](seq_tokens_list[i])` | 同序列内 token 互看，各 seq 独立 |
| [2] cross-attn | `cross_attns[i](q_tokens, seq_i)` | Q_i 只 attend 自己的 seq_i，仍独立 |
| [3] token fusion | `cat(decoded_qs + [ns_tokens], dim=1)` → shape `(B, Nq*S + N_ns, D)` | **跨序列 + 跨域唯一汇合点** |
| [4] RankMixerBlock | token mix (转置+MLP) + channel mix | 任意 token 影响任意 token |
| [5] split back | 拆回 per-seq Q + NS，进入下一 block | 多 block 累积跨域信号 |

**交互强度速查：**

| 交互方向 | 在哪发生 | 强度 |
|----------|----------|------|
| 同 seq 内 item ↔ item | [1] self-attn | 高 |
| Q_i ↔ seq_i | [2] cross-attn | 高 |
| Q_i ↔ seq_j（跨域查询） | ❌ 无直接通道，经 mixer 隐式 | 弱 |
| seq_a 细节 ↔ seq_b 细节 | ❌ [2]后压为 Nq query，跨序列只见摘要 | 弱 |
| dense ↔ sparse ↔ seq summary | [4] mixer token mixing | 中（固定 MLP 权重） |
| 跨 block 累积 | [5] split → 下一 block | 看 `num_blocks`（默认 2） |

> 为何用 2 个 block：1 个 block 跨域只交互 1 次；2 个 block = 交互 → 各自 refine → 再交互。

**Architecture 调优方向：**
- 强化跨序列细粒度交互 → 加 inter-sequence cross-attn，不能只靠 mixer 的 pooled query
- 强化 dense ↔ seq 通信 → 把 dense 展开成多 token（现在只 1 个），mixer 才能精细 mix

## 高置信误召回分析 (`neg-top-N`, 建议新增)

> 对称于 pos-top-N：找出 pred_p 最高的 Top-N 负样本（"模型最确信会转化但实际没转化"）

| 字段 | 为什么有用 |
|------|-----------|
| `hour_dist(top_neg)` | 与 pos-top-N 的小时分布对比，若夜间同时出现 → 夜间特征本身噪声大 |
| `seq_a_len top_neg` | 若误召回用户序列很长 → 模型误把"活跃"当"会转化"，需加转化专属特征 |
| `pred_p mean(top_neg)` | 模型给这些人打了多高的分；越高说明模型越"自信地错" |
| overlap with wrong_neg | top_neg ∩ wrong_neg 的占比；高度重叠说明误召回集中在少数"迷惑性"用户 |
