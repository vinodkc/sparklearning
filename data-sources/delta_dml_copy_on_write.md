# Rewriting Reality: How UPDATE, DELETE, and MERGE Change a Delta Table

This is the story of how Delta Lake applies row-level changes on top of immutable Parquet files. Object storage does not edit a row in the middle of a columnar file. Delta's classic answer is **copy-on-write**: find the files that contain matching rows, write new Parquet files with those rows changed or omitted, and commit `Remove` + `Add` actions in one transaction. Understanding that path explains why a "small" `UPDATE` can rewrite huge amounts of data, how `MERGE` becomes a planning problem as much as a SQL problem, and where newer **deletion vector** techniques change the trade-off—without abandoning the log.

---

## Immutable files, mutable tables

Parquet files, once written, are treated as immutable. You do not punch a hole in a row group to flip one status flag. A Delta table nonetheless supports `UPDATE`, `DELETE`, and `MERGE` because **the table is the log's opinion about which files count**, not the bytes inside every object forever.

A row-level change therefore becomes:

1. **Find** which data files might contain target rows (using partition filters and file stats—same skipping ideas as a read)
2. **Read** those candidate files
3. **Write** new file(s) with the desired row set
4. **Commit** atomically: `Remove` the old file(s), `Add` the new one(s)

Readers at the new snapshot see only the new files. Readers on older snapshots still see the old files until retention/vacuum says otherwise.

> **Copy-on-write is like editing a printed encyclopedia by reprinting whole volumes.** You cannot erase one sentence from a bound book on the shelf. You print a replacement volume, swap the catalog card (the log), and leave the old volume in storage until the library discards it. One wrong word can force a thick reprint if that word sat in a fat volume.

---

## DELETE: omit rows, replace files

A `DELETE` with a predicate identifies files that **might** hold matching rows. Files whose stats prove no match are left untouched—data skipping saves rewrite work. For each candidate file, Delta reads it, drops matching rows, and writes a new Parquet file with survivors (or writes nothing if a file is entirely deleted). The commit removes the old paths and adds the new ones.

Delete cost tracks **how many files the predicate touches**, not how many rows match. One row in a 1 GB file can force a 1 GB rewrite. Predicates that hit almost every file turn a delete into a table-wide rewrite.

---

## UPDATE: rewrite with new values

`UPDATE` is the same engine with a transformation: matching rows are emitted with new column values; non-matching rows pass through. Again, whole files that might contain matches are rewritten. Wide rows, many columns, and poor file skipping amplify write amplification—the ratio of bytes rewritten to bytes logically changed.

That amplification is why layout matters for DML as much as for reads: clustering on the columns you update or delete by keeps candidate file sets small.

---

## MERGE: find, change, insert in one transaction

`MERGE INTO` (upsert) combines matched and unmatched actions—update or delete on match, insert on not-matched, and variants with extra conditions. Under the hood it is still a transactional rewrite plan:

- Scan target files that could match the merge source keys (skipping what stats allow)
- Join or match logic between source and target rows
- Produce replacement target files plus new files for inserts
- Commit one version that describes all `Remove`/`Add` effects

`MERGE` is where planning quality shows. A merge source that is large and poorly filtered against an unpartitioned target can pull enormous fractions of the table into the rewrite. A merge that touches a single partition with tight key locality rewrites little. The SQL looks like a row API; the physical reality is **file-set surgery**.

> **MERGE is a controlled swap of warehouse pallets plus a delivery of new ones, logged as a single shipment receipt.** Matched pallets get replaced with restickered goods; new SKUs arrive on new pallets; the receipt (commit) makes the dock's inventory consistent in one step. Fail before the receipt and the dock still shows yesterday's inventory—even if temporary pallets were staged in the yard.

---

## Optimistic concurrency meets DML

DML commits use the same **optimistic concurrency** rules as appends: stage data files, attempt the next log version, detect conflicts. Conflicts are more likely when two writers rewrite overlapping file sets—two jobs updating different rows that happen to live in the same Parquet files can still collide because both try to `Remove` the same path.

Retries may rebase if safe; true conflicts fail. Designing partitions and file sizes is also designing **conflict granularity**: coarser files mean more conflict surface per row change.

---

## Write amplification and practical consequences

Copy-on-write DML creates predictable operational patterns:

- **Small changes, large I/O** when files are huge or predicates are unselective
- **Many small files** when frequent merges rewrite slices poorly and compaction lags
- **Longer commits** under contention on hot file sets
- **Storage growth** until **VACUUM** removes physically obsolete files outside the retention window

Append-only pipelines avoid most of this; CDC-style `MERGE` pipelines live in it. Compaction (`OPTIMIZE` and kin) periodically rewrites small or poorly laid-out files into larger ones so both reads and future DML skip better—maintenance as layout repair.

---

## Deletion vectors: a different trade (preview)

Newer Delta capabilities can record **deletes as separate bitmaps** (deletion vectors) instead of immediately rewriting every affected Parquet file. Readers honor the vectors to hide rows; later compaction materializes clean files. That shifts cost from "rewrite now on every DELETE" toward "pay more on read until compacted"—merge-on-read flavor on top of the same transaction log.

This story's mental model remains copy-on-write because it is still the clearest way to see why DML equals file replacement. Deletion vectors, column mapping, and liquid clustering deepen the format without changing the rule: **the log commits the new truth; data files are evidence the log points at.**

---

## How this ties to the rest of the lake

- [The Transaction Log](delta_lake_transaction_log.md) — the commit that publishes `Remove`/`Add`
- [Which Files Am I Reading?](delta_snapshot_read_path.md) — readers after DML resolve a new snapshot's file set
- [Writing Parquet](writing_parquet_with_spark.md) — replacement files are still ordinary Parquet writes
- [Parquet layout](parquet_layout_and_pruning.md) — clustering decides how wide a DELETE spreads

Streaming upserts into Delta are the same DML/commit machinery driven by micro-batches—see [Delta in Motion: Streaming Reads and Exactly-Once Sinks](delta_streaming.md). Deletion vectors and related format features are covered in [Beyond Copy-on-Write: Deletion Vectors, Column Mapping, and Liquid Clustering](delta_format_features.md); file tidying is [Housekeeping the Lake: OPTIMIZE, Z-Order, and VACUUM](delta_optimize_and_vacuum.md).

---

## Bringing it together

Delta row changes do not edit Parquet in place. **DELETE**, **UPDATE**, and **MERGE** identify candidate files, rewrite them (and add inserts), and commit **`Remove` + `Add`** under optimistic concurrency—**copy-on-write** at file grain. Cost follows **files touched**, not rows changed; layout and predicates decide amplification. Concurrent DML collides when writers claim the same files. Newer deletion-vector paths delay rewrite, but the table's truth remains whatever the transaction log says the active files—and any delete annotations—are. Once you see DML as transactional file replacement, "why did this one-line UPDATE scan and rewrite half the lake?" stops being a mystery.
