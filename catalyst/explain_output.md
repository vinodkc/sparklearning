# EXPLAIN Yourself: How to Read a Spark Physical Plan

This is the story of `EXPLAIN`—Spark's window into its own decision-making. When a query runs slowly, the physical plan is almost always where the diagnosis starts. The physical plan shows you exactly what Spark decided to do: which join strategy it chose, where it inserted shuffles, whether it pushed filters down to the file reader, how many rows it expected at each step. Reading an EXPLAIN output fluently is the single most powerful skill for Spark performance investigation. This story teaches you to read it from bottom to top, recognize the key operators, and translate what you see into actionable insight.

---

## How to get the plan

There are three ways to see a query's plan:

**DataFrame API:**
```python
df.explain()          # physical plan only (most common)
df.explain("extended") # parsed → analyzed → optimized → physical
df.explain("cost")    # physical plan with CBO cost estimates
df.explain("codegen") # physical plan + generated Java code
df.explain("formatted") # physical plan in a more readable tree format
```

**SQL:**
```sql
EXPLAIN SELECT * FROM orders WHERE amount > 100;
EXPLAIN EXTENDED SELECT * FROM orders WHERE amount > 100;
```

> **`explain()` is like asking a chef "tell me exactly how you're going to make this dish."** You get the full prep list: which ingredients to pull from which shelf, the order of operations, the cooking techniques. The dish hasn't been made yet—but you know the full plan before any heat is applied. If the plan looks wrong (wrong join strategy, missing index, redundant steps), you fix it before cooking, not after.

---

## Reading direction: bottom to top

Physical plans are written **top-down but read bottom-up**. The bottom of the plan is closest to the data; the top is the final output. Data flows up through the tree. When reading for performance, always start at the leaves (data sources) and work upward toward the root (output).

```
== Physical Plan ==
*(3) HashAggregate(...)                    ← top: final aggregation
+- Exchange hashpartitioning(...)          ← shuffle
   +- *(2) HashAggregate(...)              ← partial aggregation
      +- *(2) Filter (amount > 100)        ← filter on scan
         +- *(2) FileScan parquet [...]    ← bottom: reads files
```

Read this as: "scan Parquet files → filter while scanning → partially aggregate → shuffle → final aggregate."

