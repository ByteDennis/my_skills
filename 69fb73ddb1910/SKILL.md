---
id: 69fb73ddb1910
name: Compare Two Tables (Diff by PK)
tags: [data, pandas, comparison, etl, primary-key]
updated_at: 2026-05-06T17:01:17.063856Z
---

## When to use which approach

| Scenario | Approach |
|---|---|
| Both columns clean, just find unequal rows | set difference |
| 2nd col tokenized except last n digits | composite hash key + normalize |
| PK names differ but a 3rd col is shared | merge/align on shared col |

## 1. Set difference (both clean, unsorted)

```python
import pandas as pd

a = pd.read_csv('a.csv')['id']
b = pd.read_csv('b.csv')['id']

only_in_a = set(a) - set(b)
only_in_b = set(b) - set(a)
common    = set(a) & set(b)
print(f'a-only: {len(only_in_a)} | b-only: {len(only_in_b)} | both: {len(common)}')
```

## 2. Composite hash key (when PK is partially masked)

When the natural PK is tokenized except the last *n* digits, drop the
PK and build a composite key from *other* columns. Normalize each
column first so `'2026-05-06'` == `'05/06/2026'` and `0.5000` == `0.5`.

### Normalize dates → `'YYYY-MM-DD'`

```python
import pandas as pd

# pandas accepts almost any input format → strftime back to plain string
def norm_date(x):
    return pd.to_datetime(x, errors='coerce').strftime('%Y-%m-%d')

# whole column:
df['date'] = pd.to_datetime(df['date'], errors='coerce').dt.strftime('%Y-%m-%d')
# bad values become 'NaT' string — count them as a sanity check
```

### Normalize numerics (`0.5000` → `'0.5'`)

```python
def norm_num(x, decimals=4):
    if x is None or pd.isna(x): return ''
    return f'{float(x):.{decimals}f}'.rstrip('0').rstrip('.')
```

### Normalize strings (case + whitespace)

```python
def norm_str(x):
    return '' if x is None or pd.isna(x) else str(x).strip().lower()
```

### Tail-of-PK rescue (keep last n digits)

```python
def pk_tail(s, n=4):
    return str(s)[-n:] if s and not pd.isna(s) else ''
```

### Build the composite hash key

```python
import hashlib

NORM = {
    'date': norm_date, 'txn_date': norm_date,
    'amount': norm_num, 'price': norm_num,
    'pk_tail': pk_tail,
}

def row_key(row, cols):
    parts = [str(NORM.get(c, norm_str)(row[c])) for c in cols]
    return hashlib.sha1('\x1f'.join(parts).encode()).hexdigest()[:16]

a['pk_tail'] = a['id'].map(lambda x: pk_tail(x, 4))
b['pk_tail'] = b['id'].map(lambda x: pk_tail(x, 4))

KEY_COLS = ['date', 'amount', 'name', 'pk_tail']
a['_k'] = a.apply(lambda r: row_key(r, KEY_COLS), axis=1)
b['_k'] = b.apply(lambda r: row_key(r, KEY_COLS), axis=1)

only_in_a = set(a['_k']) - set(b['_k'])
```

## 3. Align via a shared third column

Table A has `(a, b)`, table B has `(A, b)` — `a`/`A` are different
representations of the same entity but both sides share `b`. Align on `b`:

```python
merged = A.merge(
    B, on='b',
    suffixes=('_left', '_right'),
    how='outer', indicator=True,
)

print('only in A :', (merged['_merge'] == 'left_only').sum())
print('only in B :', (merged['_merge'] == 'right_only').sum())
print('aligned   :', (merged['_merge'] == 'both').sum())

# Aligned (a, A) pairs you can now spot-check:
pairs = merged[merged['_merge'] == 'both'][['a', 'A', 'b']]

# Inside aligned rows, find disagreement on other shared columns:
bad = merged[(merged['_merge'] == 'both') & (merged['amount_left'] != merged['amount_right'])]
```

## Gotchas

- **Always normalize before hashing** — a stray trailing space or `.0` bites
- `pd.to_datetime(..., errors='coerce')` → `NaT` on bad input; count + log
- For floats, prefer `round(x, n)` over string formatting if you'll do math later
- Hash `NaN` to `''` consistently so both sides match
- Set diff is O(n+m) and case-sensitive — lowercase the column first if needed
