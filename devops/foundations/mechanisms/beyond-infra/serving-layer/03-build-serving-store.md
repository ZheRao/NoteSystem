# `build_serving_store.py` — Complete Walkthrough

*A line-by-line reading of the build-time loader, plus the two topics it depends on but doesn't explain: how SQLite indexes actually work, and what the integrity check really asserts.*

---

## Table of contents

1. [The 30-second model](#1-the-30-second-model)
2. [Anatomy of a run](#2-anatomy-of-a-run)
3. [Section-by-section walkthrough](#3-section-by-section-walkthrough)
4. [Deep dive: how SQLite indexes work](#4-deep-dive-how-sqlite-indexes-work)
5. [Deep dive: what `check_integrity` actually asserts](#5-deep-dive-what-check_integrity-actually-asserts)
6. [Diagnosing your 2 orphan keys](#6-diagnosing-your-2-orphan-keys)
7. [Known sharp edges in the current file](#7-known-sharp-edges-in-the-current-file)
8. [Verification recipes](#8-verification-recipes)
9. [Cheat sheet](#9-cheat-sheet)

---

## 1. The 30-second model

The file is a **projection**, not a transformation. It does four things and nothing else:

| Step | What happens | Why it's here |
|---|---|---|
| **Preflight** | Resolve every source path; check the format is readable | So a rebuild is all-or-nothing (invariant 2) |
| **Load** | `read_csv` → `to_sql(if_exists="replace")` per table | One writer, disposable store (invariants 2, 3) |
| **Index** | `CREATE INDEX` on the filter/join keys | Cheap drill-downs at serve time (invariant 7) |
| **Verify** | Orphan-key check across declared FK edges | Catch broken keys at build, not as an empty panel at serve |

Notice what is *absent*: no aggregation, no revision resolution, no joins that produce new rows. The grain arrives from the data system and survives untouched (invariant 8). If you find yourself wanting to `GROUP BY` in this file, that's a signal the data system should have done it, or that `queries.py` should do it at serve time.

The mental frame that makes the rest of the file obvious: **this is a compiler, and the store is a build artifact.** Compilers are allowed to be slow. Compilers are allowed to fail loudly. Compilers must be deterministic — same inputs, byte-identical-in-spirit output. Every design choice below follows from that.

---

## 2. Anatomy of a run

```
python build/build_serving_store.py
  │
  ├─ print header (store path, input dir, format)
  │
  ├─ build()
  │    ├─ _preflight()                      ← raises before any write
  │    ├─ mkdir -p on store's parent dir
  │    ├─ sqlite3.connect(STORE_PATH)       ← creates the .db file if absent
  │    └─ for table, spec in TABLE_SPECS:
  │         ├─ read_table(spec)             → DataFrame
  │         ├─ df.to_sql(replace)           → DROP TABLE + CREATE TABLE + INSERTs
  │         ├─ create_indexes(...)          → CREATE INDEX ×N
  │         └─ print progress line
  │       conn.commit()
  │
  ├─ summary(conn)                          ← reads sqlite_master
  │
  ├─ check_integrity(conn)                  ← N orphan-count queries
  │    └─ if problems and STRICT_INTEGRITY: SystemExit
  │
  └─ conn.close()
```

Two things worth internalizing about that shape:

- **The connection is returned, not hidden.** `build()` hands back a live `sqlite3.Connection`. That's what lets `check_integrity` and `summary` inspect the result without reopening the file, and it's what would let a test call `build()` and assert on the store directly.
- **Ordering is dictated by dict insertion order.** `TABLE_SPECS` is a plain dict, and Python dicts preserve insertion order, so `input` loads before `output` before `input_output_linkage`, and so on. Today nothing depends on that order (no real FK constraints are declared in SQLite, see §5), but if you ever add `PRAGMA foreign_keys=ON` with declared FKs, load order becomes load-bearing.

---

## 3. Section-by-section walkthrough

### 3.1 `from __future__ import annotations`

Makes every annotation in the module a **string** at runtime instead of an evaluated object. Two practical consequences:

- You can write `dict[str, dict]` and `list[tuple[str, str, str, str]]` without importing from `typing`, on Python versions where those builtins weren't yet subscriptable at runtime.
- Annotations cost nothing at import time — nothing is evaluated. This matters for large annotation graphs; here it's mostly hygiene.

Python analogy: it's the difference between `eval("Path")` at def-time and storing the literal text `"Path"` for later inspection.

### 3.2 `TABLE_SPECS` — the declarative plan

```python
TABLE_SPECS: dict[str, dict] = {
    "input": {"file": "input", "subdir": "input", "indexes": [("input_output_key",)]},
    ...
}
```

The key of the outer dict is the **DB table name**. `spec["file"]` is the **filename stem on disk**. They're deliberately decoupled — see the `clients` entry, where the export is `client.csv` (singular) but the table is `clients` (plural) so that the FK edges read naturally. That decoupling is the single most useful property of this structure: renaming an export never touches a query.

`indexes` is a **list of tuples**, not a list of strings. A one-column index is `("input_output_key",)` — note the trailing comma, without which it's just a parenthesized string and `'_'.join(cols)` would iterate characters, producing `idx_input__i_n_p_u_t...`. The tuple-of-columns shape is what allows composite indexes like `("orchestration_key", "month", "source")` to be expressed in the same structure.

The comment on the file is accurate and worth respecting: *"Adding a table later is a data change here, not new code below."* This is table-driven programming. The loop in `build()` doesn't know how many tables exist, and it never will.

One inaccuracy to flag: the comments say `input` *"exposes a unique input_output_key"* and `output` *"exposes a unique output_orchestration_key"*, but the created indexes are **not** `UNIQUE`. The comment states an invariant the code doesn't enforce. See §7.1.

### 3.3 `FOREIGN_KEYS` — the edge list

```python
FOREIGN_KEYS: list[tuple[str, str, str, str]] = [
    (child_table, child_col, parent_table, parent_col),
    ...
]
```

This is a graph, expressed as an edge list. It is **not** used to create SQLite `FOREIGN KEY` constraints — pandas' `to_sql` emits a bare `CREATE TABLE` with no constraints at all, and SQLite's FK enforcement is off by default anyway. These tuples exist for exactly one purpose: to drive the loop in `check_integrity`.

Read the edges as "every value of the child column must appear somewhere in the parent column":

```
input_output_linkage.input_output_key            → input.input_output_key
input_output_linkage.output_orchestration_key    → output.output_orchestration_key      ← your warning
cash_flow_streams.output_orchestration_key       → input_output_linkage.output_orchestration_key
cash_flow_streams.client_id                      → clients.client_id
cash_flow_streams.orchestration_key              → versions.orchestration_key
```

Note that the third edge points at a **non-unique** parent column. `input_output_linkage` has many rows per `output_orchestration_key` (that's the whole point of a link table). So this edge asserts "existence somewhere in the link table," not "references one row." That's still a meaningful assertion, but it's a different kind of assertion than the others, and it's arguably pointed at the wrong parent — the semantically natural parent for `cash_flow_streams.output_orchestration_key` is `output.output_orchestration_key`. See §7.4.

### 3.4 `READERS` — dispatch table instead of if/elif

```python
READERS = {"csv": pd.read_csv, "parquet": pd.read_parquet}
```

Functions are first-class, so this dict *is* the branch. `READERS[config.INPUT_FORMAT](path)` replaces an if/elif chain, and `sorted(READERS)` gives you the valid-values list in the error message for free — so the error message can never drift out of sync with the actual supported set. Adding Feather support is one dict entry.

### 3.5 `_source_path`

```python
def _source_path(spec: dict) -> Path:
    return config.INPUT_DIR / spec.get("subdir", "") / f"{spec['file']}.{config.INPUT_FORMAT}"
```

`spec.get("subdir", "")` makes the subdirectory optional — and `Path("/a") / "" / "b.csv"` is `/a/b.csv`, because `pathlib` discards empty components. So a spec without `subdir` reads from the input root. Nice property, easy to miss.

The leading underscore is the module-private convention: this is an implementation detail, not part of the module's contract. (Python doesn't enforce it; it's a signal to readers and to `from module import *`.)

### 3.6 `_preflight` — the all-or-nothing gate

```python
missing = [str(_source_path(spec)) for spec in TABLE_SPECS.values()
           if not _source_path(spec).exists()]
```

This is the invariant-2 enforcement point, and it's the most important defensive design in the file. Consider the alternative: no preflight, and the fifth of six exports is missing. You'd get four tables replaced with fresh data, then a `FileNotFoundError`, leaving a store where four tables are new and two are stale — a **torn rebuild**. The store would open fine, queries would run fine, and the numbers would be quietly wrong. Preflight converts a silent correctness bug into a loud startup failure.

Note that it checks *before* `sqlite3.connect`, so the `.db` file isn't even created (or touched) on a failed run.

Two small notes:

- `_source_path(spec)` is called twice per spec (once in the condition, once in the value). Harmless — `Path` construction is cheap and `.exists()` is one `stat` call — but a walrus (`if not (p := _source_path(spec)).exists()`) would halve it if you ever cared.
- Preflight is a **time-of-check-to-time-of-use** window: a file could vanish between the check and the read. In a batch build on a local filesystem this is a non-issue, which is why `read_table` still keeps its own existence check as a backstop.

### 3.7 `read_table`

Duplicates the format validation and the existence check from preflight. That duplication is deliberate defense-in-depth: `read_table` is independently callable (from a REPL, from a test) and shouldn't assume preflight ran. The format check being in both places means neither function has a hidden prerequisite.

`READERS[config.INPUT_FORMAT](path)` — note that pandas is doing all type inference here, and that inference is where a surprising number of build-time bugs originate. See §6.2.

### 3.8 `create_indexes`

```python
def create_indexes(conn, table, spec) -> None:
    for cols in spec["indexes"]:
        index_name = f"idx_{table}__{'_'.join(cols)}"
        col_list = ", ".join(f'"{c}"' for c in cols)
        conn.execute(f'CREATE INDEX IF NOT EXISTS "{index_name}" ON "{table}" ({col_list})')
```

Three details:

1. **Naming convention.** `idx_{table}__{cols_joined}` with a double underscore separating table from columns. Index names are global in a SQLite database (not scoped per table), so the table name must be in there. The double underscore makes the boundary parseable by eye when a column name itself contains underscores — which all of yours do.

2. **Identifier quoting.** Table and column names are interpolated into the SQL string, then wrapped in double quotes. You *cannot* parameterize identifiers in SQL — `?` placeholders only work for values. So string interpolation is unavoidable here; the `"..."` quoting is what makes it safe against names with spaces or reserved words. This is fine because the identifiers come from a literal dict in the source file, not from user input. If `TABLE_SPECS` ever became externally sourced (a YAML config, say), you'd need to validate identifiers against `^[A-Za-z_][A-Za-z0-9_]*$` before interpolating.

3. **`IF NOT EXISTS` is nearly vestigial.** `to_sql(if_exists="replace")` issues a `DROP TABLE` — and **dropping a table drops all its indexes**. So by the time `create_indexes` runs, the index never exists. The clause is harmless insurance for the case where someone calls `create_indexes` twice, or where a future refactor switches to truncate-and-insert instead of drop-and-recreate.

### 3.9 `_columns` and `PRAGMA table_info`

```python
def _columns(conn, table) -> set[str]:
    return {row[1] for row in conn.execute(f'PRAGMA table_info("{table}")')}
```

`PRAGMA table_info(t)` returns one row per column: `(cid, name, type, notnull, dflt_value, pk)`. So `row[1]` is the column name and `row[2]` would be the declared type — useful later when you're chasing type-affinity bugs (§6.2).

A `set` because the only operation performed is membership testing (`child_c not in _columns(...)`), which is O(1) on a set and O(n) on a list.

Worth knowing: if the table doesn't exist, `PRAGMA table_info` returns **zero rows** rather than raising. So a typo'd table name in `FOREIGN_KEYS` produces "skipped (column not present)" rather than an error. That's the one place the integrity check can be too quiet — see §7.2.

### 3.10 `build`

```python
config.STORE_PATH.parent.mkdir(parents=True, exist_ok=True)
conn = sqlite3.connect(config.STORE_PATH)
```

`parents=True` creates intermediate directories; `exist_ok=True` makes it idempotent. `sqlite3.connect` on a non-existent path **creates** the file — which is why the parent directory must exist first, and why preflight has to come before this line.

```python
df.to_sql(table, conn, if_exists="replace", index=False)
```

- `if_exists="replace"` → `DROP TABLE IF EXISTS t; CREATE TABLE t (...); INSERT ...`. This is the idempotence mechanism. Re-running the build from the same exports yields the same store, and there's no accumulation of stale rows.
- `index=False` → don't write the DataFrame's index as a column. Without this you'd get a spurious `index` integer column in every table. Since your keys are natural keys carried in the data, the pandas index carries no information.
- The `CREATE TABLE` that pandas generates has **no primary key, no NOT NULL, no foreign keys** — just column names and types. All structure lives in the indexes you add afterward and in the checks you run afterward.

```python
print(f"  loaded {table:<22} {len(df):>6} rows   <- {_source_path(spec)}")
```

Format specs: `:<22` left-aligns and pads to 22 chars; `:>6` right-aligns to 6. That's what makes the console output form clean columns. Small thing, but it's the difference between a log you can scan and a log you have to read.

```python
conn.commit()
```

Python's `sqlite3` module, at default `isolation_level=""`, implicitly opens a transaction before DML (`INSERT`/`UPDATE`/`DELETE`) and requires an explicit `commit()`. DDL behavior varies by Python version (3.6+ no longer implicitly commits before DDL). The single `commit()` at the end is the simple correct answer: everything lands or, if the process dies mid-build, the journal rolls back.

### 3.11 `check_integrity`

Covered in depth in §5. Structurally: loop the edges, skip edges whose columns are absent, count orphans, accumulate human-readable strings, return the list. Note the return-a-list-of-problems shape rather than raising — that lets the caller decide severity, which is exactly what `STRICT_INTEGRITY` does at the bottom of the file. A function that gathers facts and a caller that sets policy.

### 3.12 `summary` and `sqlite_master`

```sql
SELECT name FROM sqlite_master WHERE type='table' ORDER BY name
SELECT name FROM sqlite_master WHERE type='index' AND tbl_name=? AND name NOT LIKE 'sqlite_%' ORDER BY name
```

`sqlite_master` is SQLite's catalog table: one row per table, index, view, and trigger, with columns `(type, name, tbl_name, rootpage, sql)`. Querying it is how you introspect a database without external tooling. `sql` holds the original `CREATE` statement — very handy when you want to see exactly what pandas generated:

```sql
SELECT sql FROM sqlite_master WHERE name = 'cash_flow_streams';
```

The `NOT LIKE 'sqlite_%'` filter excludes SQLite's internal auto-created indexes (from `UNIQUE`/`PRIMARY KEY` declarations, named `sqlite_autoindex_*`). Note that this query **does** use a `?` placeholder for `tbl_name` — because that's a *value* being compared, not an identifier. Contrast with `create_indexes`, where interpolation was forced. Good illustration of the rule.

### 3.13 `if __name__ == "__main__"`

The orchestration and the **policy decision**: warnings print, and only escalate to `SystemExit` if `config.STRICT_INTEGRITY` is set. Note the `conn.close()` before raising — closing the connection before exiting means the journal is cleaned up rather than left for recovery.

The recommended posture: **`GROWLYTICS_STRICT_INTEGRITY=true` in CI and in any deploy pipeline, off for local exploration.** Warnings that nobody is forced to read are warnings that get ignored, and a broken key becomes an empty drill-down panel three weeks later.

---

## 4. Deep dive: how SQLite indexes work

This is the part you said you wanted, so it gets its own full treatment.

### 4.1 What a table physically is

Every table created by `to_sql` is a **rowid table**: SQLite gives it an implicit hidden 64-bit integer key called `rowid`, and stores the table as a **B-tree keyed on rowid**. The B-tree is a balanced multi-way tree; interior nodes hold rowid ranges and pointers to child pages, leaf nodes hold the actual row contents.

The consequence: SQLite can find *"the row with rowid 8,412"* in a handful of page reads (O(log n)), but it has no idea where *"the row with orchestration_key = 'ORCH-2026-04'"* lives. To answer that, it must read every leaf page and compare — a **full table scan**, O(n).

For 500 rows, who cares. For 4 million monthly cash-flow stream rows, a full scan on every drill-down click is the difference between 3ms and 900ms, multiplied by every panel on the page.

### 4.2 What an index physically is

An index is a **second B-tree**, stored separately in the same file, keyed on the indexed columns instead of rowid. Each entry is:

```
(indexed_col_1, indexed_col_2, ..., rowid)
```

sorted lexicographically by that whole tuple. That's it. An index is a sorted copy of a few columns plus a pointer back to the row.

So `idx_cash_flow_streams__client_id` is a B-tree of `(client_id, rowid)` pairs sorted by `client_id`. A query for `WHERE client_id = 'FARM-0042'`:

1. Binary-search the index B-tree down to the first entry with that `client_id`. → ~3 page reads.
2. Walk forward through consecutive index entries while `client_id` still matches, collecting rowids. → cheap, they're contiguous.
3. For each rowid, seek into the table B-tree to fetch the full row. → ~3 page reads *each*.

Step 3 is why an index isn't magic. If a query would match 60% of the table, the index makes things *slower* — you'd do 0.6n random seeks instead of one sequential scan. SQLite's query planner estimates this and will deliberately ignore an index it judges unhelpful. (It estimates better after you run `ANALYZE`; see §4.7.)

### 4.3 Composite indexes and the leftmost-prefix rule

Your fact table has:

```python
("orchestration_key", "month", "source")   # the level 0 -> 1 drill path
```

This is one B-tree sorted by `orchestration_key`, then `month` within that, then `source` within that. Think of a phone book sorted by (last name, first name, middle name).

**The rule: an index can serve a query only via a leftmost prefix of its columns.**

| Query predicate | Can this index help? |
|---|---|
| `WHERE orchestration_key = ?` | ✅ prefix of length 1 |
| `WHERE orchestration_key = ? AND month = ?` | ✅ prefix of length 2 |
| `WHERE orchestration_key = ? AND month = ? AND source = ?` | ✅ full |
| `WHERE orchestration_key = ? AND month BETWEEN ? AND ?` | ✅ equality then range on the last used column |
| `WHERE orchestration_key = ? AND source = ?` | ⚠️ partial — seeks on `orchestration_key`, then filters `source` row by row |
| `WHERE month = ?` | ❌ no leftmost prefix — full scan |
| `WHERE source = ?` | ❌ full scan |

Phone book intuition: you can find all the Smiths, and all the Smith Johns. You cannot find all the Johns.

The critical corollary: **column order in a composite index is a design decision, not a formatting choice.** `("orchestration_key", "month", "source")` is right for your drill path because every query filters `orchestration_key` first — it's the revision scope, always known. If you ever add a panel that filters `month` alone across all revisions, that panel gets a full scan, and it needs its own index.

There's a second, less obvious benefit: a composite index also satisfies **`ORDER BY`** on its prefix for free. `WHERE orchestration_key = ? ORDER BY month` needs no sort step at all, because the index already delivers rows in `month` order within that key. Sorts are memory-hungry and often the hidden cost in a slow query; eliminating one is a bigger win than it looks.

### 4.4 Covering indexes

If **every column a query touches** is in the index, SQLite skips step 3 entirely — no table seeks, answer comes from the index B-tree alone. That's a **covering index**, and it's often 5–10× faster than an index that requires row lookups.

```sql
-- covered by idx_cash_flow_streams__orchestration_key_month_source:
SELECT month, source FROM cash_flow_streams WHERE orchestration_key = ?;

-- NOT covered — `amount` isn't in the index, so every matching row needs a table seek:
SELECT month, source, SUM(amount) FROM cash_flow_streams
WHERE orchestration_key = ? GROUP BY month, source;
```

That second query is very likely your level-0 chart. If it's ever the bottleneck, the fix is to append the measure column to the index:

```sql
CREATE INDEX idx_cash_flow_streams__l0_covering
  ON cash_flow_streams (orchestration_key, month, source, amount);
```

Now the aggregate reads only the index, in already-grouped order. Note the tradeoff — `amount` is duplicated on disk, and the index is larger, so more pages to read per seek. Measure before and after; don't add covering columns speculatively.

### 4.5 One index per table per query

SQLite's planner generally uses **at most one index per table per query** (the exception is the OR-optimization, which can union results from several — `EXPLAIN QUERY PLAN` shows `MULTI-INDEX OR`). This is why `cash_flow_streams` has four separate index entries in your spec rather than one big one: each serves a different access pattern, and the planner picks whichever fits the incoming `WHERE` clause.

It also means adding a fifth redundant index doesn't compound the benefit. Indexes that are a leftmost prefix of another index are pure overhead: `("orchestration_key",)` is fully subsumed by `("orchestration_key", "month", "source")`, since any query the short one serves, the long one serves identically. Your spec has both. Harmless (the short one is smaller, so marginally cheaper for the planner to choose in a scan-heavy case), but if you're minimizing build time and file size, the standalone `("orchestration_key",)` is the one to drop.

### 4.6 What indexes cost

| Cost | Applies here? |
|---|---|
| Disk space (a sorted copy of those columns + rowid) | Yes — usually the dominant cost in this design |
| Slower `INSERT`/`UPDATE`/`DELETE` (every index B-tree must be updated) | Only at build time |
| Query planner has more options to consider | Negligible |

This is exactly why the architecture works. In a transactional system, indexes are a genuine tradeoff — you pay on every write. Here, writes happen once, in batch, off the request path (invariant 1), and the serving connection is read-only (invariant 3). **You pay for indexes in build seconds and get them back in serve milliseconds, forever.** Index generously.

One build-order optimization if the build ever gets slow: creating indexes **after** the bulk insert (which is what this file does — `to_sql` then `create_indexes`) is much faster than inserting into an already-indexed table, because SQLite can sort the whole column set once instead of doing n incremental B-tree insertions. The current order is already correct; don't reverse it.

### 4.7 `EXPLAIN QUERY PLAN` — how to know, not guess

Never assume an index is used. Ask:

```sql
EXPLAIN QUERY PLAN
SELECT month, source, SUM(amount) FROM cash_flow_streams
WHERE orchestration_key = 'ORCH-2026-04' GROUP BY month, source;
```

Read the output for these words:

| Output contains | Meaning |
|---|---|
| `SEARCH ... USING INDEX idx_...` | ✅ index used for lookup |
| `SEARCH ... USING COVERING INDEX idx_...` | ✅✅ index used, no table seeks |
| `SCAN table` | ❌ full table scan |
| `USE TEMP B-TREE FOR ORDER BY` / `FOR GROUP BY` | ⚠️ a sort is happening that an index could eliminate |
| `USING AUTOMATIC COVERING INDEX` | ⚠️ SQLite built a throwaway index at runtime — you're missing a permanent one |

That last one is a gift: SQLite is telling you precisely which index to create.

`ANALYZE` populates the `sqlite_stat1` table with cardinality statistics, which the planner uses to make better choices. Running it once at the end of a build is a reasonable addition:

```python
conn.execute("ANALYZE")
```

It's cheap on a store this size and makes the planner's estimates real rather than heuristic. Worth adding.

### 4.8 `UNIQUE` indexes are constraints, not just speedups

`CREATE UNIQUE INDEX` does everything a normal index does **and** rejects duplicate values at insert time. For `input.input_output_key` and `output.output_orchestration_key` — columns your own comments describe as unique — this converts a documented assumption into an enforced one. See §7.1 for how to wire it in.

---

## 5. Deep dive: what `check_integrity` actually asserts

### 5.1 The query, read carefully

```sql
SELECT COUNT(*) FROM "input_output_linkage" c
WHERE c."output_orchestration_key" IS NOT NULL
  AND c."output_orchestration_key" NOT IN (
        SELECT DISTINCT "output_orchestration_key" FROM "output"
      )
```

In English: *count child rows whose key is non-null and does not appear anywhere in the parent's key column.* Each such row is an **orphan** — a reference to something that doesn't exist. At serve time, an orphan is a join that silently drops a row, which is a number that's silently too small.

Note it counts **rows**, not distinct keys. "2 orphan key(s)" means 2 orphan *rows*, which could be 2 rows sharing one bad key or 2 rows with 2 different bad keys. Worth knowing before you go looking.

`DISTINCT` in the subquery is semantically a no-op — `IN` doesn't care about duplicates — and it may even cost you, since it can force a dedup pass. With an index on `output.output_orchestration_key` (which you have), SQLite can usually probe the index directly. Dropping `DISTINCT` is a safe micro-optimization.

### 5.2 The `NULL` trap — and this file only half-escapes it

SQL's `NOT IN` is three-valued. If the subquery result contains **even one `NULL`**, then for any value `x`:

```
x NOT IN (list containing NULL)  →  NULL   (never TRUE)
```

Because SQL can't prove `x ≠ NULL` — unknown compared to anything is unknown. So `WHERE ... AND NULL` filters out every row, and the count comes back **0**.

The file guards the **child** side (`c."key" IS NOT NULL`) but not the **parent** side. So:

> **If `output.output_orchestration_key` contains a single NULL, `check_integrity` reports zero orphans for that edge — no matter how broken the data is.**

This is the most dangerous single line in the file, because it fails in the direction of false confidence. Before trusting *any* clean result from this check, verify:

```sql
SELECT COUNT(*) FROM output WHERE output_orchestration_key IS NULL;
```

The robust rewrite avoids three-valued logic entirely by using an anti-join:

```sql
SELECT COUNT(*)
FROM "input_output_linkage" c
LEFT JOIN "output" p
  ON c."output_orchestration_key" = p."output_orchestration_key"
WHERE c."output_orchestration_key" IS NOT NULL
  AND p."output_orchestration_key" IS NULL;
```

`LEFT JOIN` keeps every child row; where no parent matched, the parent columns are `NULL`; so `WHERE p.key IS NULL` selects exactly the non-matching children. A parent-side `NULL` can't match anything and simply never participates. `NOT EXISTS (SELECT 1 FROM p WHERE p.key = c.key)` is equally correct and equally NULL-safe. Either is strictly better than `NOT IN` here.

Since your check *did* fire, the NULL trap isn't currently masking this edge. But it may be masking the other four.

### 5.3 What the check does not catch

Being precise about the guarantee matters as much as the guarantee:

- ❌ **Duplicate parent keys.** Nothing declares or enforces uniqueness, so if `output` has two rows for one `output_orchestration_key`, every join through it fans out and doubles money. This is the exact class of bug your grain invariant exists to prevent, and this check is blind to it.
- ❌ **Childless parents.** An `output` row that no linkage row references passes silently. Usually fine, sometimes a signal that an export stage dropped rows.
- ❌ **Cardinality expectations.** "Each output should have ≥1 input" is not expressed anywhere.
- ❌ **Grain/fan-out safety.** The check validates *reference existence*, never *row-count arithmetic*.
- ❌ **NULL keys in the child.** They're explicitly excluded from the count. A NULL foreign key is often a real defect worth its own separate count.

Worth adding as separate assertions once the orphan issue is closed. Especially the duplicate-parent check — it's a one-liner and it protects your most important invariant:

```sql
SELECT COUNT(*) FROM (
  SELECT output_orchestration_key FROM output
  GROUP BY output_orchestration_key HAVING COUNT(*) > 1
);
```

---

## 6. Diagnosing your 2 orphan keys

```
input_output_linkage.output_orchestration_key -> output.output_orchestration_key   2 orphan key(s)
```

Two linkage rows point at outputs that aren't in `output`. Since you just extended the forecast engine, the causes fall into three families. Work them in this order — cheapest first, and the ordering reflects which is most likely given that you recently changed the engine.

### 6.1 Family A: genuine data drift (most likely)

The engine changed, and the two sides of the export are no longer in agreement. Common shapes:

- **Revision resolution now filters `output`.** Your versioning logic resolves *"highest output revision ≤ orchestration revision."* If the `output` export now emits only resolved revisions while the `linkage` export still emits rows for superseded ones, those linkage rows become orphans by construction. **This is the most likely cause**, and if so the fix belongs in the data system: apply the same revision filter to the linkage export, or don't filter `output` at all and let the serving layer traverse.
- **Stale export.** `output.csv` was written by the old engine, `input_output_linkage.csv` by the new one — or vice versa. Check mtimes; this costs five seconds and explains a shocking share of these.
- **A pipeline stage dropped rows.** A new `dropna`, a tightened join, or a filter in the output stage that the linkage stage doesn't mirror.

### 6.2 Family B: type / representation mismatch (cheap to rule out)

Even when the underlying values agree, the *stored representations* may not. SQLite has three comparison classes: NULL, **numeric** (`INTEGER` and `REAL` together), and **TEXT** (then BLOB). Within the numeric class, values compare by magnitude — so `INTEGER 1234` **does** equal `REAL 1234.0`, and pandas' float upcasting alone won't break a join. But **all numeric values sort before all text values**, so `5` and `'5'` are never equal, no matter what.

The mechanism to watch for, then, is one side landing as text while the other lands as numeric. Pandas decides per column, per file: if `output.csv` has a clean numeric key column it becomes `int64` → `INTEGER`, while if `input_output_linkage.csv` has even one non-numeric value in that column (a header artifact, an `N/A`, a padded value) the whole column becomes `object` → `TEXT`. Same logical keys, two comparison classes, zero matches.

Float upcasting still bites in two narrower ways worth knowing: integer keys above 2^53 lose precision as `float64`, and any code path that stringifies the key gets `'1234.0'` instead of `'1234'`.

Your keys are string natural keys, which makes this less likely than it would have been with the old integer surrogates — but check anyway, along with the string-specific variants:

- **Trailing/leading whitespace** — `'ORCH-04'` vs `'ORCH-04 '`. Extremely common when a CSV was ever touched by Excel.
- **Case difference** — SQLite's default `=` is case-*sensitive* for text.
- **Unicode lookalikes** — a non-breaking space (U+00A0) or an en-dash where a hyphen belongs. Invisible in every viewer.

Diagnostic:

```sql
SELECT DISTINCT typeof(output_orchestration_key) FROM input_output_linkage;
SELECT DISTINCT typeof(output_orchestration_key) FROM output;
```

If those differ, that's your answer.

### 6.3 Family C: specification error

The FK edge is asserting something that isn't actually true of your model. If linkage legitimately references outputs that live outside this export's scope, the edge is wrong, not the data. Least likely, but worth ruling in before you "fix" correct data.

### 6.4 Run this

Save and run — it walks all three families and tells you which one you're in:

```bash
python diagnose_orphans.py
```

The script is included alongside this document. It prints, for the failing edge: the orphan rows themselves, the storage types on both sides, a whitespace/case-insensitive rematch attempt, a numeric-cast rematch attempt, and the nearest candidate keys in `output`. Reading its output should tell you the family in under a minute.

### 6.5 Interpreting the result

| What the script shows | Family | Fix |
|---|---|---|
| Types differ (`text` vs `real`/`integer`) | B | Force dtype in the export or the read; see §7.3 |
| Trimmed/case-folded match succeeds | B | Normalize keys at export; consider normalizing at load |
| Keys look like superseded revisions | A | Fix the data system's export filter — not this file |
| Keys absent entirely, no near match | A | Trace back through the engine to the stage that dropped them |
| Keys reference a legitimately external scope | C | Remove or narrow the FK edge, and document why |

**A note on the temptation:** it will be tempting to filter the orphans out at load time in `build_serving_store.py`. Don't. This file is a projection (invariant 8) — the moment it starts silently discarding rows, the store stops being a faithful copy of the data system's output, and the next inconsistency becomes invisible. Fix it upstream, or declare the edge wrong. The loader's job is to *report* the discrepancy, which it just did correctly.

---

## 7. Known sharp edges in the current file

### 7.1 Uniqueness is commented, not enforced

The comments claim uniqueness; the indexes don't. Fix: let the spec say so.

```python
"input": {
    "file": "input",
    "subdir": "input",
    "indexes": [("input_output_key",)],
    "unique_indexes": [("input_output_key",)],
},
```

```python
def create_indexes(conn: sqlite3.Connection, table: str, spec: dict) -> None:
    for cols in spec["indexes"]:
        _create_index(conn, table, cols, unique=False)
    for cols in spec.get("unique_indexes", ()):
        _create_index(conn, table, cols, unique=True)

def _create_index(conn, table: str, cols: tuple[str, ...], *, unique: bool) -> None:
    kind = "UNIQUE INDEX" if unique else "INDEX"
    suffix = "__uq" if unique else ""
    name = f"idx_{table}__{'_'.join(cols)}{suffix}"
    col_list = ", ".join(f'"{c}"' for c in cols)
    conn.execute(f'CREATE {kind} IF NOT EXISTS "{name}" ON "{table}" ({col_list})')
```

Now a duplicate parent key fails the build with `IntegrityError` instead of quietly doubling money three layers up. Given your grain invariant, this is the single highest-value change in this list.

(If you add this and the build immediately fails — that's the feature working. It means you had a latent fan-out risk.)

### 7.2 A typo'd table name is reported as "skipped"

`PRAGMA table_info` on a nonexistent table returns zero rows, so `_columns` returns an empty set, so the edge is silently skipped. Guard it:

```python
existing = {r[0] for r in conn.execute(
    "SELECT name FROM sqlite_master WHERE type='table'")}
for child_t, child_c, parent_t, parent_c in FOREIGN_KEYS:
    for t in (child_t, parent_t):
        if t not in existing:
            raise KeyError(f"FOREIGN_KEYS references unknown table {t!r}")
```

A missing *column* is a legitimate "skip" (schemas evolve). A missing *table* is always a bug in `FOREIGN_KEYS`.

### 7.3 Key dtypes are whatever pandas guessed

Nothing pins the storage type of key columns. Pinning them removes an entire failure family (§6.2) permanently. Add an optional `dtypes` to the spec and pass it through:

```python
"input_output_linkage": {
    ...
    "dtypes": {"input_output_key": "string", "output_orchestration_key": "string"},
},
```

```python
def read_table(spec: dict) -> pd.DataFrame:
    ...
    kwargs = {"dtype": spec["dtypes"]} if "dtypes" in spec and config.INPUT_FORMAT == "csv" else {}
    return READERS[config.INPUT_FORMAT](path, **kwargs)
```

Keys should be text, always, even when they look numeric. A key is an identifier, not a quantity — you never add two of them.

### 7.4 The `cash_flow_streams → input_output_linkage` edge points at a non-unique parent

As noted in §3.3, this asserts existence-in-a-link-table rather than a reference to a unique row. Consider pointing it at `output.output_orchestration_key` instead, which is where the key actually originates, and keeping the linkage edge as a separate coverage check ("every output referenced by the fact table has at least one linkage row").

### 7.5 `NOT IN` → anti-join

Per §5.2. Replacing the `NOT IN` with the `LEFT JOIN` form closes the parent-NULL blind spot on all five edges at once. Small diff, removes a class of false negatives.

### 7.6 `output_orchestration_linkage` is unloaded

The `NOTE` comment already flags this. The comment is doing its job — a documented gap is fine. Just make sure it doesn't become permanent by accident; a gap that's been "temporary" for six months is a design decision nobody made on purpose.

---

## 8. Verification recipes

Open the store read-only from a REPL and poke at it. Read-only is the right habit — it makes invariant 3 physically true for your exploratory session, not just aspirationally true:

```python
import sqlite3
conn = sqlite3.connect(f"file:{path}?mode=ro", uri=True)
```

**Confirm which indexes exist and their definitions**

```sql
SELECT name, sql FROM sqlite_master WHERE type='index' AND name NOT LIKE 'sqlite_%';
```

**Confirm an index is actually being used**

```sql
EXPLAIN QUERY PLAN SELECT * FROM cash_flow_streams WHERE orchestration_key = 'X';
-- want: SEARCH cash_flow_streams USING INDEX idx_cash_flow_streams__orchestration_key (orchestration_key=?)
```

**Prove the leftmost-prefix rule to yourself**

```sql
EXPLAIN QUERY PLAN SELECT * FROM cash_flow_streams WHERE month = '2026-04';
-- expect: SCAN cash_flow_streams        ← no prefix match, full scan
EXPLAIN QUERY PLAN SELECT * FROM cash_flow_streams
  WHERE orchestration_key = 'X' AND month = '2026-04';
-- expect: SEARCH ... USING INDEX ... (orchestration_key=? AND month=?)
```

Running these two back to back is the fastest way to make §4.3 stop being abstract.

**Check for parent-side NULLs (the §5.2 blind spot) on every edge**

```sql
SELECT 'output' t, COUNT(*) FROM output WHERE output_orchestration_key IS NULL
UNION ALL SELECT 'input',   COUNT(*) FROM input   WHERE input_output_key IS NULL
UNION ALL SELECT 'clients', COUNT(*) FROM clients WHERE client_id IS NULL
UNION ALL SELECT 'versions',COUNT(*) FROM versions WHERE orchestration_key IS NULL;
```

Any non-zero here means the corresponding edge's clean result was meaningless.

**Check for duplicate parent keys (the fan-out risk)**

```sql
SELECT output_orchestration_key, COUNT(*) c FROM output
GROUP BY 1 HAVING c > 1;
```

**Index size on disk**

```sql
-- requires the dbstat virtual table (built into most SQLite builds)
SELECT name, SUM(pgsize) bytes FROM dbstat GROUP BY name ORDER BY bytes DESC;
```

---

## 9. Cheat sheet

**Indexes**

- An index is a separate B-tree of `(indexed columns..., rowid)`, kept sorted.
- No index → full scan, O(n). Index → binary search, O(log n) + one table seek per matched row.
- Composite indexes serve **leftmost prefixes only**. `(a,b,c)` helps `a`, `a+b`, `a+b+c`. Never `b` alone.
- Equality on all leading columns, then **one** range column, is the sweet spot.
- If every column the query touches is in the index, it's **covering** — no table seeks, much faster.
- Composite indexes also eliminate `ORDER BY` sorts on their prefix.
- ~One index per table per query. Redundant prefix indexes are overhead, not compounding benefit.
- Indexes cost disk and write time. Write time is free here (batch build, read-only serve), so index generously.
- Create indexes **after** bulk insert, not before.
- `EXPLAIN QUERY PLAN`: `SEARCH ... USING INDEX` good, `USING COVERING INDEX` better, `SCAN` bad, `AUTOMATIC ... INDEX` means create a real one.
- `UNIQUE INDEX` = index + enforced constraint. Use it wherever you believe a key is unique.

**Integrity**

- The check asserts **only** "every non-null child key exists somewhere in the parent column."
- `NOT IN` with a subquery is NULL-poisoned: one parent NULL → zero orphans reported, always. Use `LEFT JOIN ... IS NULL` or `NOT EXISTS`.
- The check is blind to duplicate parents, childless parents, cardinality rules, and fan-out. Add those separately.
- "N orphan key(s)" counts **rows**, not distinct keys.
- Numeric values all sort before text values, so `'5'` ≠ `5` — but `5` **does** equal `5.0`. Pin key columns to text on read, on both sides.

**This file's contract**

- Runs in batch. Never on the request path.
- Drops and replaces everything. Re-running is always safe.
- Preflight before connect, so a failed build never leaves a torn store.
- Loads at source grain. Aggregates nothing. Resolves nothing.
- Reports discrepancies; does not paper over them. Fix data problems upstream.
- Set `GROWLYTICS_STRICT_INTEGRITY=true` in CI. Warnings nobody must read are warnings nobody reads.


# Code Appendix

## `build_serving_store.py`

```py
"""
src.growlytics_platform.serving_system.build.build_serving_store — Phase 0, the build-time loader.

WHAT THIS IS
    The first process of the serving layer. It takes the output tables produced
    by the data system and projects them into a single read-optimized SQLite
    file (the "store") that the FastAPI layer will later read.

WHERE IT SITS (the invariants this file protects)
    1  Build vs serve are separate triggers. This runs in BATCH, off the request
       path. A user click must never reach this file.
    2  The store is a disposable, rebuildable projection. Re-running is safe:
       every table is dropped and replaced, so the output is identical each time.
    3  Exactly one writer. This loader is it. The API only ever reads.
    7  Every servable slice is addressable by stable keys; the join/filter keys
       are indexed here so drill-downs are cheap.
    8  Row-level grain and lineage are preserved. We load the normalized tables
       as-is and let queries aggregate at serve time — nothing is pre-summarized.

HOW TO RUN
    From the project root:
        python build/build_serving_store.py
    Point it at your real exports by setting environment variables (see config.py):
        GROWLYTICS_INPUT_DIR=/path/to/data_system/exports python build/build_serving_store.py

WHAT IT DOES NOT DO
    It does not define the drill-down queries. Those live in queries.py (next
    step). This file only loads and indexes.
"""

from __future__ import annotations

import sqlite3
from pathlib import Path

import pandas as pd

from growlytics_platform.serving_system.utils import config


# --------------------------------------------------------------------------- #
# 1. Declarative table plan.
#    One entry per output table from the data system. Adding a table later is a
#    data change here, not new code below. `indexes` lists the column-tuples to
#    index — chosen from the keys the serving layer will FILTER or JOIN on.
# --------------------------------------------------------------------------- #
TABLE_SPECS: dict[str, dict] = {
    # input table — exposes a unique input_output_key
    "input": {
        "file": "input",
        "subdir": "input",
        "indexes": [("input_output_key",)],
    },
    # annualized output (pre-allocation) — exposes a unique output_orchestration_key
    "output": {
        "file": "output",
        "subdir": "output",
        "indexes": [("output_orchestration_key",)],
    },
    # link table: output_orchestration_key (many->1 to output) and input_output_key (many->1 to input)
    "input_output_linkage": {
        "file": "input_output_linkage",
        "subdir": "linkage",
        "indexes": [("output_orchestration_key",), ("input_output_key",)],
    },
    # NOTE: output_orchestration_linkage.csv also lives in linkage/ but is not
    # loaded yet — its keys/role aren't defined in FOREIGN_KEYS. Add a spec here
    # (and the matching FK edges below) once its schema is settled.
    #
    # the fact table: monthly allocated cash-flow streams (the grain)
    "cash_flow_streams": {
        "file": "cash_flow_streams",
        "subdir": "orchestration",
        "indexes": [
            ("orchestration_key", "month", "source"),  # the level 0 -> 1 drill path
            ("output_orchestration_key",),             # join out to linkage / lineage
            ("client_id",),                            # client control panel
            ("orchestration_key",),                    # revision control panel
        ],
    },
    # control panels
    # file is client.csv (singular); the DB table stays "clients" so FKs resolve.
    "clients": {
        "file": "clients",
        "subdir": "output",
        "indexes": [("client_id",)],
    },
    "versions": {
        "file": "versions",
        "subdir": "orchestration",
        "indexes": [("orchestration_key",)],
    },
}

# --------------------------------------------------------------------------- #
# 2. Relationships, as (child_table, child_col) -> (parent_table, parent_col).
#    Used only for a build-time integrity check: catching a broken key here is
#    far better than a silently empty drill-down at serve time.
# --------------------------------------------------------------------------- #
FOREIGN_KEYS: list[tuple[str, str, str, str]] = [
    ("input_output_linkage", "input_output_key", "input", "input_output_key"),
    ("input_output_linkage", "output_orchestration_key", "output", "output_orchestration_key"),
    ("cash_flow_streams", "output_orchestration_key", "input_output_linkage", "output_orchestration_key"),
    ("cash_flow_streams", "client_id", "clients", "client_id"),
    ("cash_flow_streams", "orchestration_key", "versions", "orchestration_key"),
]

READERS = {"csv": pd.read_csv, "parquet": pd.read_parquet}


# --------------------------------------------------------------------------- #
# Helpers
# --------------------------------------------------------------------------- #
def _source_path(spec: dict) -> Path:
    return config.INPUT_DIR / spec.get("subdir", "") /f"{spec['file']}.{config.INPUT_FORMAT}"

def _preflight() -> None:
    """Resolve every source path up front and fail before we open the store,
    so a missing/misplaced file never leaves a half-rebuilt database behind.
    (Invariant 2: a rebuild should be all-or-nothing.)"""
    if config.INPUT_FORMAT not in READERS:
        raise ValueError(
            f"Unsupported GROWLYTICS_INPUT_FORMAT={config.INPUT_FORMAT!r}; "
            f"expected one of {sorted(READERS)}"
        )
    missing = [str(_source_path(spec)) for spec in TABLE_SPECS.values()
               if not _source_path(spec).exists()]
    if missing:
        raise FileNotFoundError(
            "Expected export(s) not found:\n  " + "\n  ".join(missing)
            + "\n\nCheck GROWLYTICS_INPUT_DIR (should point at the 'consolidated' "
            "root) and each table's 'subdir' in TABLE_SPECS."
        )


def read_table(spec: dict) -> pd.DataFrame:
    """Read one export file into a DataFrame. Fail loudly if it is missing."""
    if config.INPUT_FORMAT not in READERS:
        raise ValueError(
            f"Unsupported GROWLYTICS_INPUT_FORMAT={config.INPUT_FORMAT!r}; "
            f"expected one of {sorted(READERS)}"
        )
    path = _source_path(spec)
    if not path.exists():
        raise FileNotFoundError(
            f"Expected export not found: {path}\n"
            f"Set GROWLYTICS_INPUT_DIR to the data system's export directory."
        )
    return READERS[config.INPUT_FORMAT](path)


def create_indexes(conn: sqlite3.Connection, table: str, spec: dict) -> None:
    """Build the indexes for one table. IF NOT EXISTS keeps re-runs clean."""
    for cols in spec["indexes"]:
        index_name = f"idx_{table}__{'_'.join(cols)}"
        col_list = ", ".join(f'"{c}"' for c in cols)
        conn.execute(
            f'CREATE INDEX IF NOT EXISTS "{index_name}" ON "{table}" ({col_list})'
        )


def _columns(conn: sqlite3.Connection, table: str) -> set[str]:
    return {row[1] for row in conn.execute(f'PRAGMA table_info("{table}")')}


# --------------------------------------------------------------------------- #
# The build
# --------------------------------------------------------------------------- #
def build() -> sqlite3.Connection:
    """Load every table from the data system into the store. Returns the open
    connection so callers (or the integrity check) can inspect the result."""

    _preflight()  # every file present and format supported before we write anything

    config.STORE_PATH.parent.mkdir(parents=True, exist_ok=True)
    conn = sqlite3.connect(config.STORE_PATH)

    for table, spec in TABLE_SPECS.items():
        df = read_table(spec)
        # if_exists="replace" makes the whole build idempotent (Invariant 2).
        df.to_sql(table, conn, if_exists="replace", index=False)
        create_indexes(conn, table, spec)
        print(f"  loaded {table:<22} {len(df):>6} rows   <- {_source_path(spec)}")

    conn.commit()
    return conn


def check_integrity(conn: sqlite3.Connection) -> list[str]:
    """Report child keys that have no matching parent (orphans). Returns a list
    of human-readable problems; empty means every relationship resolves."""
    problems: list[str] = []
    for child_t, child_c, parent_t, parent_c in FOREIGN_KEYS:
        edge = f"{child_t}.{child_c} -> {parent_t}.{parent_c}"
        if child_c not in _columns(conn, child_t) or parent_c not in _columns(conn, parent_t):
            problems.append(f"  ?  {edge:<55} skipped (column not present)")
            continue
        orphans = conn.execute(
            f'SELECT COUNT(*) FROM "{child_t}" c '
            f'WHERE c."{child_c}" IS NOT NULL '
            f'AND c."{child_c}" NOT IN (SELECT DISTINCT "{parent_c}" FROM "{parent_t}")'
        ).fetchone()[0]
        if orphans:
            problems.append(f"  X  {edge:<55} {orphans} orphan key(s)")
        else:
            print(f"  ok {edge:<55} all keys resolve")
    return problems


def summary(conn: sqlite3.Connection) -> None:
    """Print the tables and the indexes that were created on the store."""
    print("\nStore tables:")
    tables = [
        r[0]
        for r in conn.execute(
            "SELECT name FROM sqlite_master WHERE type='table' ORDER BY name"
        )
    ]
    for t in tables:
        count = conn.execute(f'SELECT COUNT(*) FROM "{t}"').fetchone()[0]
        idx = [
            r[0]
            for r in conn.execute(
                "SELECT name FROM sqlite_master WHERE type='index' "
                "AND tbl_name=? AND name NOT LIKE 'sqlite_%' ORDER BY name",
                (t,),
            )
        ]
        print(f"  {t:<22} {count:>6} rows   indexes: {', '.join(idx) or '(none)'}")


if __name__ == "__main__":
    print(f"Building store at: {config.STORE_PATH}")
    print(f"Reading exports from: {config.INPUT_DIR} (*.{config.INPUT_FORMAT})\n")

    conn = build()
    summary(conn)

    print("\nReferential integrity:")
    problems = check_integrity(conn)
    if problems:
        print("\n".join(problems))
        if config.STRICT_INTEGRITY:
            conn.close()
            raise SystemExit("Aborting: orphan keys found and STRICT_INTEGRITY is on.")
        print("\n(Warnings only. Set GROWLYTICS_STRICT_INTEGRITY=true to make these fatal.)")
    else:
        print("  every relationship resolves cleanly.")

    conn.close()
    print(f"\nDone. {config.STORE_PATH} is ready for queries.py.")

    ```