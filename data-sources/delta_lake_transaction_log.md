# The Transaction Log: How Delta Lake Brings ACID to Object Storage

This is the story of how Delta Lake turns a directory of Parquet files on cloud object storage into a transactional table. Object storage—S3, GCS, Azure Blob—was designed for scalable, durable file storage, not for the atomic multi-file updates that databases rely on. Delta Lake bridges this gap with a single elegant mechanism: the **transaction log**. Understanding the transaction log explains how concurrent writers don't corrupt each other's data, how you can query a table as it existed an hour ago, how failed writes stay invisible to readers (even if orphan files briefly sit on storage), and why Delta Lake can provide snapshot isolation on a system that doesn't natively support it.

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
- **Protocol**: records the minimum reader/writer protocol versions (and, in modern tables, which **table features** are required). Enabling deletion vectors or column mapping is ultimately a protocol/feature commitment every client must honor.
- **CommitInfo**: a human-readable record of what operation was performed, who performed it, and when.
- **SetTransaction** (optional): records that an application id completed a named version/batch—used by some streaming and exactly-once writers so a retried batch does not commit twice under the same app identity.

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
2. **Compute the write**: the writer runs its Spark job, producing new Parquet files. The files are written to the table directory immediately, but they are not yet referenced in any log entry; they are "staged." Orphan staging files from crashed jobs can remain on storage; they are invisible to readers until (and unless) a commit names them, and cleanup/VACUUM eventually removes unreferenced objects.
3. **Attempt to commit**: the writer tries to create the next log entry (`…043.json`) through Delta's **LogStore**, a thin layer that provides atomic, mutually exclusive commit creation on top of the underlying filesystem or object store (HDFS-style renames, conditional writes, or an external store such as DynamoDB on S3; not a guarantee of raw S3 PUT alone). Exactly one committer wins for that version name.
4. **Conflict handling**: if creating version 43 fails because another writer already committed it, the loser reads the intervening commits and checks whether its own changes conflict under Delta's isolation rules. Two outcomes matter, and they treat the already-written Parquet files differently:
   - **Compatible race (rebase)**: the concurrent commits do not logically invalidate this writer's staged work (classic case: two blind appends). Delta **retries only the commit** at a higher version (44, then 45, …) and **reuses the same staged Parquet files**. It does not throw those files away or rewrite their bytes. The expensive Spark write already finished; only the log-entry race is retried.
   - **Logical conflict (fail)**: the intervening commits *do* invalidate the work (classic case: two writers both try to `Remove` the same data file). Delta raises a concurrent-modification exception. The staged Parquet files are **not** committed. They stay on storage as invisible orphans until VACUUM. Delta does **not** automatically rewrite them. If your job or pipeline retries, that retry is a **new transaction**: read a fresh snapshot, write **new** Parquet files, and attempt a new commit. The previous attempt's files are abandoned, not reused.

> **OCC is like two people trying to check out the last copy of a book from the library.** Both approach the counter thinking the book is available. One checks it out first (commits version 43). The second person finds that version taken: if they wanted a *different* book that is still available, they can complete a later checkout (rebase to version 44). If they both needed the *same* last copy, the second checkout fails. Nobody locked the shelf in advance; the conflict is resolved at the counter.

### What happens to the files that were already written?

The key mental model: **data files are cheap to leave lying around; the commit is the moment of truth.** Staging happens before the race. Winning or losing the race decides whether those staged paths become part of the table.

**Real-life example 1: two ingest jobs appending events (reuse).**

Two pipelines append clickstream events into the same Delta table. Job A and Job B both read version 100, then each write a handful of Parquet files under the table path. Job A wins `…101.json` and publishes its `Add` actions. Job B's attempt at version 101 fails on the LogStore race. Delta checks: Job B only adds files; Job A's commit only added different files; under WriteSerializable this is compatible. Job B **keeps** the Parquet files it already wrote and commits them as version 102. Readers never saw a partial append; Job B did not re-run its Spark write just because the version number was taken.

Think of two delivery trucks that both loaded boxes overnight and both drive to the warehouse dock. The dock clerk can stamp only one receipt at a time. Truck A gets receipt #101. Truck B's boxes are already on the dock; the clerk does not make Truck B drive back to the factory and reload. The clerk stamps receipt #102 for the same boxes.

**Real-life example 2: two UPDATEs touching the same file (throw away and rewrite on retry).**

Two jobs each try to correct different rows that happen to live in the same Parquet file `part-0007.parquet`. Both read version 50, both rewrite that file into a new staged file (`part-A-new.parquet` and `part-B-new.parquet`), and both plan to `Remove` `part-0007.parquet`. Job A commits first as version 51. Job B's commit attempt sees that the file it read was removed by a concurrent writer: a logical conflict. Job B fails with a concurrent-modification exception. `part-B-new.parquet` is never added to the log. It is an orphan. If an orchestrator retries Job B, the retry must read the **new** snapshot (which already includes Job A's rewrite), compute a fresh rewrite against whatever file is active now, write **new** staged bytes, and commit again. Reusing `part-B-new.parquet` would be wrong: it was computed against an outdated base file.

