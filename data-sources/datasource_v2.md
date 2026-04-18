# The DataSource V2 API: How Spark Talks to Storage Systems

This is the story of how Spark connects to external data sources—files, databases, streaming systems, and custom storage backends—through a pluggable API. The **DataSource V2 API** (introduced in Spark 2.3 and significantly refined through Spark 3.x) is the contract that any storage system must implement to integrate with Spark as a first-class citizen. Understanding this API explains how pushdowns work, why some connectors support filter and projection pushdown while others don't, how Spark reads and writes in parallel, and why custom connectors look the way they do.

---

## The problem V1 solved badly

The original DataSource API (now called V1) was designed for simple file-based sources. It had significant limitations:

- **No pushdown support**: V1 connectors couldn't tell Spark "I can evaluate this filter myself"—Spark always read all data and then applied filters in the JVM.
- **No streaming**: V1 was batch-only; streaming sources needed a completely separate API.
- **No transactional writes**: V1 had no mechanism for atomic multi-partition writes or for aborting a write cleanly.
- **Driver-centric partition planning**: the driver computed the full list of partitions upfront. For sources with many partitions or dynamic partition structures, this was a bottleneck.
- **No columnar reads**: V1 returned rows; returning Arrow or columnar batches was not possible.

DataSource V2 was designed to address all of these limitations with a clean, extensible interface.

---

## The V2 API hierarchy

DataSource V2 is organized as a set of **capability traits** that a connector implements, indicating which features it supports. The main hierarchy:

**`TableProvider`**: the entry point. A connector implements `TableProvider` and its `getTable()` method, which returns a `Table` object representing the specific table/dataset being accessed. The driver calls `getTable()` during query planning.

**`Table`**: represents the dataset. It has a name, a schema, a set of capabilities (read, write, streaming read, etc.), and methods to create readers and writers. A `Table` is the thing Spark plans against—it provides the schema for Catalyst analysis and declares what pushdowns it can accept.

**`ScanBuilder`** (for reads): created from the `Table`, it is the negotiation point for pushdowns. Spark calls methods on the `ScanBuilder` to offer pushdowns: `pushFilters(filters)`, `pruneColumns(schema)`, `pushLimit(limit)`. The connector examines each offered pushdown and either accepts it (returning `true` or accepting the filter) or rejects it (Spark will apply the pushdown itself after reading). After negotiation, `build()` returns a `Scan`.

**`Scan`**: describes the planned read. It knows the final schema (after column pruning), the accepted filters, and produces `InputPartition` objects—one per Spark task.

**`InputPartition`**: a serializable description of one slice of data to read. The driver creates `InputPartition` objects; executors deserialize them and use them to create `PartitionReader` instances that do the actual reading.

**`PartitionReader`**: runs on the executor. It reads records from its assigned `InputPartition` and returns them one at a time (or as Arrow batches, for columnar connectors).

> **The V2 API is like a staffing agency that brokers between a company (Spark) and contractors (connectors).** The company tells the agency what skills it needs (pushdowns, column pruning). The agency asks each contractor candidate (ScanBuilder) "can you do filtering yourself?" Some say yes; others say no and the company (Spark) does it for them. Once the contractor is hired (a Scan is built), they get their task assignments (InputPartitions) and start working independently (PartitionReaders on executors).

---

## Filter and projection pushdown: the negotiation

The heart of V2's efficiency gains over V1 is the **pushdown negotiation**. During query planning:

1. Catalyst identifies filters in the query that could be applied to the source (predicates involving only source columns, standard comparison operations).
2. The physical planner calls `ScanBuilder.pushFilters(filters)` with those filter expressions.
3. The connector examines each filter. For filters it can evaluate internally (e.g., a JDBC connector can translate `amount > 1000` to `WHERE amount > 1000` in the SQL query), it **accepts** them. For filters it cannot handle (complex expressions, UDF results, non-standard types), it **rejects** them.
4. Accepted filters are removed from Spark's plan—Spark trusts the connector to apply them. Rejected filters remain as `Filter` operators in Spark's plan.
5. Similarly, `pruneColumns(requiredSchema)` tells the connector which columns the query needs. A JDBC connector can translate this to `SELECT col1, col2` instead of `SELECT *`. A Parquet connector reads only the requested column chunks.

The result: only the data that matches the query's filters and projects is returned to Spark's executors. Bytes that can't match are never transferred.

> **Pushdown negotiation is like ordering at a restaurant with dietary restrictions.** You tell the waiter "no gluten, no dairy" (filters). The kitchen (connector) either accommodates your restriction internally (pushdown accepted: the dish arrives already gluten-free) or says "we can't do that in the kitchen, but you can pick around it" (pushdown rejected: Spark applies the filter after reading). Either way, you get the right food—but having the kitchen handle it is more efficient.

---

## Columnar reads: Arrow integration

