# Subqueries Untangled: How Spark Rewrites Nested Queries

This is the story of subqueries—SQL's nested query mechanism—and what Spark does with them. A subquery appears inside another query: in a `WHERE` clause as a filter, in a `SELECT` as a scalar value, or in a `FROM` clause as a derived table. The challenge for Spark is that naive execution of subqueries is expensive: a correlated subquery that filters by a value from the outer query could, in principle, execute once for every row in the outer table. Understanding how Spark detects, classifies, and rewrites subqueries explains why nested SQL is not always as expensive as it looks—and when it is.

---

## The two kinds of subqueries

Subqueries come in two fundamentally different varieties: **uncorrelated** and **correlated**.

An **uncorrelated subquery** does not depend on any column from the outer query. It can be executed once, its result is fixed, and that result is reused by the outer query. For example:

```sql
SELECT * FROM orders WHERE customer_id IN (SELECT id FROM premium_customers)
```

The inner query `SELECT id FROM premium_customers` doesn't reference anything from `orders`. It produces a fixed set of IDs that can be computed once and then applied to `orders`. The inner query is independent of the outer one.

A **correlated subquery** references a column from the outer query inside the subquery. The subquery's result depends on the current row of the outer query:

```sql
SELECT * FROM orders o
WHERE o.amount > (SELECT AVG(amount) FROM orders WHERE customer_id = o.customer_id)
```

The inner query references `o.customer_id`—a column from the outer query. In principle, this means executing the inner query once for every row in `orders`. For a table with 100 million rows, that's 100 million executions of the inner query. Naïve execution is catastrophic.

> **Think of a correlated subquery like a question that changes based on who you're asking about.** An uncorrelated subquery is "which cities have more than 1 million people?"—one answer, used for everyone. A correlated subquery is "for this customer specifically, what's their average order value?"—the answer differs for every customer, so you'd naïvely have to ask the question once per customer. Smart rewriting turns the "once per customer" version into a single group-by query asked once for all customers simultaneously.

---

## Subquery forms in SQL

Subqueries appear in several positions in SQL syntax, each with a different name:

**IN subquery** (semi-join pattern): `WHERE col IN (SELECT col FROM t)`. The outer query filters to rows where the column appears in the subquery's result set. The complement, `NOT IN`, excludes matching rows.

**EXISTS subquery**: `WHERE EXISTS (SELECT 1 FROM t WHERE condition)`. Returns rows where the subquery produces at least one row. Used for correlated existence checks.

**Scalar subquery**: `SELECT (SELECT MAX(price) FROM products WHERE category = o.category)`. A subquery in the SELECT list that must return exactly one value per row.

**Lateral subquery** (FROM clause with LATERAL): `FROM t, LATERAL (SELECT ... FROM ... WHERE ... = t.col)`. The subquery in the FROM clause can reference columns from the preceding table.

---

## How Catalyst handles uncorrelated subqueries

For uncorrelated subqueries, the rewriting is straightforward. Catalyst detects that the subquery has no references to the outer query and plans it as an independent subplan. At execution time, the subplan is run once—typically before the outer query begins—and its result is collected on the driver. This collected result is then used as a runtime filter or a broadcast variable in the outer query.

For an `IN` subquery with a small result set, Catalyst rewrites it as an **IN-list filter** or a **semi-join**. A semi-join is a join where only the left side's rows are emitted (not the right side's columns), and only when a matching row exists on the right. For large right-hand sides, this becomes a hash join; for small ones, it may be a broadcast hash join.

> **An uncorrelated IN subquery is like a guest list check at a venue.** The guest list (inner query) is compiled once before the event and pinned to the wall. Each arriving guest (outer query row) is checked against the list. The list doesn't change during the event; it's computed once and used for everyone.

The key optimizations Catalyst applies to uncorrelated subqueries:
- **Subquery reuse**: if the same uncorrelated subquery appears multiple times in the plan, it is executed only once and its result is shared.
- **Semi-join rewriting**: `IN` and `EXISTS` are rewritten to semi-join operators, which the physical planner can execute efficiently.
- **Broadcast semi-join**: for small subquery results, the result is broadcast to all executors and each executor filters locally without a shuffle.

---

## How Catalyst handles correlated subqueries: decorrelation

The magic of Catalyst's subquery handling is **decorrelation**: converting a correlated subquery (execute-once-per-row) into an equivalent join or aggregate that executes the inner query once for all outer rows simultaneously.

