# The Optimizer's Rulebook: How Catalyst Makes Plans Cheaper

This is the story of Spark's **logical optimizer**—the phase of Catalyst that takes a correct, fully resolved plan and transforms it into a cheaper equivalent. The analyzed plan faithfully represents what you asked for; the optimized plan represents the most efficient way to compute the same answer. The optimizer does this by applying a library of **rewrite rules**, each of which recognizes a pattern in the plan tree and replaces it with a logically equivalent but less costly subtree. No rule changes the result—only the cost.

Understanding which rules fire, what they look for, and what they produce explains why Spark sometimes reads far less data than you'd expect, why certain seemingly redundant queries are automatically simplified, why filters appear to "move" in the plan, and why the plan you write is often not the plan that executes.

---

## How the optimizer works: RuleExecutor and batches

The optimizer, like the Analyzer, is a `RuleExecutor`: it holds named **rule batches**, applies them in order, and repeats each batch until no rule fires (fixed-point convergence). Rules are pure functions `LogicalPlan => LogicalPlan`—they produce new trees without mutating existing nodes.

Optimizer rule batches (abbreviated list in approximate order):

| Batch | Purpose |
|-------|---------|
| Eliminate Distinct | Remove redundant DISTINCT |
| Finish Analysis | Clean up analysis artifacts |
| Union | Combine and simplify UNION nodes |
| **Subquery** | Decorrelate and rewrite subqueries |
| **Replace Operators** | Rewrite high-level operators into primitives |
| **Aggregate** | Push aggregates and simplify |
| **Early Filter & Projection Pushdown** | Move filters and projections down |
| **Operator Optimization** | Core rule batch: most rewrite rules live here |
| **Push Extra Predicate Through Join** | Join condition simplification |
| **Join Reordering (CBO)** | Cost-based join ordering |

Each rule is independently testable, and new rules can be added or disabled. You can inspect which rules are enabled:

```scala
spark.sessionState.optimizer.batches.foreach { batch =>
  println(s"Batch: ${batch.name}")
  batch.rules.foreach(r => println(s"  - ${r.ruleName}"))
}
```

> **The optimizer is like an editor who applies a style guide.** Each rule in the guide says "whenever you see pattern X, rewrite it as Y." The editor applies the guide repeatedly until no more changes are needed. No rule changes the meaning of the text—only its style and efficiency.

---

## Rule group 1: Predicate pushdown

### PushDownPredicate

The most impactful optimizer rule. It moves `Filter` nodes **down** through the plan tree, as close to the data source as possible. When a filter reaches a scan node, the DataSource connector can evaluate it at read time—skipping rows and, for columnar formats like Parquet, skipping entire row groups.

**Before**:
```
Filter (status = 'complete')
+- Project [order_id, customer_id, amount, status, order_date]
   +- Relation[...] parquet
```

**After**:
```
Project [order_id, customer_id, amount, status, order_date]
+- Filter (status = 'complete')            ← pushed below Project
   +- Relation[...] parquet
```

And after DataSource V2 pushdown negotiation, the scan itself absorbs the filter:
```
Project [order_id, customer_id, amount, status, order_date]
+- Relation[...] parquet  (PushedFilters: [IsNotNull(status), EqualTo(status, complete)])
```

**Across joins**: filters on one side of a join are pushed to that side before the join executes. This is critical for star schema queries—a filter on the dimension table that reduces it from 10M to 100 rows is pushed down before the join, dramatically reducing the number of rows that need to be joined.

```sql
SELECT o.*, c.name
FROM orders o JOIN customers c ON o.customer_id = c.id
WHERE c.country = 'US'    -- this filter on the dim table is pushed to the customers scan
  AND o.amount > 1000     -- this filter on the fact table is pushed to the orders scan
```

**What to check in EXPLAIN**: look at `PushedFilters` in the scan node. If your filter doesn't appear there, it wasn't pushed. Common reasons:

**1. The filter uses a UDF (opaque to Catalyst)**

Catalyst cannot look inside a UDF — it treats it as a black box. Even if the UDF simply checks a value, Catalyst cannot push it to the scan.

