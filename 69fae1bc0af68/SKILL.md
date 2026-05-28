---
id: 69fae1bc0af68
name: RankMixer
tags: []
updated_at: 2026-05-19T09:45:06.043850Z
---

## 在模型中的位置（model.py:966-970）

RankMixer 是 `MultiSeqHyFormerBlock` 中**唯一的跨序列融合通路**。

```python
combined = torch.cat(decoded_qs + [ns_tokens], dim=1)  # (B, Nq*S + Nns, D)
boosted = self.mixer(combined)
```

```
seq_a (B,La,D) ─┐ encoder_a ─┐ cross-attn_a ─┐ decoded_q_a (B,Nq,D)
seq_b (B,Lb,D) ─┤ encoder_b ─┤ cross-attn_b ─┤ decoded_q_b (B,Nq,D)
seq_c (B,Lc,D) ─┤ encoder_c ─┤ cross-attn_c ─┤ decoded_q_c (B,Nq,D)
seq_d (B,Ld,D) ─┘ encoder_d ─┘ cross-attn_d ─┘ decoded_q_d (B,Nq,D)
                                                        │
               ns_tokens (B,Nns,D) ─────────────────────┤
                                                        ▼
                           concat → (B, Nq*4+Nns, D)  ← T 个 token
                                                        │
                                          ┌─ RankMixerBlock ─┐
                                          │  token-mixing 在此 │
                                          └──────────────────┘
                                                        │
                                                        ▼
                         split → next_q_a/b/c/d, next_ns
```

默认参数：`Nq=2, S=4, Nns≈3` → `T = 2×4+3 = 11` 个 token。

step 1/2 各 seq 独立 encode / cross-attn，彼此隔离；所有跨序列（a↔b↔c↔d）及跨域（seq↔NS）交互**全在这一步**。

## 名字来历

`RankMixer` 是 PCVRHyFormer 内部模块名（`RankMixerBlock` / `RankMixerNSTokenizer`），不是独立论文。

| 词 | 含义 |
|----|------|
| Rank | 张量的秩/维度重排（≠ 排序 ranking） |
| Mixer | 同 MLP-Mixer：token-mixing + channel-mixing 的思路 |

`full` 模式下将 `(B, n_total, D)` reshape 为 `(B, n_total', D')` 再 transpose，本质是低秩 token 重组——靠维度重排做 token 交互，无可学参数。

## Block 模式对比

| mode | token-mixing（无参 reshape） | per-token FFN（SwiGLU+LN） | seq 间交流 |
|--------------|-----------|----------|-----------------|
| `full` | ✅ | ✅ | reshape rewire + 各自 FFN |
| `ffn_only` | ❌ | ✅ | 无；4 seq query 互不可见 |
| `none` | ❌ | ❌ | 零；block 退化为 identity |

## 两步做了什么

**token-mixing（无参，纯 reshape）**

```python
# (B,T,D) → (B,T,T,d_sub) → transpose(1,2) → (B,T,D)
# 把每个 token 的 D 维拆成 T 块，与其它 token 的对应块互换位置
# 无可学参数，纯位置 rewire
```

**为什么要转置**

`nn.Linear` 只对最后一维做矩阵乘（`y = x @ W`，W 动 `x.shape[-1]`）。
要让 MLP 跨 T 轴混合 token，必须把 T 挪到最后：

```python
# 标准 MLP-Mixer token-mixing
x = x.transpose(-1, -2)   # [B,T,D] → [B,D,T]  ← T 到末尾
x = token_mlp(x)          # Linear(T,T) 对 T 轴 mix
x = x.transpose(-1, -2)   # 转回 [B,T,D]
```

不转置 → MLP 只能做 channel-mixing（D 轴）。RankMixer 用 reshape 替代了 `Linear(T,T)`（无参数），但转置的逻辑相同：把想 mix 的轴挪到 Linear 能碰到的位置。

| | Self-Attention | MLP-Mixer | RankMixer |
|---|---|---|---|
| 跨 token 权重来源 | softmax(QKᵀ)，data-dependent | `Linear(T,T)`，学到的 | reshape rewire，无参 |
| 是否需要转置 | 否，矩阵乘自带 T×T | 是 | 是（reshape + transpose） |
| 复杂度 | O(T²D) | O(T²D) | O(T²D)，常数小 |

**per-token FFN（有参，参数共享）**

```
LayerNorm → Linear(D→4D) → GELU → Linear(4D→D)
每个 token 独立过同一份 FFN，增加非线性深度，不交换 token 信息
```

**Residual: `Q_boost = post_norm(Q + Q_e)`**

```
Q ───────────────────────────────────────────────────┐  (恒等通路)
│                                                    │
├─→ token_mixing → norm → fc1 → GELU → dropout → fc2 → Q_e ──┤
   (token-cross-attn)  (D→4D)   (gating)        (4D→D)       │
                                                          Q + Q_e
                                                             │
                                                          post_norm
                                                             │
                                                           Q_boost
```

- `Q`：输入 token 集（cross-attn 后的 decoded_qs + ns_tokens 拼接）
- `Q_e`：token_mixing + FFN 算出的增量（"Q + 交互作用 Qe"），不替换 Q
- 初期 fc2 权重 ≈ 0 → `Q_e ≈ 0` → `Q_boost ≈ Q`，等价恒等映射，梯度直通，训练稳定
- FFN 是 block **唯一有学习容量**的组件：token_mixing 零参，norm 近线性 → 去掉 FFN 等于零参 permutation，训练无效



## vs v7.6 selective_query（TQFBlock.boost）

| 维度 | v2 RankMixerBlock | v7.6 TQFBlock.boost |
|------|-------------------|---------------------|
| token 交互 | reshape 固定 permutation（无参） | boost_token MLP（可学） |
| channel mixing | 共享 FFN | boost_chan MLP |
| 参数量 | 轻（一个 FFN） | 重（两个 MLP） |
| 表达力 | 弱（routing 固定） | 强（两个 MLP 都能学） |
| 设计哲学 | 省参数，让 attention 学交互，mixer 只做 reshape + 非线性 | mixer 也学，配合 reliability-weighted summary 实现 selective |

## 消融意义

- `full → ffn_only`：token-mixing 去掉后 AUC 几乎不掉 → attention 已完成跨序列交互，reshape 冗余
- `full → none`：AUC 显著掉 → FFN 非线性深度是真正贡献
- `ffn_only → none`：隔离纯 FFN 边际价值
- `none` 掉得少 → 说明核心信号不在跨序列交互（也是有意义的发现）

TAGS: ranking, token-mixing, ffn, ablation, transformer
