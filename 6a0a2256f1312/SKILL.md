---
id: 6a0a2256f1312
name: Experiment Tracking
tags: []
updated_at: 2026-05-19T00:08:20.941699Z
---

## Run Comparison

| 维度 | 5/10 win_strtgy | 5/16 sweep best (50k) |
|-------------|-----------------|----------------------|
| `seq_c` / `seq_d` maxlen | 512 / 512 | 256 / 128 |
| `time_feature_mode` | `sample_periodic_seq_stats`（offsets 918–934）| `none` |
| `drop_user_int_fids` | [101–109]（9 个）| [] |
| `drop_item_int_fids` | [83, 84, 85]（3 个）| [] |
| `user_int_dim` 实际 | 402（drop 后，base=411）| 411（no drop）|
| `user_dense_dim` | 940（含 22 个 time feature 位）| 918（无 time feature）|
| 模型总参数 | 239,793,985（240M）| ⚠️ 见下方注释 |
| loss | `bce, focal α=0.1 γ=2.0` | `bce, focal α=0 γ=0` |
| `emb_skip_threshold` | 1,000,000 | 50,000 |
| `batch_size` | 256 | 1024 |
| **best val_auc** | **0.86453**（ep5）| 0.8438（ep10）|

> [!NOTE]
> 上表"5/16 sweep best (50k)"列的模型总参数标注为 5.1M **是错的**——5.1M 来自 `emb_skip=5000` variant，不是 `emb_skip=50000` best variant。真正的 50k best 约 30–80M。

## win_strtgy vs sweep：4 项关键差异（按贡献排）

| # | 改动 | Δ AUC 估 | 为什么有效 |
|--|---------|------|--------------|
| 1 | `seq_max_lens` 加倍（seq_a 128→256，seq_c/d 256→512）| +0.002–0.004 | 每条样本能看到的用户行为历史翻倍；更长 history 让 cross-attn 更容易找到与候选广告相关的 (item, behavior) 配对。5/18 错例 F- 聚集在"cold-start / 长尾"用户，正是 history 太短导致 |
| 2 | `time_feature_mode`<br>=sample_periodic_seq_stats | +0.002–0.003 | 在每个 seq token 上加周期性时间统计（hour-of-day sin/cos, token-target time_diff 等），直接对应 5/18 错例中 morning AUC=0.807（gap=0.087）这个时段 gap |
| 3 | `drop_*_fids` 12 个（user 101–109，item 83–85）| +0.001–0.002 | 这批 fid `drop_nonseq_sparse` probe Δ=0，删掉减噪声 + 释放容量给有效 fid |
| 4 | `focal α=0.1 γ=2.0` | +0.001 | γ=2.0 强压 hard-positive；α=0.1 与 γ 组合净效果是 mild regularization，ep1 收益小，ep4–6 可能更大 |

## emb_skip Variants（5/16 sweep）

`sparse 参数 = Σ vocab_size × emb_dim(64)`，阈值越低踢掉的 fid 越多，模型越小。

| variant | giant 表 skipped（user/item）| seq fids skipped | sparse 参数 |
|-----------------|-------------|--------------------|-------------|
| `emb_skip=5000` | 0 / 2 | a 3/8, b 7/13, c 6/11, d 3/9, item_ns 2/14（19 个）| **3.05M**（实测）|
| `emb_skip=10000` | 0 / 2 | a 2/8, b 6/13, c 4/11, d 3/9, item_ns 2/14（17 个）| ~5–10M |
| `emb_skip=50000`（best）| 0 / 0 | a 1/8, b 4/13, c 4/11, d 2/9（11 个）| ~30–80M |
| `emb_skip=200000` | 0 / 0 | 更少 | ~150M |
| `emb_skip=1000000` | 0 / 0 | a 0/8, b 1/13, c 3/11, d 0/9（~4 个）| ~220–240M |
| `win_strtgy emb_skip=1M` | 0 / 0 | b 1/13, c 3/11（4 个）| **237M**（实测）|

**结论**：5/16 sweep `emb_skip=1M` variant ≈ 220–240M ≈ `win_strtgy`，两者可比。
5.1M（`emb_skip=5000`）与 240M（`win_strtgy`）踢掉的 fid 差异悬殊，**不可直接比较**。

### emb_skip 作用范围

`emb_skip_threshold` 对每个 fid 的 vocab_size 独立判断，只影响走 `nn.Embedding` 的 sparse 输入：

