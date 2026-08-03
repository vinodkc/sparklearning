# From DataFrame to Disk: How Spark Writes a Parquet File

This is the story of how Spark turns an in-memory DataFrame into Parquet files on storage—and why that write path is where many lake tables quietly go wrong. Reading Parquet well depends on what the writer put on disk: how many files, how large each file is, how big the row groups are, which compression codec was used, and whether dictionaries were built. Understanding the write side explains the small-file problem, why a job that "just writes" can leave a directory that is painful to scan later, and which knobs actually matter before you hit save.

---

## One task, one (or a few) files

When Spark writes Parquet without directory partitioning, each **output task** typically produces **one file**. The number of output tasks is driven by the number of **partitions** in the DataFrame at write time. A DataFrame with 200 partitions writing with `df.write.parquet(...)` tends to create about 200 Parquet files (plus a `_SUCCESS` marker in some setups). A streaming micro-batch with 50 partitions tends to append about 50 new files every trigger. With **`partitionBy`**, a single task may write **one file per partition-column value** it sees—so file count is task count times distinct partition keys in that task, not simply one file per task.

That one-to-one relationship is the root of the small-file problem. If a filter or a frequent micro-batch leaves you with hundreds of partitions that each hold only a few megabytes—or a few kilobytes—of data, you get hundreds of tiny Parquet files. Each tiny file still has a footer, still needs to be listed and opened on read, and still costs a task (or part of a task) later. The data volume may be small; the file count is not.

> **Writing Parquet is like packing an order into shipping boxes on an assembly line.** Each worker (task) packs their own box (file). If you staff 500 workers for an order that only fills 500 tiny boxes, you get 500 packages to label, ship, and open later—even though the goods would have fit in five. The packing step decides how painful receiving will be.

---

## What goes into each file as it is written

Inside each file, the writer fills **row groups** up to a target size (commonly on the order of 128 MB of buffered data, depending on Spark and Parquet settings), then starts another row group. Within each row group, values are organized into **column chunks**, encoded, and compressed into **pages**—the same structure described in the Parquet anatomy story.

Several decisions happen at write time and then freeze into the file:

**Compression codec.** Snappy is a common default: fast to compress and decompress, decent size reduction. ZSTD (or gzip in older stacks) often shrinks further at higher CPU cost. The codec is a trade-off between write CPU, read CPU, and bytes on disk or over the network to object storage.

**Dictionary encoding.** For low-cardinality columns, the writer builds a dictionary and stores indices instead of raw values. If a column's cardinality explodes inside a row group, the writer may fall back to plain encoding for that column. Sorted or clustered data often compresses and dictionary-encodes better because repeated values sit together.

**Page and row-group sizing.** Larger row groups mean fewer row groups per file and sometimes better compression, but a task must buffer more data while writing, and readers that cannot skip a row group may pull a larger chunk. Very small row groups increase footer and metadata overhead relative to useful data.

> **Think of each file as a crate packed on the line.** Row group size is how full each compartment gets before you close it. Compression is how tightly you shrink the packing material. Dictionary encoding is labeling repeated items with a short code instead of restamping the full name on every piece. Pack well once; every future reader benefits.

---

## Too many small files

The classic failure mode: a pipeline writes often, from many partitions, with little data per partition.

**Batch symptom.** You repartition to 1,000 for a heavy transform, then write immediately. You get ~1,000 files even if the output is only a few gigabytes—or worse, after a selective filter, a few hundred megabytes.

**Streaming symptom.** Every micro-batch appends one file per partition. Hourly triggers with 200 partitions create thousands of files per day. Readers then spend more time listing and opening files than decoding useful column chunks.

Small files hurt readers in several ways: more footer reads, more task scheduling overhead, worse throughput against object stores that charge per request, and weaker opportunities to amortize decompression. The lake looks "correct"—the data is there—but every scan pays a tax proportional to file count.

**Remedies on the write path** are about controlling *how many* output partitions hold *how much* data before write:

