# Housekeeping the Lake: OPTIMIZE, Z-Order, and VACUUM

This is the story of how a Delta table stays usable after months of appends, merges, and deletes. The transaction log records every version; it does not automatically tidy the physical mess those versions leave behind—too many tiny files, poorly clustered keys, data files that are logically gone but still occupying storage, deletion vectors that deserve materialization. **OPTIMIZE**, **Z-Order** (and clustering), and **VACUUM** are the housekeeping verbs. Understanding them explains why streaming lakes rot into small-file sludge, why skipping suddenly improves after a weekend job, and why vacuuming too eagerly breaks time travel.

---

## Why tables need maintenance

Healthy commits optimize for **correctness and latency of the writer**, not for eternal read shape.

- Micro-batches and concurrent writers create **many small Parquet files**
- Copy-on-write DML leaves **replaced files** on disk until retention expires
- Deletion vectors leave **logically deleted rows** physically present
- Random ingest order weakens **file-level stats** even when the logical model is fine

Readers pay: more footers, worse skipping, longer listing of log/checkpoint state, higher cloud request costs. Maintenance rewrites or deletes physical artifacts **while committing new log versions** that describe the tidier state—or, for vacuum, removing objects the log no longer needs within policy.

> **A Delta table without housekeeping is like a workshop that never throws away cut-offs or sorts the scrap bin.** Every project still finishes (commits succeed). Finding a clean board later takes forever. OPTIMIZE sorts and glues usable stock into full-length boards; VACUUM hauls scrap past the retention date to the dumpster.

---

## OPTIMIZE: compaction into fewer, fuller files

**OPTIMIZE** (compaction) reads groups of small files and writes fewer larger Parquet files, then commits `Remove` on the small ones and `Add` on the new. The logical table content for the latest snapshot stays the same; the **file grain** changes.

Effects:

- Fewer files → fewer tasks/footer opens on the next snapshot read
- More even sizes → steadier scan parallelism
- Opportunity to **materialize** cleaner files (including applying deletion vectors into rows that simply disappear)

Compaction is itself a write-heavy transaction. It competes under optimistic concurrency with other writers touching the same files. Running it during quiet windows, or on partitions that are "closed" to ingest, reduces conflict pain.

OPTIMIZE does not invent selectivity by itself—it mainly fixes **file count and size**. Clustering/Z-Order during optimize is what reshapes **which values share a file**.

---

## Z-Order: clustering for data skipping

**Z-Order** (multi-dimensional clustering during a rewrite) colocates related values across one or more columns so that file-level min/max ranges become tighter for common filters. After a Z-Order optimize on `user_id` and `event_date`, a query for one user on one day should skip far more files than after random append layout.

Z-Order is not magic indexing:

- It costs a **large rewrite** of the scoped data (often a partition or whole table)
- It helps predicates on the z-ordered columns; unrelated filters gain little
- Benefits decay as **new unsorted files** append on top—hence periodic re-optimize, or liquid clustering policies that keep reshaping incrementally

Liquid clustering (format-features story) aims at the same outcome—locality for skipping—with less "big bang weekend Z-Order" dependence. Z-Order remains the mental model for "rewrite so stats tell the truth."

> **Z-Order is re-shelving the library so books on the same topic sit near each other.** You still walk the stacks (scan files), but whole shelves are irrelevant at a glance (data skipping). Leave new returns in a random pile by the door (unsorted appends) and the neat shelves slowly stop representing reality.

---

## VACUUM: deleting physically obsolete files

`Remove` in the log makes a file invisible to new snapshots; **VACUUM** deletes orphaned **data** files from storage after a **data retention** interval. Default retention for data files is deliberately long—commonly **7 days** (`delta.deletedFileRetentionDuration`), not minutes—so that:

- Time travel to recent versions still finds the bytes it needs
- Concurrent readers with slightly stale snapshots are less likely to hit missing objects
- You have a safety margin against clock skew and long-running jobs

Vacuum too aggressively and you **break time travel** and risk failures for readers still on old snapshots. Vacuum never (when used correctly) deletes files still referenced by the current snapshot within retention rules—its job is garbage collection of *logically dead* objects, not shrinking the live table.

**Log retention is a separate dial.** Old JSON commits and obsolete checkpoints are cleaned under **log retention** (commonly around 30 days via `delta.logRetentionDuration`), independent of how long removed Parquet files linger for VACUUM. You can imagine needing the log long enough to time-travel *metadata* and needing data files long enough to time-travel *bytes*. Shorten either without understanding the other and you get "version exists in conversation but cannot be reconstructed" failures.

> **Data retention is how long scrap wood stays in the yard after a rebuild; log retention is how long you keep the signed work orders.** Empty the yard too soon and yesterday's blueprint still says a beam should be there. Shred the work orders too soon and you cannot prove which beams were ever approved—even if wood remains.

The headline lesson for operators: **history is a storage and compliance choice**, not free—and it has **two clocks**, not one.

---

## How maintenance interacts with DML and streaming

- After heavy `MERGE`/`DELETE`, optimize recovers from small or fragmented files and DV buildup
- Streaming sinks that commit every batch **need** scheduled compaction or they recreate the small-file problem the Parquet write story warned about—only now with a transaction log on top
- Clustering keys should match **filters and merge keys**, or you pay rewrite cost without skip benefit

Maintenance jobs are part of the table's contract with production, not optional polish.

---

## Bringing it together

Delta housekeeping is three complementary acts. **OPTIMIZE** rewrites many small or dirty data files into fewer healthier ones for the latest snapshot. **Z-Order / clustering** uses those rewrites to colocate filter keys so **data skipping** works. **VACUUM** removes storage objects that the log has long since abandoned, bounded by **data** retention, while **log retention** separately expires old commits and checkpoints—two clocks for reconstructible history. Skip maintenance and the log remains correct while every scan gets slower and every cloud bill gets chatty; schedule it and the snapshot-read and Parquet paths see the lake you meant to build.
