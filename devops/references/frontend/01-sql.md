# SQL Reference Sheet

> **Status:** first draft — living document.
> **Primary dialects tracked:** PostgreSQL and SQLite. Where they diverge, the divergence is called out inline. `[PG]` = PostgreSQL only, `[SQLite]` = SQLite only, `[std]` = ANSI standard supported by both.

---

## Table of contents

| Part | Topic |
|---|---|
| 0 | How to use / conventions |
| 1 | The core mental model: logical query processing |
| 2 | Data types and type systems |
| 3 | DDL — tables, constraints, schema evolution |
| 4 | DML — INSERT, UPDATE, DELETE, UPSERT |
| 5 | SELECT — the clause-by-clause reference |
| 6 | Expressions, operators, NULL semantics |
| 7 | Joins |
| 8 | Aggregation |
| 9 | Subqueries |
| 10 | CTEs and recursion |
| 11 | Window functions |
| 12 | Set operations |
| 13 | Scalar function reference (string, numeric, date, JSON) |
| 14 | Views and materialized views |
| 15 | Indexes |
| 16 | Transactions and concurrency |
| 17 | Query planning and EXPLAIN |
| 18 | Recipes — the patterns you actually reach for |
| 19 | Traps ledger |
| 20 | Dialect divergence table |

---

## Part 0 — How to use / conventions

**Conventions in this document**

- `UPPERCASE` = SQL keyword. `lowercase` = identifier you supply.
- `[optional]`, `{a | b}` = choose one, `...` = repeatable.
- Every part ends with an **Invariants** block: the things that are true regardless of dialect, and that you can reason from when you've forgotten the syntax.

**How to grow this document**

Each time you hit something that surprises you, it belongs in one of two places: the relevant Part (if it's syntax) or the **Traps ledger** in Part 19 (if it's a semantic surprise that cost you debugging time). The traps ledger is the highest-value section over time — syntax is searchable, semantics are not.

---

## Part 1 — The core mental model: logical query processing

This is the single most useful invariant in SQL. The order you *write* a query is not the order it is *evaluated*.

**Written order:**

```
SELECT → FROM → WHERE → GROUP BY → HAVING → ORDER BY → LIMIT
```

**Logical evaluation order:**

```
1. FROM          -- produce the source rowset
2. JOIN / ON     -- combine sources, apply join predicates
3. WHERE         -- filter rows (pre-aggregation)
4. GROUP BY      -- collapse rows into groups
5. HAVING        -- filter groups (post-aggregation)
6. SELECT        -- evaluate the output expressions, assign aliases
6b. WINDOW fns   -- evaluated here, after grouping, over the result set
7. DISTINCT      -- deduplicate output rows
8. ORDER BY      -- sort
9. LIMIT/OFFSET  -- truncate
```

Almost every "why doesn't this work" question in SQL falls out of this list:

| Symptom | Explanation |
|---|---|
| `WHERE` can't see a `SELECT` alias | Step 3 runs before step 6. |
| `ORDER BY` *can* see a `SELECT` alias | Step 8 runs after step 6. |
| `WHERE` can't contain an aggregate | Aggregates don't exist until step 4. Use `HAVING`. |
| `WHERE` can't contain a window function | Windows are computed at 6b. Wrap in a CTE/subquery and filter outside. |
| `GROUP BY` can't see a `SELECT` alias `[std]` | Step 4 before step 6. (PostgreSQL permits it as an extension; SQLite permits it too. Don't rely on it if portability matters.) |
| `LIMIT` without `ORDER BY` returns different rows each run | Step 9 truncates an unordered set. Row order is never guaranteed without `ORDER BY`. |

**A second model: the grain of a rowset.**

Every rowset has a *grain* — the set of columns that uniquely identify one row. Reasoning about grain prevents most correctness bugs:

- `JOIN` can multiply grain (fan-out) if the join key isn't unique on the right side.
- `GROUP BY` sets grain explicitly to the grouping columns.
- `DISTINCT` is usually a symptom that you lost track of grain upstream, not a fix.
- **Never sum a money column after a join that changed the grain.** Aggregate first, join second.

### Invariants

1. A SQL query is a declaration of *what*, not *how*. The planner chooses the physical order; only the logical order above is guaranteed semantically.
2. Rows in a table have no inherent order. Any order you observe without `ORDER BY` is an artifact of the plan and can change.
3. Every clause consumes a rowset and produces a rowset. If you can name the grain going in and coming out of each clause, the query is correct.

---

## Part 2 — Data types and type systems

### PostgreSQL — static, strict

| Category | Types | Notes |
|---|---|---|
| Integer | `SMALLINT` (2B), `INTEGER`/`INT` (4B), `BIGINT` (8B) | |
| Serial | `SMALLSERIAL`, `SERIAL`, `BIGSERIAL` | Legacy sugar for a sequence + default. Prefer `GENERATED ALWAYS AS IDENTITY`. |
| Exact numeric | `NUMERIC(p, s)` / `DECIMAL(p, s)` | **Use for money.** Arbitrary precision, no binary rounding error. |
| Float | `REAL` (4B), `DOUBLE PRECISION` (8B) | Never for money. |
| Text | `TEXT`, `VARCHAR(n)`, `CHAR(n)` | `TEXT` and `VARCHAR` are the same speed in PG. `CHAR(n)` blank-pads — avoid. |
| Boolean | `BOOLEAN` | `TRUE`/`FALSE`/`NULL`. |
| Date/time | `DATE`, `TIME`, `TIMESTAMP`, `TIMESTAMPTZ`, `INTERVAL` | **Default to `TIMESTAMPTZ`.** `TIMESTAMP` has no zone and will silently lie to you. |
| Binary | `BYTEA` | |
| JSON | `JSON`, `JSONB` | `JSONB` is parsed, indexable, reorders keys. Default to `JSONB`. |
| UUID | `UUID` | |
| Array | `type[]` | e.g. `INTEGER[]`, `TEXT[]`. 1-indexed. |
| Range | `INT4RANGE`, `DATERANGE`, `TSTZRANGE`, … | Pairs with exclusion constraints for no-overlap rules. |
| Enum | `CREATE TYPE mood AS ENUM ('a','b')` | Adding values is easy; removing is not. |

### SQLite — dynamic, affinity-based

SQLite stores one of five **storage classes**: `NULL`, `INTEGER`, `REAL`, `TEXT`, `BLOB`. A column has an **affinity**, which is a *preference*, not a constraint.

| Declared type contains | Affinity |
|---|---|
| `INT` | INTEGER |
| `CHAR`, `CLOB`, `TEXT` | TEXT |
| `BLOB` or no type | BLOB (none) |
| `REAL`, `FLOA`, `DOUB` | REAL |
| anything else | NUMERIC |

Consequences that bite:

- There is **no `BOOLEAN`**. Use `INTEGER` `0`/`1`. `TRUE`/`FALSE` keywords are accepted and mean `1`/`0`.
- There is **no date/time type**. Store ISO-8601 `TEXT` (`'2026-08-29'`, `'2026-08-29T14:03:00Z'`), Unix epoch `INTEGER`, or Julian day `REAL`. ISO-8601 text sorts and compares correctly lexicographically — this is why it's the default choice.
- There is **no `DECIMAL`**. `NUMERIC` affinity falls back to `REAL` for non-integers, so money in SQLite is safest as **integer minor units** (cents) or as `TEXT` with application-side decimal handling.
- `VARCHAR(10)` does **not** enforce length. Nothing does, unless you add a `CHECK`.

**`STRICT` tables** (SQLite ≥ 3.37) opt into type enforcement:

```sql
CREATE TABLE t (
  id     INTEGER PRIMARY KEY,
  amount INTEGER NOT NULL,
  label  TEXT    NOT NULL
) STRICT;
```

In a `STRICT` table only `INT`, `INTEGER`, `REAL`, `TEXT`, `BLOB`, `ANY` are permitted as declared types, and inserts that can't be losslessly converted are rejected. Worth defaulting to for any new schema.

### Casting

```sql
CAST(expr AS type)          -- [std], portable
expr::type                  -- [PG] shorthand
```

`CAST` in SQLite performs affinity conversion and will silently produce `0` from `CAST('abc' AS INTEGER)`. PostgreSQL raises an error. This asymmetry is a common source of "the numbers are wrong but nothing failed" in cross-dialect pipelines.