The `*(N)` prefix indicates a **whole-stage codegen stage number**—all operators with the same number are fused into one tight loop. An `Exchange` operator is a codegen boundary (a shuffle can't be fused with surrounding operators).

---

## Key operators and what they mean

**FileScan / Scan parquet / Scan orc**: reading data from storage. The annotations show which columns are read (`output columns`), which filters were pushed into the reader (`PushedFilters`), and whether partition pruning was applied (`PartitionFilters`). A scan with `PushedFilters: [IsNotNull(amount), GreaterThan(amount,100.0)]` means Spark successfully pushed the filter into the Parquet reader—rows that can't match are skipped at the file level, before any processing.

> **PushedFilters in a scan are like a pre-screening call before a job interview.** Instead of flying all 10,000 applicants in for an in-person interview (reading all rows), the recruiter calls candidates first and only invites the 200 who pass the phone screen (satisfy the filter). The cost of the plane ticket (reading and decoding a row) is only paid for candidates that have a chance.

**Filter**: a filter that could NOT be pushed to the scan. Rows are read first, then filtered. If you see a `Filter` above a `FileScan` for a filter that should be pushable (e.g., a simple equality on a column), investigate whether the expression uses a UDF, a cast, or some other pattern that blocks pushdown.

**Project**: column selection. A `Project [col1, col2]` above a scan means only `col1` and `col2` are needed—but if it's *below* the scan, column pruning was applied and the scan itself only reads those two columns.

**Exchange**: a shuffle. This is one of the most expensive operators. The annotation shows the partitioning: `hashpartitioning(customer_id, 200)` means shuffle by `customer_id` into 200 partitions. `RoundRobinPartitioning(n)` is a repartition with no key. `SinglePartition` gathers everything to one reducer. Every `Exchange` is a potential bottleneck—look at how many there are and how large their inputs are.

**BroadcastExchange**: the small side of a broadcast hash join being distributed to all executors. If you see this, the query planner decided one table is small enough to broadcast. If you expected a `BroadcastHashJoin` but see `SortMergeJoin` instead, the small table's size estimate may be above the broadcast threshold (or statistics are missing).

**Sort**: sorts rows within each partition. Required by `SortMergeJoin` and `repartitionByRange`. A sort above an exchange is typically the sort phase of a sort-merge join.

**BroadcastHashJoin**: the fast join—one side broadcast, the other probed locally. No `Exchange` on the build side (the broadcast side). An `Exchange` on only the probe side (if the large table isn't already partitioned correctly) may appear.

**SortMergeJoin**: the scalable join—`Exchange` on both sides, then `Sort` on both sides, then a merge. Two shuffles, two sorts. This is correct and scalable but expensive. If you see this for a join where one side is small, check your statistics (`ANALYZE TABLE`) or add a `broadcast()` hint.

**ShuffledHashJoin**: like sort-merge but without the sort—one side builds an in-memory hash table after the shuffle.

**HashAggregate**: a two-phase aggregation. You'll usually see two: a partial `HashAggregate` close to the scan (computing local partial results) and a final `HashAggregate` above the shuffle (merging partial results). This is the normal and optimal pattern.

**ObjectHashAggregate / SortAggregate**: fallback aggregation modes used when the aggregation function doesn't support hash-based partial aggregation (e.g., `collect_list`, custom UDAFs). These are generally slower.

**Window**: a WindowExec operator. Check what it's partitioned by and sorted by—these imply a preceding exchange and sort.

---

## Operator-level statistics: rows and size

With `explain("formatted")` or with AQE enabled, each operator shows the **estimated (or actual) row count and size**. These numbers are key:

- A Filter with `rows output: 1,000,000,000` when you expected 1,000 means the filter isn't pushing down—all rows are read and then filtered.
- An Exchange with `data size: 500 GB` for a join explains exactly why the job is slow.
- A `BroadcastHashJoin` with `rows output: 50` from the broadcast side confirms the broadcast is working correctly.

When AQE is active, the plan shows **actual** post-execution statistics at each operator (for completed stages), giving you ground truth rather than estimates.

---

## What to look for: a diagnostic checklist

**1. Are filters pushing down to the scan?**
Look at the `FileScan` operator. Do `PushedFilters` list your WHERE clause conditions? If not, the filter might be blocked by a UDF, a non-supported expression, or a column not in the file's schema. All data is being read and then filtered in memory—expensive.

**2. Are the right join strategies being used?**
For every join, check whether you see `BroadcastHashJoin` or `SortMergeJoin`. If a small table is using `SortMergeJoin`, run `ANALYZE TABLE` or add a `broadcast()` hint. If a large table is unexpectedly being broadcast, check for a wrong size estimate and reduce `spark.sql.autoBroadcastJoinThreshold`.

**3. How many shuffles are there?**
Count the `Exchange` operators. Each is a full data redistribution. More than 2–3 shuffles in a single query usually indicates a join or aggregation that could be restructured. Pre-partitioned or bucketed tables can eliminate shuffles for common joins.

**4. Are there unnecessary sorts?**
A `Sort` that's not feeding a join or a window function may be an artefact of a mismatched partitioning. Check whether the data could be sorted earlier in the pipeline.

**5. What's the codegen boundary count?**
Each `*(N)` stage number change is a codegen boundary (an operator that can't be fused). Boundaries are expected at exchanges and certain aggregations. An unusually high number of boundaries in a simple pipeline may indicate UDFs, non-codegen-supported functions, or Python UDF calls breaking the pipeline.

**6. Are there AdaptiveSparkPlan nodes?**
With AQE on, the plan root shows `AdaptiveSparkPlan`. Click through in the UI or use `explain("formatted")` to see both the initial plan and the final adapted plan. Changes from `SortMergeJoin` to `BroadcastHashJoin` will be visible here.

---

## A worked example

```
== Physical Plan ==
*(5) Project [order_id#1, customer_name#22]
+- *(5) BroadcastHashJoin [customer_id#2], [id#20], Inner, BuildRight
   :- *(5) Filter (isnotnull(customer_id#2))
   :  +- *(5) FileScan parquet orders[order_id#1,customer_id#2]
   :        PushedFilters: [IsNotNull(customer_id)]
   :        PartitionFilters: []
   +- BroadcastExchange HashedRelationBroadcastMode([id#20])
      +- *(4) Filter (isnotnull(id#20))
         +- *(4) FileScan parquet customers[id#20,customer_name#22]
               PushedFilters: [IsNotNull(id)]
```

Reading bottom-up:
- Stage `*(4)`: scan `customers` (reading only `id` and `customer_name` columns—column pruning applied), filter nulls, broadcast the result.
- Stage `*(5)`: scan `orders` (reading only `order_id` and `customer_id`—pruned), filter nulls, probe the broadcast hash map of customers, project the final columns.

Observations:
- `BroadcastHashJoin` with `BuildRight`: Spark broadcast the customers table. Good—it's small.
- `PushedFilters`: `IsNotNull` was pushed down to both scans. Basic but correct.
- `PartitionFilters: []`: no partition column filter on `orders`. If `orders` is partitioned by `date` and the query had a date filter, we'd want to see it here; its absence would indicate missed partition pruning.
- The whole right side (customers scan + broadcast) is in stage `*(4)`, and the probe side is `*(5)`—two codegen stages, reasonable.

---

## Bringing it together

Reading an EXPLAIN output is reading Spark's decisions about your data. The physical plan is a tree of operators—read **bottom-up, from data sources to output**. Key operators to recognize: `FileScan` (check `PushedFilters` and `PartitionFilters`), `Exchange` (every shuffle, count them), `BroadcastHashJoin` vs. `SortMergeJoin` (join strategy), `HashAggregate` pairs (two-phase aggregation), `Window` (requires sort + exchange), and `AdaptiveSparkPlan` (AQE adaptation visible here). The numbers annotated on each operator—estimated or actual row counts and data sizes—are the evidence that connects the plan to the performance. A plan with the right filters pushed down, the right joins, and the fewest possible shuffles is the goal. `explain()` is the tool that tells you whether you've achieved it.