```python
from pyspark.sql.functions import udf
from pyspark.sql.types import BooleanType

is_premium = udf(lambda status: status == "premium", BooleanType())

df = spark.read.parquet("/data/customers")
df.filter(is_premium(df.status)).explain()
```

```
== Physical Plan ==
*(1) Filter <lambda>(status#3)           ← filter sits ABOVE the scan, not inside it
+- *(1) FileScan parquet [id#1, status#3]
         PushedFilters: []               ← nothing pushed — UDF is opaque
         ReadSchema: struct<id:bigint, status:string>
```

**Fix**: replace the UDF with a native Spark expression that Catalyst understands:

```python
df.filter(df.status == "premium").explain()
# PushedFilters: [IsNotNull(status), EqualTo(status,premium)]  ← pushed
```

---

**2. The filter references a computed column that doesn't exist in the raw data**

If you derive a new column with `withColumn` and then filter on it, the filter is on a computed expression — not a raw column — so Catalyst cannot push it into the file scan.

```python
from pyspark.sql.functions import col, year

df = spark.read.parquet("/data/orders")  # has a raw 'order_ts' timestamp column

df.withColumn("order_year", year(col("order_ts"))) \
  .filter(col("order_year") == 2024) \
  .explain()
```

```
== Physical Plan ==
*(1) Filter (year(order_ts#5) = 2024)    ← filter on derived column, above scan
+- *(1) FileScan parquet [order_ts#5, amount#6]
         PushedFilters: []               ← year(order_ts) is not a raw column
```

**Fix**: filter on the raw column directly, or use partition pruning if `order_ts` is a partition column:

```python
from pyspark.sql.functions import lit, to_timestamp

# Option 1: filter on the raw timestamp column
df.filter((col("order_ts") >= "2024-01-01") & (col("order_ts") < "2025-01-01")).explain()
# PushedFilters: [GreaterThanOrEqual(order_ts,...), LessThan(order_ts,...)]  ← pushed

# Option 2: if 'year' is a Parquet partition column (folder like year=2024/)
df.filter(col("year") == 2024).explain()
# PartitionFilters: [isnotnull(year#9), (year#9 = 2024)]  ← partition pruning
```

---

**3. The filter mixes columns from both sides of a join**

A filter that references columns from *both* the left and right tables is a **join condition**, not a scan filter. Catalyst cannot push it to either side because neither scan alone can evaluate it — the rows from both sides must be present first.

```python
orders    = spark.read.parquet("/data/orders")     # has customer_id, amount
customers = spark.read.parquet("/data/customers")  # has id, credit_limit

joined = orders.join(customers, orders.customer_id == customers.id)

# This filter touches columns from BOTH sides
joined.filter(col("amount") > col("credit_limit")).explain()
```

```
== Physical Plan ==
*(3) Filter (amount#3 > credit_limit#12)        ← post-join filter, above SortMergeJoin
+- *(3) SortMergeJoin [customer_id#2],[id#11]
   :- *(1) FileScan parquet orders[customer_id#2, amount#3]
   :        PushedFilters: []                   ← can't push cross-table condition
   +- *(2) FileScan parquet customers[id#11, credit_limit#12]
            PushedFilters: []
```

Compare this with a filter that touches only *one* side — Catalyst pushes it freely:

```python
# This filter only touches the orders table
joined.filter(col("amount") > 1000).explain()
```

```
== Physical Plan ==
*(3) SortMergeJoin [customer_id#2],[id#11]
:- *(1) Filter (isnotnull(amount#3) AND (amount#3 > 1000))   ← pushed to orders scan
:  +- *(1) FileScan parquet orders[customer_id#2, amount#3]
:           PushedFilters: [IsNotNull(amount), GreaterThan(amount,1000.0)]
+- *(2) FileScan parquet customers[id#11, credit_limit#12]
```

**Fix**: if possible, split the filter so each part touches only one side:

```python
# Pre-filter each side independently before the join
orders_filtered    = orders.filter(col("amount") > 500)       # pushed to orders scan
customers_filtered = customers.filter(col("credit_limit") > 0) # pushed to customers scan
orders_filtered.join(customers_filtered, "customer_id") \
               .filter(col("amount") > col("credit_limit"))    # cross-table check after join
```

---

### CombineFilters