### Invariants

1. Money is never a binary float. `NUMERIC` in PostgreSQL; integer minor units in SQLite.
2. Timestamps are stored with a zone or stored as UTC by convention that is documented and enforced. "Local time, no zone" is a bug waiting for a DST boundary.
3. In SQLite, the declared type is documentation unless the table is `STRICT`. Enforcement lives in `CHECK` constraints or in the table option.

---

## Part 3 — DDL: tables, constraints, schema evolution

### CREATE TABLE

```sql
CREATE TABLE [IF NOT EXISTS] schema.table_name (
    column_name  data_type  [column_constraints...],
    ...
    [table_constraints...]
);
```

Full-shape example:

```sql
CREATE TABLE crop_input_streams (
    stream_id      INTEGER PRIMARY KEY,
    revision_id    INTEGER NOT NULL,
    crop_code      TEXT    NOT NULL,
    category       TEXT    NOT NULL,
    month_num      INTEGER NOT NULL,
    amount_cents   INTEGER NOT NULL DEFAULT 0,
    created_at     TEXT    NOT NULL DEFAULT (datetime('now')),

    CONSTRAINT fk_revision
        FOREIGN KEY (revision_id) REFERENCES revisions(revision_id)
        ON DELETE CASCADE
        ON UPDATE RESTRICT,
    CONSTRAINT ck_month
        CHECK (month_num BETWEEN 1 AND 12),
    CONSTRAINT uq_stream_grain
        UNIQUE (revision_id, crop_code, category, month_num)
) STRICT;
```

### Column constraints

| Constraint | Meaning |
|---|---|
| `NOT NULL` | Rejects NULL. |
| `DEFAULT expr` | Applied when the column is omitted from `INSERT`. Not applied on explicit `NULL`. |
| `PRIMARY KEY` | Unique + not null (see SQLite caveat below). One per table. |
| `UNIQUE` | Unique across non-NULL values. **Multiple NULLs are allowed** — NULLs aren't equal to each other. |
| `CHECK (expr)` | Row-level predicate. Rejects when the expression is `FALSE`; **passes when `UNKNOWN`/NULL**. |
| `REFERENCES t(col)` | Inline foreign key. |
| `GENERATED ALWAYS AS (expr) [STORED\|VIRTUAL]` | Computed column. `VIRTUAL` is SQLite default; PG supports `STORED` only. |
| `COLLATE name` | Comparison/sort rule. `NOCASE`/`BINARY`/`RTRIM` in SQLite; ICU/libc collations in PG. |

**SQLite `PRIMARY KEY` caveat:** in a non-`STRICT`, non-`WITHOUT ROWID` table, `PRIMARY KEY` columns other than `INTEGER PRIMARY KEY` **do not imply `NOT NULL`** — a long-standing bug preserved for compatibility. Always write `NOT NULL` explicitly.

**`INTEGER PRIMARY KEY` in SQLite** is an alias for the internal `rowid` — this is the only true clustered key and the fastest lookup path. `INT PRIMARY KEY` (note: not `INTEGER`) is *not* the same thing.

### Foreign key referential actions

```sql
ON DELETE { NO ACTION | RESTRICT | CASCADE | SET NULL | SET DEFAULT }
ON UPDATE { ... same ... }
```

- `NO ACTION` (default) — check deferred to end of statement.
- `RESTRICT` — check immediately, cannot be deferred.
- `CASCADE` — propagate. Powerful and dangerous; audit every cascade path before enabling.

**SQLite enforces foreign keys only if you ask:**

```sql
PRAGMA foreign_keys = ON;   -- per connection, resets every connection
```

This is per-connection, not per-database. Any connection pool, ORM, or script that doesn't set it silently skips FK enforcement. Set it in the connection factory, once, and add an integrity check to CI:

```sql
PRAGMA foreign_key_check;   -- returns rows for every violation
PRAGMA integrity_check;
```

**FK direction rule:** point at the owner — the table where the key is born and is unique — never at a borrower.

### ALTER TABLE

```sql
-- PostgreSQL: broad support
ALTER TABLE t ADD COLUMN c TEXT NOT NULL DEFAULT 'x';
ALTER TABLE t DROP COLUMN c;
ALTER TABLE t RENAME COLUMN a TO b;
ALTER TABLE t RENAME TO t2;
ALTER TABLE t ALTER COLUMN c TYPE NUMERIC(12,2) USING c::numeric;
ALTER TABLE t ALTER COLUMN c SET NOT NULL;
ALTER TABLE t ALTER COLUMN c SET DEFAULT 0;
ALTER TABLE t ADD CONSTRAINT ck CHECK (c >= 0);
ALTER TABLE t DROP CONSTRAINT ck;
```

```sql
-- SQLite: only these four
ALTER TABLE t RENAME TO t2;
ALTER TABLE t RENAME COLUMN a TO b;
ALTER TABLE t ADD COLUMN c TEXT;         -- restrictions apply, see below
ALTER TABLE t DROP COLUMN c;             -- ≥ 3.35
```

SQLite `ADD COLUMN` restrictions: cannot be `PRIMARY KEY` or `UNIQUE`; if `NOT NULL`, must have a non-null default; a `DEFAULT` that isn't a constant is rejected; can't add a `STORED` generated column.

**The SQLite 12-step table rebuild** (for anything else — type changes, adding constraints, reordering):

```sql
PRAGMA foreign_keys = OFF;
BEGIN;
  CREATE TABLE t_new ( ...new definition... );
  INSERT INTO t_new (cols...) SELECT cols... FROM t;
  DROP TABLE t;
  ALTER TABLE t_new RENAME TO t;
  -- recreate indexes, triggers, views
COMMIT;
PRAGMA foreign_key_check;
PRAGMA foreign_keys = ON;
```

### DROP / TRUNCATE

```sql
DROP TABLE [IF EXISTS] t [CASCADE | RESTRICT];   -- CASCADE is [PG]
TRUNCATE TABLE t [RESTART IDENTITY] [CASCADE];   -- [PG] only; DDL, fast, not row-by-row
DELETE FROM t;                                   -- portable equivalent, logged per row
```

### Invariants

1. Constraints are the cheapest tests you will ever write. Anything the database can refuse to store is a class of bug that can't reach your application.
2. A `CHECK` constraint passes on NULL. If NULL is not acceptable, `NOT NULL` must be stated separately.
3. `UNIQUE` does not mean "one row per key" when the key is nullable. Use a partial unique index or `NOT NULL`.
4. In SQLite, referential integrity is opt-in per connection. Treat `PRAGMA foreign_keys = ON` as part of the schema, not part of the session.

---

## Part 4 — DML: INSERT, UPDATE, DELETE, UPSERT

### INSERT

```sql
INSERT INTO t (col1, col2)
VALUES (v1, v2),
       (v3, v4);                      -- multi-row: one statement, one round trip

INSERT INTO t (col1, col2)
SELECT a, b FROM source WHERE ...;    -- insert-select: the ETL workhorse

INSERT INTO t DEFAULT VALUES;
```

Always name the target columns. Positional inserts break silently the day someone adds a column.

### RETURNING

```sql
INSERT INTO t (a) VALUES (1) RETURNING id, created_at;
UPDATE t SET a = 2 WHERE id = 5 RETURNING *;
DELETE FROM t WHERE id = 5 RETURNING id;
```

`[PG]` since forever, `[SQLite]` since 3.35. Removes the insert-then-select round trip and is the correct way to capture generated keys.

### UPDATE

```sql
UPDATE t
SET col1 = expr1,
    col2 = expr2
WHERE predicate;

-- update from another table
-- [PG]
UPDATE forecasts f
SET amount = s.amount
FROM staging s
WHERE f.id = s.id;

-- [SQLite] ≥ 3.33 supports UPDATE ... FROM with the same shape;
-- portable fallback is a correlated subquery:
UPDATE forecasts
SET amount = (SELECT s.amount FROM staging s WHERE s.id = forecasts.id)
WHERE EXISTS (SELECT 1 FROM staging s WHERE s.id = forecasts.id);
```

The `WHERE EXISTS` guard is not optional in the fallback form. Without it, rows with no match get set to `NULL`.

