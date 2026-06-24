---
id: 6a3bbe7736cb9
name: dtrack: column match / stats preset
tags: [dtrack, column-mapping, stats, preset, runbook, rss, public]
updated_at: 2026-06-24T11:27:33.508300Z
---

# dtrack: column match / stats preset

>col 阶段两件事：先把左右列对上（`match-columns` → `col_map`），再决定每列算多深（`--preset` / `--stats-override`）。CTE 不能自省的表用 `{qname}_columns.csv` 手喂列。<

配套：[[dtrack: CLI runbook]] · 规则实现细节 [[dtrack: general knowledge]] · 统计语义 [[Data Engineer]]。

## 一、列匹配：`match-columns`

```bash
dtrack match-columns project.db --config pairs.json --yes \
  [--map-csv csv/col_map.csv]   # 显式映射 (pair,left,right)，优先级最高
  [--mode merge|replace]        # merge=保留已有 col_map 再补；replace=重来
```
匹配三来源（优先级：显式 > 规则 > 自动同名）：
1. **auto**：左右**同名**列直接配对。
2. **col_rules**：wildcard / regex 批量改名映射（如 `xxxx_([1-5])` → `yyyy_$1`）。
3. **--map-csv**：手写 `pair,left,right` 三列 CSV，强制指定。

结果落进该 pair 的 `col_map = {左列: 右列}`，**只有 col_map 进抽取**。`--yes` 跳过手动确认（justfile 用它）。

> [!NOTE] 三层概念别混（详见 [[dtrack: general knowledge]]）
> `col_map`（最终 {左:右}，进抽取）· `col_rules`（批量生成规则，**不**进抽取）· `col_filter`(`col_include`/`col_exclude`，决定抽哪些列，进抽取)。列注释 / 时间映射在 **web 的 col_mapping 页**做，不在 CLI。

## 二、stats preset：每列算多深

`--preset`（挂在 col-gen，烘进 col SQL）三档：

| 统计 | quick | standard | detailed |
|---|:--:|:--:|:--:|
| `n_total`（行数） | ✓ | ✓ | ✓ |
| `n_missing`（空/NULL） | ✓ | ✓ | ✓ |
| `n_unique`（去重计数） | — | ✓ | ✓ |
| `min_max` | 标量 | 标量 | `值=cnt` |
| `length`（avg+max） | ✓ | ✓ | ✓ |
| `hash_sum`（可移植 md5→int16 求和） | — | ✓ | ✓ |
| `top_10`（频次表 top N） | — | — | ✓ |

- **quick**：纯扁平聚合，无频次表 → 最便宜、利于分区裁剪；分类列 min/max 是标量（字母序 MIN/MAX）。
- **standard**：加 `n_unique` + `hash_sum`，跨引擎对比有了实质信号；仍无频次表。
- **detailed**：加 `top_10` + `值=cnt` 形态的 min/max → 需要 WITH-chain 频次表（Hive 唯一会 GROUP BY 的档）。

> [!NOTE] mean/std 不在任何 preset
> 经验信噪比差且翻倍查询成本，恒为 `''`；14 列 schema 不变。

### 单项覆盖：`--stats-override`
逗号分隔，键：`n_total, n_missing, n_unique, min_max, length, hash_sum, top_10`：
```bash
dtrack generate … --type col --preset standard --stats-override n_unique=on,hash_sum=off
```

## 三、14 列 col CSV schema（固定）

每个 `{qname}_col.csv` 恒为这 14 列（`COL_CSV_HEADERS`），不在 preset 里的统计出 `''`，跨平台 diff 才对得齐：
```text
dt, column_name, col_type, n_total, n_missing, n_unique,
mean, std, min_val, max_val, top_10, length_avg, length_max, hash_sum
```

## 四、CTE / 无法自省的表：手喂列

`processed`（CTE join 多表）的一侧没有单一物理表可 `DESCRIBE`，自动发现会跳过并提示。手动提供 `{qname}_columns.csv`（放进 `--columns-dir`），表头灵活：

```text
column_name,data_type        # 或 variable,type / name,type 都认
cust_id,VARCHAR
amt,DOUBLE
```
`load-columns` 读它写进 `_column_meta`，col-stats builder 才能挑 sentinel family。

| 来源 | 列怎么来 |
|---|---|
| oracle/aws/databricks/hadoop 普通表 | `load-columns` 在线 DESCRIBE |
| **csv 源** | 自动从 CSV 表头发现（DuckDB DESCRIBE / pandas）——**无需**手喂 |
| **processed / CTE** | **必须**手写 `{qname}_columns.csv` |

## 观察什么
- [ ] `match-columns` 打印 `Auto-matched: N` / `Rule-matched: N`；该配对的 key 列也映上了（sample-hash 需要）
- [ ] `extract_col.*.sql` 只覆盖 `col_map` 里的列、按 preset 含/略各统计
- [ ] CTE 侧：先放好 `{qname}_columns.csv` 再 `load-columns`，否则该侧列为空
- [ ] 跨引擎对比看 `hash_sum`/`length_avg` 信号（[[Data Engineer]]）；存疑跑 [[dtrack: CSV → CSV + sample-hash]]
