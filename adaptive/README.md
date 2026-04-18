# Adaptive Query Execution

Stories about how Spark rewrites query plans at runtime using real shuffle statistics.

## Stories

- [AQE: How Spark Rewrites Plans After the Shuffle](aqe_rewriting_plans.md) — coalescing shuffle partitions, join conversion, skew join handling, and dynamic partition pruning

## Related stories

- [How Spark Chooses a Join](../joins/how_spark_chooses_a_join.md) — the join strategies AQE can switch between at runtime
- [What Spark Knows About Your Data: Statistics and the Cost-Based Optimizer](../catalyst/statistics_and_cbo.md) — how AQE complements static CBO with real runtime statistics
- [The Journey of a Shuffle Record](../shuffle/journey_of_a_shuffle_record.md) — the shuffle mechanism AQE observes to make its decisions
- [When One Partition Holds Up Everyone: The Data Skew Story](../partitioning/data_skew_story.md) — the skew problem AQE's skew join handler solves
