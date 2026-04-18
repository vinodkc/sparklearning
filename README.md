# Spark Internals: Learning Through Stories

**📖 Published as [GitHub Pages](https://vinodkc.github.io/sparklearning/)**

A growing collection of **story-style** explanations of Apache Spark internals. Each story focuses on one concept or subsystem and explains it as a narrative—what problem it solves, how it works, and how the pieces fit together. Every story includes **analogies and real-world examples** to make the ideas concrete and memorable, without diving into code.

**50 stories published**



Stories are grouped by **topic** (each has its own directory); related topics are grouped into **themes** in the index below.

---

## How to use this doc

- **Browse by theme** — Execution core, Query & planning, Streaming, Data & I/O, and more.
- **Read in any order** — Stories are self-contained; follow your curiosity.
- **Look for the blockquotes** — Each story contains `> **Think of it like...`** analogies that anchor the technical concept to something familiar.

### Suggested reading paths

| Goal | Path |
|------|------|
| New to Spark internals | [Driver & Executors](execution/driver_executors_and_the_execution_model.md) → [Scheduler](scheduler/from_action_to_tasks.md) → [Shuffle](shuffle/journey_of_a_shuffle_record.md) → [Memory](memory/unified_memory_and_block_manager.md) → [Fault tolerance](fault-tolerance/lineage_and_fault_tolerance.md) |
| Understanding SQL/DataFrame performance | [Catalyst](catalyst/from_sql_to_physical_plan.md) → [Statistics & CBO](catalyst/statistics_and_cbo.md) → [Joins](joins/how_spark_chooses_a_join.md) → [AQE](adaptive/aqe_rewriting_plans.md) → [EXPLAIN output](catalyst/explain_output.md) → [Parquet](data-sources/inside_a_parquet_file.md) → [Tungsten](tungsten/tungsten_and_binary_rows.md) |
| Debugging slow jobs | [Spark UI](ui-metrics/reading_the_spark_ui.md) → [EXPLAIN output](catalyst/explain_output.md) → [Partitions](partitioning/partitions_coalesce_repartition_pruning.md) → [Data skew](partitioning/data_skew_story.md) → [Shuffle tuning](shuffle/shuffle_tuning.md) → [OOM diagnosis](memory/oom_diagnosis.md) |
| PySpark & UDF performance | [PySpark bridge](python/pyspark_bridge.md) → [UDF tax](udfs/udf_tax.md) → [Pandas UDFs](python/pandas_udfs.md) → [Arrow columnar](data-sources/arrow_columnar.md) → [Serialization](serialization/bytes_on_the_wire.md) |
| Streaming systems | [Micro-batch engine](ss/micro_batch_engine.md) → [Watermarks](ss/watermarks_and_late_data.md) → [Stateful operations](ss/stateful_operations.md) → [Kafka source](ss/kafka_source.md) → [Trigger types](ss/trigger_types.md) → [Exactly-once](ss/exactly_once_delivery.md) → [RocksDB state store](ss/rocksdb_structured_streaming_story.md) |
| Scheduling & fairness | [Scheduler](scheduler/from_action_to_tasks.md) → [Locality](scheduler/locality_and_delay_scheduling.md) → [Fair sharing](scheduler/scheduling_pools_and_fair_sharing.md) → [Dynamic allocation](execution/dynamic_allocation.md) |
| Data lake & storage | [Parquet internals](data-sources/inside_a_parquet_file.md) → [DataSource V2](data-sources/datasource_v2.md) → [Delta Lake](data-sources/delta_lake_transaction_log.md) → [Catalog & tables](catalog/what_is_a_table_to_spark.md) → [Unity Catalog](catalog/unity_catalog.md) |
| Cluster & deploy | [spark-submit](cluster/spark_submit.md) → [YARN mode](cluster/spark_on_yarn.md) → [Kubernetes mode](cluster/spark_on_kubernetes.md) → [SparkConf](config/sparkconf_configuration.md) → [Event log](ui-metrics/event_log_and_history_server.md) |
| Join & shuffle optimization | [Join strategies](joins/how_spark_chooses_a_join.md) → [Join patterns](joins/join_optimization_patterns.md) → [Shuffle tuning](shuffle/shuffle_tuning.md) → [Caching strategy](memory/caching_strategy.md) → [GC tuning](tungsten/gc_tuning.md) |
| Advanced internals | [Tungsten](tungsten/tungsten_and_binary_rows.md) → [Expression tree](catalyst/expression_tree.md) → [Encoders & Datasets](catalyst/encoders_and_datasets.md) → [Subqueries](catalyst/subqueries_untangled.md) → [Window functions](catalyst/window_functions.md) |

---

## Index by theme

### Execution core

How jobs become stages and tasks, how data moves, and how memory and fault tolerance work.

| Topic | Description | Stories |
|-------|-------------|---------|
| [Execution & scheduling](execution/) | From actions to DAG, stages, tasks; driver and executors | [The Driver, the Executors, and How a Job Actually Runs](execution/driver_executors_and_the_execution_model.md) |
| [Dynamic allocation](execution/) | Requesting and releasing executors at runtime; elasticity under load | [Elastic Executors: How Dynamic Allocation Grows and Shrinks the Cluster](execution/dynamic_allocation.md) |
| [Scheduler](scheduler/) | DAG Scheduler, Task Scheduler; how stages and tasks are submitted and run | [From One Action to Many Tasks](scheduler/from_action_to_tasks.md) |
| [Locality and delay scheduling](scheduler/) | Preferred locations, locality levels, delay scheduling; when Spark waits for a good executor | [Locality and Delay Scheduling](scheduler/locality_and_delay_scheduling.md) |
| [Scheduling pools and fair sharing](scheduler/) | Pools, minimum share, weight; how multiple jobs share resources in fair mode | [Scheduling Pools and Fair Sharing](scheduler/scheduling_pools_and_fair_sharing.md) |
| [Shuffle](shuffle/) | Shuffle write/read, sort shuffle, external shuffle service | [The Journey of a Shuffle Record](shuffle/journey_of_a_shuffle_record.md) |
| [Memory & storage](memory/) | Unified memory, BlockManager, caching and eviction | [The Two Lives of Spark's Memory](memory/unified_memory_and_block_manager.md) |
| [Fault tolerance](fault-tolerance/) | Lineage, recomputation, checkpointing, speculation | [How Spark Survives Failure](fault-tolerance/lineage_and_fault_tolerance.md) |
| [Partitioning](partitioning/) | Partitions, coalesce vs repartition, partition pruning | [Partitions: The Grain of Parallelism](partitioning/partitions_coalesce_repartition_pruning.md) |
| [Broadcast & shared state](broadcast/) | Broadcast variables, accumulators | [Shared State in a Distributed Job](broadcast/broadcast_variables_and_accumulators.md) |
| [Data skew](partitioning/) | Detecting and handling skewed partitions; salting, AQE skew join | [When One Partition Holds Up Everyone: The Data Skew Story](partitioning/data_skew_story.md) |

---

### Query & planning

How DataFrame/SQL becomes a plan, how it's optimized, and how joins and adaptive execution work.

| Topic | Description | Stories |
|-------|-------------|---------|
| [Query planning (Catalyst)](catalyst/) | Logical plan, optimization rules, physical plan, codegen | [From SQL to a Running Plan: The Catalyst Story](catalyst/from_sql_to_physical_plan.md) |
| [Statistics & CBO](catalyst/) | Table statistics, column histograms, cost-based optimizer decisions | [What Spark Knows About Your Data: Statistics and the Cost-Based Optimizer](catalyst/statistics_and_cbo.md) |
| [Adaptive & runtime](adaptive/) | AQE: coalescing partitions, join conversion, skew handling, dynamic partition pruning | [AQE: How Spark Rewrites Plans After the Shuffle](adaptive/aqe_rewriting_plans.md) |
| [Join strategies](joins/) | Sort-merge, broadcast, hash join; when each is chosen | [How Spark Chooses a Join](joins/how_spark_chooses_a_join.md) |
| [Subqueries](catalyst/) | Correlated and uncorrelated subqueries; how they are rewritten and executed | [Subqueries Untangled: How Spark Rewrites Nested Queries](catalyst/subqueries_untangled.md) |
| [Window functions](catalyst/) | Window specs, frame boundaries, ranking and analytic functions | [Windows into Your Data: How Window Functions Are Planned and Executed](catalyst/window_functions.md) |
| [Encoders & Datasets](catalyst/) | How Dataset[T] maps JVM types to Spark's internal row format | [The Encoder Contract: How Spark Converts Between JVM Objects and Binary Rows](catalyst/encoders_and_datasets.md) |
| [Expression tree](catalyst/) | How computations are represented as trees of expressions; evaluation model | [Expressions All the Way Down: How Spark Represents and Evaluates Computations](catalyst/expression_tree.md) |
| [Reading EXPLAIN output](catalyst/) | Parsing physical plans to find shuffles, broadcast decisions, and skipped filters | [EXPLAIN Yourself: How to Read a Spark Physical Plan](catalyst/explain_output.md) |

---

### Streaming

State, checkpointing, watermarks, and the lifecycle of micro-batches.

| Topic | Description | Stories |
|-------|-------------|---------|
| [Micro-batch engine](ss/) | How each batch is planned, executed, and committed; the StreamExecution thread | [Batch by Batch: Inside the Structured Streaming Micro-Batch Engine](ss/micro_batch_engine.md) |
| [Watermarks & late data](ss/) | Event time, watermarks, how late records are handled or dropped | [Watermarks: How Structured Streaming Decides When to Stop Waiting](ss/watermarks_and_late_data.md) |
| [Exactly-once delivery](ss/) | Sources, sinks, idempotent writes, transactional commits | [Exactly Once, For Real: How Structured Streaming Guarantees No Duplicates](ss/exactly_once_delivery.md) |
| [Structured Streaming](ss/) | State stores, checkpointing; RocksDB as the state backend | [RocksDB in Structured Streaming](ss/rocksdb_structured_streaming_story.md) |
| [Stateful operations](ss/) | Aggregations over time windows, mapGroupsWithState, flatMapGroupsWithState | [Keeping Score: How Spark Maintains State Across Micro-Batches](ss/stateful_operations.md) |
| [Kafka integration](ss/) | Offset management, partition assignment, rate limiting in the Kafka source | [Spark Meets Kafka: How Offsets, Partitions, and Backpressure Work Together](ss/kafka_source.md) |
| [Trigger types](ss/) | ProcessingTime, Once, AvailableNow, Continuous; what changes under the hood | [When Should the Next Batch Run? The Story of Trigger Types](ss/trigger_types.md) |

---

### Data & I/O

Reading and writing data, formats, and data source APIs.

| Topic | Description | Stories |
|-------|-------------|---------|
| [Parquet internals](data-sources/) | Row groups, column chunks, page encoding, predicate and projection pushdown | [Inside a Parquet File: Row Groups, Column Chunks, and Why Spark Loves It](data-sources/inside_a_parquet_file.md) |
| [Delta Lake basics](data-sources/) | Transaction log, snapshot isolation, schema enforcement, time travel | [The Transaction Log: How Delta Lake Brings ACID to Object Storage](data-sources/delta_lake_transaction_log.md) |
| [Serialization](serialization/) | Tungsten binary format, Kryo, Java serialization; when each is used | [Bytes on the Wire: How Spark Serializes Data for Tasks and Shuffles](serialization/bytes_on_the_wire.md) |
| [DataSource V2 API](data-sources/) | Pluggable connector API; pushdown negotiation, transactional writes, streaming sources | [The DataSource V2 API: How Spark Talks to Storage Systems](data-sources/datasource_v2.md) |
| [Arrow & columnar transfer](data-sources/) | Apache Arrow format, columnar batches, zero-copy transfer in PySpark | [The Columnar Fast Lane: How Apache Arrow Speeds Up PySpark](data-sources/arrow_columnar.md) |

---

### Python & UDFs

How PySpark and UDFs integrate with the JVM.

| Topic | Description | Stories |
|-------|-------------|---------|
| [Python (PySpark)](python/) | JVM ↔ Python bridge, Py4J, Arrow, serialization overhead | [Two Runtimes, One Job: How PySpark Bridges Python and the JVM](python/pyspark_bridge.md) |
| [Pandas UDFs](python/) | Arrow-based columnar UDFs; why they are faster than row-at-a-time UDFs | [Pandas UDFs: How Arrow Makes Python Functions Fast Enough for Spark](python/pandas_udfs.md) |
| [UDFs](udfs/) | Scalar UDF execution path, deserialization cost, why UDFs block Catalyst | [The UDF Tax: Why User-Defined Functions Are a Black Box to the Optimizer](udfs/udf_tax.md) |
| [UDTFs & table functions](udfs/) | User-defined table functions, how they expand one row into many | [One Row In, Many Rows Out: The Story of User-Defined Table Functions](udfs/udtfs.md) |

---

### Cluster & observability

How Spark runs on clusters and how you observe it.

| Topic | Description | Stories |
|-------|-------------|---------|
| [UI & metrics](ui-metrics/) | Spark UI tabs — Jobs, Stages, SQL, Executors, Storage — and what each reveals | [Reading the Spark UI: What Every Tab Is Actually Telling You](ui-metrics/reading_the_spark_ui.md) |
| [spark-submit & resource negotiation](cluster/) | From spark-submit to running tasks; driver launch, executor acquisition | [From spark-submit to Running Tasks: The Resource Negotiation Story](cluster/spark_submit.md) |
| [YARN mode](cluster/) | How Spark runs on YARN; AM lifecycle, container allocation, queue policies | [Spark on YARN: ApplicationMaster, Containers, and the Queue](cluster/spark_on_yarn.md) |
| [Kubernetes mode](cluster/) | Pod lifecycle, driver pod, executor pods, dynamic allocation on K8s | [Spark on Kubernetes: Pods, Namespaces, and Ephemeral Executors](cluster/spark_on_kubernetes.md) |
| [Event log & history server](ui-metrics/) | What goes into the event log, how the history server replays it | [The Event Log: A Complete Record of Everything That Happened in Your Job](ui-metrics/event_log_and_history_server.md) |
| [Configuration](config/) | SparkConf, config sources and precedence, how settings flow through the stack | [SparkConf to Code: How Configuration Reaches the Component That Needs It](config/sparkconf_configuration.md) |

---

### Advanced / internals

Deeper internals: Tungsten, encoders, catalog, and expression trees.

| Topic | Description | Stories |
|-------|-------------|---------|
| [Tungsten](tungsten/) | Binary rows, off-heap memory, cache-friendly layout, UnsafeRow | [Tungsten: How Spark Stopped Trusting the JVM](tungsten/tungsten_and_binary_rows.md) |
| [Catalog & tables](catalog/) | Spark catalog, session catalog, Hive metastore, managed vs external tables | [What Is a Table to Spark? The Catalog, Metadata, and the Metastore](catalog/what_is_a_table_to_spark.md) |
| [Unity Catalog](catalog/) | Governance layer beyond the session catalog; lineage, fine-grained access control | [Beyond the Session Catalog: Unity Catalog and the Governed Lakehouse](catalog/unity_catalog.md) |

---

### Performance & tuning

Practical stories about diagnosing and fixing common Spark performance problems.

| Topic | Description | Stories |
|-------|-------------|---------|
| [OOM diagnosis](memory/) | Heap vs off-heap OOMs, driver vs executor, common causes and fixes | [Out of Memory: A Field Guide to Spark OOM Errors](memory/oom_diagnosis.md) |
| [Reading EXPLAIN output](catalyst/) | Parsing physical plans to find shuffles, broadcast decisions, and skipped filters | [EXPLAIN Yourself: How to Read a Spark Physical Plan](catalyst/explain_output.md) |
| [Shuffle tuning](shuffle/) | Shuffle partition count, spill, sort vs bypass; tuning for job size | [Taming the Shuffle: Partition Count, Spill, and the Right Shuffle for Your Job](shuffle/shuffle_tuning.md) |
| [Join optimization patterns](joins/) | When to broadcast, pre-partition, bucket, or cache to eliminate shuffle | [Join Without Pain: Patterns for Fast Joins on Large Tables](joins/join_optimization_patterns.md) |
| [Caching strategy](memory/) | What to cache, what not to, storage levels, when caching hurts | [Cache Wisely: When Persisting Data Helps and When It Hurts](memory/caching_strategy.md) |
| [GC tuning](tungsten/) | G1GC vs ZGC, heap sizing, off-heap trade-offs, diagnosing GC pauses | [Garbage Collection in Spark: Why the JVM Pauses and How to Make It Stop](tungsten/gc_tuning.md) |

---

## Story map at a glance

| Theme | Stories |
|-------|---------|
| Execution core | 11 |
| Query & planning | 9 |
| Streaming | 7 |
| Data & I/O | 5 |
| Python & UDFs | 4 |
| Cluster & observability | 6 |
| Advanced / internals | 3 |
| Performance & tuning | 6 |
| **Unique total** | **50** |

> Note: [EXPLAIN Yourself](catalyst/explain_output.md) appears under both Query & planning and Performance & tuning — same story, two relevant homes.

---

## Adding new stories

- Put each new story in the **directory for its topic** (create the directory if it's the first story in that group).
- Use a **descriptive filename** (e.g. `rocksdb_structured_streaming_story.md`).
- **Update this README** — add the story link in the table and update the Story map counts.
