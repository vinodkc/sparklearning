# Data Sources & I/O

Stories about reading and writing data — file formats, storage APIs, and the columnar data exchange layer.

## Stories

- [Inside a Parquet File: Row Groups, Column Chunks, and Why Spark Loves It](inside_a_parquet_file.md) — row groups, column chunks, encoding, predicate/projection pushdown, bloom filters
- [The Transaction Log: How Delta Lake Brings ACID to Object Storage](delta_lake_transaction_log.md) — transaction log, snapshot isolation, optimistic concurrency, time travel, checkpoints
- [The DataSource V2 API: How Spark Talks to Storage Systems](datasource_v2.md) — pluggable connector API, pushdown negotiation, transactional writes, streaming source support
- [The Columnar Fast Lane: How Apache Arrow Speeds Up PySpark](arrow_columnar.md) — Arrow columnar format, zero-copy transfer, toPandas() and pandas UDF performance

## Related stories

- [What Is a Table to Spark? The Catalog, Metadata, and the Metastore](../catalog/what_is_a_table_to_spark.md) — how catalog metadata points to the physical files these stories describe
- [Bytes on the Wire: How Spark Serializes Data for Tasks and Shuffles](../serialization/bytes_on_the_wire.md) — how data is serialized once it leaves the file reader
- [Cache Wisely: When Persisting Data Helps and When It Hurts](../memory/caching_strategy.md) — when to cache the results of expensive reads from these formats
- [Two Runtimes, One Job: How PySpark Bridges Python and the JVM](../python/pyspark_bridge.md) — Arrow is the fast path for getting data from Spark into Python
- [Batch by Batch: Inside the Structured Streaming Micro-Batch Engine](../ss/micro_batch_engine.md) — DataSource V2's MicroBatchStream is how streaming sources plug into Structured Streaming
