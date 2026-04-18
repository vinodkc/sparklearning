# Memory & Storage

Stories about how Spark manages memory — from the unified memory model to OOM diagnosis and caching strategy.

## Stories

- [The Two Lives of Spark's Memory](unified_memory_and_block_manager.md) — unified memory manager, execution vs storage memory, BlockManager, LRU eviction
- [Out of Memory: A Field Guide to Spark OOM Errors](oom_diagnosis.md) — heap vs off-heap OOMs, driver vs executor, common causes and fixes
- [Cache Wisely: When Persisting Data Helps and When It Hurts](caching_strategy.md) — when to cache, storage levels, cache invalidation, unpersist discipline

## Related stories

- [Tungsten: How Spark Stopped Trusting the JVM](../tungsten/tungsten_and_binary_rows.md) — off-heap Tungsten memory sits outside the unified memory pool
- [Taming the Shuffle: Partition Count, Spill, and the Right Shuffle for Your Job](../shuffle/shuffle_tuning.md) — shuffle spill is directly caused by insufficient execution memory
- [Partitions: The Grain of Parallelism](../partitioning/partitions_coalesce_repartition_pruning.md) — partition count determines how much data each task holds in memory at once
- [Garbage Collection in Spark: Why the JVM Pauses and How to Make It Stop](../tungsten/gc_tuning.md) — GC pressure is closely related to how memory is used and how data is cached
- [RocksDB in Structured Streaming](../ss/rocksdb_structured_streaming_story.md) — RocksDB state store moves streaming state off the JVM heap to reduce GC pressure