| 输入 | fid 数 | 说明 | emb_skip=1M 时跳过 |
|------------|----|------------------|---------------|
| `user_int_feats` | 402 | → `RankMixerNSTokenizer.user`，每 fid 独立 emb 表，concat 后切 5 块投影 | 0 / 402（最大 vocab=2,848）|
| `item_int_feats` | 30 | → `RankMixerNSTokenizer.item`，切 2 块 | 0 / 30（最大 vocab=23,700）|
| `seq_a`（8 fids/token）| 8 | `_make_seq_embs`；跳过的 fid → zero vector | 0 / 8 |
| `seq_b`（13 fids/token）| 13 | 同上 | 1 / 13（item_id 64M 级）|
| `seq_c`（11 fids/token）| 11 | 同上 | 3 / 11（高基数 sideinfo）|
| `seq_d`（9 fids/token）| 9 | 同上 | 0 / 9 |

每个 seq token 是 `(item_id, brand_id, category_id, ..., time_bucket_id)` 的复合 token，各 fid embedding 相加表示一个时刻的行为；跳过的 fid 对该 token 贡献 zero。

**不受影响**：

| 输入 | 原因 |
|------|-----------|
| `user_dense_feats`（918 维）| 走 `nn.Linear(918, 64)`，dense float，不查表 |
| `item_dense_feats`（0 维）| TAAC 数据 item 无 dense feature |
| `seq_*_time_bucket` | 独立 emb 表，vocab=64，远低于任何阈值 |
| `seq_*_len` | scalar，只进 attention masking |

## Loss Spike 分析（~step 150 飙到 2.65）

**不是 BF16 精度问题**：cascade 全程 `amp_enabled=False`，FP32 训练，5/15、5/16 sweep、5/10 win_strtgy 的启动 log 均确认。

**根因：sparse Adagrad cold-start touch**

| 步骤 | 说明 |
|------|---------------------------|
| 时间窗 | 每 epoch ≈ 193 step；spike 在 ~step 150，mid epoch 1，`reinit_sparse_after_epoch=1` 在 step 193 后才触发，排除 epoch 边界 reinit |
| 触发条件 | 前 ~150 step 扫了 ~150K samples（bs=1024），第一次撞到某罕见高基数 fid（vocab 几千到几万的尾部 fid） |
| 机制 | 该 fid 的 embedding row 仍是随机初始化；sparse Adagrad `accumulator ≈ 0` → `update = lr · grad / (√accum + ε)`，有效幅度接近 `lr · sign(grad)`，远大于已热身 fid 的更新幅度 |
| 放大 | 同一 batch 该 fid 落在正样本 → 一次 update 把 row 推得极远 → 下一 batch 给出极端 logit → loss 飙升 |
| 自愈 | spike 后 Adagrad `accum += grad²` 单调增，该 fid 的有效 lr 自动衰减，不再重演；可见 loss 从高方差 0.2–0.3 收敛为超稳 0.2–0.3 |

> [!NOTE]
> dense AdamW 有 momentum + variance estimate，一次 update 不会飞；spike 几乎只出现在 **sparse Adagrad** 上，属正常 cold-start 特征，无需干预。

## ideas.yaml（18 个 idea，5 个 disabled）

| 类别 | idea | enabled | 备注 |
|------|------|---------|------|
| sparse-side | `lr_sparse_aggressive` | ✓ | 5/18 L2 winner |
| sparse-side | `wd_sparse` | ✓ | 5/18 加 |
| reinit | `reinit_warmup0` | ✓ L3-only | `skip_levels=[L1,L2]` |
| dense-side | `lr_dense_anneal` | ✗ | 0.25 ee 没信号 |
| dense-side | `wd_dense` | ✓ | 5/18 加 |
| regularizer | `dropout_005` | ✓ | key bug 修了 |
| regularizer | `dropout_010` | ✓ L3-only, reset_optim | 5/18 改 |
| combined | `lr_anneal` | ✓ | 5/18 L1 winner |
| loss | `focal_soft` | ✓ | 5/17 L1 winner |
| loss | `focal_soft_v2` | ✗ | γ=0.3 方向反 |
| loss | `focal_strong`（新）| ✓ | γ=2.0/α=0.75 + pw=15，攻 F-/F+=7.3× |
| loss | `loss_bce_pos_weight_v2` | ✓ | bce.py 静默 bug 修了 |
| loss | `label_smoothing` | ✓ | 5/18 加 |
| sample | `parquet_boost` v1 | ✗ | warm-start 过训，用 v2 |
| sample | `parquet_boost_v2`（新）| ✓ | reset_optim + 2ep + boost=3，攻 pred_p valley@q3 |
| arch | `hash_bucket_100k` | ✓ | 优先级降 |
| arch | `ns_groups` | ✓ | 没跑过 |
| feature_mask | `dense_only_probe`（新）| ✗ 待基建 | 验证 drop_nonseq_sparse Δ=0 假设 |
| feature_mask | `drop_more_sparse`（新）| ✗ 待 schema | 扩 `drop_user_int_fids` 到 zero_frac>0.85 |
| feature_mask | `hour_feature_explicit`（新）| ✗ 待 baseline | hour bucket valley@q2 信号 |