The decorrelation algorithm works by identifying the correlation predicate (the part of the inner query that references the outer query), pulling it out of the subquery, and turning it into a join condition between the outer query and the inner query. The inner query is then executed once (computing results for all possible values of the correlated column), and the result is joined back to the outer query.

For the example above:
```sql
-- Correlated (naive: execute inner query once per row)
SELECT * FROM orders o
WHERE o.amount > (SELECT AVG(amount) FROM orders WHERE customer_id = o.customer_id)

-- After decorrelation (execute inner query once for all customers)
SELECT o.*
FROM orders o
JOIN (SELECT customer_id, AVG(amount) AS avg_amt FROM orders GROUP BY customer_id) sub
  ON o.customer_id = sub.customer_id
WHERE o.amount > sub.avg_amt
```

The correlated subquery that ran once per row has been turned into a single `GROUP BY` aggregation (run once) joined to the outer table. The execution cost is now O(n + m) instead of O(n × m).

> **Decorrelation is like replacing "ask each customer one-on-one" with "run a report for all customers at once and then match the results."** Instead of interviewing 10 million customers individually, you produce a report grouped by customer_id and then join it to the order table. One report, one join—not 10 million interviews.

Not all correlated subqueries can be decorrelated—some involve non-deterministic functions, complex correlations, or patterns that don't translate cleanly to a join. When decorrelation fails, Spark falls back to the interpreted subquery execution model, which may be slow for large inputs.

---

## Scalar subqueries and their execution

A **scalar subquery** must return exactly one row with one column. If it returns zero rows, the result is `NULL`. If it returns more than one row, it throws a runtime error.

Catalyst treats scalar subqueries specially: after planning, the subquery plan is embedded in the physical plan as a **subquery node**. At execution time, the subquery is evaluated once (for uncorrelated scalar subqueries) or once per group (for correlated scalar subqueries after decorrelation). The result is materialized and injected into the outer query as a literal value.

> **A scalar subquery is like asking "what is the current exchange rate?" before a financial calculation.** You ask the question once, get one number, and use that number throughout the rest of the calculation. If the exchange rate query returned two conflicting numbers, or no number at all, the calculation would fail—exactly like a scalar subquery that returns multiple rows.

---

## The EXPLAIN output for subqueries

When you `explain()` a query with subqueries, you'll see **Subquery** nodes in the physical plan. Uncorrelated subqueries appear as separate subplan fragments, often labelled `Subquery #1`, `Subquery #2`, etc. The main plan references these subplans by ID. Correlated subqueries that were decorrelated will appear as join operators in the plan, often with a note about the rewrite.

If you see a subquery that was NOT decorrelated—usually labelled `Filter` with a nested `Subquery` inside a projection—that is the slow path: the subquery will execute once per row. This is worth investigating if the outer table is large.

---

## Common pitfalls with subqueries

**NOT IN with NULLs**: `NOT IN (SELECT col FROM t)` behaves unexpectedly when the subquery returns any NULL value. SQL's three-valued logic means the entire `NOT IN` expression evaluates to unknown (not true) for every row, effectively returning zero rows. If nullability is possible, use `NOT EXISTS` instead.

**Scalar subquery returning multiple rows**: a scalar subquery that can return multiple rows fails at runtime. Always ensure the inner query is constrained (by a `LIMIT 1`, a unique key join, or an aggregate) to return at most one row.

**Correlated subqueries in SELECT lists on large tables**: each distinct value of the correlated column triggers a separate inner query execution if decorrelation fails. Profile with `explain()` before running on large datasets.

---

## Bringing it together

Subqueries in Spark SQL are handled through Catalyst's subquery analysis and rewriting rules. **Uncorrelated subqueries** are planned as independent subplans, executed once, and used as semi-joins or broadcast filters in the outer query. **Correlated subqueries** are rewritten through **decorrelation**: the correlation predicate is lifted out, the inner query is transformed into a group-by aggregate or join, and the result is joined back to the outer query—turning O(n × m) repeated execution into a single O(n + m) join. Scalar subqueries are materialized once and injected as literal values. When decorrelation fails, execution falls back to the per-row model, which is slow for large inputs. So the story of a subquery is: **detect correlation → decorrelate if possible → execute once → join back → use result**, turning SQL's most natural recursive pattern into the same efficient joins and aggregates Spark uses for everything else.