- **`coalesce(n)`** reduces partition count without a full shuffle—good when you already filtered down and just need fewer, larger output files.
- **`repartition(n)`** reshuffles to exactly `n` roughly even partitions—better when you need balanced file sizes and are willing to pay for a shuffle.
- Target file sizes in a practical band (often tens to a few hundreds of MB per file for analytics tables), not "one file per core you happened to use upstream."

> **Small files are like mailing a novel one page per envelope.** Correct content, absurd logistics. Coalesce or repartition before write is choosing fewer envelopes so the post office—and tomorrow's reader—can cope.

---

## Too few, oversized files

The opposite failure mode is rarer in streaming but common after an aggressive coalesce to 1 or a tiny `n`.

One enormous file (or a handful of multi‑GB files) means:

- Less read parallelism: fewer tasks can scan the table at once if splits cannot be subdivided enough.
- Heavier write memory pressure: a single task buffers a huge row group or a huge output stream.
- Painful retries: if that task fails near the end, a large write may be redone.

Row groups inside a large file still allow some parallelism and skipping, but extremely unbalanced output—one giant file next to many tiny ones—leaves the job as slow as its largest scan tasks. Aim for **even, moderate-sized files**, not a single monolithic dump.

---

## Repartition and coalesce as write tools

Upstream partition count is an accident of the plan (shuffle partitions, Kafka partitions, previous reads). **Write-time partition count should be a deliberate choice.**

Use **coalesce** when the data is already roughly where you want it and you only need to merge small partitions before landing files—cheap, no full reshuffle, possibly uneven.

Use **repartition** when file count and evenness matter more than avoiding a shuffle—for example, landing a daily table that will be queried heavily and should not be a carpet of 2 KB objects.

Use **repartition by columns** (including range-style repartition where appropriate) when you also want values that will be filtered together to land in the same files—this sets up the layout story: sorting and clustering make in-file min/max statistics useful.

None of these replace table formats that compact files in the background; they are what you control on a raw Parquet write. Compaction later is cleanup. Sensible writes are prevention.

---

## Knobs that matter (without a config dump)

A few settings shape the files more than the rest:

- **Compression codec** — balances CPU vs size for every reader of those files.
- **Row group / block size** — how much data a writer accumulates per row group; influences skip granularity and writer memory.
- **Page size** — granularity of compression units inside a column chunk.
- **Dictionary encoding enablement** — usually leave on for analytics; high-cardinality noise columns may not benefit.
- **Bloom filters / page indexes** — optional write-time metadata that makes point and range skips sharper at read time; readers cannot invent them after the fact.

Treat these as **table-level design choices**, not something to flip randomly per job. Changing codec or row-group targets mid-lake creates a mix of file shapes that all share one directory but not one performance profile.

For the full map of how Spark configuration reaches components, [SparkConf to Code](../config/sparkconf_configuration.md) covers precedence; here the point is simpler: **the write configures the physical shape that every future scan inherits.** Bloom filters and similar optional metadata exist in files only when the **writer** enabled them—readers cannot invent skips the write never recorded.

---

## Bringing it together

Spark writes Parquet by turning **output partitions into files**, and packing each file with **row groups, encodings, and a compression codec**. That packing step decides whether readers enjoy large, skip-friendly columnar scans or drown in tiny footers and undersized tasks. Too many partitions at write time produce the **small-file problem**; too few produce **oversized, under-parallel** outputs. **Coalesce** and **repartition** are the practical tools to set file count and balance before save; compression and dictionary settings decide how dense each file becomes. Once the bytes are on disk, the anatomy of those files—and how layout and pruning use them—determines how cheap tomorrow's query will be.

Continue with [Folders, Filters, and Skipping: How Layout Makes or Breaks Parquet Scans](parquet_layout_and_pruning.md) for directory vs in-file skipping, then [From Footer to Batches: How Spark Reads Parquet](spark_parquet_read_path.md) for how those files are scanned. Frequent small commits in Delta streaming recreate this problem transactionally—see [OPTIMIZE & VACUUM](delta_optimize_and_vacuum.md) and [Delta streaming](delta_streaming.md).
