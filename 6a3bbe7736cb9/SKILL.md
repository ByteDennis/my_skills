---
id: 6a3bbe7736cb9
name: dtrack: column match / stats preset
tags: [dtrack, column-mapping, stats, preset, runbook, rss, public]
updated_at: 2026-06-24T12:12:37.950008Z
---

# dtrack: column match / stats preset

>Two jobs in the col phase: first align left/right columns (`match-columns` → `col_map`), then decide how deep to compute stats per column (`--preset` / `--stats-override`). For tables that CTEs cannot introspect, feed columns manually via `{qname}_columns.csv`.<

Companion cards: [[dtrack: CLI runbook|6a3b8031de3c0]] · Rule implementation details [[dtrack: general knowledge|6a3a7b11b439b]] · Statistics semantics [[Data Engineer|6a05342d3068b]].

## 1. Column matching: `match-columns`

```bash
dtrack match-columns project.db --config pairs.json --yes \
  [--map-csv csv/col_map.csv]   # explicit mapping (pair,left,right), highest priority
  [--mode merge|replace]        # merge=keep existing col_map then append; replace=start over
```
Three matching sources (priority: explicit > rules > auto same-name):
1. **auto**: left/right columns with **identical names** are paired directly.
2. **col_rules**: wildcard / regex bulk rename mappings (e.g. `xxxx_([1-5])` → `yyyy_$1`).
3. **--map-csv**: a hand-written `pair,left,right` three-column CSV for forced assignment.

Results are stored in that pair's `col_map = {left_col: right_col}`; **only col_map entries enter extraction**. `--yes` skips manual confirmation (used by the justfile).

> [!NOTE] Don't confuse the three layers (see [[dtrack: general knowledge|6a3a7b11b439b]])
> `col_map` (final {left:right}, enters extraction) · `col_rules` (bulk generation rules, do **not** enter extraction) · `col_filter` (`col_include`/`col_exclude`, decides which columns to extract, enters extraction). Column annotations / time mappings are done on the **web's col_mapping page**, not in the CLI.

## 2. stats preset: how deep to compute per column

`--preset` (attached to col-gen, baked into col SQL) has three levels:

| Statistic | quick | standard | detailed |
|---|:--:|:--:|:--:|
| `n_total` (row count) | ✓ | ✓ | ✓ |
| `n_missing` (empty/NULL) | ✓ | ✓ | ✓ |
| `n_unique` (distinct count) | — | ✓ | ✓ |
| `min_max` | scalar | scalar | `value=cnt` |
| `length` (avg+max) | ✓ | ✓ | ✓ |
| `hash_sum` (portable md5→int16 sum) | — | ✓ | ✓ |
| `top_10` (frequency table top N) | — | — | ✓ |

- **quick**: pure flat aggregation, no frequency table → cheapest, friendly to partition pruning; min/max for categorical columns is scalar (lexicographic MIN/MAX).
- **standard**: adds `n_unique` + `hash_sum`, giving meaningful signals for cross-engine comparison; still no frequency table.
- **detailed**: adds `top_10` + `value=cnt`-form min/max → requires WITH-chain frequency table (the only level that Hive handles with GROUP BY).

> [!NOTE] mean/std are not in any preset
> Empirically poor signal-to-noise ratio and they double query cost; always `''`; the 14-column schema is unchanged.

### Per-item override: `--stats-override`
Comma-separated, keys: `n_total, n_missing, n_unique, min_max, length, hash_sum, top_10`:
```bash
dtrack generate … --type col --preset standard --stats-override n_unique=on,hash_sum=off
```

## 3. 14-column col CSV schema (fixed)

Every `{qname}_col.csv` always has these 14 columns (`COL_CSV_HEADERS`); statistics not included in the preset output `''`, so cross-platform diffs align correctly:
```text
dt, column_name, col_type, n_total, n_missing, n_unique,
mean, std, min_val, max_val, top_10, length_avg, length_max, hash_sum
```

## 4. CTE / non-introspectable tables: manual column feed

One side of `processed` (CTE joining multiple tables) has no single physical table to `DESCRIBE`, so auto-discovery skips it and shows a prompt. Provide `{qname}_columns.csv` manually (place it in `--columns-dir`); the header is flexible:

```text
column_name,data_type        # or variable,type / name,type are also accepted
cust_id,VARCHAR
amt,DOUBLE
```
`load-columns` reads it and writes into `_column_meta`, enabling the col-stats builder to pick the correct sentinel family.

| Source | How columns are obtained |
|---|---|
| oracle/aws/databricks/hadoop regular tables | `load-columns` online DESCRIBE |
| **csv source** | auto-discovered from CSV header (DuckDB DESCRIBE / pandas) — **no** manual feed needed |
| **processed / CTE** | **must** hand-write `{qname}_columns.csv` |

## Checklist
- [ ] `match-columns` prints `Auto-matched: N` / `Rule-matched: N`; the key columns for that pair are also mapped (needed for sample-hash)
- [ ] `extract_col.*.sql` covers only columns in `col_map`, including/omitting statistics per preset
- [ ] CTE side: place `{qname}_columns.csv` before running `load-columns`, otherwise that side's columns will be empty
- [ ] For cross-engine comparison, check `hash_sum`/`length_avg` signals ([[Data Engineer|6a05342d3068b]]); if in doubt run [[dtrack: CSV → CSV + sample-hash|6a3b87c0905a7]]