### DELETE

```sql
DELETE FROM t WHERE predicate;

-- delete with reference to another table [PG]
DELETE FROM t USING other o WHERE t.id = o.id;

-- portable
DELETE FROM t WHERE id IN (SELECT id FROM other);
```

### UPSERT — `ON CONFLICT`

```sql
INSERT INTO t (id, a, b)
VALUES (1, 'x', 2)
ON CONFLICT (id) DO UPDATE
    SET a = excluded.a,
        b = excluded.b,
        updated_at = CURRENT_TIMESTAMP
WHERE t.b IS DISTINCT FROM excluded.b;   -- optional: skip no-op writes

INSERT INTO t (id, a) VALUES (1, 'x')
ON CONFLICT DO NOTHING;
```

- `excluded` is the pseudo-table holding the row that failed to insert.
- The conflict target must match an existing unique index or constraint — a plain `UNIQUE` on the same columns, or `ON CONFLICT ON CONSTRAINT name` `[PG]`.
- Supported in `[PG]` ≥ 9.5 and `[SQLite]` ≥ 3.24 with near-identical syntax. This is the single best-aligned advanced feature across the two.

### MERGE `[PG ≥ 15]`

```sql
MERGE INTO target t
USING source s ON t.id = s.id
WHEN MATCHED AND s.deleted THEN DELETE
WHEN MATCHED THEN UPDATE SET amount = s.amount
WHEN NOT MATCHED THEN INSERT (id, amount) VALUES (s.id, s.amount);
```

More expressive than `ON CONFLICT` (it can delete, and it matches on an arbitrary join condition rather than a unique index), but not available in SQLite. For portable pipelines, prefer `ON CONFLICT`.

### Invariants

1. An `UPDATE` or `DELETE` without a `WHERE` clause affects every row. Write the `WHERE` before the `SET`.
2. Before running a destructive DML statement by hand, run it as a `SELECT COUNT(*)` with the same `WHERE`. The number should be the number you expect.
3. Upserts require a unique constraint to conflict against. If there is no unique constraint, there is no upsert — there is a race condition.

---

## Part 5 — SELECT, clause by clause

```sql
[WITH [RECURSIVE] cte_name AS (...) [, ...]]
SELECT [ALL | DISTINCT | DISTINCT ON (expr) /* [PG] */]
    select_expr [AS alias], ...
FROM table_expr
    [join_clause ...]
[WHERE predicate]
[GROUP BY expr, ... [WITH ROLLUP /* dialect-specific */]]
[HAVING predicate]
[WINDOW w AS (window_definition)]
[{UNION | INTERSECT | EXCEPT} [ALL] select_stmt]
[ORDER BY expr [ASC|DESC] [NULLS {FIRST|LAST}], ...]
[LIMIT n] [OFFSET m]
```

### SELECT list

```sql
SELECT *                     -- never in production code or views
SELECT t.*                   -- acceptable in ad-hoc exploration
SELECT col AS alias          -- AS is optional but write it anyway
SELECT DISTINCT a, b         -- dedup across the whole output row
SELECT DISTINCT ON (a) a, b  -- [PG] first row per a, per ORDER BY
    FROM t ORDER BY a, b DESC;
```

`SELECT *` in a view or a materialized artifact freezes an implicit contract that changes whenever the table changes. Enumerate columns anywhere the output is consumed by other code.

### FROM

```sql
FROM t
FROM t AS alias                 -- alias shadows the name: t is no longer usable
FROM (SELECT ...) AS sub        -- derived table; alias is REQUIRED in PG
FROM t1, t2                     -- implicit cross join; avoid, use explicit JOIN
FROM generate_series(1, 12) AS m(month_num)   -- [PG] set-returning function
```

### WHERE

Filters rows before grouping. Predicates here are **sargable** or not — a predicate that wraps the indexed column in a function usually cannot use the index:

```sql
WHERE date_trunc('month', ts) = '2026-08-01'    -- not sargable on an index on ts
WHERE ts >= '2026-08-01' AND ts < '2026-09-01'  -- sargable, range scan
```

(An expression index on `date_trunc('month', ts)` makes the first form sargable too — but the range form is portable and needs no extra index.)

### ORDER BY

```sql
ORDER BY col ASC
ORDER BY col DESC NULLS LAST     -- NULLS clause: [PG] always, [SQLite] ≥ 3.30
ORDER BY 2, 1                    -- by output position; brittle, avoid
ORDER BY alias                   -- legal, aliases are visible here
ORDER BY CASE WHEN ... END       -- custom ordering
```

Default NULL placement differs: PostgreSQL treats NULLs as **largest** (so `ASC` → last, `DESC` → first); SQLite treats them as **smallest** (`ASC` → first). Always write `NULLS FIRST`/`NULLS LAST` explicitly when NULLs are possible.

### LIMIT / OFFSET

```sql
LIMIT 20 OFFSET 40                      -- [PG], [SQLite]
OFFSET 40 ROWS FETCH NEXT 20 ROWS ONLY  -- [std], PG supports it
```

`OFFSET` is O(offset) — the database still produces and discards the skipped rows. For deep pagination use **keyset pagination** instead (Part 18).

### Invariants

1. `LIMIT` without `ORDER BY` is nondeterministic. Always pair them.
2. An `ORDER BY` that isn't a total order is also nondeterministic. Add a tiebreaker column (usually the primary key) to any pagination sort.
3. Aliases are visible in `ORDER BY`, not in `WHERE`/`GROUP BY`/`HAVING` (per the standard). Repeat the expression or wrap the query in a CTE.

---

## Part 6 — Expressions, operators, NULL semantics

### Comparison and logic

```sql
=  <>  !=  <  <=  >  >=
BETWEEN a AND b              -- inclusive on both ends
IN (a, b, c) / IN (SELECT …)
LIKE 'pat%'                  -- % = any run, _ = single char
ILIKE 'pat%'                 -- [PG] case-insensitive
NOT LIKE / SIMILAR TO / ~ '<regex>'  -- [PG] regex operators
GLOB 'pat*'                  -- [SQLite] case-sensitive glob syntax
IS NULL / IS NOT NULL
IS DISTINCT FROM             -- NULL-safe inequality
IS NOT DISTINCT FROM         -- NULL-safe equality
```

`IS DISTINCT FROM` is available in `[PG]` and in `[SQLite]` ≥ 3.39. It is the correct comparison for change detection, since `a <> b` is `UNKNOWN` when either side is NULL.

### Three-valued logic — the central NULL model

NULL means *unknown*, not *empty* and not *zero*. Any comparison involving NULL yields `UNKNOWN`, and `WHERE` keeps only rows where the predicate is `TRUE`.

| Expression | Result |
|---|---|
| `NULL = NULL` | `UNKNOWN` (not TRUE) |
| `NULL <> 1` | `UNKNOWN` |
| `TRUE OR NULL` | `TRUE` |
| `FALSE OR NULL` | `UNKNOWN` |
| `TRUE AND NULL` | `UNKNOWN` |
| `FALSE AND NULL` | `FALSE` |
| `NOT NULL` (the operator) | `UNKNOWN` |

Where NULLs *are* treated as equal, by special-case rules: `GROUP BY`, `DISTINCT`, `UNION`, `ORDER BY`, and `IS NOT DISTINCT FROM`. Everywhere else they are not.

### The `NOT IN` trap

```sql
-- If the subquery returns ANY NULL, this returns ZERO rows. Always.
SELECT * FROM a WHERE id NOT IN (SELECT fk FROM b);
```

`x NOT IN (1, 2, NULL)` expands to `x<>1 AND x<>2 AND x<>NULL` → `TRUE AND TRUE AND UNKNOWN` → `UNKNOWN` → row dropped. Use `NOT EXISTS` instead, which is NULL-safe and usually plans better:

```sql
SELECT * FROM a WHERE NOT EXISTS (SELECT 1 FROM b WHERE b.fk = a.id);
```

### NULL-handling functions

```sql
COALESCE(a, b, c)        -- first non-NULL argument
NULLIF(a, b)             -- NULL if a = b, else a  (useful for divide-by-zero guards)
IFNULL(a, b)             -- [SQLite] two-arg COALESCE
a / NULLIF(b, 0)         -- returns NULL instead of raising on zero divisor
```

