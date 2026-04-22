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
- The filter uses a UDF (opaque to Catalyst)
- The filter references a computed column that doesn't exist in the raw data
- The filter mixes columns from both sides of a join

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

When joining three or more tables, the order matters enormously. Joining a 1B-row fact table with a 100-row dimension table first produces 1B rows for the next join; joining the two dimension tables first produces 100 rows. `ReorderJoin` uses table statistics (row counts, sizes) to find the join order that minimizes intermediate result sizes.

**Example** (orders: 1B rows, customers: 1M rows, regions: 50 rows):
```sql
SELECT * FROM orders o JOIN customers c ON o.customer_id = c.id
                       JOIN regions r ON c.region_id = r.id
WHERE r.name = 'APAC'
```

**Without CBO (default order—may be suboptimal)**:
```
orders (1B) ⋈ customers (1M) → 1B rows  ⋈  regions (50, filtered to ~5)
```

**With CBO** (`spark.sql.cbo.enabled=true` and statistics collected):
```
regions (50, filtered to ~5) ⋈ customers (1M, filtered by region) → ~50K  ⋈  orders (1B)
```

The filter on `regions` is pushed down first, regions and customers join to produce a much smaller intermediate, and only then does the large orders table join.

---

### EliminateJoin

If a join is provably unnecessary—e.g., joining on a unique key where no columns from the right side are selected—the join can be eliminated entirely.

```sql
SELECT o.order_id FROM orders o JOIN customers c ON o.customer_id = c.id
-- If customer_id is a foreign key to a unique customers.id, the join adds nothing to the output
-- (no customer columns are selected, and every order has exactly one customer)
-- EliminateJoin removes the join
```

This requires statistics and declared constraints (primary/foreign keys), which are rare in practice outside of catalog-managed tables.

---

## Seeing optimizer rules fire: before and after

The best way to observe optimizer rules is with `explain("extended")`:

```python
df = spark.sql("""
    SELECT order_id, amount
    FROM orders
    WHERE status = 'complete'
      AND 1 = 1
""")

df.explain("extended")
```

**Analyzed plan** (before optimization):
```
== Analyzed Logical Plan ==
Project [order_id#1L, amount#3]
+- Filter ((status#4 = complete) AND (1 = 1))
   +- Relation[order_id#1L, customer_id#2L, amount#3, status#4, order_date#5] parquet
```

**Optimized plan** (after optimization):
```
== Optimized Logical Plan ==
Project [order_id#1L, amount#3]
+- Filter (isnotnull(status#4) AND (status#4 = complete))
   +- Relation[order_id#1L, amount#3, status#4] parquet
```

Changes visible:
- `1 = 1` is gone (ConstantFolding + BooleanSimplification)
- `isnotnull(status#4)` was added (NullPropagation adds null check for EqualTo)
- `customer_id#2L` and `order_date#5` are gone from the scan (ColumnPruning)
- Filter was pushed below Project (PushDownPredicate)

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
