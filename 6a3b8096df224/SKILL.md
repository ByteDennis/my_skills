---
id: 6a3b8096df224
name: dtrack: Oracle → Databricks (row + col)
tags: [dtrack, oracle, databricks, runbook, row, col, rss, public]
updated_at: 2026-06-24T11:22:49.378094Z
---

# dtrack: Oracle → Databricks (row + col)

>左 Oracle、右 Databricks(Spark SQL) 的对数。两个独有点：`conn_macro` 是 `catalog.schema`；连接走 OAuth(`DB_PROFILE`+`DB_HTTPS`) + 复用 Athena 的代理。col 阶段直接借用 Athena 的 SQL 模板（Spark 与 Trino 方言够近）。<

基座流程见 [[dtrack: CLI runbook]]；跨引擎语义坑见 [[Data Engineer]]。

## 样例 pair

目前仓库里**还没有** oracle→databricks 的 pair（只有 mock fixture `databricks_pos_daily`）。要测就照此建一个，左 Oracle `pb30` 的 `POS_DAILY`，右 Databricks 的 `pos_daily`：

```json+ testing/oracle_databricks.json
{
  "pairs": {
    "oracle_vs_databricks_pos": {
      "description": "Oracle DATE POS_DAILY vs Databricks pos_daily（日粒度）。",
      "left":  { "name": "pos_daily", "table": "POS_DAILY", "source": "oracle",
                 "conn_macro": "pb30", "date_col": "POS_DT", "date_type": "date" },
      "right": { "name": "pos_daily", "table": "pos_daily", "source": "databricks",
                 "conn_macro": "main.analytics", "date_col": "pos_dt", "date_type": "date" },
      "mode": "incremental", "fromDate": "2024-10-01", "toDate": "2024-12-31"
    }
  }
}
```

> [!IMPORTANT] `conn_macro` = `catalog.schema`
> Databricks 的 `conn_macro` 必须是点分的 `catalog.schema`（如 `main.analytics`），最终引用 `{catalog}.{schema}.{table}`。
> - 只给 schema（`analytics`）→ dtrack 前置 `DB_DEFAULT_CATALOG` 拼成 `{DB_DEFAULT_CATALOG}.analytics.pos_daily`。
> - 完全留空 → 落到 warehouse 的默认 catalog.schema。

## 真机 env（`.env`）

```bash
# Oracle（左）
PCDS_USR=your_user
PB30_PWD=...                 # conn_macro=pb30 → 找 PB30_PWD
# Databricks（右）
DB_PROFILE=your_profile      # databricks-sdk Config 的 OAuth profile
DB_HTTPS=/sql/1.0/warehouses/xxxxxxxx   # SQL warehouse 的 http_path
DB_DEFAULT_CATALOG=main      # 可选；conn_macro 只给 schema 时前置
# 代理：复用 Athena 那套（仅设 HTTPS_PROXY，不做 AWS token 续期）
AWS_USR=... ; AWS_PWD=... ; AWS_HOST=proxy.host
```

> [!NOTE] 连接内幕（来自 `databricks.py`）
> `databricks_connect` 用 `Config(profile=DB_PROFILE).authenticate()` 取 Bearer token，host 从 profile 解析，`http_path=DB_HTTPS`。若 `AWS_USR/PWD/HOST` 齐全则设 `HTTPS_PROXY=http://usr:pwd@host:8080`。**不**跑 AWS token dance——那曾让 col_gen runner 间歇性失败。缺 `DB_PROFILE`/`DB_HTTPS` 会直接 `RuntimeError`。

## 跑（justfile）

```bash
just pair=oracle_databricks e2e row
just pair=oracle_databricks e2e col-gen     # 审 testing/csv/extract_col.databricks.sql
just pair=oracle_databricks e2e col-run
# 一把梭：just pair=oracle_databricks e2e all
```
mock 演练（无真库时验证流程）：`.env` 里设 `DTRACK_MOCK=testing/mock`，会拷 `databricks_pos_daily_row.csv` 当结果。

## 观察什么

### Row 阶段
- [ ] `extract_row.databricks.sql`：表名是 `main.analytics.pos_daily`（catalog.schema 正确拼接）
- [ ] `pos_dt`（Spark `DATE`）按天分桶，窗口 `2024-10-01..12-31`
- [ ] `run-sql` 成功；失败先看是不是 `DB_PROFILE`/`DB_HTTPS` 缺，或代理没通
- [ ] `compare_row.html`：同 Oracle→Athena，看缺日期 / count 差

### Col 阶段
- [ ] col SQL 用的是 **Athena/Trino 模板**（`date_parse`/`date_trunc`/`approx_count_distinct`/`SUBSTR`）——Spark 兼容
- [ ] 哈希表达式应是 Databricks 方言的 16-bit MD5：
  ```sql
  CAST(conv(SUBSTR(LOWER(md5(TRIM(col))), 1, 4), 16, 10) AS BIGINT)
  ```
  与 Oracle 的 `int(md5(trim(value))[:4],16)` **数值等价**——这是跨引擎 `hash_sum` 能对上的关键。
- [ ] `compare_col.html` 四类信号判读同 [[dtrack: Oracle → Athena (row + col)]]

## Oracle ↔ Databricks 专属坑

> [!WARNING] 时区丢失
> Databricks `TIMESTAMP` **不带时区**，Oracle `TIMESTAMP WITH TIME ZONE` 带。跨时区时间戳直接比会偏。对"日"粒度对数影响小（都 TRUNC 到天），但若 `date_col` 含时分秒且两边时区不同，会错位一天——优先用纯 `date`/按天归一。

> [!CAUTION] 方言分歧的兜底
> col 借 Athena 模板，绝大多数函数 Spark 都有。万一某个表达式 Spark 不认，在 pair 上挂 `vintage_transform` 覆写该 SQL（见 `databricks.py` 头注）。

- **空串 ≠ NULL**（同 Athena，与 Oracle 相反）→ `hash_sum`+`length_avg` 同差时先查 NULL 口径。
- **catalog 权限**：`pos_daily` 在 `main.analytics` 但 warehouse 默认别的 catalog → 必须把 catalog 写进 `conn_macro` 或设 `DB_DEFAULT_CATALOG`，否则 "table not found"。
- 怀疑哈希：跑 [[dtrack: CSV → CSV + sample-hash]]（支持 databricks 端按 key 取同样行、后端算 hash16 逐格核对）。
