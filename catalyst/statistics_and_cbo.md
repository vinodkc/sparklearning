# What Spark Knows About Your Data: Statistics and the Cost-Based Optimizer

This is the story of how Spark learns facts about your data and uses them to make better decisions. The query optimizer's job is to choose, from many logically equivalent query plans, the one that will run fastest. Without data about the data—how many rows a table has, how many distinct values a column contains, how large a filtered result will be—the optimizer is flying blind, relying on conservative heuristics. Statistics give it real information, and the **cost-based optimizer** (CBO) turns that information into better plans: better join orders, better join strategies, better estimates throughout the entire pipeline.

---

## Why statistics matter: the join order problem

The most consequential decision the optimizer makes for complex queries is **join order**. For a query that joins five tables, there are 120 ways to order the pairwise joins. Some orderings filter the data down to almost nothing early in the plan; others produce intermediate results that are much larger than the inputs, making subsequent joins expensive. The difference between the best and worst ordering can be orders of magnitude.

Without statistics, the optimizer can't distinguish these orderings. It uses the same heuristic for every plan—usually "smaller table first"—but without knowing actual table sizes, it may guess wrong. With statistics, it can model the cost of each candidate plan: how many rows each join produces, how much data moves across the network, how large the hash map for a hash join would be. The optimizer picks the plan with the lowest estimated cost.

---

## What statistics Spark collects

Spark collects two categories of statistics: **table-level** and **column-level**.

**Table-level statistics** are the basics: the total number of rows and the total size in bytes. These let the optimizer estimate whether a table is small enough to broadcast (broadcast hash join threshold check) and give a baseline for estimating downstream result sizes.

**Column-level statistics** are richer. For each column you analyze, Spark can collect:

- **Distinct count** (approximate, via HyperLogLog): how many unique values the column has. High cardinality means a `GROUP BY` produces many groups; low cardinality means potential for high join fanout.
- **Min and max values**: the range of the column. Used to estimate how many rows a range filter will match—a filter `WHERE amount BETWEEN 100 AND 200` on a column with min=0 and max=10000 is estimated to select about 1% of rows.
- **Null count**: how many rows have null values. Filters on nullable columns can be adjusted for null fraction.
- **Average column length**: used to estimate the byte size of intermediate results.
- **Height-balanced histograms**: the most accurate structure. Instead of just min/max, a histogram divides the value range into buckets and records how many values fall in each. With a histogram, the optimizer can estimate selectivity for filters that target a specific part of the distribution—even on skewed columns where values cluster around certain ranges.

---

## Collecting statistics: ANALYZE TABLE

Statistics are not collected automatically. You must explicitly run `ANALYZE TABLE` (or the equivalent DataFrame command) to populate the catalog with statistics.

The basic command collects row count and table size:
`ANALYZE TABLE my_table COMPUTE STATISTICS`

To collect per-column statistics:
`ANALYZE TABLE my_table COMPUTE STATISTICS FOR ALL COLUMNS`

Or for specific columns:
`ANALYZE TABLE my_table COMPUTE STATISTICS FOR COLUMNS col1, col2, col3`

For histograms (which provide more accurate selectivity estimates than plain min/max):
`ANALYZE TABLE my_table COMPUTE STATISTICS FOR COLUMNS col1 WITH HISTOGRAM`

Statistics are written to the **metastore** (Hive Metastore or Spark's internal catalog) and read by the optimizer at query planning time. They are **not updated automatically** when data changes. After loading new data into a table, you should re-run `ANALYZE TABLE` or the statistics will reflect the old state and may mislead the optimizer.

For partitioned tables, you can also collect per-partition statistics, which allows the optimizer to use them after partition pruning—estimating the size of the post-pruning result rather than the full table.

---

## How the CBO uses statistics

With statistics in hand, the CBO's **join reordering** rule (enabled by `spark.sql.cbo.enabled = true` and `spark.sql.cbo.joinReorder.enabled = true`) can enumerate candidate join orderings for a multi-table query and score each one. The score is an estimate of total intermediate result size across all joins in the plan. Lower total intermediate size means less data flowing through the plan, less memory needed for hash tables, and faster execution.

The scoring model is simple but effective. For each join, the optimizer estimates the output row count as:

```
output_rows ≈ left_rows × right_rows / max(left_distinct_count, right_distinct_count)
```

This is the "uniform distribution" assumption: if the left side has 1M rows and the right side has 100K rows, and the join column has 100K distinct values on the right, about 1 row from the left matches each right row on average, so the output is about 1M rows. A column with fewer distinct values implies higher fanout; a filter that reduces cardinality implies a smaller input to the next join.

Histograms improve this estimate significantly for skewed data. If the join column has a histogram showing that 80% of values fall in one bucket and 20% in another, the optimizer can estimate the join output size more precisely for queries that filter on that column before joining.

---

## Statistics propagation through the plan

The CBO doesn't just use statistics for the base tables—it **propagates** estimated statistics through the plan. After estimating the output size of a filter, it uses that estimate as the input statistics for the next operator. After estimating a join's output, it uses that for the next join's input. This means a multi-join query benefits from statistics all the way through the plan, not just at the leaf tables.

Statistics propagation is also what allows the broadcast threshold to work correctly in the absence of a shuffle. The physical planner checks whether a plan node's estimated output size is below `spark.sql.autoBroadcastJoinThreshold`. With accurate propagated statistics, a table that starts large but is heavily filtered before the join will have a small estimated size at the join point, correctly triggering a broadcast. Without statistics, the planner sees only the base table size and may not broadcast even when the filtered result would easily fit.

---

## Statistics vs. AQE: complementary tools

The CBO and AQE address the statistics problem from different angles.

The CBO works **before execution**, using stored statistics to choose better plans at planning time. It is most effective for well-understood, stable data that has been analyzed. Its weakness is that stored statistics can become stale after data changes, and it can't anticipate runtime conditions like executor failures or actual shuffle sizes.

AQE works **during execution**, using real measured statistics from completed shuffle stages to adapt the plan for remaining stages. It doesn't need pre-collected statistics—it gets the ground truth at runtime. Its weakness is that it can only adapt at shuffle boundaries, not within a stage.

The two are complementary: good statistics enable better initial plans (CBO), and AQE catches whatever the initial plan got wrong (runtime adaptation). For a production environment with a stable data model and regular `ANALYZE TABLE` runs, both should be enabled. CBO produces better initial join orderings and broadcast decisions; AQE handles the cases where estimates were still wrong despite statistics.

---

## Bringing it together

Statistics are the facts that let Spark's optimizer reason about data rather than guess. **Table-level statistics** (row count, size) give the optimizer a scale sense for each table. **Column-level statistics** (distinct count, min, max, nulls, histograms) let it estimate filter selectivity and join fanout. These are collected via `ANALYZE TABLE` and stored in the catalog—they must be refreshed after data changes. The **cost-based optimizer** uses statistics to enumerate join orderings and score each by estimated intermediate result size, picking the ordering that minimizes total data flow. Statistics propagate through the plan, so a heavily-filtered intermediate result correctly shows as small at the next join. The CBO and AQE are complementary: CBO makes better plans before execution; AQE fixes plans mid-execution with real measured sizes. Together they close the loop between what Spark thinks about your data and what your data actually is.
