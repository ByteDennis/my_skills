---
id: 6a0568e069007
name: Bootstrap Cascade
tags: [cascade, beam-search, curriculum, ml-training, scheduler]
updated_at: 2026-05-18T23:26:56.270868Z
---

## Bundle 文件清单

| 文件 | 说明 |
|------|------|
| `baseline.tar.gz` | bundle，md5=`0dd41b57…` |
| `run.sh` | 入口（原 `smoke-online-run.sh`） |
| `cascade-curriculum-run.sh` | L0→L3 主调度，305 行 |
| `cascade-curriculum-config.yaml` | 含 `prior_predictions_dump` |
| `ideas.yaml` | 候选 idea 池 |
| `beam_scheduler.py` | 输出 `plan_LX.yaml`（上次漏传） |

## 关键改动（SMOKE 通 → Taiji 挂）

| 改动 | 原因 |
|------|------|
| `CASCADE_PY` 按平台选 python 路径 | 本地用 `.venv/bin/python`，Taiji 用系统 `python3` |
| `export TRAIN_CKPT_PATH / LOG_PATH / TF_EVENTS_PATH="$ROOT_*"` | 之前 `TRAIN_LOG_PATH` 没赋值，bundle fallback 路径出错 |
| `python3 -m experiments.20260512.beam_scheduler` → `"\$SCRIPT_DIR/beam_scheduler.py"` | Taiji 无包目录结构，`-m` 找不到 namespace |
| README 删除 `--bootstrap-name cascade-curriculum` | 该参数会让脚本被自动覆盖 |

## 为什么 baseline 只有 27 MB（不是 bug）

两个 config 参数决定了模型体积：

```yaml
emb_skip_threshold: 10000   # vocab > 10k 的 fid 跳过独立 embedding 表
hash_bucket_for_skipped: 65536  # 改用 65k hash bucket 共享
```

| 方案 | 体积 |
|------|------|
| 原始 sparse 表（naive）| user_id 10M × dim16 × 4B = **640 MB** 一张；多表合计 1-2 GB |
| 当前：高基数 fid 全 hash 到 65k 桶 | 65k × 16 × 4B = **4 MB**/张，3-5 张 ≈ 15-20 MB |
| dense transformer (layer=2 head=4 hidden=64) + 杂项 | ~7 MB |
| **合计** | **~27 MB** ✓ |

**trade-off**：小模型让 cascade 4 层跑得起（ckpt I/O 快、不 OOM），代价是 hash collision 严重（1000 万用户挤 65k 桶 ≈ 150:1），user 个性化信号被摊薄 — 这也是 `drop_nonseq_sparse` probe Δ≈0 的根因：那些表实际上已经退化成接近全零。

`ideas.yaml` 里的 `hash_bucket_65k` idea 就是反向实验：把桶调大（65k→1M），换回信号但 ckpt 涨到几百 MB。当前路线是"小模型 + 多 idea cascade"，而不是"big model 一次 ship"。

## Two-Step 筛选逻辑

### Step 1 — eligible_for_level（`beam_scheduler.py:96`）

```python
budget_for_level = {"L1": 0.25, "L2": 0.5, "L3": 1.0}
return [s for s in ideas.values()
        if s.min_budget <= cur and level not in s.skip_levels]
```

- `min_budget`：该 idea 至少需要多少 `train_sample_ratio` 才能有效（L1=0.25，L2=0.5，L3=1.0）
- `skip_levels`：显式跳过某几级

### Idea 资格矩阵

