# Windows into Your Data: How Window Functions Are Planned and Executed

This is the story of window functions—one of SQL's most powerful analytical features—and how Spark plans and executes them. A window function computes a value for each row based on a set of related rows (its "window") without collapsing those rows into a single output row. Ranking every employee by salary within their department, computing a rolling 7-day average, or finding the difference between a row and the one before it—all of these are window function use cases. Understanding how Spark handles window specs, frames, and execution explains why window functions require a specific physical operator, why they can be expensive, and how to use them without paying unnecessary costs.

---

## What makes window functions different

Aggregate functions collapse many rows into one: `SUM(amount)` over a table of 1,000 rows returns one row. Window functions compute a value **for each input row** based on a related set of rows, without reducing the row count. For every row in the input, you get one row in the output—but each row's value may depend on other rows in its "window."

> **Think of a window function like grading on a curve.** Each student (row) still has their own individual grade in the output. But the grade is calculated relative to the other students in the same class (partition). No student disappears from the gradebook; every student gets one result. The "window" is the class the student belongs to.

The SQL syntax for window functions uses the `OVER` clause:

```sql
SELECT
  employee_id,
  department,
  salary,
  RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank,
  SUM(salary) OVER (PARTITION BY department) AS dept_total
FROM employees
```

Each `OVER` clause defines a **window specification**: which partition to compute within, how to order the rows, and optionally a **frame** (a range of rows relative to the current row).

---

## The three parts of a window spec

A window specification has three optional components:

**PARTITION BY**: divides the data into groups (windows). The window function computes independently within each partition. If omitted, the whole dataset is one partition—which is valid but expensive for large tables, since all data must be gathered on one executor.

**ORDER BY**: defines the ordering of rows within each partition. Required for ranking and navigational functions (RANK, ROW_NUMBER, LAG, LEAD, NTILE), and for ordered frames (ROWS BETWEEN, RANGE BETWEEN). For functions like SUM or AVG without an ORDER BY, the frame defaults to the entire partition.

**Frame specification** (ROWS BETWEEN or RANGE BETWEEN): defines which rows relative to the current row are included in the window frame. `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` is a running total (all rows from the start to the current row). `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` is a rolling 7-row window. `RANGE BETWEEN INTERVAL 7 DAYS PRECEDING AND CURRENT ROW` is a rolling 7-day window based on a date column's values.

> **The frame is like the "lookback window" on a windshield.** UNBOUNDED PRECEDING means you can see the entire road behind you. ROWS BETWEEN 3 PRECEDING means you can only see the last 3 cars behind you, regardless of distance. RANGE BETWEEN with a value means you can see cars within 100 metres, regardless of how many cars that is. Different frame specs answer different analytical questions.

---

## Window function categories

**Ranking functions** assign a number to each row based on its position in the ordered partition:
- `ROW_NUMBER()`: unique sequential number, 1 to N, with no ties.
- `RANK()`: same number for tied rows; gaps after ties (1, 2, 2, 4).
- `DENSE_RANK()`: same number for tied rows; no gaps (1, 2, 2, 3).
- `NTILE(n)`: divides the partition into n equal buckets and assigns each row to a bucket.

**Analytic (navigational) functions** access values from other rows in the partition:
- `LAG(col, n)`: value of `col` from n rows earlier in the partition. Used for period-over-period comparisons.
- `LEAD(col, n)`: value of `col` from n rows later.
- `FIRST_VALUE(col)` / `LAST_VALUE(col)`: value at the start or end of the window frame.

**Aggregate window functions**: any aggregate function (`SUM`, `AVG`, `COUNT`, `MAX`, `MIN`) can be used as a window function. Applied with a frame, these compute rolling or cumulative aggregates.

---

## How Spark plans window functions: the WindowExec operator

When Catalyst encounters window functions in a query, it inserts a **WindowExec** operator in the physical plan. This operator:

1. **Sorts** the data by the partition columns and the order-by columns (so all rows of the same partition are contiguous and in order).
2. **Groups** consecutive rows that share the same partition key.
3. **Evaluates** the window function over each partition's rows and the specified frame.

