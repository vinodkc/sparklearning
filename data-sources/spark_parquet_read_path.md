# From Footer to Batches: How Spark Reads Parquet

This is the story of what happens between `spark.read.parquet(...)` (or a table scan) and the first rows flowing through a query. Anatomy describes what is inside a file; writing and layout describe how files got that way; schema describes what the columns mean. This story follows the **read path**: how Spark discovers files, negotiates pushdowns, opens footers, skips what it can, and turns surviving column pages into vectorized batches executors can process. Understanding that path explains Spark UI scan metrics, why planning sometimes dominates runtime, and when a filter in SQL never becomes a skip on disk.

---

## From logical scan to file tasks

A Parquet read starts in **Catalyst**. The logical plan contains a relation or scan over paths / a table. After analysis and optimization, filters and projections are pushed as close to the scan as possible. Through **DataSource V2** (or the file-source path that implements the same ideas), Spark **negotiates** with the Parquet connector: which columns are needed, which filters the reader can accept, and what the residual filters will still apply in the engine.

Planning then builds a set of **input splits**—work units for tasks. For Parquet, splits often align with **files** or **row groups** inside large files, so parallelism tracks how data was written. A thousand tiny files mean a thousand little tasks (or heavy combining); a few well-sized files with multiple row groups can still parallelize if splitting allows it.

> **Planning a Parquet read is like assigning warehouse pickers before anyone touches a crate.** First you decide which aisles and pallets matter (partitions and filters), then you hand each picker a clipboard of locations (splits). Bad clipboards—every micro-file as its own stop—mean pickers spend the day walking between doors.

---

## Partition pruning before footers

If the dataset is **Hive-style partitioned**, Spark applies **partition pruning** while enumerating paths: directories that cannot match partition filters are never part of the read set. That step is pure metadata—no Parquet footer required—and it is often the largest win for date-scoped queries.

Only the surviving files proceed to footer-aware work. Layout design (coarse partitions, sensible folder cardinality) decides how small that surviving set is. The layout story covers *why*; here the read path simply **consumes** that decision as a shorter file list.

---

## Footers: the map before the journey

For files that remain, Spark reads the **Parquet footer** (and may cache footer metadata across tasks or jobs depending on configuration and source). The footer supplies:

- Schema and column paths for this file
- Row-group offsets and sizes
- Per-column chunk statistics (min, max, null counts)
- Optional bloom filter references

With footers, the reader can **skip row groups** whose statistics contradict pushed filters, optionally use **page indexes** and **bloom filters** when present, and **seek only the column chunks** (and pages) required by projection. A selective query on a well-laid-out file may read a thin slice of bytes relative to file length; a query with no usable pushed filter may still project fewer columns but must visit every relevant row group's selected chunks.

Footer read cost scales with **file count**. Lakes with enormous numbers of small files pay a planning and open tax before vectorized decoding ever runs—the write story's small-file problem showing up on the read side as "jobs stuck at 0% with lots of driver or scanner activity."

---

## Decoding into vectorized batches

After skipping what it can, the reader decompresses and decodes **pages** into memory. Spark's **vectorized Parquet reader** assembles **column batches** (often thousands of rows at a time) rather than building one row object at a time. Batches flow into whole-stage codegen operators: filters that were not fully applied at skip time run on dense column vectors; projections are already narrow.

This is where encoding choices from write time matter: dictionary pages decode into compact IDs; plain pages carry full values; nested definition/repetition levels reconstruct arrays and structs. CPU time in "scan" is often decode + filter, not only disk or S3 wait.

**When vectorization falls back.** If the scan includes types or layouts the vectorized path does not handle (certain nested columns, some decimals, legacy INT96 timestamps, and similar cases), Spark switches those reads to the older row-at-a-time path. The job still succeeds; it just burns more CPU per row and benefits less from batch filters. Wide nested event payloads are a common reason a "Parquet scan" feels surprisingly heavy despite good projection.

Large files are split using size-based planning (Spark's max-partition-bytes style settings): a multi-gigabyte Parquet file can become several tasks, each owning a byte range that typically aligns with row-group boundaries when possible—reinforcing that parallelism is about **splits**, not strictly "one row group = one task."

> **Vectorized reading is like unloading a truck by the pallet, not by the single SKU in your hand.** You stage a block of values, run the check (`amount > 1000`) across the block, and pass survivors up the line. Fallback is unloading piece by piece when the cargo won't fit the pallet jack. The crate size was decided when the file was written; the unload method is the reader mode.

---

## What residual work stays in the engine

Not every SQL filter becomes a row-group skip. Common cases:

- The connector **rejects** a filter during pushdown negotiation (non-deterministic expressions, certain casts, complex predicates).
- Statistics **cannot prove** a skip (unsorted data, wide min/max ranges)—the row group is read; **page indexes** may still skip some pages when present, otherwise rows are filtered in Spark.
- The scan fell back to **non-vectorized** decode for some columns—bytes may still be pruned, but CPU per row climbs.
- Partition filters were not on partition columns—no directory prune; only in-file mechanisms apply.

So a physical plan that *shows* a filter near a scan does not guarantee minimal I/O. Confirm with metrics: bytes read, rows scanned vs rows output, number of files.

---

## What the Spark UI is telling you

On the SQL / stage view, Parquet scans surface clues:

- **Number of files / partitions** — hints at small-file or over-partition pain
- **Files pruned / row groups skipped** (when exposed) — layout and stats effectiveness
- **Bytes read vs table size** — projection and skip success
- **Scan time vs shuffle time** — whether the bottleneck is still storage decode or already elsewhere

If scan time dominates and bytes read are huge despite selective `WHERE` clauses, suspect layout, type casts killing pushdown, or schema mismatch. If planning or task launch dominates with tiny peak throughput, suspect file count. The UI does not replace the mental model—but it confirms which chapter of the Parquet stories you are living in.

---

## How the Parquet stories connect on a read

1. **Schema** — unify what columns exist and which types to decode  
2. **Partition layout** — drop irrelevant directories  
3. **Footer + statistics** — drop irrelevant row groups; open needed column chunks  
4. **Vectorized decode** — turn pages into batches  
5. **Residual filters / upstream operators** — finish what I/O could not skip  

Table formats (Delta and friends) insert a step **before** (1)–(2): the transaction log chooses the **active file set** for a snapshot. After that set is known, this same Parquet read path applies file by file. That handoff is [Which Files Am I Reading? How Delta Resolves a Snapshot](delta_snapshot_read_path.md).

---

## Bringing it together

Spark reads Parquet by **planning a scan**, **pruning partitions**, **reading footers**, **skipping row groups**, **projecting column chunks**, and **decoding vectorized batches** into the execution engine. Pushdowns and layout decide how much work disappears before decode; schema and expression shape decide whether pushdowns attach at all. UI metrics—files touched, bytes read, scan time—are the read path made visible. When those numbers look wrong, the fix is rarely "Parquet is slow"; it is usually write shape, directory layout, or schema drift showing up at the moment the footer meets the filter.