When two consecutive `Filter` nodes exist in the plan, they are merged into one `Filter` with an `AND` condition. This avoids iterating over the data twice.

**Before**:
```
Filter (amount > 100)
+- Filter (status = 'complete')
   +- Relation[...]
```

**After**:
```
Filter ((status = 'complete') AND (amount > 100))
+- Relation[...]
```

---

### EliminateOuterJoin

A `LEFT OUTER JOIN` that is then filtered on a non-null condition from the right side is equivalent to an `INNER JOIN`—any row where the right side was null would have been filtered out anyway. Catalyst detects this and converts the outer join to an inner join, which is cheaper (can use broadcast hash join, more flexible strategy selection).

**Before**:
```sql
SELECT o.*, c.name
FROM orders o LEFT JOIN customers c ON o.customer_id = c.id
WHERE c.name IS NOT NULL   -- ← this filter eliminates the "outer" rows
```

**Logical plan before optimization**:
```
Filter (c.name IS NOT NULL)
+- Join LeftOuter, (o.customer_id = c.id)
```

**After optimization**:
```
Join Inner, (o.customer_id = c.id)    ← LEFT JOIN converted to INNER JOIN
```

> **This is like booking a restaurant for a group with "possible plus-ones," then cancelling the extra seats once you know everyone is coming.** The outer join reserved space for non-matching rows (the nulls). The filter said "we only want rows with matches." The optimizer realizes: just use an inner join—no null rows will survive the filter anyway.

---

## Rule group 2: Column pruning

### ColumnPruning

Removes columns from the plan that are never used by any operator above their producing node. For a scan of a wide table (hundreds of columns), pruning the unused columns means the file reader never reads or decodes them—enormous savings for columnar formats like Parquet.

**Query**:
```sql
SELECT order_id, amount FROM orders WHERE status = 'complete'
-- orders has 50 columns; only 3 are needed: order_id, amount, status
```

**Before pruning** (scan reads all 50 columns):
```
Project [order_id#1, amount#3]
+- Filter (status#4 = 'complete')
   +- Relation[order_id#1, customer_id#2, amount#3, status#4, ... (50 cols)] parquet
```

**After pruning** (scan reads only 3 columns):
```
Project [order_id#1, amount#3]
+- Filter (status#4 = 'complete')
   +- Relation[order_id#1, amount#3, status#4] parquet  ← only 3 cols
```

**What to check in EXPLAIN**: the `FileScan` node lists `ReadSchema`. If it contains only the columns your query needs, pruning worked. If it lists all columns despite your SELECT, something blocked pruning (often a `SELECT *` inside a subquery or a schema mismatch).

---

### PruneFilters (EliminateSorts)

Removes `Sort` nodes that don't affect the final result. An `ORDER BY` inside a subquery that feeds into an aggregation or join has no semantic effect—the aggregate ignores order; the join doesn't care about input order.

```sql
SELECT customer_id, SUM(amount) FROM (
  SELECT * FROM orders ORDER BY order_date   -- ← this sort is useless
) GROUP BY customer_id
```

The inner `ORDER BY order_date` is eliminated because the outer `GROUP BY` doesn't need or preserve any order.

---

## Rule group 3: Constant folding and expression simplification

### ConstantFolding

Evaluates expressions that consist entirely of literals at **plan time** rather than at task execution time. The result is a single `Literal` node, computed once.

**Before**:
```
Filter (amount > (50 + 50))
```

**After**:
```
Filter (amount > 100)   ← 50+50 computed at plan time
```

**More examples**:
```sql
WHERE 1 = 1                → removed (always true)
WHERE DATEDIFF('2024-01-31', '2024-01-01') > 20   → WHERE 30 > 20 → removed
WHERE amount > 0 AND 5 > 3  → WHERE amount > 0   (5>3 is always true, eliminated)
```

---

### BooleanSimplification

Simplifies boolean expressions using standard logical identities:

| Pattern | Simplified to |
|---------|--------------|
| `x AND TRUE` | `x` |
| `x OR FALSE` | `x` |
| `x AND FALSE` | `FALSE` (and the whole branch can be eliminated) |
| `x OR TRUE` | `TRUE` |
| `NOT (NOT x)` | `x` |
| `x AND x` | `x` |