Think of two editors who each photocopied the same chapter, marked up their own copy, and raced to file the replacement with the records office. Editor A's replacement is accepted. Editor B's marked-up photocopy is based on the old chapter and cannot be filed as-is. The office rejects it. If Editor B tries again, they must photocopy the **current** chapter (A's version), mark it up again, and submit a new replacement. Yesterday's rejected photocopy stays in the recycling bin until cleanup day (VACUUM).

**Real-life example 3: crash after staging, before commit (orphans, invisible).**

A job writes 40 Parquet files, then the cluster dies before creating the next `_delta_log` JSON file. Those 40 files sit on object storage but no log entry names them, so every reader ignores them. A later successful write (or a retry of the same job) writes its own files and commits normally. VACUUM eventually deletes the unreferenced crash leftovers after the retention window.

So: **compatible commit races reuse staged files; logical conflicts abandon them; application retries rewrite.** The transaction log is what makes "already written" safe either way, because readers only trust files named by a committed version.

### Isolation levels: what counts as a conflict

"Compatible" is not vibes; it is defined by the table's **isolation level**.

**WriteSerializable** (Delta's common default) allows concurrent transactions that commute in safe ways. Most importantly, concurrent **blind appends** that only `Add` files typically succeed after rebase, reusing their staged files as above. Operations that read the table and then rewrite files (updates, deletes, merges, overwrite patterns) conflict when their read/write sets disagree with what landed in intervening commits.

**Serializable** is stricter: more pairs of concurrent transactions are treated as conflicts, favoring a serial order of writes over throughput under contention.

Readers still see **snapshot isolation** either way: they pick a version and ignore later commits. Isolation levels govern **which concurrent writers may both commit**, not whether a reader mid-scan suddenly sees new files.

---

## Time travel: reading historical versions

Because the transaction log is an append-only record of every version of the table, you can reconstruct any historical version by replaying the log only up to that version. This is **time travel**.

You can query a specific version: `spark.read.format("delta").option("versionAsOf", 5).load(path)` reads the table as it was after transaction 5. You can query by timestamp: `spark.read.format("delta").option("timestampAsOf", "2024-01-15 09:00:00").load(path)`.

> **Time travel is like the "undo history" in a document editor.** You can jump back to version 5 of your document and see exactly what it looked like at that point—every deletion, every addition is recorded. You can't un-save a version; the history is permanent (within the retention window). Delta Lake's time travel gives you this for petabyte-scale data tables.

---

## Checkpoints: compacting the log for fast reads

The transaction log grows with every committed transaction. Replaying a log with 100,000 entries would be slow. Delta Lake solves this with **checkpoints**: periodic compaction of the log into a Parquet snapshot (sometimes accompanied by sidecar files in newer layouts) that represents the complete table state at a given version.

A checkpoint at version 1000 contains the full list of all active files as of that version—every Add and Remove from versions 0 through 1000 has been resolved into a definitive file list. To reconstruct the current state after version 1000, you only need to read the checkpoint and then replay the small number of JSON log entries after version 1000. By default, Delta Lake creates a checkpoint every 10 transactions.

> **Checkpoints are like a "save state" in a long video game.** Instead of replaying the entire game from the beginning to get back to Level 100, you load the save file at Level 98 and replay only the last 2 levels. The checkpoint is that save file; the transaction log after the checkpoint is those 2 remaining levels.

---

## Schema enforcement and evolution

Every Delta table has a schema, recorded in the `Metadata` action in the log. When you write to a Delta table, Delta checks that the new data's schema is compatible with the table's current schema. By default, writing data with extra columns or incompatible types fails—schema enforcement catches accidental schema drift.

**Schema evolution** can be enabled to allow adding new columns: `option("mergeSchema", "true")` on a write will update the table's schema to include any new columns in the incoming data. This schema change is recorded as a new `Metadata` action in the transaction log, versioning the schema change alongside the data change.

---

## Bringing it together

Delta Lake's transaction log is an append-only directory of JSON files in `_delta_log`, each representing one committed version of the table. Every write is a transaction that records which Parquet files were added and which were removed. **Snapshot isolation** is achieved because readers record a version at query start and ignore subsequent log entries. **Optimistic concurrency control** is achieved because committing the next log version is an atomic race through Delta's LogStore—one writer wins that version name; others detect the conflict and rebase or fail under **WriteSerializable** or stricter **Serializable** rules. **Protocol** actions record which table features readers must understand. **Time travel** is possible because the log is never deleted within the retention window—any past version can be reconstructed by replaying the log up to that point. **Checkpoints** compact the log periodically so reading the current state doesn't require replaying thousands of entries. **Schema enforcement** prevents accidental incompatible writes, and **schema evolution** allows deliberate additions. Together, these mechanisms give Delta Lake the ACID properties of a database on infrastructure that supports only simple file writes.

How a reader turns a version into a Parquet file list—and how `UPDATE` / `DELETE` / `MERGE` rewrite that list—are the next stories: [Which Files Am I Reading? How Delta Resolves a Snapshot](delta_snapshot_read_path.md) and [Rewriting Reality: How UPDATE, DELETE, and MERGE Change a Delta Table](delta_dml_copy_on_write.md). Format features, housekeeping, and streaming continue in [Beyond Copy-on-Write](delta_format_features.md), [OPTIMIZE, Z-Order, and VACUUM](delta_optimize_and_vacuum.md), and [Delta in Motion](delta_streaming.md).