### CASE

```sql
-- searched form (preferred: handles NULL correctly)
CASE WHEN cond1 THEN r1
     WHEN cond2 THEN r2
     ELSE r3
END

-- simple form (equality comparison; will never match NULL)
CASE expr WHEN v1 THEN r1 ELSE r2 END
```

Without `ELSE`, unmatched rows yield `NULL`. All branches must resolve to a compatible type.

### Invariants

1. NULL is not a value; it is the absence of information. Nothing is equal to it, including itself.
2. `WHERE` keeps only `TRUE`. `CHECK` rejects only `FALSE`. That asymmetry is the reason a `CHECK` passes on NULL.
3. Any predicate you write on a nullable column needs an explicit answer to "what should happen when this is NULL?"
4. `NOT IN (subquery)` is a bug unless the subquery column is `NOT NULL`. Default to `NOT EXISTS`.

---

## Part 7 — Joins

```sql
FROM a [INNER] JOIN b ON a.k = b.k
FROM a LEFT  [OUTER] JOIN b ON …     -- all of a, matched or NULL-extended
FROM a RIGHT [OUTER] JOIN b ON …     -- [SQLite] ≥ 3.39
FROM a FULL  [OUTER] JOIN b ON …     -- [SQLite] ≥ 3.39
FROM a CROSS JOIN b                  -- cartesian product
FROM a NATURAL JOIN b                -- joins on all same-named columns; avoid
FROM a JOIN b USING (k)              -- ON a.k = b.k, and k appears once in output
```

### `ON` vs `WHERE` for outer joins

This distinction is the most common outer-join bug.

```sql
-- keeps all rows of a; b-side condition applied during the join
FROM a LEFT JOIN b ON a.k = b.k AND b.status = 'active'

-- silently converts the LEFT JOIN into an INNER JOIN:
-- NULL-extended rows fail the WHERE test and are dropped
FROM a LEFT JOIN b ON a.k = b.k
WHERE b.status = 'active'
```

For inner joins, `ON` and `WHERE` are semantically identical and the planner treats them the same. For outer joins they are not interchangeable.

### Semi- and anti-joins

There is no `SEMI JOIN` keyword. You express them with `EXISTS`:

```sql
-- semi-join: rows of a that have at least one match, no fan-out
SELECT a.* FROM a WHERE EXISTS (SELECT 1 FROM b WHERE b.k = a.k);

-- anti-join: rows of a with no match
SELECT a.* FROM a WHERE NOT EXISTS (SELECT 1 FROM b WHERE b.k = a.k);
```

The value of `EXISTS` over a join is that it **cannot change the grain of `a`**, no matter how many matching rows exist in `b`.

### LATERAL `[PG]`

A subquery in `FROM` that may reference columns from tables to its left — a correlated subquery that can return multiple rows and columns.

```sql
SELECT c.crop_code, top.month_num, top.amount
FROM crops c
CROSS JOIN LATERAL (
    SELECT month_num, amount
    FROM crop_input_streams s
    WHERE s.crop_code = c.crop_code
    ORDER BY amount DESC
    LIMIT 3
) AS top;
```

Use `LEFT JOIN LATERAL (...) ON TRUE` to keep left rows with no matches. Not available in SQLite; the portable equivalent is a window function with `ROW_NUMBER() <= 3` filtered in an outer query.

### Fan-out and the grain rule

```sql
-- WRONG: if input_output_linkage has multiple rows per stream,
-- every crop_input_streams.amount is counted once per linkage row
SELECT SUM(s.amount)
FROM crop_input_streams s
JOIN input_output_linkage l ON l.stream_id = s.stream_id;

-- RIGHT: aggregate to the target grain first, then join
WITH stream_totals AS (
    SELECT stream_id, SUM(amount) AS amount
    FROM crop_input_streams
    GROUP BY stream_id
)
SELECT ...
FROM stream_totals st
JOIN ...
```

The general diagnostic: after any join, `COUNT(*)` should equal the count of the driving table if the join was meant to be 1:1 or 1:0..1. If it doesn't, the grain changed and every downstream `SUM` is inflated.

### Join ordering and equivalence classes

The planner is free to reorder inner joins. When it sees `a.k = b.k AND b.k = c.k` it builds an **equivalence class** `{a.k, b.k, c.k}` and may infer the transitive predicate `a.k = c.k`, opening up join orders you never wrote. This is generally a gift.

The caveat is that equivalence requires the comparisons to genuinely be transitive. In SQLite, if the columns have different **affinities** or different **collating sequences**, `a.k = b.k` and `b.k = c.k` do not imply `a.k = c.k` — text `'1'` and integer `1` compare differently depending on which side carries which affinity. Keep join-key types and collations identical across tables and the problem disappears.

### Invariants

1. A join's effect on grain is determined entirely by the uniqueness of the join key on each side. Know which side is unique before you write it.
2. A filter on the null-extended side of an outer join belongs in `ON`, not `WHERE`.
3. If you only need existence, use `EXISTS` — it is the only join form that cannot fan out.
4. `NATURAL JOIN` and `USING` bind to column names, so adding a column can change the join. Prefer explicit `ON`.

---

## Part 8 — Aggregation

### Aggregate functions

```sql
COUNT(*)                     -- rows, including all-NULL rows
COUNT(col)                   -- non-NULL values of col
COUNT(DISTINCT col)
SUM(col) / AVG(col)          -- ignore NULLs; SUM of zero rows is NULL, not 0
MIN(col) / MAX(col)
STRING_AGG(col, ',' ORDER BY col)   -- [PG]
GROUP_CONCAT(col, ',')              -- [SQLite]
ARRAY_AGG(col ORDER BY col)         -- [PG]
JSON_AGG(col) / JSONB_AGG(col)      -- [PG]
BOOL_AND(col) / BOOL_OR(col)        -- [PG]
STDDEV / VARIANCE / PERCENTILE_CONT -- [PG]
```

`SUM` over an empty group returns `NULL`. Wrap in `COALESCE(SUM(x), 0)` anywhere the result feeds arithmetic.

### GROUP BY / HAVING

```sql
SELECT crop_code, month_num, SUM(amount) AS total
FROM crop_input_streams
GROUP BY crop_code, month_num
HAVING SUM(amount) > 0
ORDER BY crop_code, month_num;
```

Rule: every non-aggregated `SELECT` expression must appear in `GROUP BY`. PostgreSQL relaxes this when you group by a primary key (functional dependency); SQLite silently picks an arbitrary row's value for un-grouped columns, which is a correctness hazard — don't rely on it.

`GROUP BY` collapses NULLs into a single group.

### FILTER — conditional aggregation

```sql
SELECT
    crop_code,
    SUM(amount) FILTER (WHERE category = 'inflow')  AS inflows,
    SUM(amount) FILTER (WHERE category = 'outflow') AS outflows,
    COUNT(*)    FILTER (WHERE amount < 0)           AS negative_rows
FROM crop_input_streams
GROUP BY crop_code;
```

`[PG]` and `[SQLite]` ≥ 3.30. The portable pre-`FILTER` idiom is `SUM(CASE WHEN cond THEN amount ELSE 0 END)` — note that `COUNT(CASE WHEN cond THEN 1 END)` needs no `ELSE`, since `COUNT` skips NULLs.

### GROUPING SETS / ROLLUP / CUBE `[PG]`

```sql
SELECT crop_code, month_num, SUM(amount)
FROM crop_input_streams
GROUP BY GROUPING SETS ((crop_code, month_num), (crop_code), ());

GROUP BY ROLLUP (crop_code, month_num)   -- (a,b), (a), ()  — hierarchical subtotals
GROUP BY CUBE (crop_code, month_num)     -- every combination
```

`GROUPING(col)` returns 1 when the column was aggregated away in that row — use it to distinguish a subtotal row from a real NULL. Not available in SQLite; emulate with `UNION ALL` of separate aggregate queries.

### Invariants

1. `COUNT(*)` counts rows; `COUNT(col)` counts non-NULLs. The gap between them is your NULL count.
2. Aggregates ignore NULLs — which means `AVG` divides by the count of non-NULLs, not the count of rows. If NULL should mean zero, `COALESCE` before aggregating, not after.
3. `WHERE` filters rows, `HAVING` filters groups. If a predicate doesn't reference an aggregate, it belongs in `WHERE`, where it reduces work.
4. Aggregate to the target grain *before* joining, whenever the join could fan out.

