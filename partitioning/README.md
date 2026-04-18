# Partitioning

Stories about how Spark divides data into partitions, and how to deal with skew.

## Stories

- [Partitions: The Grain of Parallelism](partitions_coalesce_repartition_pruning.md) — partition basics, coalesce vs repartition, repartitionByRange, partition pruning
- [When One Partition Holds Up Everyone: The Data Skew Story](data_skew_story.md) — detecting skew, hot keys, salting, AQE skew join handling

## Related stories

- [The Journey of a Shuffle Record](../shuffle/journey_of_a_shuffle_record.md) — shuffles are how data is repartitioned; partition count directly controls shuffle output
- [AQE: How Spark Rewrites Plans After the Shuffle](../adaptive/aqe_rewriting_plans.md) — AQE coalesces small shuffle partitions and splits skewed ones at runtime
- [Join Without Pain: Patterns for Fast Joins on Large Tables](../joins/join_optimization_patterns.md) — skewed join keys and bucket-based pre-partitioning are covered here
- [Taming the Shuffle: Partition Count, Spill, and the Right Shuffle for Your Job](../shuffle/shuffle_tuning.md) — how to choose the right number of shuffle partitions
- [Windows into Your Data: How Window Functions Are Planned and Executed](../catalyst/window_functions.md) — window functions PARTITION BY creates the same data-per-partition tradeoffs
