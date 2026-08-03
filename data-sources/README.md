# Data Sources & I/O

Stories about reading and writing data — file formats, storage APIs, and the columnar data exchange layer.

## Stories

### Parquet

- [Inside a Parquet File: Row Groups, Column Chunks, and Why Spark Loves It](inside_a_parquet_file.md) — row groups, column chunks, encoding, predicate/projection pushdown, page indexes, bloom filters, vectorized reader fallbacks
- [From DataFrame to Disk: How Spark Writes a Parquet File](writing_parquet_with_spark.md) — output tasks and files, row groups at write time, compression, small-file problem, coalesce/repartition before save
- [Folders, Filters, and Skipping: How Layout Makes or Breaks Parquet Scans](parquet_layout_and_pruning.md) — Hive-style partitions, partition pruning vs row-group skipping, clustering, over-partitioning
- [Columns That Change: Schema, Nested Types, and Evolution in Parquet](parquet_schema_and_evolution.md) — footer schemas, nested types, additive evolution vs renames, mergeSchema cost
- [From Footer to Batches: How Spark Reads Parquet](spark_parquet_read_path.md) — planning, pruning, footers, vectorized batches and fallbacks, UI scan metrics

### Delta Lake

- [The Transaction Log: How Delta Lake Brings ACID to Object Storage](delta_lake_transaction_log.md) — transaction log, protocol/features, snapshot isolation, OCC and isolation levels, time travel, checkpoints
- [Which Files Am I Reading? How Delta Resolves a Snapshot](delta_snapshot_read_path.md) — version → checkpoint + JSON replay → active files, data skipping, hand-off to Parquet
- [Rewriting Reality: How UPDATE, DELETE, and MERGE Change a Delta Table](delta_dml_copy_on_write.md) — copy-on-write file replacement, write amplification, OCC conflicts, deletion-vector preview
- [Beyond Copy-on-Write: Deletion Vectors, Column Mapping, and Liquid Clustering](delta_format_features.md) — DVs, column IDs/renames, liquid clustering, protocol features
- [Housekeeping the Lake: OPTIMIZE, Z-Order, and VACUUM](delta_optimize_and_vacuum.md) — compaction, clustering rewrites, data vs log retention
- [Delta in Motion: Streaming Reads and Exactly-Once Sinks](delta_streaming.md) — version-based source, CDF, per-batch commits, SetTransaction, small-file loop

### Connectors & columnar transfer

- [The DataSource V2 API: How Spark Talks to Storage Systems](datasource_v2.md) — pluggable connector API, pushdown negotiation, transactional writes, streaming source support
- [The Columnar Fast Lane: How Apache Arrow Speeds Up PySpark](arrow_columnar.md) — Arrow columnar format, zero-copy transfer, toPandas() and pandas UDF performance

## Related stories

- [What Is a Table to Spark? The Catalog, Metadata, and the Metastore](../catalog/what_is_a_table_to_spark.md) — how catalog metadata points to the physical files these stories describe
- [Partitions: The Grain of Parallelism](../partitioning/partitions_coalesce_repartition_pruning.md) — coalesce/repartition and partition pruning as Spark-wide concepts
- [Bytes on the Wire: How Spark Serializes Data for Tasks and Shuffles](../serialization/bytes_on_the_wire.md) — how data is serialized once it leaves the file reader
- [Cache Wisely: When Persisting Data Helps and When It Hurts](../memory/caching_strategy.md) — when to cache the results of expensive reads from these formats
- [Two Runtimes, One Job: How PySpark Bridges Python and the JVM](../python/pyspark_bridge.md) — Arrow is the fast path for getting data from Spark into Python
- [Batch by Batch: Inside the Structured Streaming Micro-Batch Engine](../ss/micro_batch_engine.md) — DataSource V2's MicroBatchStream is how streaming sources plug into Structured Streaming
- [Exactly Once, For Real: How Structured Streaming Guarantees No Duplicates](../ss/exactly_once_delivery.md) — Delta as an exactly-once sink via transactional commits
