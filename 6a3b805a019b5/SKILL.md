---
id: 6a3b805a019b5
name: dtrack: Oracle → Athena (row + col)
tags: [dtrack, oracle, athena, runbook, row, col, rss, public]
updated_at: 2026-06-24T11:22:56.604609Z
---

# dtrack: Oracle → Athena (row + col)

>左 Oracle、右 Athena 的对数实操。重点：两侧 `date_type` 通常不同（Oracle `timestamp`/`date` ↔ Athena `string_dash`/`num`），dtrack 各自归一到同一日期桶再比。<

基座流程见 [[dtrack: CLI runbook]]；跨引擎语义坑见 [[Data Engineer]]。

## 样例 pair

来自 `testing/pairs.json` 的 `oracle_ts_vs_aws_str`——Oracle `TIMESTAMP`（含 00:00:00）对 Athena `VARCHAR` 的 `YYYY-MM-DD`：

```json
"oracle_ts_vs_aws_str": {
  "left":  { "name": "ts_cust_daily", "table": "CUST_DAILY_VW", "source": "pcds",
             "conn_macro": "pcds", "date_col": "RPT_TS", "date_type": "timestamp" },
  "right": { "name": "str_cust_daily", "table": "cust_daily", "source": "aws",
             "conn_macro": "analytics_db", "date_col": "rpt_dt", "date_type": "string_dash" },
  "mode": "incremental", "fromDate": "2024-10-01", "toDate": "2024-12-31"
}
```

> [!NOTE] `source: "pcds"` vs `"oracle"`
> 两者都走 Oracle 抽取器；`pcds` 是一个具名 macro。右侧 Athena 的 `conn_macro` 是 **Glue database 名**（这里 `analytics_db`），`table` 是其下的表。

## 真机 env（`.env`）

```bash
# Oracle（左）
PCDS_USR=your_user
PCDS_PWD=...                 # conn_macro=pcds → 找 PCDS_PWD
# Athena（右）
AWS_DEFAULT_REGION=us-east-1
AWS_S3_WORK_GROUP=primary
AWS_S3_STAGING_DIR=s3://your-bucket/athena-output/
AWS_USR=... ; AWS_PWD=...
```
先 `dtrack doctor` 确认 `PCDS_USR=set`、`pcds_pwd` 不 MISSING、四个 AWS 变量都在。

## 跑（justfile）

```bash+ 推荐：分组只跑这一对
# 把上面 pair 单独放 testing/oracle_athena.json，然后：
just pair=oracle_athena e2e row          # 行数阶段
just pair=oracle_athena e2e col-gen      # 生成 col SQL，停下来审
#   → 审 testing/csv/extract_col.aws.sql 与 extract_col.oracle.sql
just pair=oracle_athena e2e col-run      # 执行 → load → compare
# 或一把梭：just pair=oracle_athena e2e all
```

裸命令对照见 [[dtrack: CLI runbook]] 的"完整命令序列"。

## 观察什么

### Row 阶段
- [ ] `extract_row.oracle.sql`：`RPT_TS` 被 `TRUNC(...)`/`CAST AS DATE` 归一到天，窗口 `2024-10-01..12-31`
- [ ] `extract_row.aws.sql`：`rpt_dt`（字符串 `YYYY-MM-DD`）被 `date_parse`/直接比较，窗口一致
- [ ] `compare_row.html`：
  - **只在一侧的日期** → 抽取窗口或源数据缺失
  - **同日 count 不等** → 真实行数差异，记下日期供 col 阶段聚焦
- [ ] `run-sql` 末行 `N ok, M failed`；Athena 失败常见于 staging dir / workgroup 权限

### Col 阶段
- [ ] col SQL 的日期被**裁剪到 row 认定两边都齐的交集**（铁律的体现）
- [ ] `compare_col.html` 逐列看四类信号：

| 信号 | 多半原因 |
|---|---|
| `n_total` / `n_missing` 差 | 行数或 NULL 口径不一致 |
| 仅 `length_avg` 差，`hash_sum` 同 | **LENGTH 字符/字节语义**不同（见 [[Data Engineer]]） |
| `hash_sum` 与 `length_avg` **同时**差 | NULL/哨兵映射不一致 |
| 仅 `hash_sum` 差 | 值本身有差异 → 跑 [[dtrack: CSV → CSV + sample-hash]] 定位 |

## Oracle ↔ Athena 专属坑

> [!WARNING] 日期类型不对称
> Oracle `TIMESTAMP` 带时分秒，Athena 侧多为纯日期字符串。`date_type` 必须如实标注（`timestamp` vs `string_dash`），dtrack 才能把两边都 TRUNC 到天。标错会导致全表"不匹配"。

> [!CAUTION] Athena 分区
> `rpt_dt` 若是分区列，WHERE 没带上会全表扫描、又慢又贵。确认 `extract_row.aws.sql` 的 WHERE 命中分区列。

- **空串 = NULL 仅 Oracle 成立**：`hash_sum`/`length_avg` 同差时，先怀疑这个（Athena `''≠NULL`）。
- **`SUBSTR(...,1,4000)` 截断**：Oracle 侧长文本会被截到 4000 字符算长度；若有超长值，`length_avg` 会偏——确认无 >4000 字符的列。
- **哈希字节一致性**：MD5 两边都对字节算，相同字节必同值；分叉只来自 TRIM/LENGTH/NULL 这些前处理（见 [[Data Engineer]] 的哈希小节）。