| idea | family | min_budget | skip_levels | L1 | L2 | L3 |
|------|--------|-----------|-------------|----|----|-----|
| `lr_anneal` | optim | 0.25 | — | ✓ | ✓ | ✓ |
| `dropout_005` | regularizer | 0.25 | — | ✓ | ✓ | ✓ |
| `loss_combine_bce_focal` | loss | 0.25 | — | ✓ | ✓ | ✓ |
| `patience_2` | train_control | 0.25 | — | ✓ | ✓ | ✓ |
| `bpr_semihard` | loss | 0.50 | L1 | — | ✓ | ✓ |
| `reinit_warmup0` | reinit | 0.50 | L1 | — | ✓ | ✓ |
| `focal_prior` | sample | 1.00 | L1, L2 | — | — | ✓ |
| `parquet_boost` | sample | 1.00 | L1, L2 | — | — | ✓ |
| `hash_bucket_65k` | arch | 1.00 | L1, L2 | — | — | ✓ |
| `ns_groups` | arch | 1.00 | L1, L2 | — | — | ✓ |

eligible 数量：L1=4，L2=6，L3=10

### Step 2 — Family 互斥 + beam_width 截断（`beam_scheduler.py:284-296`）

```python
candidates_sorted = sorted(candidates,
    key=lambda c: (-score_by_name[c.name].score, c.family, c.name))
seen_families = set()
for c in candidates_sorted:
    if c.family in seen_families:
        continue
    seen_families.add(c.family)
    kept.append(c)
    if len(kept) >= beam_width:
        break
```

排序键：`(-score, family asc, name asc)`。冷启动 score 全 0 时退化为字典序。

**本次 smoke（beam_width=3）实际 plan：**

| Level | 进 plan 的 idea |
|-------|----------------|
| L1 | `loss_combine_bce_focal` / `lr_anneal` / `dropout_005` |
| L2 | `bpr_semihard` / `dropout_005` / `lr_anneal` |
| L3 | `hash_bucket_65k` / `bpr_semihard` / `lr_anneal` |

真有 valid_db 数据后：`score = α·ΔAUC + β·(net_flip/N) + γ·(1-ρ_with_best)`，先按 score 降序，字典序仅作 tie-break。

## Backtrack 现状

### ✅ 已工作：隐式纵向 backtrack

`beam_scheduler.find_current_best_ver` 扫全部 valid_db → 返回 AUC 最高 model_ver → 作为下层 `parent_ver`。

效果：L1 全输 → L2/L3 自动 fork 自 R0；L2 全输 → L3 fork 自 L1 best。本次 SMOKE winner = `L1/loss_combine_bce_focal`（L2/L3 均未超过它）。

### ❌ 声明了但未启用（`apply_prunes` 是死代码）

| 规则 | 应该干啥 | 状态 |
|------|---------|------|
| Quality Gate | ΔAUC < −0.002 永久淘汰 | ❌ 未调用 |
| Noise Floor | \|ΔAUC\| < 0.0005 跳过 | ❌ 未调用 |
| Subtree Upper Bound | upper bound < 已知 best → 剪枝 | ❌ 未调用 |

`build_plan` 只做 score 排序 + family 互斥 + beam_width 截断，从未调用 `apply_prunes`。

### ❌ 完全缺失

- 无 abort-level：L1 全降仍继续推 L2/L3
- 无 idea blacklist 持久化：被淘汰的 idea 下次 cascade run 仍会重选
- 无跨 run 状态：`valid_db` 是单 job 内 parquet，`hard_blob` 需手工传（README §5.5 Step D），scheduler 不消费

## 让 Backtrack 真正生效（3 个改动）

```python
# 1. build_plan 里调 apply_prunes（10 行内）
scores = apply_prunes(scores, ideas, cold_start=(predictions_path is None))
# kept 集合排除 pruned_reason is not None 的 idea

# 2. Abort-level：加 --min-delta-vs-baseline 0.0
#    plan 里全部 candidate Δ 都为负 → 退出 cascade，返回信号：
#    [curriculum] L1 all degraded, aborting

# 3. 跨 run blacklist：valid_db schema 已有 pruned_reason
#    scheduler 跨日读历史 prune 决定 → 自动跳过烂 idea
```

## 待补充的设计项

