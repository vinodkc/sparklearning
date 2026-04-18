# Shuffle

Stories about how Spark redistributes data across executors and how to tune it.

## Stories

- [The Journey of a Shuffle Record](journey_of_a_shuffle_record.md) — map phase, sort-merge shuffle, shuffle files, reduce phase, External Shuffle Service
- [Taming the Shuffle: Partition Count, Spill, and the Right Shuffle for Your Job](shuffle_tuning.md) — partition sizing, spill management, codec selection, diagnosing shuffle bottlenecks

## Related stories

- [Partitions: The Grain of Parallelism](../partitioning/partitions_coalesce_repartition_pruning.md) — the number of shuffle partitions is the single most impactful tuning parameter
- [AQE: How Spark Rewrites Plans After the Shuffle](../adaptive/aqe_rewriting_plans.md) — AQE coalesces small shuffle output partitions automatically after the map phase
- [How Spark Chooses a Join](../joins/how_spark_chooses_a_join.md) — sort-merge join is built on top of the shuffle; broadcast hash join avoids it entirely
- [The Two Lives of Spark's Memory](../memory/unified_memory_and_block_manager.md) — shuffle spill is triggered when execution memory is insufficient for the in-memory sort buffer
- [Bytes on the Wire: How Spark Serializes Data for Tasks and Shuffles](../serialization/bytes_on_the_wire.md) — shuffle data is serialized and optionally compressed before being written to disk