---

## Part 9 — Subqueries

| Form | Position | Returns | Notes |
|---|---|---|---|
| Scalar subquery | anywhere an expression goes | exactly 1 row × 1 col | errors (PG) or returns first row (SQLite) if more |
| Derived table | `FROM` | rowset | alias required in PG |
| `IN` / `NOT IN` | `WHERE` | 1 column | NULL hazard on `NOT IN` |
| `EXISTS` / `NOT EXISTS` | `WHERE` | boolean | NULL-safe, no fan-out |
| `ANY` / `ALL` | `WHERE` | 1 column | `= ANY` ≡ `IN`; `<> ALL` ≡ `NOT IN` |
| Row constructor | `WHERE` | tuple | `(a,b) IN (SELECT x,y FROM t)` |

```sql
-- scalar
SELECT crop_code,
       amount,
       amount * 1.0 / (SELECT SUM(amount) FROM crop_input_streams) AS share
FROM crop_input_streams;

-- correlated: re-evaluated per outer row (conceptually)
SELECT c.crop_code
FROM crops c
WHERE (SELECT COUNT(*) FROM crop_input_streams s WHERE s.crop_code = c.crop_code) > 5;

-- ANY / ALL
WHERE amount > ALL (SELECT threshold FROM limits)
WHERE amount = ANY (SELECT allowed FROM limits)
```

Correlated subqueries in the `SELECT` list are often better written as a `LEFT JOIN` to a pre-aggregated CTE, especially when there are several of them scanning the same table — the CTE scans once, the subqueries scan once each.

---

## Part 10 — CTEs and recursion

```sql
WITH monthly AS (
    SELECT crop_code, month_num, SUM(amount) AS total
    FROM crop_input_streams
    GROUP BY crop_code, month_num
),
annual AS (
    SELECT crop_code, SUM(total) AS annual_total
    FROM monthly
    GROUP BY crop_code
)
SELECT m.*, a.annual_total,
       m.total * 1.0 / NULLIF(a.annual_total, 0) AS share_of_year
FROM monthly m
JOIN annual a USING (crop_code);
```

CTEs are the main tool for making a complex query readable: each one names an intermediate rowset with a known grain.

**Materialization:** PostgreSQL ≥ 12 inlines CTEs by default when they're referenced once and side-effect free; before 12 they were always an optimization fence. Control it explicitly:

```sql
WITH x AS MATERIALIZED     (SELECT ...)   -- force a temp result
WITH x AS NOT MATERIALIZED (SELECT ...)   -- force inlining
```

`[SQLite]` supports both hints since 3.35.

### Recursive CTEs

```sql
WITH RECURSIVE months(n) AS (
    SELECT 1                        -- anchor / base case
    UNION ALL
    SELECT n + 1 FROM months        -- recursive term, references itself
    WHERE n < 12                    -- termination condition
)
SELECT n FROM months;
```

Hierarchy traversal with cycle safety:

```sql
WITH RECURSIVE tree AS (
    SELECT id, parent_id, name, 1 AS depth, CAST(id AS TEXT) AS path
    FROM nodes WHERE parent_id IS NULL
    UNION ALL
    SELECT n.id, n.parent_id, n.name, t.depth + 1, t.path || '/' || n.id
    FROM nodes n
    JOIN tree t ON n.parent_id = t.id
    WHERE t.depth < 100                     -- hard depth guard
      AND t.path NOT LIKE '%/' || n.id || '/%'   -- crude cycle guard
)
SELECT * FROM tree ORDER BY path;
```

`[PG]` also offers `CYCLE id SET is_cycle USING path` (≥ 14) as a first-class cycle detector.

- `UNION ALL` = keep duplicates, faster; `UNION` = dedup each iteration, terminates on cyclic graphs but hides multiplicity.
- The `RECURSIVE` keyword goes once after `WITH`, even with multiple CTEs.

### Data-modifying CTEs `[PG]`

```sql
WITH archived AS (
    DELETE FROM staging WHERE processed
    RETURNING *
)
INSERT INTO archive SELECT * FROM archived;
```

All statements see the same snapshot; ordering between them is not guaranteed beyond the data flow. Not supported in SQLite.

### Invariants

1. A CTE is a named rowset with a grain. Write the grain in a comment above each one and most bugs become visible before you run it.
2. Recursion needs an anchor, a recursive term, and a termination guarantee. If you can't state why it terminates, add a depth counter.

---

## Part 11 — Window functions

The tool for computing a value **per row** that depends on **other rows**, without collapsing the result set.

### Syntax

```sql
function(args) OVER (
    [PARTITION BY expr, ...]
    [ORDER BY expr [ASC|DESC] [NULLS FIRST|LAST], ...]
    [frame_clause]
)
```

Named windows avoid repetition:

```sql
SELECT
    month_num,
    SUM(amount)      OVER w AS running_total,
    ROW_NUMBER()     OVER w AS rn
FROM crop_input_streams
WINDOW w AS (PARTITION BY crop_code ORDER BY month_num);
```

### The three families

**Ranking**

| Function | Behaviour on ties |
|---|---|
| `ROW_NUMBER()` | arbitrary distinct numbers — add a tiebreaker to make it deterministic |
| `RANK()` | same rank, gaps after (1,1,3) |
| `DENSE_RANK()` | same rank, no gaps (1,1,2) |
| `NTILE(n)` | split into n buckets |
| `PERCENT_RANK()`, `CUME_DIST()` | relative position, 0–1 |

**Offset / navigation**

```sql
LAG(col, offset, default)  OVER (...)   -- previous row
LEAD(col, offset, default) OVER (...)   -- next row
FIRST_VALUE(col)  OVER (...)
LAST_VALUE(col)   OVER (...)            -- see frame trap below
NTH_VALUE(col, n) OVER (...)
```

**Aggregates as windows** — any aggregate (`SUM`, `AVG`, `COUNT`, `MIN`, `MAX`, `STRING_AGG`, …) becomes a window function with `OVER`.

### Frames

```sql
{ROWS | RANGE | GROUPS} BETWEEN frame_start AND frame_end
-- bounds:
UNBOUNDED PRECEDING | n PRECEDING | CURRENT ROW | n FOLLOWING | UNBOUNDED FOLLOWING
```

**Default frames** — the most consequential detail in the whole feature:

- With `ORDER BY`: `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`
- Without `ORDER BY`: `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` (the whole partition)

`RANGE` operates on *values*, so `CURRENT ROW` includes all **peer rows** with the same `ORDER BY` value. `ROWS` operates on *positions*. This is why:

```sql
-- BUG: with the default RANGE frame, this returns the current row's value,
-- because the frame ends at the last peer of the current row, not the partition end
LAST_VALUE(amount) OVER (PARTITION BY crop_code ORDER BY month_num)

-- FIX: state the frame
LAST_VALUE(amount) OVER (
    PARTITION BY crop_code ORDER BY month_num
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
```

And why a running total over a column with duplicate sort keys jumps in steps under `RANGE` but increments per row under `ROWS`.

Common frames:

```sql
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW   -- running total
ROWS BETWEEN 2 PRECEDING AND CURRENT ROW           -- 3-period trailing average
ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING           -- centred window
RANGE BETWEEN INTERVAL '7 days' PRECEDING
          AND CURRENT ROW                          -- [PG] time-based window
```

### Placement rules

Window functions are evaluated **after** `WHERE`, `GROUP BY`, and `HAVING`. They cannot appear in those clauses. To filter on one, wrap it:

```sql
WITH ranked AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY crop_code ORDER BY loaded_at DESC) AS rn
    FROM crop_input_streams
)
SELECT * FROM ranked WHERE rn = 1;
```

Window functions *can* be combined with `GROUP BY` — they then operate over the grouped rows:

```sql
SELECT crop_code, SUM(amount) AS total,
       SUM(SUM(amount)) OVER () AS grand_total
FROM crop_input_streams GROUP BY crop_code;
```

Availability: `[PG]` since 8.4, `[SQLite]` since 3.25 (with `GROUPS` frames and `EXCLUDE` since 3.28).