**Practical impact**: generated queries from ORMs or query builders often produce redundant boolean expressions. `BooleanSimplification` cleans them up before execution.

---

### NullPropagation

Propagates `NULL` through expressions where the result is always `NULL` regardless of other inputs. Avoids computing sub-expressions that can never contribute a non-null result.

| Pattern | Result |
|---------|--------|
| `NULL + 5` | `NULL` |
| `NULL * amount` | `NULL` |
| `CONCAT(NULL, 'hello')` | `NULL` |
| `NULL > 100` | `NULL` (not false—SQL three-valued logic) |

**Why this matters**: if a column is declared `NOT NULL` in the schema, Spark knows it can never be null and skips null checks in generated code—slightly tightening the execution loop for every row.

---

### SimplifyConditionals

Simplifies `IF`, `CASE WHEN`, and `COALESCE` expressions where a branch is always or never taken:

```sql
-- Always-true condition
CASE WHEN 1=1 THEN amount ELSE 0 END  →  amount

-- Always-false condition
IF(FALSE, expensive_udf(col), default_value)  →  default_value
-- The expensive UDF is never called

-- COALESCE with a non-null literal
COALESCE(amount, 0)  -- if amount is declared NOT NULL  →  amount
```

> **SimplifyConditionals is like crossing out "if it rains" from your plans when you're in a desert.** The branch is dead—it will never execute. Removing it makes the code simpler and avoids evaluating expressions that will never contribute to the result.

---

## Rule group 4: Subquery rewriting

### RewriteCorrelatedScalarSubquery / DecorrelateInnerQuery

Correlated subqueries that run once per outer row are rewritten into joins that run once for all rows. This is the most important subquery optimization. (Covered in depth in the [Subqueries Untangled](subqueries_untangled.md) story.)

**Before** (executes inner query once per outer row—potentially millions of times):
```sql
SELECT o.*
FROM orders o
WHERE o.amount > (SELECT AVG(amount) FROM orders WHERE customer_id = o.customer_id)
```

**After** (inner query runs once as a GROUP BY, joined back):
```
Join Inner, (o.customer_id = sub.customer_id AND o.amount > sub.avg_amt)
:- Relation[orders]
+- Aggregate [customer_id], [customer_id, avg(amount) AS avg_amt]
   +- Relation[orders]
```

---

### RewriteInToOr / OptimizeIn

For `IN` expressions with a small, static list of literals, Catalyst may rewrite them as a series of `OR` conditions (which codegen handles efficiently) or keep them as an `In` expression with a `HashSet` lookup. For large `IN` lists (over `spark.sql.optimizer.inSetConversionThreshold`, default 10), it converts to `InSet` which uses a hash set for O(1) lookup per row instead of O(n) linear scan.

```sql
WHERE status IN ('pending', 'processing', 'complete', 'shipped', ... 20 values)
-- Converted to InSet with HashSet lookup: O(1) per row instead of 20 comparisons
```

---

## Rule group 5: Join optimization

### ReorderJoin (CBO-dependent)

When joining three or more tables, the order in which pairs are joined matters enormously. Consider three tables:

| Table | Row count | Size |
|---|---|---|
| `orders` | 1 billion | 500 GB |
| `customers` | 1 million | 2 GB |
| `regions` | 50 | 10 KB |

If Spark joins in the written order (`orders ⋈ customers` first), the intermediate result is up to 1 billion rows before the small `regions` table ever narrows it down. If instead Spark joins `regions ⋈ customers` first (producing a small filtered set), the final join against `orders` touches far fewer rows on the right side.

`ReorderJoin` uses **table statistics** — row count and byte size collected by `ANALYZE TABLE` — to find the join order that minimises intermediate result sizes.

**The query**:
```sql
-- Find total order amount for customers in the APAC region
SELECT o.order_id, o.amount, c.name, r.name AS region
FROM   orders    o
JOIN   customers c ON o.customer_id = c.id
JOIN   regions   r ON c.region_id   = r.id
WHERE  r.name = 'APAC'
```

**Step 1 — Predicate pushdown fires first**

`PushDownPredicate` pushes `r.name = 'APAC'` into the `regions` scan before `ReorderJoin` even runs. After pushdown, the effective table sizes are:

| Table after pushdown | Estimated rows |
|---|---|
| `orders` | 1,000,000,000 |
| `customers` | 1,000,000 |
| `regions` (filtered) | ~5 (only APAC sub-regions) |

**Step 2 — Without CBO (default, no statistics)**

Without statistics, Spark has no size estimates and follows the written order:

```
Join order:  orders ⋈ customers  →  1B-row intermediate  ⋈  regions(~5)

Logical plan (Optimized, no CBO):
Join (c.region_id = r.id)                 ← final join
:- Join (o.customer_id = c.id)            ← first join: orders × customers = 1B rows
:  :- Relation orders
:  +- Relation customers
+- Filter (name = APAC)
   +- Relation regions
```

The first join shuffles and processes 1 billion + 1 million rows to produce up to 1 billion output rows, most of which are discarded by the second join.

**Step 3 — With CBO (statistics collected)**

```sql
-- Collect statistics so CBO can estimate sizes
ANALYZE TABLE orders    COMPUTE STATISTICS;
ANALYZE TABLE customers COMPUTE STATISTICS;
ANALYZE TABLE regions   COMPUTE STATISTICS FOR COLUMNS name, region_id;
```

Enable CBO:
```python
spark.conf.set("spark.sql.cbo.enabled", "true")
spark.conf.set("spark.sql.cbo.joinReorder.enabled", "true")
```

Now `ReorderJoin` scores all possible join orders using the statistics. The winning order:

```
regions(~5)  ⋈  customers(1M)  →  ~5,000-row intermediate  ⋈  orders(1B)

Logical plan (Optimized, with CBO):
Join (o.customer_id = c.id)               ← final join: small left side × orders
:- Join (c.region_id = r.id)              ← first join: tiny regions × customers
:  :- Filter (name = APAC)
:  :  +- Relation regions                 ← ~5 rows after filter
:  +- Relation customers                  ← 1M rows
+- Relation orders                        ← 1B rows, probed last
```

The first join produces only ~5,000 rows (5 APAC sub-regions × some customers each). The second join then probes the 1 billion orders table with a 5,000-row left side — far cheaper than a 1 billion-row left side.

**Step 4 — Seeing the difference in `explain("extended")`**

```python
df.explain("extended")
```

Without CBO you see `orders` at the bottom-left (first table to be processed):
```
== Optimized Logical Plan ==
Join region_id#8 = id#15
:- Join customer_id#2 = id#7         ← orders joined first
:  :- Relation[order_id,customer_id,amount] parquet
:  +- Relation[id,name,region_id] parquet
+- Filter (name = APAC)
   +- Relation[id,name] parquet
```

With CBO you see `regions` at the bottom-left (smallest table processed first):
```
== Optimized Logical Plan ==
Join customer_id#2 = id#7
:- Join region_id#8 = id#15          ← regions joined first (smallest)
:  :- Filter (name = APAC)
:  :  +- Relation[id,name] parquet   ← ~5 rows
:  +- Relation[id,name,region_id] parquet   ← 1M rows
+- Relation[order_id,customer_id,amount] parquet  ← 1B rows, joined last
```

> **ReorderJoin is like choosing which ingredient to chop first in a recipe.** If you're making a dish for 2 people and 200 people, you prep the 2-person portion first so you have a small, manageable base to add ingredients to — rather than starting with the 200-person pot and scaling down at the end. CBO is the chef who reads the recipe and figures out the efficient prep order before the cooking starts.

**When ReorderJoin does NOT fire**:
- `spark.sql.cbo.enabled` or `spark.sql.cbo.joinReorder.enabled` is `false` (both default to `false` in most Spark versions)
- Statistics have not been collected (`ANALYZE TABLE` was never run)
- The query has only two tables (no reordering possible)
- A join hint is present (`/*+ BROADCAST */`, `/*+ MERGE */`) — hints override CBO decisions

**Quick check: did CBO reorder your joins?**
```python
# Compare optimized plan with and without CBO
spark.conf.set("spark.sql.cbo.enabled", "false")
df.explain("extended")   # note join order in Optimized Logical Plan

spark.conf.set("spark.sql.cbo.enabled", "true")
df.explain("extended")   # if order changed, CBO rewrote the plan
```

