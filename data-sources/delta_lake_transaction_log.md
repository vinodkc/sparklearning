# The Transaction Log: How Delta Lake Brings ACID to Object Storage

This is the story of how Delta Lake turns a directory of Parquet files on cloud object storage into a transactional table. Object storage—S3, GCS, Azure Blob—was designed for scalable, durable file storage, not for the atomic multi-file updates that databases rely on. Delta Lake bridges this gap with a single elegant mechanism: the **transaction log**. Understanding the transaction log explains how concurrent writers don't corrupt each other's data, how you can query a table as it existed an hour ago, how failed writes leave no trace, and why Delta Lake can provide snapshot isolation on a system that doesn't natively support it.

---

## The problem: object storage has no transactions

A database table is often updated in one atomic operation: you insert 10,000 rows and either all of them appear or none do. Object storage doesn't work that way. You can write files independently, but there is no native primitive to say "write these 50 files atomically." If your Spark job writes 50 Parquet files and crashes after writing 30, the table directory contains 30 new files and 20 missing ones. Any reader that queries now sees a partially written table.

Concurrency makes this worse. Two writers may each read the table's current state, compute their updates on the same set of files, and then independently write new files. The second writer's update silently overwrites the first writer's files, producing a corrupted table.

> **Without transactions, a shared data directory is like a shared whiteboard with no rules.** Anyone can walk up and erase or rewrite at any time, without knowing what others are writing. You might walk in mid-sentence and see half of one person's work and half of another's. The result is incoherent. Delta Lake's transaction log is the rule that says "take turns, sign your changes, and keep a history."

---

## The transaction log: a directory of JSON files

The transaction log lives in a subdirectory called `_delta_log` inside the table's root directory. It is simply a **directory of JSON files**, each representing one committed transaction. The files are named sequentially: `000000000000000000000.json`, `000000000000000000001.json`, and so on. Each file is one transaction; each transaction corresponds to one version of the table.

A transaction log entry is a list of **actions**. The most important actions are:

- **Add**: a new Parquet file has been added to the table. The entry records the file's path, size, partition values, and statistics.
- **Remove**: a Parquet file has been logically deleted from the table. The file is not physically deleted immediately—it stays on disk for a configurable retention period—but it is no longer part of the table's current state.
- **Metadata**: records the table's schema, partition columns, and table-level properties.
- **CommitInfo**: a human-readable record of what operation was performed, who performed it, and when.

To reconstruct the current state of the table, you replay the transaction log from the beginning: start with an empty set, process every Add and Remove action in order, and the result is the set of currently active Parquet files.

> **The transaction log is like a land registry.** Each entry records "Plot 42 was transferred from A to B on this date." To find who owns Plot 42 today, you don't call each previous owner—you read the registry forward from the beginning and find the most recent transfer. The history is permanent; the current owner is the last entry that touched that plot.

---

## Snapshot isolation: reading a consistent view

Every read against a Delta table reads at a **snapshot**—a specific version of the table defined by the transaction log up to a certain point. When a Spark query starts, it records which version of the log it sees (the latest committed transaction). It reads only the files active at that version. Any files added or removed by concurrent writers after that snapshot point are invisible to this query.

This is **snapshot isolation**: a reader sees a consistent, frozen view of the table for the duration of its query, regardless of concurrent writes.

> **Snapshot isolation is like taking a photograph of a busy street.** The moment the shutter clicks, you capture one consistent frame—even though cars are still moving. A reader who looks at your photo sees a stable, consistent scene; they don't see the blur of cars that arrived after the photo was taken. Concurrent writers are those moving cars; your query is the photograph.

Snapshot isolation is achieved purely through the transaction log and file naming—no locks, no coordination with other readers. The reader simply ignores any log entries with versions higher than its snapshot version.

---

## Optimistic concurrency control: how writers avoid conflicts

Delta Lake uses **optimistic concurrency control** (OCC) for concurrent writes. Unlike pessimistic locking (where a writer locks the table before writing), OCC assumes conflicts are rare and checks for them only at commit time.