## 超参覆盖度（v2/train.py，按优先级排）

🔴 高优先 · 🟡 中优先 · ✅ 已对齐

| 超参 | v2 default | 当前 ursl / cascade | 含义 |
|-------|---|---|------------|
| 🔴 `dense_transform` | `'none'` | `'none'` | 918 维 dense feature 喂进 linear 前的预处理。`'none'`=原始值；`'signed_log1p_clip'`=压缩长尾（`sign(x)·log1p(|x|)` clip ±20）；`'robust_clip'`=用 median+IQR 标准化后 clip ±20。dense 混合 counts/rates/timestamps 等多量纲，raw 值让梯度被最大量级支配 |
| 🔴 `warmup_steps` | `0` | `0` | 训练前 N 步将 AdamW + Adagrad lr 从 0 线性升到目标值。`0`=第一步直接用目标 lr。主要作用：让 Adagrad accumulator 先攒起来再加速，抑制 ep1 sparse cold-start spike |
| 🔴 `d_model` / `emb_dim` | `64 / 64` | `64 / 64` | transformer 内部维度 / sparse embedding 维度，通常绑定。翻倍到 128 ≈ 参数×4、显存×4、耗时×4。当前 dense+transformer 才 2.4M params，64 维 attention 在 user-item 交互上表达空间有限 |
| 🔴 `num_hyformer_blocks` | `2` | `2` | PCVRHyFormer 的 transformer block 数（attention+FFN 层数）。2→4 加深一倍，≈2× 计算量。更多层让 user/item feature 有更多交互机会；与 `d_model` 互补（宽度 vs 深度）|
| 🟡 `use_rope` | `False` | `False` | 用 Rotary Position Embedding 替换默认的 learned absolute position emb。RoPE 把位置编进 QK rotation，天然支持任意长度；learned pos emb 在尾部位置（e.g. 510）数据稀疏，表征差 |
| 🟡 `seq_top_k` | `50` | `50` | seq attention 只看每个序列里 top-k 个最相关 item。过小丢信息，过大退化为 full attention。当前 50 未扫过，调 20/100 可能给信号 |
| 🟡 `hidden_mult` | `4` | `4` | transformer FFN 内部维度倍率：FFN 宽度 = `d_model × hidden_mult`。FFN 占 transformer ≈80% 参数，`d_model=64` 时 4× 可能不够，试 8（FFN 512 维）|
| 🟡 `amp_dtype` | `'none'`| `'bf16'` | 混合精度训练。`'none'`=全 FP32；`'bf16'`=前向/反向用 bf16，权重/optimizer state 仍 FP32。不影响 AUC，但训练快 1.5–2×、显存少 30–50%；是跑更大模型的 enabler |
| ✅ `num_queries` | `1` | `2`（5/18 修）| 每个 seq domain 配几个 query token。`=2` + 4 seq → 8 query token；`=1` → 4 个。query token 从每个 seq attention 出一个 d_model vector 进 transformer，更多 query 增加表达力 |
| ✅ `user_ns_tokens` / `item_ns_tokens` | `0/0`（fallback 7/4）| `5/2`（5/18 修）| RankMixerNSTokenizer 把 user_int/item_int 等分成 N 段，每段投影到 d_model 当 NS token。chunk 越大每个 token 信息越浓；win_strtgy 用 5/2（chunk 474/352），默认 fallback 7/4（chunk 339/176）|
| ✅ `seq_max_lens` | 128/256 | win_strtgy 配置（sweep）| 每个 seq domain 截断/padding 的最大长度。直接决定输入 tensor 的 seq 维度，与显存正相关, 比如 `{a:128,b:256,c:256,d:128}` |
| ✅ `batch_size` | `256` | `256`（sweep）/ `1024`（cascade）| 每次梯度更新看几个样本。小 batch 给 sparse Adagrad 更多 update 次数，有助于 rare fid 收敛；cascade 1024 是 OOM-safe 选择 |
| ✅ `time_feature_mode` | `'none'` | - | 给 dense 加时间衍生特征。`'sample_periodic_seq_stats'`=对每个 seq 的 timestamp 算 max/min/mean/std（共 22 位，offsets 918–934）|
| ✅ `drop_*_fids` | `''` | - | 跳过指定 fid（不进 dataset）。高缺失（zero_frac 高）的 fid 留着是噪声，删掉减少干扰, 比如 `[101–109]` + `[83–85]` |
| 🟡 `weight_decay`（AdamW）| `1e-2` hardcoded | `1e-2`  | AdamW 对 dense param 的 L2 正则系数。dense 才 2.4M params，1e-2 可能过度压缩 transformer 权重，试 1e-3 放松 |
| 🟡 `sparse_weight_decay` | `0.0` | `1e-5` | Adagrad 对 sparse emb 的 L2 正则。默认 0=不正则；加小 L2 让长尾 fid 收向 0，软等价于 drop 高缺失 fid |
| 🟡 `sparse_lr` | `0.05` | `0.1` | sparse Adagrad lr。比 dense lr 高 500× 是因为 sparse row 一年才被 touch 一次，必须大 lr 才有更新机会 |
| 🟡 `lr`（dense）| `1e-4` | `5e-5` idea | dense AdamW lr。`lr_dense_anneal` L1 无信号已 disabled，L3 4 ee 上可能给信号 |
| 🟡 `focal_alpha` / `focal_gamma` | `0.1 / 2.0` | 多版本 | focal loss 参数：`loss = -α(1-p)^γ · log(p)`。α 控正样本权重，γ 控 hard-example emphasis。F-/F+=7.3× 说明严重欠预测正样本，`focal_strong` 试 α=0.75/γ=2.0+pos_weight=15 |

