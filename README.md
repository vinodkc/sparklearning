# Spark Internals: Learning Through Stories

This doc is a growing collection of **story-style** explanations of Apache Spark internals. Each story focuses on one concept or subsystem and explains it as a narrative—what problem it solves, how it works, and how the pieces fit together—with minimal code, so the ideas stick.

Stories are grouped by topic. Each topic has its own directory; related topics are grouped into **themes** below.

---

## How to use this doc

- **Browse by theme**: the index is organized into Execution core, Query & planning, Streaming, Data & I/O, and so on.
- **Read in any order**: stories are self-contained; you can follow your curiosity.

New stories will be added over time and linked from this README.

---

## Index by theme

### Execution core

How jobs become stages and tasks, how data moves, and how memory and fault tolerance work.

| Topic | Description | Stories |
|-------|-------------|---------|
| [Execution & scheduling](execution/) | From actions to DAG, stages, tasks; driver and executors | *Coming soon* |
| [Scheduler](scheduler/) | DAG Scheduler, Task Scheduler; how stages and tasks are submitted and run | *Coming soon* |
| [Shuffle](shuffle/) | Shuffle write/read, sort shuffle, external shuffle service | *Coming soon* |
| [Memory & storage](memory/) | Unified memory, BlockManager, caching and eviction | *Coming soon* |
| [Fault tolerance](fault-tolerance/) | Lineage, recomputation, checkpointing, speculation | *Coming soon* |
| [Partitioning](partitioning/) | Partitions, coalesce vs repartition, partition pruning | *Coming soon* |
| [Broadcast & shared state](broadcast/) | Broadcast variables, accumulators | *Coming soon* |

---

### Query & planning

How DataFrame/SQL becomes a plan, how it’s optimized, and how joins and adaptive execution work.

| Topic | Description | Stories |
|-------|-------------|---------|
| [Query planning (Catalyst)](catalyst/) | Logical plan, optimization rules, physical plan, codegen | *Coming soon* |
| [Adaptive & runtime](adaptive/) | AQE, dynamic partition pruning | *Coming soon* |
| [Join strategies](joins/) | Sort-merge, broadcast, hash join; when each is chosen | *Coming soon* |

---

### Streaming

State, checkpointing, and the lifecycle of micro-batches.

| Topic | Description | Stories |
|-------|-------------|---------|
| [Structured Streaming](ss/) | State stores, checkpointing, micro-batches, exactly-once | [RocksDB in Structured Streaming](ss/rocksdb_structured_streaming_story.md) |

---

### Data & I/O

Reading and writing data, formats, and data source APIs.

| Topic | Description | Stories |
|-------|-------------|---------|
| [Data sources](data-sources/) | Reading/writing, V1 vs V2 API, file formats | *Coming soon* |
| [Serialization](serialization/) | Tungsten binary format, Kryo, wire format | *Coming soon* |

---

### Python & UDFs

How PySpark and UDFs integrate with the JVM.

| Topic | Description | Stories |
|-------|-------------|---------|
| [Python (PySpark)](python/) | JVM ↔ Python, Arrow, Pandas UDFs | *Coming soon* |
| [UDFs](udfs/) | Scala/Java UDFs, registration, execution path | *Coming soon* |

---

### Cluster & observability

How Spark runs on clusters and how you observe it.

| Topic | Description | Stories |
|-------|-------------|---------|
| [Cluster & deploy](cluster/) | Cluster managers, driver/executor lifecycle, resource negotiation | *Coming soon* |
| [UI & metrics](ui-metrics/) | Spark UI, event log, history server, where metrics come from | *Coming soon* |
| [Configuration](config/) | SparkConf, important configs, how they flow through the app | *Coming soon* |

---

### Advanced / internals

Deeper internals: Tungsten, catalog, and table metadata.

| Topic | Description | Stories |
|-------|-------------|---------|
| [Tungsten](tungsten/) | Binary rows, off-heap, cache-friendly layout | *Coming soon* |
| [Catalog & tables](catalog/) | Spark catalog, table metadata, session catalog | *Coming soon* |

---

## Adding new stories

- Put each new story in the **directory for its topic** (create the directory if it’s the first story in that group).
- Use a **descriptive filename** (e.g. `rocksdb_structured_streaming_story.md`).
- **Update this README**: add the story under the right topic in the table (or add a new topic row and directory if needed).

---

## Quick link to existing content

- **[RocksDB in Structured Streaming: The Story](ss/rocksdb_structured_streaming_story.md)** — Why streaming needs state, why RocksDB is an option, where state lives (local vs checkpoint), and the rhythm of load → work → commit.