Here is how a write works:

1. **Read the current version**: the writer records the current log version (say, version 42).
2. **Compute the write**: the writer runs its Spark job, producing new Parquet files. The files are written to the table directory immediately, but they are not yet referenced in any log entry—they are "staged."
3. **Attempt to commit**: the writer tries to create log entry `000000000000000000043.json`. This is an atomic PUT operation—object storage guarantees that only one writer can successfully create a given file name. The log entry lists the staged files as Add actions.
4. **Conflict check**: before the PUT, the Delta Lake client checks whether any transaction was committed between version 42 and 43. If another writer committed version 43 first, the current writer detects the conflict and may rebase and retry as version 44 (if the conflict is on different partitions) or fail (if it is a true conflict).

> **OCC is like two people trying to check out the last copy of a book from the library.** Both approach the counter thinking the book is available. One checks it out first (commits first, writes version 43). The second person arrives at the counter and finds the book gone—they have to either take a different book (rebase on a different partition) or leave empty-handed (fail with a conflict exception). Nobody locked the book on the shelf in advance; the conflict was resolved at the checkout moment.

---

## Time travel: reading historical versions

Because the transaction log is an append-only record of every version of the table, you can reconstruct any historical version by replaying the log only up to that version. This is **time travel**.

You can query a specific version: `spark.read.format("delta").option("versionAsOf", 5).load(path)` reads the table as it was after transaction 5. You can query by timestamp: `spark.read.format("delta").option("timestampAsOf", "2024-01-15 09:00:00").load(path)`.

> **Time travel is like the "undo history" in a document editor.** You can jump back to version 5 of your document and see exactly what it looked like at that point—every deletion, every addition is recorded. You can't un-save a version; the history is permanent (within the retention window). Delta Lake's time travel gives you this for petabyte-scale data tables.

---

## Checkpoints: compacting the log for fast reads

The transaction log grows with every committed transaction. Replaying a log with 100,000 entries would be slow. Delta Lake solves this with **checkpoints**: periodic compactions of the log into a Parquet file that represents the complete table state at a given version.

A checkpoint at version 1000 contains the full list of all active files as of that version—every Add and Remove from versions 0 through 1000 has been resolved into a definitive file list. To reconstruct the current state after version 1000, you only need to read the checkpoint and then replay the small number of JSON log entries after version 1000. By default, Delta Lake creates a checkpoint every 10 transactions.

> **Checkpoints are like a "save state" in a long video game.** Instead of replaying the entire game from the beginning to get back to Level 100, you load the save file at Level 98 and replay only the last 2 levels. The checkpoint is that save file; the transaction log after the checkpoint is those 2 remaining levels.

---

## Schema enforcement and evolution

Every Delta table has a schema, recorded in the `Metadata` action in the log. When you write to a Delta table, Delta checks that the new data's schema is compatible with the table's current schema. By default, writing data with extra columns or incompatible types fails—schema enforcement catches accidental schema drift.

**Schema evolution** can be enabled to allow adding new columns: `option("mergeSchema", "true")` on a write will update the table's schema to include any new columns in the incoming data. This schema change is recorded as a new `Metadata` action in the transaction log, versioning the schema change alongside the data change.

---

## Bringing it together

Delta Lake's transaction log is an append-only directory of JSON files in `_delta_log`, each representing one committed version of the table. Every write is a transaction that records which Parquet files were added and which were removed. **Snapshot isolation** is achieved because readers record a version at query start and ignore subsequent log entries. **Optimistic concurrency control** is achieved because committing a log entry is an atomic object-creation race—one writer wins, others detect the conflict and retry or fail. **Time travel** is possible because the log is never deleted within the retention window—any past version can be reconstructed by replaying the log up to that point. **Checkpoints** compact the log periodically so reading the current state doesn't require replaying thousands of entries. **Schema enforcement** prevents accidental incompatible writes, and **schema evolution** allows deliberate additions. Together, these mechanisms give Delta Lake the ACID properties of a database on infrastructure that supports only simple file writes.