## 下一轮调参候选

### Tier 1 — 中影响 + 中成本

| 超参 | 默认 | 试值 | 成本 | 影响的模型能力 |
|------|------|------|------|--------------|
| `use_rope` | `False` | `True` | ~0（只改 position emb 方式）| **理解用户行为序列的位置顺序**。广告推荐中，用户点击/曝光序列有时间顺序，近期行为比早期行为预测力强 10×。learned absolute pos emb 在尾部位置（e.g. pos 510）训练数据稀疏，表征退化；RoPE 用 sin/cos rotation 不依赖数据密度，长序列尾部位置同样准确 |
| `seq_top_k` | `50` | `100` / `20` | top-k=100 约 +20% attention 计算 | **从用户长历史中选出最相关的行为信号**。用户历史里大部分 token 是噪声（无关品类、cold-start padding）；top-k attention 让模型主动选与当前候选广告最相关的历史点击，类似"召回用户兴趣子集"。太小（20）丢信号，太大（100）退化为 full attention |
| `num_queries` | `2`| `3` | T=20（3×4+8），transformer 序列长+20%，计算+25% | **每个行为域对当前广告的表达维度**。广告推荐有 4 个行为域（a/b/c/d），每域 2 个 query token 各自 attend 该域序列后进 transformer 交互。query 数=每域能汇总的用户兴趣侧面数：2 query 可能只区分"高频兴趣"vs"低频兴趣"，3 query 可额外捕捉"时序趋势" |

### Tier 2 — 高影响 + 高成本（单独大跑，不组合 sweep）

| 超参 | 默认 | 试值 | 成本 | 影响的模型能力 |
|-----------|----|-----|------|--------------|
| `d_model` + `emb_dim` | `64/64` | `128/128` | sparse 1×，dense ~4×，显存压力大，单 ep 30–40 min | **区分相似用户/广告的精细度**。64 维 embedding 在 attention 里区分同品类不同兴趣强度的用户有限；128 维给 user-item 交互更多正交方向。当前模型对 pred_p valley@q3（中等概率区）AUC 仅 0.546，是典型"表达空间不够"的表现 |
| `num_hyformer_blocks` | `2` | `4` | dense ~2×，单 ep 25–30 min | **user feature 与 item feature 的交互深度**。每个 block 做一轮 multi-head attention + FFN：block 1 让 user query token 看到 item NS token，block 2 再做一轮 refinement。2 层可能不足以让稀疏用户特征（冷启动用户只有少量历史）充分与 item 特征对齐；4 层给更多迭代机会 |
| `hidden_mult` | `4` | `8` | dense ~1.5×，单 ep ~20 min | **每个 token 内部的非线性特征组合能力**。FFN 是 transformer 里唯一 per-token 的非线性变换，学的是"给定这个用户-广告交互表示，激活哪些特征组合"。广告推荐的 CTR 依赖复杂交叉（用户年龄 × 广告品类 × 时段），FFN 宽度直接限制能表达的交叉数量；`d_model=64` 时 4×=256 维，试 8×=512 维 |