| # | 名称 | 目的 | 当前缺失表现 |
|---|------|------|-------------|
| 1 | Plugin 矩阵测试 `test_components_on_sample1k.py` | 每个 contrib plugin 在真实 1k 行数据上跑一步 forward+backward，本地 5 秒报错 | CI 全过，上 Taiji 才挂（如 `loss_combine_bce_focal` 的 `@json:` bug） |
| 2 | 双 forward 验证探针（Step 18b `validation_probe`） | eval 时额外跑一次"屏蔽部分特征"的 forward，暴露模型靠单一特征死记的虚胖 AUC | `probe_auc` 永远是 `None`，`use_probe=true` 开关空摆，只能 fallback warn |
| 3 | Optimizer sidecar 落盘 | 存 ckpt 时同时存 optimizer 状态，续训时接力 momentum 而不是冷启动 | cascade 每层续训只 load `model.pt`，L2 ep1 optimizer 从 0 重启，AUC 反比 parent 低（日志：`dropped 0.0214 vs prev winner → optimizer cold-start suspected`） |
| 4 | `wrong_cluster_analysis_v2` | 按自信度 + 方向切错例，区分过拟合 vs 欠拟合，输出两个 blob 喂不同 idea | v1 只按 pred bucket 切，两种错误用同一个 blob，调度器无法区分该降权还是加特征 |

> [!NOTE] **主线**：
> (1) 让 plugin 挂在本地提前暴露；(2) 让 early-stop 选出真正鲁棒的 epoch 而非虚胖的；(3) 让 cascade 续训不丢"惯性"；(4) 让错误分析有诊断力，驱动调度器选对 idea 方向。

### wrong_cluster_analysis_v2 设计要点

**核心切片**

| 维度 | 切法 | 含义 |
|------|------|------|
| 自信度 | `\|pred−0.5\| > 0.4` = confident wrong | 模型笃定但错 → 过拟合到误导特征 → 降权/regularize |
| 自信度 | `\|pred−0.5\| < 0.1` = ambiguous wrong | 模型本来不知道 → 欠拟合 → 加特征/加深 |
| 方向 | F+（label=0, pred>0.5） | 过激预测 → 降热度信号权重 |
| 方向 | F−（label=1, pred<0.5） | 过保守 → `focal_prior γ` 调高 |

**baseline 对照（过拟 vs 欠拟判定）**

对每个特征 `X` 算 `KL(D_wrong ‖ D_all)`：

- KL > 0.1 → 错例在 X 上偏移显著 → 过拟合，该 idea 在 X 方向 overpredict
- KL ≈ 0 但 X 与 label 相关 → 欠拟合，模型对 X 无感

`feature drift KL` 作为第 4 项加入 `beam_scheduler` score 公式，让调度器自动避开"过拟合到某特征"的 idea。

**两个 blob 喂不同 idea（关键）**

```text
confident_wrong_blob  →  降权 idea（模型太自信反而错，软化它）
ambiguous_wrong_blob  →  加权+加特征 idea（模型不知道，boost + 新维度）
```

新增指标：`pred_overlap_area` = pos/neg pred 直方图重叠面积（0=完美分开，比 AUC 更直观）

## 下一步任务

| Step | 内容 | 状态 |
|------|------|------|
| A. 本地1000样本测试 | 本地测试 | ✅ 314 passed |
| B. 打包上传 | `package-train baseline --force` | ✅ md5=`9240eefb…` |
| C. 线上全量样本测试 | 完整 4 层 | ⚠️ 仅跑过 SMOKE，未真做 |
| D. 错误样本分析 | 解 `hard_blob` → user_id | ⏳ 跑完 C 才能做 |
| E. 错误聚类分析 | `wrong_cluster_v2` → next iter | ⏳ 跑完 C 才能做 |

### SMOKE vs FULL

