# Catalyst: Query Planning & Optimization

Stories about how Spark SQL turns a query into an optimized physical plan and executes it.

## Stories

- [From SQL to a Running Plan: The Catalyst Story](from_sql_to_physical_plan.md) — parsing, analysis, logical optimization, physical planning, whole-stage codegen
- [What Spark Knows About Your Data: Statistics and the Cost-Based Optimizer](statistics_and_cbo.md) — table/column statistics, histograms, CBO join reordering
- [Subqueries Untangled: How Spark Rewrites Nested Queries](subqueries_untangled.md) — correlated vs uncorrelated subqueries, decorrelation, semi-join rewriting
- [Windows into Your Data: How Window Functions Are Planned and Executed](window_functions.md) — window specs, frames, WindowExec, memory and skew considerations
- [The Encoder Contract: How Spark Converts Between JVM Objects and Binary Rows](encoders_and_datasets.md) — ExpressionEncoder, Dataset[T] vs DataFrame, Kryo fallback
- [Expressions All the Way Down: How Spark Represents and Evaluates Computations](expression_tree.md) — expression trees, leaf/unary/binary nodes, interpreted vs codegen evaluation
- [EXPLAIN Yourself: How to Read a Spark Physical Plan](explain_output.md) — reading physical plans, key operators, diagnostic checklist

## Related stories

- [AQE: How Spark Rewrites Plans After the Shuffle](../adaptive/aqe_rewriting_plans.md) — runtime plan rewriting that picks up where static Catalyst optimization leaves off
- [How Spark Chooses a Join](../joins/how_spark_chooses_a_join.md) — the physical join strategies Catalyst selects during planning
- [Tungsten: How Spark Stopped Trusting the JVM](../tungsten/tungsten_and_binary_rows.md) — the binary execution layer that whole-stage codegen targets
- [Out of Memory: A Field Guide to Spark OOM Errors](../memory/oom_diagnosis.md) — diagnosing memory problems often starts with reading the physical plan