### Invariants

1. Window functions never change the number of rows. Aggregates do. That's the whole distinction.
2. `PARTITION BY` = "reset per group". `ORDER BY` inside `OVER` = "in this sequence". Frame = "over which slice of that sequence".
3. If you use `ORDER BY` in a window and don't state a frame, you have chosen `RANGE … CURRENT ROW` — state it explicitly whenever the answer depends on it.
4. `ROW_NUMBER()` is only deterministic if the window `ORDER BY` is a total order.

---

## Part 12 — Set operations

```sql
SELECT a, b FROM t1
UNION [ALL]      -- ALL keeps duplicates and skips the sort/hash: prefer it when you know rows are distinct
SELECT a, b FROM t2;

INTERSECT [ALL]
EXCEPT [ALL]     -- called MINUS in some other dialects
```

Rules: same number of columns, compatible types, column names taken from the first branch. `ORDER BY` and `LIMIT` apply to the whole result and go at the end (parenthesize a branch if you need a per-branch `LIMIT`).

For set operations, NULLs **are** treated as equal for deduplication.

Diffing two tables — a useful reconciliation idiom:

```sql
(SELECT * FROM expected EXCEPT SELECT * FROM actual)
UNION ALL
(SELECT * FROM actual   EXCEPT SELECT * FROM expected);
-- empty result = identical sets
```

---

## Part 13 — Scalar function reference

### String

| Purpose | `[PG]` | `[SQLite]` |
|---|---|---|
| Concatenate | `a \|\| b`, `CONCAT(a,b)` (NULL-safe) | `a \|\| b` (NULL-poisoning), `CONCAT` ≥ 3.44 |
| Length | `LENGTH(s)`, `CHAR_LENGTH(s)` | `LENGTH(s)` |
| Case | `UPPER`, `LOWER`, `INITCAP` | `UPPER`, `LOWER` (ASCII only) |
| Trim | `TRIM([LEADING\|TRAILING\|BOTH] c FROM s)`, `LTRIM`, `RTRIM`, `BTRIM` | `TRIM(s, c)`, `LTRIM`, `RTRIM` |
| Substring | `SUBSTRING(s FROM p FOR n)`, `LEFT`, `RIGHT` | `SUBSTR(s, p, n)` |
| Replace | `REPLACE(s, from, to)` | same |
| Position | `POSITION(sub IN s)`, `STRPOS(s, sub)` | `INSTR(s, sub)` |
| Pad | `LPAD(s, n, c)`, `RPAD` | `FORMAT('%0*d', …)` or manual |
| Split | `SPLIT_PART(s, delim, n)`, `STRING_TO_ARRAY` | — (recursive CTE) |
| Regex | `~`, `REGEXP_REPLACE`, `REGEXP_MATCHES` | — (extension required) |
| Format | `FORMAT('%s %I %L', …)` | `PRINTF`/`FORMAT('%s', …)` |

`||` returns NULL if either operand is NULL, in both dialects. `CONCAT` in PG ignores NULLs instead.

### Numeric

```sql
ABS(x)  CEIL(x)  FLOOR(x)  ROUND(x, d)  TRUNC(x, d)  SIGN(x)
MOD(a, b)  a % b   POWER(a, b)  SQRT(x)  EXP(x)  LN(x)  LOG(b, x)
GREATEST(a, b, …)  LEAST(a, b, …)   -- [PG]; [SQLite] uses MAX(a,b)/MIN(a,b) scalar forms
```

**Integer division:** `5 / 2` is `2` in both PostgreSQL (integer types) and SQLite (integer operands). Force real division with `5 * 1.0 / 2` or `CAST(5 AS REAL) / 2`.

`ROUND` on floats uses banker's/binary rounding; `ROUND` on PG `NUMERIC` uses half-up. One more reason money is `NUMERIC`.

### Date and time — PostgreSQL

```sql
CURRENT_DATE, CURRENT_TIMESTAMP, NOW(), LOCALTIMESTAMP
AGE(ts1, ts2)
ts + INTERVAL '1 month'
DATE_TRUNC('month', ts)            -- 'year','quarter','month','week','day','hour'
EXTRACT(YEAR FROM ts)  /  DATE_PART('month', ts)
TO_CHAR(ts, 'YYYY-MM-DD')          -- format to text
TO_DATE('2026-08-29', 'YYYY-MM-DD')
ts AT TIME ZONE 'America/Regina'    -- tz-aware conversion
GENERATE_SERIES('2026-01-01'::date, '2026-12-01'::date, '1 month')
```

### Date and time — SQLite

```sql
date('now')                              -- 'YYYY-MM-DD'
time('now')                              -- 'HH:MM:SS'
datetime('now')                          -- 'YYYY-MM-DD HH:MM:SS' (UTC by default)
datetime('now', 'localtime')
julianday(a) - julianday(b)              -- difference in days
unixepoch('now')                         -- ≥ 3.38
strftime('%Y-%m', ts)                    -- format / extract
date(ts, 'start of month')
date(ts, '+1 month', '-1 day')           -- modifiers chain left to right
CAST(strftime('%m', ts) AS INTEGER)      -- strftime returns TEXT — cast for arithmetic
```

Modifiers: `'+N days'`, `'-N months'`, `'start of month'`, `'start of year'`, `'weekday N'`, `'utc'`, `'localtime'`.

The `strftime` → `CAST` step is easy to forget: `strftime('%m', ts) = 8` is false because `'08' <> 8`.

### JSON

```sql
-- [PG], JSONB
j -> 'key'          -- returns jsonb
j ->> 'key'         -- returns text
j #> '{a,b}'        -- path, returns jsonb
j #>> '{a,b}'       -- path, returns text
j @> '{"k":1}'      -- containment (GIN-indexable)
j ? 'key'           -- key exists
jsonb_build_object('a', 1)   jsonb_agg(x)   jsonb_set(j, '{a}', '2')
jsonb_array_elements(j)      jsonb_each(j)     -- set-returning
```

```sql
-- [SQLite], JSON1 (built in since 3.38)
json_extract(j, '$.key')     j -> '$.key'      j ->> '$.key'   -- ≥ 3.38
json_array_length(j)         json_type(j)      json_valid(j)
json_object('a', 1)          json_group_array(x)   json_group_object(k, v)
json_each(j)                 json_tree(j)      -- table-valued
```

Index a JSON field by indexing the extraction expression:

```sql
CREATE INDEX idx_j_key ON t ((j ->> 'key'));                 -- [PG]
CREATE INDEX idx_j_key ON t (json_extract(j, '$.key'));      -- [SQLite]
```

---

## Part 14 — Views and materialized views

```sql
CREATE [OR REPLACE] VIEW v AS SELECT ...;      -- OR REPLACE is [PG]
CREATE VIEW v (a, b) AS SELECT ...;            -- explicit column names
DROP VIEW [IF EXISTS] v;
```

A view is a stored query, re-executed on every reference. It costs nothing to store and is the right way to publish a stable contract over a volatile schema.

**Updatable views:** `[PG]` auto-updates simple single-table views; anything more needs an `INSTEAD OF` trigger. `[SQLite]` views are always read-only; use `INSTEAD OF` triggers to make them writable.

**Materialized views `[PG]`:**

```sql
CREATE MATERIALIZED VIEW mv AS SELECT ... WITH [NO] DATA;
REFRESH MATERIALIZED VIEW mv;
REFRESH MATERIALIZED VIEW CONCURRENTLY mv;   -- requires a UNIQUE index on mv
```

`CONCURRENTLY` avoids an exclusive lock but is slower and requires the unique index. SQLite has no materialized views — the equivalent is a real table plus a refresh routine, or a trigger-maintained summary table.

---

## Part 15 — Indexes

```sql
CREATE [UNIQUE] INDEX [IF NOT EXISTS] idx_name
    ON table (col1 [ASC|DESC], col2, ...)
    [WHERE predicate];                    -- partial index

CREATE INDEX CONCURRENTLY idx ON t (c);   -- [PG] no write lock, can't be in a transaction
DROP INDEX [IF EXISTS] idx;
```

### What to know

**Composite index column order — the leftmost-prefix rule.** An index on `(a, b, c)` serves queries filtering on `a`, `(a,b)`, or `(a,b,c)`. It does *not* serve a query filtering only on `b`. General ordering heuristic: equality columns first, then the range/sort column last.