| 参数 | SMOKE | FULL |
|------|-------|--------|
| 数据 | `max_files=2` ≈ 2k 行 | 全量 ≈ 1M 行 |
| epoch | 1/level | L0=4, L1=2, L2=2, L3=4 |
| `max_steps` | 50 | unbounded |
| batch size | 32 | 1024 |
| `beam_width` | 1 | 3 |
| AUC 噪声 | ±0.02（信号不可读） | ±0.005 |
| FINAL retrain | 跳过 | 跑 |
| 时长 | ~3 分钟 | ~6-8 小时 |


> [!NOTE]
> - 12 个真 model_ver 的 valid AUC（100k+ 行）
> - 完整 `wrong_cluster_v2`：KL drift / F+/F- / `hard_blob`
> - FINAL ckpt（95% data）
> - Step D 需要的 `hard500_blob`
> ```bash
> grep 'hard500_blob' run.log | tail -1 | python decode_blob.py > wrong_users.txt
> ```
> - 看 net_flip / wrong-cluster top-cluster / hard_blob，决定下一批次迭代模型所需要的验证的想法

## FULL Run 实测（Step C 观测）

| 维度 | 现象 | 对比设计 |
|------|------|---------|
| L0 R0 baseline | ep1-4 AUC 0.841/0.842/**0.844**/0.841，best=ep3 落盘 | ✅ AUC-based ckpt + ep4 退步符合 4 epoch 边际 |
| optim sidecar 续训 | R0 ep1/2/3 各 dump 35.4MB，L1 各 idea `[optim-resume]` 恢复 AdamW+Adagrad | ✅ momentum 接力正常 |
| valid_db 累积 | R0/L1/L2 共 6 model_ver，1.4M 行，跨 round append | ✅ append-only schema 正常 |
| wrong_cluster_v2 | 每 best epoch 输出 4 段日志 + 2 个 blob | ✅ 全段都出 |
| BEAM 决策 | L3 `best_ver=R0`，candidates=10，kept=3，5 条剪枝 + family 互斥 | ✅ 隐式 backtrack：L1/L2 全 ≤ R0 → 选回 R0 |
| `reset_optim=true` | `L3/hash_bucket_65k` resume 为空，从头训 | ✅ 改架构 idea 跳过 warm-start |
| L3/hash_bucket_65k | ep3 AUC **0.8519** vs R0 0.8436 = **+0.0083** | ✅ 架构改动真涨，L3 全量预算起效 |
| probe drop_dense | 出 17 次 drop_all_sparse + 17 次 drop_nonseq_sparse，**0 次** drop_dense | ❌ config 有 3 个 probe 只跑 2 个 |
| KL drift 全 neutral | 4 个 feature KL 实测 0.001-0.019，全 < `kl_neutral_threshold` | ⚠️ scheduler 第 4 项惩罚拿不到信号，阈值需再降或换 feature_keys |
| F+/F- 不平衡 | F+≈900 / F-≈7700，比例 ~1:8，`mean_pred_pos=0.30` | ⚠️ 模型严重 under-predict 正例，需调高 `focal_prior γ` 或加 class_weight |
| L1/L2 全 ≤ R0 | L1 AUC=0.838，L2=0.833-0.837，R0=0.844 | ⚠️ 25% 数据 × 2 epoch 不够热，建议 L1 epoch→3 或 sample_ratio→0.5 |
| L2 idea 与 L1 重叠 | L2 plan = `bpr_semihard / dropout_005 / lr_anneal`，后两个与 L1 一致 | ⚠️ 无机制排除"L1 已跑过且未赢"的 idea，跨层探索不足 |


> [!WARNING]
> - v2 启用的优秀配置(robust_clip / time features)要从 train.py CLI flag 走 —— 这个改动比较大,先不在本批做(等 step_D 用 v1 拿一个对照点,然后再开 v2 实验)

## 建模信号维度候选（23 个）

选标准：① extras 已有、零额外计算；② 能直接驱动新 idea；③ 不依赖 train-time 频次表。