---

### EliminateJoin

A join can be eliminated entirely when Catalyst can prove it has **no effect on the output**. This requires two conditions to be true simultaneously:

1. **No columns from the joined table appear in the output** (SELECT, WHERE, GROUP BY, ORDER BY)
2. **The join cannot change the row count** of the driving table — meaning the join key on the right side is unique (a primary key), so no row on the left side is duplicated or dropped by the join

If both conditions hold, every row in the left table matches exactly one row in the right table, and since no right-side column is used, the join result is identical to the left table alone. Catalyst removes it.

**Example — the join that looks necessary but isn't**:

```python
# orders has: order_id, customer_id, amount
# customers has: id (PK), name, email, region

# Query: we only want order_id from orders, but someone wrote a JOIN anyway
df = spark.sql("""
    SELECT o.order_id
    FROM   orders    o
    JOIN   customers c ON o.customer_id = c.id
""")
df.explain("extended")
```

**Without declared constraints (most Spark deployments) — join is kept**:
```
== Optimized Logical Plan ==
Project [order_id#1]
+- Join Inner, (customer_id#2 = id#7)    ← join is still here
   :- Relation[order_id#1, customer_id#2, amount#3] parquet
   +- Relation[id#7, name#8, email#9] parquet

== Physical Plan ==
*(3) Project [order_id#1]
+- *(3) SortMergeJoin [customer_id#2], [id#7], Inner   ← full shuffle + sort-merge
   :- *(1) FileScan parquet orders[order_id#1, customer_id#2]
   +- *(2) FileScan parquet customers[id#7]
```

Spark cannot prove `customers.id` is unique without being told — so it keeps the join to be safe. A customer_id that has no matching row in customers would be dropped by the inner join, potentially reducing the row count.

**With declared constraints — join is eliminated**:

In catalogs that support constraint declarations (Hive Metastore with `NOT NULL` + `UNIQUE`, Delta Lake with `ALTER TABLE ... ADD CONSTRAINT`, or Unity Catalog), you can declare:

```sql
ALTER TABLE customers ADD CONSTRAINT pk_customers PRIMARY KEY (id);
ALTER TABLE orders    ADD CONSTRAINT fk_orders_customers
    FOREIGN KEY (customer_id) REFERENCES customers(id);
```

Or in code using `hint`-based approaches for testing:

```python
# Simulating with a DataFrame that Catalyst can reason about statically
from pyspark.sql.functions import col

# Mark the join key as unique by creating a deduplicated view
customers_unique = spark.sql("SELECT DISTINCT id FROM customers")  # Catalyst sees this as unique
orders = spark.table("orders")

df = orders.join(customers_unique, orders.customer_id == customers_unique.id, "inner") \
           .select("order_id")   # no customer column selected

df.explain("extended")
```

```
== Optimized Logical Plan ==
Project [order_id#1]                      ← join is GONE
+- Relation[order_id#1, customer_id#2, amount#3] parquet

== Physical Plan ==
*(1) Project [order_id#1]
+- *(1) FileScan parquet orders[order_id#1]   ← scans only orders, no shuffle at all
         ReadSchema: struct<order_id:bigint>
```

The join and the entire customers scan are eliminated. No shuffle, no sort, no network transfer.

**The two conditions illustrated**:

```python
# Condition 1 FAILS — a customer column appears in SELECT → join cannot be eliminated
spark.sql("""
    SELECT o.order_id, c.name          ← c.name is from customers → join must stay
    FROM orders o JOIN customers c ON o.customer_id = c.id
""")

# Condition 2 FAILS — join key is not unique → row count could change
# Imagine customers had duplicate ids (badly loaded data):
# customer_id=101 appears twice in customers → each order for 101 would produce 2 output rows
# Catalyst keeps the join because it cannot guarantee uniqueness
spark.sql("""
    SELECT o.order_id
    FROM orders o JOIN customers_with_dupes c ON o.customer_id = c.id
    ← row count may increase if c.id is not unique → join must stay
""")

# Both conditions MET — join is a candidate for elimination
spark.sql("""
    SELECT o.order_id                  ← no customer column
    FROM orders o JOIN customers c ON o.customer_id = c.id
    ← customers.id declared as PK (unique) → every order matches exactly one customer
    ← join result == orders, so eliminate the join
""")
```

