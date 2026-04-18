# Joins

Stories about how Spark plans and executes joins, and how to make them faster.

## Stories

- [How Spark Chooses a Join](how_spark_chooses_a_join.md) — broadcast hash join, sort-merge join, shuffled hash join, AQE join conversion
- [Join Without Pain: Patterns for Fast Joins on Large Tables](join_optimization_patterns.md) — broadcast, bucketing, AQE conversion, skew handling, salting, partition pruning

## Related stories

- [AQE: How Spark Rewrites Plans After the Shuffle](../adaptive/aqe_rewriting_plans.md) — AQE can convert sort-merge joins to broadcast hash joins at runtime
- [When One Partition Holds Up Everyone: The Data Skew Story](../partitioning/data_skew_story.md) — skewed join keys are a primary cause of join slowdowns
- [The Journey of a Shuffle Record](../shuffle/journey_of_a_shuffle_record.md) — sort-merge joins are built on top of the shuffle mechanism
- [From SQL to a Running Plan: The Catalyst Story](../catalyst/from_sql_to_physical_plan.md) — join strategy selection happens during physical planning
- [What Spark Knows About Your Data: Statistics and the Cost-Based Optimizer](../catalyst/statistics_and_cbo.md) — the CBO uses table statistics to choose optimal join order and strategy
- [EXPLAIN Yourself: How to Read a Spark Physical Plan](../catalyst/explain_output.md) — how to verify which join strategy was actually chosen