### MVP 6（立即可用）

| # | 维度 | bins | 能驱动的 idea |
|---|------|------|--------------|
| 1 | `hour_of_day` | 深夜/上午/下午/晚上（4） | hour-aware embedding / 周期 sinusoidal feature |
| 2 | `seq_a_len` | 冷/浅/中/长（4） | seq encoder 改 / 冷启动 dense fallback |
| 3 | `seq_b_len` | 同上（4） | multi-domain attention 调参 |
| 4 | `total_history` | 0-8/8-32/32-128/128+（4） | 全域 history pooling 策略 |
| 5 | `user_int_zero_frac` | 完整/部分/稀疏（3） | 稀疏 mask awareness / hash bucket size |
| 6 | `predicted_p_bucket` | 5 桶 calibration | loss family 调整（focal / pos_weight） |

### 全量候选池（供后续迭代选）

| 组 | 维度 | 说明 |
|----|------|------|
| 时间 | `day_of_week` | 周末 vs 工作日 conversion 模式 |
| 时间 | `valid_window_position` | valid 集内部 temporal drift |
| 时间 | `time_since_last_event` | 用户活跃度衰减（0-1h/1-24h/1-7d/>7d） |
| 时间 | `prediction_lead_time` | 即时转化 vs 长尾转化 |
| 用户历史 | `history_diversity` | unique_items/len，专注用户 vs 浏览型 |
| 用户特征 | `user_dense_filled` | dense 特征 non-null 数（0-5/5-10/10+） |
| Item 侧 | `item_freq_decile` | 长尾 vs 头部，直接评估 hash bucket 65k 够不够 |
| Item 侧 | `item_age` | 新品 vs 老品 |
| 预测分布 | `prediction_uncertainty` | `\|p−0.5\|`，模糊 vs 确定样本 AUC 差 |
| 预测分布 | `label_bucket` | pos 看 logloss，neg 看 mean_pred |
| 交互 | `freshness` | `timestamp − max(seq_ts)`，刚浏览 vs 沉睡用户 |
| 交互 | `seq_b/a_ratio` | 次域 vs 主域比例，multi-domain 用户偏好 |
| 交互 | `action_type_distribution` | 序列 action 类型熵，单一 vs 多元行为 |
| 联合 | `hour × seq_len` | 16 桶，凌晨冷启动 vs 凌晨长历史 |
| 联合 | `is_cold × user_int_zero_frac` | 6 桶，最难 corner：真冷启动 + 稀疏特征 |
| 联合 | `seq_a_len × seq_b_len` | 16 桶，单域 vs 多域活跃 |

## TensorBoard Profiles

```bash
export URSL_TB_LOGDIR=runs/selq_smoke/tb
export URSL_TB_PROFILES=core,errors,selective_query,subgroup  # 不设默认=core,errors
bash cascade-curriculum-run.sh
tensorboard --logdir runs/selq_smoke/tb
```

`URSL_TB_LOGDIR` 不设 → writer no-op，训练管道零回归。

| Profile | 关键 metric | 解决的诊断盲点 |
|---------|------------|--------------|
| `core` | `Calib/ratio` | R0 calib=0.975 ep4 过训，能看出哪个 step 开始偏 |
| `errors` | `Loss/pos_over_neg` | F-/F+ 7.3× 是 focal_strong 真生效还是没生效 |
| `lr_grad` | `Grad/sparse_norm` | ep1 spike 配合 LR 曲线，区分 cold-start vs 数据 spike |
| `selective_query` | `Alpha/entropy_block_{0..2}` | Selective 核心验收，否则"是否真生效"永远是猜 |
| `subgroup` | `AUC/valid_pred_q3` | pred_p valley@q3 AUC=0.546，看每 epoch 是否真有改善 |
| `embedding` | `Emb/zero_row_frac` | Adagrad 末期 dead embedding 比例，验证 `reinit_warmup0` |