>[!NOTE] 
> ==emb_skip sweep== → 对齐 v2 baseline (0.864) — 修了 5 个配置 bug (ns_tokens=7/4→5/2, num_queries=1→2, loss.pos_weight 吃 kwargs, model.dropout key 错位, train_sample_ratio 漏写) + 顺手把 AMP bf16 默认开。
> 
> ==emb_skip_threshold== 决定每个 sparse fid (user_id/item_id 等离散 ID) 是走 独立 embedding 表 (model 给它一个唯一的 d_model=64 向量, 能 memorize 这个 ID 的 identity) 还是 hash bucket 共享 vector (上百个 ID撞到同一个 65k 桶里, model 只能学 ID 类别的统计规律, 不能区分个体) — 相当于把 N 个 vocabulary 给 hash 到 65K桶里，

## 想法：Sparse → Dense，统一用 AdamW

用 Adam/AdamW 统一优化所有参数，把 sparse feature 变成 dense feature（去掉 sparse Adagrad 路径）。

**风险**

| 风险 | 机制 |
|------|-------------------|
| 显存爆炸 | sparse emb 用 Adagrad 只更新被命中的 row；改 dense 后整张 emb 表每步算梯度 + 存 Adam 的 momentum 和二阶矩（≈2× 参数量），百万级 id 表直接 OOM |
| 低频 id 学习被破坏 | Adagrad per-parameter lr 天然放大低频 id 的步长；Adam 全局自适应后低频 id 每步被 momentum 平均稀释，长尾 id 可能学不出有效 embedding |
| 训练速度下降 | sparse 更新跳过未命中行；dense 更新全量计算，单步耗时显著上升 |
| 正则化语义变了 | sparse 通常配 lazy weight decay；dense 化后 weight decay 对所有 id 持续衰减，对冷启动 id 等价于注入噪声 |

**收益**

| 收益 | 说明 |
|------|-------------------|
| 优化器一致性 | 全模型一个 AdamW，warmup / cosine schedule / grad clip 统一作用，不用维护两套配置 |
| 支持更复杂正则 | dense 路径可无痛接入 LayerNorm、EMA、SWA、二阶方法；sparse 优化器很多技巧不支持 |
| 梯度信息更充分 | Adam 二阶矩对不平稳 loss 曲面适应更好，理论上比 Adagrad 单调衰减步长更适合长训练 |
| 工程链路简化 | 去掉 sparse 专属的 hash table、shard、checkpoint 逻辑 |
| 解锁新结构 | 可对 embedding 做全局正交约束、低秩分解、跨 id 共享 basis——sparse 优化下难以实现 |

### 为什么值得做 + 具体做法

**关键证据**：`drop_nonseq_sparse Δ=+0.0000`（5/17+5/18 两次复现）——46 个 user_int + 14 个 item_int fid 各有独立 Embedding 但 AUC 贡献 ≈ 0，暗示"每 fid 独立 Embedding"本身是错的表示方式，合并/压缩反而可能拿到 +0.005–0.010。`ns_groups.json` 已手工分了 9 个 super-group 但只到 aggregator 层做平均，这个想法是把"group"再往下推到 vocab 层。

**三种做法**

| 方向 | 做法 | sparse/dense | 推荐 |
|-----------|--------------------|-------------|----|
| A. tuple super-fid | 低基数 fid `(v₁,…,vₖ)` hash 到 65k super-vocab，单 `nn.Embedding(65k, D)` | 还是 sparse，表数 46→1 | |
| **B. one-hot + Linear** | 低基数 fid 各 one-hot → 拼 ~600 维 → `nn.Linear(600, D)` | **全 dense，57k 参数走 AdamW** | ✓ |
| C. 频次编码 | 每 fid value 替换成 `log1p(freq)` → 46 floats → MLP | 全 dense，需 offline freq table | |

B 与"unified AdamW"这条线汇合：单做 unified AdamW 显存爆，单做 compact features 优化器还分两套，合起来才可行。

**风险**

- `drop_nonseq_sparse Δ=0` 是 zero-fill probe，5/18 batch-permutation 没回归——先跑一次 batch-permutation probe 锁信号再开 idea
- 低基数口径（vocab < 32 / 100 / 1024）影响 fid 数量，先用 `schema_full_train.json` 跑频率直方图定口径
- B/C 方向需把 vocab/freq table 带进 infer bundle
- `H_ns` 信息密度大变会影响 selective_query α 路由训练动力学——不一定坏，但需要重训
