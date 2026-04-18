# Structured Streaming

Stories about how Spark processes continuous data streams — micro-batches, state, watermarks, and sources.

## Stories

- [Batch by Batch: Inside the Structured Streaming Micro-Batch Engine](micro_batch_engine.md) — StreamExecution thread, offset log, commit log, checkpoint directory
- [Watermarks: How Structured Streaming Decides When to Stop Waiting](watermarks_and_late_data.md) — event time vs processing time, watermark semantics, late data handling, output modes
- [Exactly Once, For Real: How Structured Streaming Guarantees No Duplicates](exactly_once_delivery.md) — at-most-once vs at-least-once vs exactly-once, idempotent writes, transactional commits
- [RocksDB in Structured Streaming](rocksdb_structured_streaming_story.md) — default in-memory state store vs RocksDB, changelog-based checkpointing, GC pressure
- [Keeping Score: How Spark Maintains State Across Micro-Batches](stateful_operations.md) — streaming aggregations, windowed state, deduplication, mapGroupsWithState, timeouts
- [Spark Meets Kafka: How Offsets, Partitions, and Backpressure Work Together](kafka_source.md) — Kafka offset management, partition-to-task mapping, rate limiting, exactly-once recovery
- [When Should the Next Batch Run? The Story of Trigger Types](trigger_types.md) — ProcessingTime, Once, AvailableNow, Continuous triggers and their trade-offs

## Related stories

- [How Spark Survives Failure](../fault-tolerance/lineage_and_fault_tolerance.md) — Structured Streaming's checkpoint-based recovery is an extension of Spark's fault tolerance model
- [The DataSource V2 API: How Spark Talks to Storage Systems](../data-sources/datasource_v2.md) — streaming sources implement the MicroBatchStream interface from DataSource V2
- [The Two Lives of Spark's Memory](../memory/unified_memory_and_block_manager.md) — stateful streaming stores state in the BlockManager or RocksDB
- [The Transaction Log: How Delta Lake Brings ACID to Object Storage](../data-sources/delta_lake_transaction_log.md) — Delta Lake is a common exactly-once sink for Structured Streaming pipelines
- [Partitions: The Grain of Parallelism](../partitioning/partitions_coalesce_repartition_pruning.md) — the number of streaming shuffle partitions controls state store parallelism