A significant V2 feature is support for **columnar reads**. A connector can implement `SupportsScanColumnarBatch`, indicating that it returns data as `ColumnarBatch` (Apache Arrow format) rather than row-by-row. This enables:
- Zero-copy reads from Arrow-native formats (e.g., Arrow IPC files)
- Parquet's vectorized reader, which returns Arrow batches
- Efficient interop with PySpark's `toPandas()` via the Arrow pathway

Connectors that return `ColumnarBatch` bypass the row-by-row deserialization path entirely—the Arrow buffer is used directly by Spark's columnar operators, which are codegen'd to process arrays rather than individual rows.

---

## Write support: transactional and streaming

V2 adds structured write support via `WriteBuilder`, `Write`, and `DataWriter`:

**`WriteBuilder`**: created from the `Table`, accepts write options and produces a `Write`.

**`Write`**: describes the planned write. It returns `WriterFactory` objects.

**`DataWriter[T]`**: runs on the executor for each output partition. It receives rows (or Arrow batches for columnar writes), writes them to the target storage, and produces a `WriterCommitMessage` when done.

**`BatchWrite` and `StreamingWrite`**: at commit time, the driver calls the `Table`'s `commit(messages)` method with all partition commit messages. This is the point where the connector can implement **atomic, transactional commits**: only if all partitions succeed does the driver call `commit`; if any fail, it calls `abort`. Delta Lake implements exactly this—all written Parquet files are staged, and only the transaction log entry is written in `commit()`, making the entire write atomic.

> **The V2 write API is like building construction with a sign-off process.** Each subcontractor (DataWriter on each executor) builds their section and files a completion report (WriterCommitMessage). Only when every section is complete and signed off does the general contractor (driver) file the final building permit (commit). If any section fails inspection, the entire project is aborted and the incomplete sections are demolished (abort). The building either opens fully or not at all—never in a half-built state.

---

## Streaming source support

V2 unifies batch and streaming sources through the `MicroBatchStream` and `ContinuousStream` interfaces. A connector that implements `MicroBatchStream` can be used as a Structured Streaming source:

- `latestOffset()`: returns the latest available offset.
- `planInputPartitions(start, end)`: given a range of offsets, returns the `InputPartition` objects to read for this batch.
- `createReaderFactory()`: creates the factory for `PartitionReader` objects on executors.
- `commit(end)`: called after a batch commits; the connector can clean up processed data.

The Kafka connector implements `MicroBatchStream` to provide the offset-based streaming semantics described in the Kafka story. File-based streaming sources (reading new files from a directory) also implement `MicroBatchStream`.

---

## How built-in formats use V2

Most of Spark's built-in format connectors have been migrated to V2 internally:

- **Parquet, ORC, CSV, JSON, Avro**: use V2 for planning (pushdowns, column pruning) while the actual file reading continues to use the well-optimized file reader implementations.
- **Delta Lake**: fully V2-native, implementing transactional writes via the commit protocol.
- **JDBC**: V2 for filter and column pruning pushdown, translating to SQL WHERE clauses and column lists.
- **Kafka**: V2 `MicroBatchStream` for streaming, V2 batch for historical replay.

---

## Writing a custom connector

To implement a simple read-only V2 connector:

1. Implement `TableProvider` (the entry point—given load options, return a `Table`).
2. Implement `Table` (return schema, capabilities, and create a `ScanBuilder`).
3. Implement `ScanBuilder` (accept pushdowns, build a `Scan`).
4. Implement `Scan` (return `InputPartition` objects—one per Spark task).
5. Implement `InputPartition` (serializable; describes one chunk of data).
6. Implement `PartitionReaderFactory` (creates `PartitionReader` on the executor).
7. Implement `PartitionReader` (reads rows from the chunk on the executor).

Register the connector in `META-INF/services/org.apache.spark.sql.sources.DataSourceRegister` and configure it with `.format("my.connector.ClassName")`.

---

## Bringing it together

DataSource V2 is Spark's pluggable storage API, designed to replace V1's limitations with a principled, capability-based contract. A connector exposes a `TableProvider` → `Table` → `ScanBuilder` hierarchy that enables **filter pushdown** (accepted filters are applied by the connector, not Spark), **column pruning** (only requested columns are read), and **columnar reads** (Arrow batches instead of rows). Writes use a two-phase commit pattern (`DataWriter` on executors + atomic `commit` on the driver) that enables transactional, all-or-nothing writes. Streaming is unified through `MicroBatchStream`, eliminating the separate streaming source API of V1. Custom connectors implement only the capabilities they support; Spark handles the rest. So the story of DataSource V2 is: **negotiate capabilities at plan time → distribute partition assignments → execute in parallel on executors → commit atomically**, giving every storage system the ability to participate in Spark's query optimization and execution as a fully first-class data source.
