---
id: 69fae1bc0af68
name: RankMixer
tags: []
updated_at: 2026-05-08T04:31:52.144146Z
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
               ns_tokens (B,Nns,D) ────────────────────┤
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
|------|------------------------------|----------------------------|-----------|
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

**per-token FFN（有参，参数共享）**

```
LayerNorm → Linear(D→4D) → GELU → Linear(4D→D)
每个 token 独立过同一份 FFN，增加非线性深度，不交换 token 信息
```

## 消融意义

- `full → ffn_only`：token-mixing 去掉后 AUC 几乎不掉 → attention 已完成跨序列交互，reshape 冗余
- `full → none`：AUC 显著掉 → FFN 非线性深度是真正贡献
- `ffn_only → none`：隔离纯 FFN 边际价值
- `none` 掉得少 → 说明核心信号不在跨序列交互（也是有意义的发现）

TAGS: ranking, token-mixing, ffn, ablation, transformer