**Sargability.** An index on `col` is unusable if the query wraps `col` in a function or applies an implicit cast. This is the single most common cause of a "dead" index:

```sql
-- index on (month) as TEXT, query filters month_num as INTEGER → index unused
WHERE month_num = 8
```

Type mismatch between the indexed column and the query predicate silently disables the index. Check `EXPLAIN` after creating any index — an index that is never chosen is pure write overhead.

**Covering / index-only scans.** If every column the query needs is in the index, the table itself is never read.

```sql
CREATE INDEX idx ON t (a, b) INCLUDE (c);   -- [PG] payload columns, not part of the key
```

**Partial indexes** — index only the rows you query:

```sql
CREATE UNIQUE INDEX uq_one_published
    ON revisions (dataset_id) WHERE status = 'published';
```

This is also the canonical way to express "at most one published revision per dataset", which a plain `UNIQUE` cannot say.

**Expression indexes:**

```sql
CREATE INDEX idx_lower_name ON t (LOWER(name));   -- must match the query expression exactly
```

**PostgreSQL index types:** `BTREE` (default, ordered), `HASH` (equality only), `GIN` (arrays, jsonb, full-text), `GiST` (ranges, geometry), `BRIN` (huge naturally-ordered tables), `SP-GiST`. SQLite has B-tree only.

**Maintenance:**

```sql
ANALYZE;              -- refresh planner statistics; both dialects
VACUUM;               -- [PG] reclaim dead tuples; [SQLite] rebuild the file
REINDEX INDEX idx;
```

Stale statistics are a frequent cause of a good query suddenly planning badly. `ANALYZE` after any bulk load.

### Invariants

1. Every index makes reads potentially faster and writes definitely slower. An unused index is all cost, no benefit.
2. An index is used only when the predicate's type, collation, and expression match the index's. "Almost matching" means "not used".
3. Primary keys and unique constraints create indexes implicitly. Foreign keys do **not** — index the referencing column yourself, or joins and cascading deletes will scan.

---

## Part 16 — Transactions and concurrency

```sql
BEGIN;              -- or BEGIN TRANSACTION / START TRANSACTION
    ...
    SAVEPOINT sp1;
    ...
    ROLLBACK TO SAVEPOINT sp1;
    RELEASE SAVEPOINT sp1;
COMMIT;             -- or ROLLBACK;
```

### ACID, briefly

- **Atomicity** — all or nothing.
- **Consistency** — constraints hold at transaction boundaries.
- **Isolation** — degree to which concurrent transactions can observe each other.
- **Durability** — committed means survives a crash.

### Isolation levels and the anomalies they permit

