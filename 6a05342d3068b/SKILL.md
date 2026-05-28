---
id: 6a05342d3068b
name: Data Engineer
tags: [oracle, databricks, athena, mongodb, sql, data-engineering]
updated_at: 2026-05-14T02:37:30.980197Z
---

## 字符串语义：LENGTH 与 TRIM

| 行为 | Oracle | Athena / Trino | Databricks (Spark) | MongoDB |
|---|---|---|---|---|
| `LENGTH()` 单位 | 字符（默认），`NLS_LENGTH_SEMANTICS=BYTE` 时改为字节 | 字符 | 字符 | 字节（BSON） |
| 取字节长度 | `LENGTHB()` | `OCTET_LENGTH(s)` | `OCTET_LENGTH(s)` | `$strLenBytes` |
| `café` 的长度 | 4（字符）/ 5（字节） | 4 | 4 | 5 |
| `TRIM()` 去除范围 | 仅 ASCII 空格 | 仅 ASCII 空格 | ASCII 空格 + Unicode 空白 | — |
| 不间断空格 `\xa0` | **不**去除 | 默认**不**去除 | 去除 | — |

> **常见坑**：`hash_sum` 一致但 `length_avg` 不同 → 几乎必定是字符/字节 LENGTH 语义不一致，而非数据本身有差异。

## 跨引擎哈希与数据指纹

```sql
-- Oracle
RAWTOHEX(DBMS_CRYPTO.HASH(UTL_RAW.CAST_TO_RAW(col), 2))  -- MD5
-- 截断上限：SUBSTR(TO_CHAR(expr), 1, 4000) 以适配 VARCHAR2 长度限制
-- ⚠ 截断只影响 length_avg，若所有值 <4000 字符则 hash 不变

-- Athena / Trino
MD5(TO_UTF8(col))         -- 返回 varbinary，用 TO_HEX() 转十六进制
TO_HEX(MD5(TO_UTF8(col)))

-- Databricks
MD5(col)                  -- 直接返回十六进制字符串
SHA2(col, 256)

-- MongoDB aggregation（无原生 MD5）
{ $function: { body: "...", lang: "js" } }
```

> MD5 在所有引擎中均对**字节**操作，相同字节序列哈希必然一致；只有长度函数会因语义不同而分叉。

## NULL 与哨兵值的不对称性

| 行为 | Oracle | Athena | Databricks | MongoDB |
|---|---|---|---|---|
| 空字符串等于 NULL | **是**（`'' IS NULL`） | 否 | 否 | 否 |
| NULL 替换函数 | `NVL(x, y)` | 仅 `COALESCE` | `COALESCE` / `IFNULL` | `$ifNull` |
| 哨兵值 `'NA'` → NULL | `NULLIF(col, 'NA')` | 同左 | 同左 | `$cond` |

> 诊断规律：`hash_sum` 与 `length_avg` **同时**不同 → NULL/哨兵映射不一致；**仅** `length_avg` 不同 → LENGTH 语义问题。

## 数据类型速查

| 概念 | Oracle | Athena / Trino | Databricks | MongoDB |
|---|---|---|---|---|
| 可变字符串 | `VARCHAR2(n)` | `VARCHAR(n)` | `STRING` | 隐式 |
| 大文本 | `CLOB` | `VARCHAR`（无上限） | `STRING` | `String`（≤16 MB） |
| 整数 | `NUMBER(p,0)` | `BIGINT` | `BIGINT` | `Int32` / `Int64` |
| 小数 | `NUMBER(p,s)` | `DECIMAL(p,s)` | `DECIMAL(p,s)` | `Decimal128` |
| 时间戳 | `TIMESTAMP WITH TIME ZONE` | `TIMESTAMP WITH TIME ZONE` | `TIMESTAMP`（无时区） | `Date`（UTC） |
| 布尔 | 无原生类型（用 `NUMBER(1)`） | `BOOLEAN` | `BOOLEAN` | `Bool` |
| 数组/嵌套 | `VARRAY` / 嵌套表 | `ARRAY(T)` | `ARRAY<T>` | 原生 BSON 数组 |
| JSON | `JSON`（21c+）/ `CLOB` | `MAP` / `VARCHAR` + JSON 函数 | `VARIANT`（Delta） | 原生 BSON 文档 |

## 查询模式常见坑

```sql
-- Oracle：ROWNUM 在排序前过滤，切勿直接用于 TOP-N 排序
SELECT * FROM t WHERE ROWNUM <= 10;   -- 快，但先截断再排序（结果不对）
SELECT * FROM (SELECT *, ROW_NUMBER() OVER (ORDER BY id) rn FROM t) WHERE rn <= 10;

-- Athena：始终在 WHERE 中过滤分区列，否则全表扫描
SELECT * FROM t WHERE dt = '2024-01-01';

-- Databricks：Z-ORDER 提升多维过滤性能；Delta 时间旅行回溯历史版本
OPTIMIZE table ZORDER BY (col1, col2);
SELECT * FROM table VERSION AS OF 42;
SELECT * FROM table TIMESTAMP AS OF '2024-01-01';

-- MongoDB：投影仅返回索引覆盖字段，避免回表
db.col.find({ status: "A" }, { _id: 0, status: 1, score: 1 })
db.col.find({...}).explain("executionStats")   -- 查看执行计划
```

## 并发与事务模型

| 特性 | Oracle | Athena | Databricks Delta | MongoDB |
|---|---|---|---|---|
| ACID 事务 | 完整（跨行跨表） | 只读（S3） | 完整（Delta Log） | 多文档（4.0+） |
| 默认隔离级别 | Read Committed | Snapshot（单次查询） | Serializable（写） | Snapshot |
| 行级锁 | 有 | 无 | 乐观并发（OCC） | 文档级锁 |
| DDL 在事务内 | 否（DDL 自动提交） | 无 | 是（Delta） | 无 |

## 性能调优速查

```sql
-- Oracle：查看执行计划 + 收集统计信息
EXPLAIN PLAN FOR <query>;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
EXEC DBMS_STATS.GATHER_TABLE_STATS('schema', 'table');

-- Athena：用列式格式（Parquet/ORC）+ 分区降低扫描成本
-- CTAS 物化中间结果
CREATE TABLE result WITH (format='PARQUET', partitioned_by=ARRAY['dt']) AS SELECT ...;

-- Databricks：AQE 默认开启（DBR 10+）；热表缓存
SET spark.sql.adaptive.enabled = true;
CACHE TABLE hot_dim;

-- MongoDB：复合索引顺序遵循「等值 → 排序 → 范围」原则
db.col.createIndex({ status: 1, score: -1, date: 1 });
db.currentOp({ "secs_running": { $gt: 5 } })   -- 监控慢查询
```

## 跨引擎数据核对清单

- [ ] 统一 `LENGTH` 语义：两侧同用字符长度或同用字节长度
- [ ] 哈希前对两侧做相同的 `TRIM`；注意 `\xa0`、`\t` 的处理差异
- [ ] Oracle 空字符串等于 NULL → 比较前用 `COALESCE(col, '')` 对齐
- [ ] Oracle `SUBSTR(..., 1, 4000)` 截断 → 确认无超过 4000 字符的值
- [ ] 小基数场景注意哈希碰撞：16-bit hash sum 在 K≈100 时碰撞概率约 7%
- [ ] 时间戳时区：Databricks 丢失时区信息，Oracle/Athena 保留

TAGS: oracle, databricks, athena, mongodb, sql, data-engineering