> **EliminateJoin is like checking whether a detour is necessary.** If the side road (the joined table) contributes nothing to your destination (no output columns) and you're guaranteed the road is a 1-to-1 mapping (unique key — no duplicates, no dead ends), you can skip it entirely and take the direct route. But if either condition is uncertain, you must take the detour to be safe.

**In practice**: this optimisation fires most reliably with:
- **Databricks Unity Catalog** or **Delta Lake** tables with declared primary key constraints
- **Hive Metastore** tables with `NOT NULL` + uniqueness hints set via `TBLPROPERTIES`
- **View rewriting** where Catalyst can prove uniqueness from the view definition itself

For plain Parquet/CSV tables with no constraint metadata, Catalyst cannot prove uniqueness and the join is always kept. This is the most common case.

---

## Seeing optimizer rules fire: before and after

The best way to observe optimizer rules is with `explain("extended")`. To make **every** rule visible — including `PushDownPredicate` — the query must have a filter sitting *above* an intermediate projection layer (a subquery). In a flat `SELECT … FROM table WHERE …` the filter is already adjacent to the scan, so there is nothing to push through.

```python
# The subquery creates an intermediate Project layer.
# The outer WHERE filter starts above that inner Project in the analyzed plan,
# giving PushDownPredicate something visible to do.
df = spark.sql("""
    SELECT order_id, amount
    FROM (
        SELECT order_id, customer_id, amount, status, order_date
        FROM orders
        WHERE 1 = 1
    ) sub
    WHERE status = 'complete'
""")

df.explain("extended")
```

---

### Analyzed plan (before any optimization)

The analyzer resolves names and types but does **not** reorder or simplify anything. The plan mirrors the query structure exactly:

```
== Analyzed Logical Plan ==
order_id: bigint, amount: double

Project [order_id#1L, amount#3]               ← outer SELECT (order_id, amount)
+- Filter (status#4 = complete)               ← outer WHERE — sits ABOVE inner Project
   +- Project [order_id#1L, customer_id#2L,   ← inner subquery SELECT (all 5 columns)
               amount#3, status#4, order_date#5]
      +- Filter (1 = 1)                       ← inner WHERE 1=1 — not yet removed
         +- Relation[order_id#1L, customer_id#2L,
                     amount#3, status#4, order_date#5] parquet
```

Key observations:
- The outer `Filter (status = complete)` is sitting *above* the inner `Project` — they are in the wrong order for efficient execution. The inner Project expands all 5 columns needlessly before the filter narrows rows.
- `1 = 1` is still present — not evaluated yet.
- All 5 columns are still in the inner scan — pruning hasn't happened.

---

### Optimized plan (after all rules have fired)

```
== Optimized Logical Plan ==
Project [order_id#1L, amount#3]               ← outer Project kept (column selection)
+- Filter (isnotnull(status#4)                ← Filter moved DOWN — now directly above scan
       AND (status#4 = complete))
   +- Relation[order_id#1L, amount#3,         ← only 3 columns remain (customer_id, order_date gone)
               status#4] parquet
```

---

### What changed — rule by rule

**Rule 1 — PushDownPredicate** (the movement that was missing before)

The outer `Filter (status = complete)` started above the inner `Project`. `PushDownPredicate` pushes it through the `Project` and down toward the scan:

```
Before PushDownPredicate:
  Project [order_id, customer_id, amount, status, order_date]   ← inner projection
  +- Filter (status = complete)                                  ← filter ABOVE projection
     +- Relation orders

After PushDownPredicate:
  Project [order_id, customer_id, amount, status, order_date]   ← inner projection
  +- Filter (status = complete)                                  ← filter now BELOW projection
     +- Relation orders
```

Wait — visually the tree looks the same here because `Project` is above `Filter`. The movement is:

```
BEFORE:
  outer Project [order_id, amount]
  +- outer Filter (status = complete)       ← filter is ABOVE the inner Project
     +- inner Project [order_id, customer_id, amount, status, order_date]
        +- Relation orders

AFTER PushDownPredicate:
  outer Project [order_id, amount]
  +- inner Project [order_id, customer_id, amount, status, order_date]
     +- Filter (status = complete)          ← filter pushed BELOW the inner Project
        +- Relation orders
```