| Level | Dirty read | Non-repeatable read | Phantom read |
|---|---|---|---|
| READ UNCOMMITTED | possible (not in PG) | possible | possible |
| READ COMMITTED | no | possible | possible |
| REPEATABLE READ | no | no | possible (not in PG) |
| SERIALIZABLE | no | no | no |

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;   -- [PG]
```

PostgreSQL defaults to `READ COMMITTED` and implements MVCC — readers never block writers. Its `REPEATABLE READ` is snapshot isolation and already prevents phantoms; `SERIALIZABLE` adds predicate-based conflict detection and can abort with a serialization failure, which the application must retry.

**SQLite** is single-writer: a write transaction takes an exclusive lock on the whole database. Isolation is effectively serializable. The knobs that matter:

```sql
PRAGMA journal_mode = WAL;      -- readers don't block the writer; persistent setting
PRAGMA synchronous = NORMAL;    -- reasonable durability/speed tradeoff under WAL
PRAGMA busy_timeout = 5000;     -- ms to wait on a lock before SQLITE_BUSY
BEGIN IMMEDIATE;                -- take the write lock now, not on first write
```

`BEGIN IMMEDIATE` avoids the classic SQLite deadlock where two connections start deferred read transactions and both then try to upgrade to write.

### Invariants

1. Transaction scope should match the unit of business meaning. A batch load that must be all-or-nothing is one transaction, not one per row.
2. Long-running transactions hold resources: in PG they block vacuum and bloat tables; in SQLite they block all other writers. Keep them short.
3. Under `SERIALIZABLE`, serialization failures are normal operating conditions, not errors. Write the retry loop.
4. DDL is transactional in PostgreSQL and in SQLite. Migrations can and should be wrapped in a transaction (with the exception of `CREATE INDEX CONCURRENTLY`).

---

## Part 17 — Query planning and EXPLAIN

```sql
EXPLAIN SELECT ...;                          -- estimated plan
EXPLAIN ANALYZE SELECT ...;                  -- [PG] actually runs it, shows real timings
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT JSON) SELECT ...;   -- [PG]
EXPLAIN QUERY PLAN SELECT ...;               -- [SQLite]
```

**`EXPLAIN ANALYZE` executes the statement.** Wrap DML in `BEGIN; … ROLLBACK;` if you don't want the effect.

### Reading a PostgreSQL plan

The plan is a tree; read innermost/most-indented first. Per node, compare `rows=` (estimate) with `actual rows=`. A large divergence means the statistics are wrong, and everything above that node is planned on bad information.

| Scan node | Meaning |
|---|---|
| `Seq Scan` | full table read — fine for small tables, suspicious on large ones |
| `Index Scan` | index lookup then heap fetch |
| `Index Only Scan` | satisfied entirely from the index |
| `Bitmap Heap Scan` | many matches; index gathers pages, then reads them in order |

| Join node | Good when |
|---|---|
| `Nested Loop` | outer side small, inner side indexed |
| `Hash Join` | one side fits in memory, no useful index |
| `Merge Join` | both inputs already sorted on the join key |

Watch for: `Rows Removed by Filter` (a predicate that should have been an index lookup), `Sort Method: external merge Disk` (`work_mem` too small), and nested loops over large row counts.

### SQLite plans

Terse by comparison. Key lines: `SCAN t` (full scan), `SEARCH t USING INDEX idx (col=?)` (index used), `USE TEMP B-TREE FOR ORDER BY` (sort not served by an index). If you created an index and still see `SCAN`, the index isn't matching — check type affinity and collation first.

---

## Part 18 — Recipes

### Deduplicate, keeping the newest row per key

```sql
WITH ranked AS (
    SELECT *, ROW_NUMBER() OVER (
        PARTITION BY natural_key ORDER BY loaded_at DESC, id DESC
    ) AS rn
    FROM staging
)
SELECT * FROM ranked WHERE rn = 1;
```

The `id DESC` tiebreaker makes it deterministic when timestamps collide.

### Top-N per group

Same shape, `WHERE rn <= N`. `[PG]` alternative: `DISTINCT ON (key) … ORDER BY key, sort_col DESC` for N = 1, or `LATERAL` with `LIMIT N`.

### Running total and period-over-period change

```sql
SELECT month_num, amount,
       SUM(amount) OVER (ORDER BY month_num
                         ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running,
       amount - LAG(amount) OVER (ORDER BY month_num) AS mom_change,
       AVG(amount) OVER (ORDER BY month_num
                         ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS ma3
FROM monthly_totals;
```

### Pivot — rows to columns

```sql
SELECT crop_code,
       SUM(amount) FILTER (WHERE month_num = 1)  AS jan,
       SUM(amount) FILTER (WHERE month_num = 2)  AS feb,
       ...
FROM crop_input_streams
GROUP BY crop_code;
```

Column lists must be static. Dynamic pivots require generating the SQL in application code.

### Unpivot — columns to rows

```sql
-- portable
SELECT crop_code, 1 AS month_num, jan AS amount FROM wide
UNION ALL SELECT crop_code, 2, feb FROM wide
UNION ALL ...;

-- [PG] compact
SELECT crop_code, m.month_num, m.amount
FROM wide,
     LATERAL (VALUES (1, jan), (2, feb), (3, mar)) AS m(month_num, amount);
```

### Gaps: dense calendar with zero-filled months

```sql
-- [PG]
WITH months AS (SELECT generate_series(1, 12) AS month_num)
SELECT m.month_num, COALESCE(SUM(s.amount), 0) AS amount
FROM months m
LEFT JOIN crop_input_streams s ON s.month_num = m.month_num
GROUP BY m.month_num ORDER BY m.month_num;

-- [SQLite] recursive CTE instead of generate_series (see Part 10)
```

Note the `COALESCE` on the aggregate: `SUM` of no rows is NULL.

### Islands: group consecutive runs

```sql
WITH marked AS (
    SELECT month_num, active,
           month_num - ROW_NUMBER() OVER (PARTITION BY active ORDER BY month_num) AS grp
    FROM t
)
SELECT active, MIN(month_num) AS from_m, MAX(month_num) AS to_m, COUNT(*) AS len
FROM marked GROUP BY active, grp ORDER BY from_m;
```

The trick: for a consecutive run, position and value increase in lockstep, so their difference is constant across the run.

### Keyset pagination (instead of OFFSET)

```sql
SELECT * FROM t
WHERE (created_at, id) < (:last_created_at, :last_id)   -- row comparison
ORDER BY created_at DESC, id DESC
LIMIT 50;
```

Constant time regardless of depth, and stable under concurrent inserts.

### Slowly changing dimension, type 2

```sql
-- close the current row
UPDATE dim SET valid_to = :now, is_current = 0
WHERE natural_key = :k AND is_current = 1;

-- open a new one
INSERT INTO dim (natural_key, attrs..., valid_from, valid_to, is_current)
VALUES (:k, ..., :now, NULL, 1);
```

Guard with a partial unique index: `CREATE UNIQUE INDEX ON dim (natural_key) WHERE is_current = 1;`

### Reconciliation: does the detail sum to the total?

```sql
SELECT d.parent_id,
       p.total            AS declared,
       SUM(d.amount)      AS derived,
       p.total - SUM(d.amount) AS diff
FROM detail d
JOIN parent p ON p.id = d.parent_id
GROUP BY d.parent_id, p.total
HAVING p.total <> SUM(d.amount);   -- empty result = conserved
```

Run this as an assertion in the pipeline, not as an ad-hoc check.

### Find duplicates on a supposed key

```sql
SELECT k1, k2, COUNT(*) AS n
FROM t GROUP BY k1, k2 HAVING COUNT(*) > 1 ORDER BY n DESC;
```

---

## Part 19 — Traps ledger

Each entry: the symptom, the cause, the fix. Add to this as you hit them.

| # | Trap | Why | Fix |
|---|---|---|---|
| 1 | `NOT IN (subquery)` returns nothing | one NULL in the subquery makes every comparison `UNKNOWN` | `NOT EXISTS` |
| 2 | `LEFT JOIN` silently becomes `INNER` | right-side predicate in `WHERE` kills NULL-extended rows | move it to `ON` |
| 3 | Sums are inflated after a join | join fanned out the grain | pre-aggregate, or use `EXISTS` |
| 4 | `SUM` returns NULL | zero qualifying rows | `COALESCE(SUM(x), 0)` |
| 5 | `LAST_VALUE` returns the current row | default frame is `RANGE … CURRENT ROW` | state `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` |
| 6 | `5 / 2 = 2` | integer division | multiply by `1.0` or cast |
| 7 | Index exists but is never used | type/collation/expression mismatch with the predicate | check `EXPLAIN`; align types |
| 8 | FK violations reach the data in SQLite | `PRAGMA foreign_keys` defaults off, per connection | set it in the connection factory |
| 9 | `UNIQUE` column has repeated blanks | NULLs are not equal, so `UNIQUE` permits many | `NOT NULL`, or a partial unique index |
| 10 | `CHECK` constraint doesn't catch a bad row | `CHECK` passes on `UNKNOWN` | add `NOT NULL` |
| 11 | Pagination shows duplicate/missing rows | non-total `ORDER BY`, or `OFFSET` over shifting data | add a tiebreaker; use keyset pagination |
| 12 | `strftime('%m', ts) = 8` is never true | `strftime` returns TEXT | `CAST(... AS INTEGER)` |
| 13 | `a \|\| b` becomes NULL | concatenation is NULL-poisoning | `COALESCE` each operand, or PG `CONCAT` |
| 14 | Money is off by cents | float arithmetic | `NUMERIC` (PG) / integer cents (SQLite) |
| 15 | Query fast in dev, slow in prod | stale statistics or different data volume | `ANALYZE`; compare estimated vs actual rows |
| 16 | Different results across dialects on the same join | SQLite affinity/collation breaks join-key transitivity | make join key types and collations identical |
| 17 | Un-grouped column in a `GROUP BY` query returns arbitrary values | SQLite permits it; PG rejects it | group by it, or aggregate it explicitly |
| 18 | Rows appear in different order between runs | no `ORDER BY` | there is no default order, ever |

---

## Part 20 — Dialect divergence table

| Feature | PostgreSQL | SQLite |
|---|---|---|
| Type enforcement | static, strict | affinity; `STRICT` tables ≥ 3.37 |
| Boolean type | yes | no (0/1) |
| Native date/time type | yes (`TIMESTAMPTZ`) | no (TEXT/INTEGER/REAL conventions) |
| Exact decimal | `NUMERIC` | no — use integer minor units |
| `RIGHT`/`FULL JOIN` | yes | ≥ 3.39 |
| `LATERAL` | yes | no |
| `DISTINCT ON` | yes | no |
| Window functions | ≥ 8.4 | ≥ 3.25 |
| `FILTER` on aggregates | yes | ≥ 3.30 |
| `GROUPING SETS`/`ROLLUP`/`CUBE` | yes | no |
| `ON CONFLICT` upsert | ≥ 9.5 | ≥ 3.24 |
| `MERGE` | ≥ 15 | no |
| `RETURNING` | yes | ≥ 3.35 |
| `UPDATE … FROM` | yes | ≥ 3.33 |
| Recursive CTE | yes | yes |
| CTE materialization hints | ≥ 12 | ≥ 3.35 |
| Data-modifying CTEs | yes | no |
| Materialized views | yes | no |
| `ALTER TABLE` breadth | wide | rename / add / drop column only |
| `TRUNCATE` | yes | no (`DELETE FROM`) |
| FK enforcement default | on | **off** (per-connection pragma) |
| Index types | btree, hash, gin, gist, brin, spgist | btree only |
| Partial + expression indexes | yes | yes |
| Generated columns | `STORED` only | `VIRTUAL` (default) and `STORED` |
| Arrays | yes | no (JSON as substitute) |
| JSON | `JSON`, `JSONB` + operators | JSON1 functions, `->`/`->>` ≥ 3.38 |
| Regex | built in | extension only |
| `generate_series` | yes | recursive CTE |
| Stored procedures / functions | PL/pgSQL, SQL, others | no (application-defined functions only) |
| Concurrency model | MVCC, many writers | single writer; WAL for concurrent readers |
| Default isolation | READ COMMITTED | serializable in practice |
| NULL sort default | largest (ASC → last) | smallest (ASC → first) |
| `EXPLAIN` | rich, `ANALYZE`/`BUFFERS` | `EXPLAIN QUERY PLAN` |

---

## Appendix A — Style conventions worth keeping

- Keywords uppercase, identifiers `snake_case` lowercase.
- One clause per line; leading commas or trailing commas — pick one and hold it.
- Always qualify columns with a table alias when more than one table is in scope.
- Alias tables with short meaningful names (`s` for streams), not `t1`/`t2`.
- Comment the **grain** of every CTE.
- Prefer `EXISTS` over `IN` for subqueries, `NOT EXISTS` over `NOT IN` always.
- Enumerate columns in anything durable; `SELECT *` only in the REPL.
- Name every constraint and index (`fk_`, `uq_`, `ck_`, `idx_`), so error messages tell you what broke.

## Appendix B — To add as this evolves

- PostgreSQL `EXPLAIN` cost model in detail; `work_mem` / `random_page_cost` tuning
- Full-text search (`tsvector`/`tsquery`; SQLite FTS5)
- Triggers: syntax, `BEFORE`/`AFTER`/`INSTEAD OF`, and when to prefer constraints
- Row-level security `[PG]`
- Partitioning `[PG]`
- `PERCENTILE_CONT` / ordered-set aggregates
- Range types and exclusion constraints for temporal validity
- SQL:2011 system-versioned tables and how to emulate them
- Isolation-level anomalies worked through with concrete interleavings