Because all rows of a partition must be processed together, WindowExec requires that the data be **sorted and co-located**. This means a **sort** (and usually a **shuffle** to partition by the window's PARTITION BY keys) is required before WindowExec can run. The shuffle ensures that all rows of the same partition go to the same executor; the sort ensures they are ordered correctly for frame evaluation.

> **WindowExec is like an accountant processing journal entries department by department.** First, all entries are sorted: "all entries for Accounting come first, then all for Engineering, then Marketing." Then the accountant processes each department's block in order—never mixing rows from different departments. The sort and shuffle are the filing system; WindowExec is the accountant who can only work on one department's pile at a time.

---

## Multiple window functions: batching and separating

A single query may have multiple window functions with different window specs:

```sql
SELECT
  RANK() OVER (PARTITION BY dept ORDER BY salary DESC) AS rank_in_dept,
  SUM(salary) OVER (PARTITION BY dept ORDER BY hire_date) AS running_total
FROM employees
```

These two window functions have the same PARTITION BY column but different ORDER BY columns—their window specs are different. Spark cannot evaluate both in a single pass because the sort order required is different for each.

Catalyst groups window functions by their window spec: functions with identical PARTITION BY and ORDER BY and frame specifications are evaluated together in one WindowExec pass. Functions with different window specs require separate WindowExec operators, each with its own sort (and possibly shuffle).

> **Multiple window functions with different ORDER BY clauses are like running two different sorts on the same dataset.** You can't sort a deck of cards by both suit and number simultaneously in a single pass—you need two sorted versions. Each WindowExec with a different ORDER BY is one such sort pass. Each additional unique window spec adds one more sort stage to the plan.

If performance is critical and a query has many window functions with different specs, it may be worth refactoring to reduce the number of distinct window specifications, or pre-sorting data and caching before applying window functions.

---

## Frames and their execution cost

The frame specification significantly affects execution cost.

**ROWS frame with CURRENT ROW or UNBOUNDED PRECEDING**: these are running totals or cumulative aggregates. The evaluation is O(n) per partition: iterate through rows once, maintaining a running aggregator.

**ROWS frame with both bounds relative to the current row** (e.g., `BETWEEN 6 PRECEDING AND CURRENT ROW`): for each row, the window shrinks and grows as the pointer advances. For aggregate functions, Spark maintains a sliding window accumulator. Still O(n) per partition with a bounded constant factor.

**RANGE frame with ORDER BY**: rows within a value-based range of the current row. For a `RANGE BETWEEN INTERVAL 7 DAYS PRECEDING AND CURRENT ROW` on a date column, for each row, Spark includes all rows whose date is within 7 days before. This requires finding the range boundaries for each row, which is O(n log n) in general. The result depends on actual column values, not just row positions.

**UNBOUNDED FOLLOWING**: any frame that looks ahead (e.g., `BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING`) requires buffering the entire partition before computing any row's result, because the last row's value affects the first row's result. Memory usage is proportional to the partition size.

---

## Memory considerations and spill

WindowExec must buffer an entire partition's rows before producing any output. For a partition that contains millions of rows (e.g., all orders for the most active customer in a `PARTITION BY customer_id` window), the executor must hold all those rows in memory simultaneously.

If memory is insufficient, WindowExec can **spill** the partition's rows to disk, sort them, and process them in a streaming fashion. Spill is correct but slow—it writes the partition's data to disk and reads it back. For window functions with large partitions, spill is a common performance problem.

The same data skew problems that affect shuffles affect window functions: a few very large partitions in one executor's window will be slow and memory-intensive while other executors finish quickly.

---

## Window functions vs. self-joins: when to use which

Before window functions, period-over-period comparisons required a self-join:

```sql
-- Self-join approach (before window functions)
SELECT a.order_id, a.amount - b.amount AS change
FROM orders a
JOIN orders b ON a.customer_id = b.customer_id
             AND a.order_date = b.order_date - INTERVAL 1 DAY
```

The equivalent with window functions:
```sql
SELECT order_id, amount - LAG(amount, 1) OVER (PARTITION BY customer_id ORDER BY order_date) AS change
FROM orders
```

The window function approach is almost always better: it requires one shuffle (to co-locate rows by customer_id), while the self-join requires the same shuffle plus a join. For large tables, the self-join also risks creating a Cartesian explosion if keys are not unique.

---

## Bringing it together

Window functions compute per-row results that depend on a set of related rows, defined by a **window specification** (PARTITION BY, ORDER BY, and a frame). Catalyst plans them using the **WindowExec** operator, which requires a **shuffle** to co-locate partition rows and a **sort** to order them within each partition. Window functions with identical specs share one WindowExec pass; different specs require separate passes with their own sorts. **Frames** control which rows contribute to each computation—running totals, rolling windows, and unbounded look-ahead all have different memory and CPU profiles. Skewed partitions (one large partition) translate directly to slow, memory-intensive WindowExec tasks. So the story of a window function is: **sort and co-locate → process partition by partition → compute frame for each row → emit every input row with its computed value.** The power of window functions comes from computing complex analytics without collapsing rows; the cost comes from the sort and the need to buffer each partition's rows in memory.