The filter now runs on raw scan rows — before the inner `Project` expands the column set — so fewer rows flow upward.

**Rule 2 — ConstantFolding + BooleanSimplification**

`1 = 1` evaluates to `true` at plan time. `AND true` is removed entirely.

```
Filter ((status = complete) AND (1 = 1))
→ Filter (status = complete)               ← 1=1 gone
```

**Rule 3 — NullPropagation**

`EqualTo(status, complete)` silently drops NULL values, which could produce confusing results. Catalyst adds an explicit `isnotnull` guard to make the semantics precise and allow the file format to apply the null check as a pushed filter:

```
Filter (status = complete)
→ Filter (isnotnull(status) AND (status = complete))
```

**Rule 4 — ColumnPruning**

The inner `Project` referenced 5 columns. After the outer `Project` and `Filter` only use `order_id`, `amount`, and `status`, `ColumnPruning` removes the unnecessary columns from the subquery's projection and from the scan's `ReadSchema`:

```
inner Project [order_id, customer_id, amount, status, order_date]
→ inner Project [order_id, amount, status]   ← customer_id and order_date dropped

Relation[order_id, customer_id, amount, status, order_date]
→ Relation[order_id, amount, status]          ← scan reads 3 columns instead of 5
```

**Rule 5 — CollapseProject**

After column pruning, the outer and inner `Project` nodes select the same columns (`order_id`, `amount`, `status` → `order_id`, `amount`). Having two consecutive `Project` nodes is redundant — `CollapseProject` merges them into one.

```
outer Project [order_id, amount]
+- inner Project [order_id, amount, status]
→ single Project [order_id, amount]
```

---

### Summary of all changes at a glance

| Rule | Before | After |
|---|---|---|
| `PushDownPredicate` | Filter sits above inner Project | Filter pushed below inner Project, closer to scan |
| `ConstantFolding` + `BooleanSimplification` | `AND (1 = 1)` present | `1 = 1` removed |
| `NullPropagation` | `status = complete` | `isnotnull(status) AND status = complete` |
| `ColumnPruning` | 5 columns in scan + inner Project | 3 columns (`order_id`, `amount`, `status`) |
| `CollapseProject` | Two nested Project nodes | Single Project node |

---

## Disabling and customizing optimizer rules

For debugging, you can disable specific rules:

```python
# Disable a specific optimizer rule
spark.conf.set("spark.sql.optimizer.excludedRules",
               "org.apache.spark.sql.catalyst.optimizer.PushDownPredicate")

# Or add custom optimizer rules (Scala/Java)
spark.experimental.extraOptimizations = Seq(MyCustomRule)
```

Disabling `PushDownPredicate` will cause filters to stay above scans—useful for debugging whether a pushdown is causing incorrect results, but very harmful for performance.

---

## Bringing it together

The logical optimizer transforms a correct plan into a cheaper plan through **pattern-matching rewrite rules** organized in batches:

- **PushDownPredicate** — moves filters toward scans; what reaches `PushedFilters` in the scan gets evaluated at read time, skipping data before it enters the JVM
- **ColumnPruning** — removes unused columns from scans; columnar formats skip unread columns entirely
- **CombineFilters** — merges consecutive filters into one pass
- **EliminateOuterJoin** — converts `LEFT JOIN + IS NOT NULL` filter to `INNER JOIN`
- **ConstantFolding** — evaluates constant expressions at plan time
- **BooleanSimplification** — removes tautologies and contradictions
- **NullPropagation** — propagates NULL through expressions statically
- **SimplifyConditionals** — eliminates dead branches in CASE/IF/COALESCE
- **Subquery decorrelation** — rewrites per-row subqueries into single-pass joins
- **ReorderJoin (CBO)** — reorders multi-table joins by estimated intermediate size

Every rule preserves correctness while reducing cost. The optimizer doesn't know about physical execution—it only works with logical plan nodes and expression trees. Physical decisions (which join algorithm to use, where to put shuffles) happen in the next phase: physical planning.
