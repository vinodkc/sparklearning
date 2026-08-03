# Folders, Filters, and Skipping: How Layout Makes or Breaks Parquet Scans

This is the story of how data is *arranged* on disk—across directories and inside Parquet files—and why that arrangement decides whether a filter skips almost everything or reads almost everything. Parquet's footer statistics and Spark's partition pruning are powerful, but they only help when the physical layout matches the way you query. Understanding layout explains why `WHERE date = '2024-06-01'` can open one folder and three row groups—or scan a petabyte of unrelated files with useless min/max ranges.

---

## Two layers of layout

Analytical lakes usually combine two layouts:

**Directory (Hive-style) partitioning.** Paths like `/events/date=2024-06-01/country=US/` mean every file under that folder only contains rows for that date and country. The partition columns often do not even need to be stored inside every row; they are implied by the path. Spark can **prune partitions**: if the query says `date = '2024-06-01'`, directories for other dates are never listed as candidates.

**In-file layout.** Inside each Parquet file, rows sit in **row groups** with per-column **min/max** (and optional bloom filters). Spark can **skip row groups** whose statistics prove no row can match a filter—even within a directory that was not pruned away.

> **Directories are the warehouse aisles; row groups are the labeled pallets in an aisle.** Partition pruning chooses which aisles to walk. Predicate pushdown chooses which pallets to open. A great aisle plan with randomly piled pallets still wastes time; perfectly sorted pallets in one giant unsorted aisle do too. You want both.

These layers are complementary. Partitioning answers "which files exist for this key?" Row-group stats answer "within this file, which horizontal slices can I ignore?"

---

## Partition pruning: skip whole directories

When a table is partitioned by `date`, a query filtered on `date` should touch only matching folders. Catalyst plans this as **partition pruning**: the file index or directory listing is narrowed before tasks read Parquet footers for irrelevant days.

That win scales with partition selectivity. One day out of three years of daily partitions is a tiny fraction of files. The same query on an **unpartitioned** directory of flat Parquet files must discover and open far more files, then hope in-file statistics can skip row groups—if the data was clustered by date at all.

Partition pruning fails or weakens when:

- The filter is not on a partition column (you partitioned by `date` but filter only on `user_id`).
- The filter is hidden behind a cast or expression Spark cannot match to the partition value.
- Partition columns were chosen poorly so almost every query still hits most folders.

> **Partition pruning is like a library sorted by publication year in separate rooms.** Asking for "2020" sends you to one room. Asking for "books by a specific author" with no author rooms means you still visit every year-room—unless something inside each book helps you skip chapters.

---

## Row-group skipping: skip slices inside a file

Once Spark opens a file (or schedules a split), the **footer** lists row groups and column statistics. For `amount > 1000`, a row group whose max `amount` is 50 can be skipped entirely. That is **predicate pushdown** into the Parquet reader—covered in the anatomy story—and it is only as good as the **value locality** inside the file.

If rows are inserted in random order by `amount`, every row group's min/max for `amount` may span nearly the full domain. Statistics say "might match"; almost nothing is skipped. If rows are **sorted or clustered** by `amount` (or by `date` when you filter on dates), row groups form tight ranges and filters become surgical.

Bloom filters help **point lookups** on columns that are not sorted: they can rule out row groups that definitely do not contain a key. **Page indexes** refine range skips inside a row group when the writer stored them. Neither replaces clustering for selective ranges, and neither replaces directory pruning for coarse keys like date.

---

## When partitioning helps—and when it hurts

**Good partition columns** are common filter dimensions with **moderate cardinality**: date, hour (carefully), region, tenant. They align with how people query and produce folders large enough to hold sensible file sizes.

**Over-partitioning** is the trap. Partitioning by `user_id` or `event_id` can create millions of tiny directories, each with one tiny file. Listing the table becomes expensive; the small-file problem returns wearing a hive-partition costume. Partitioning by several high-cardinality columns multiplies the explosion (`date` × `country` × `device` × `campaign` …).

**Under-partitioning** leaves huge folders. A single `year=` partition for a hot table may still force readers to open many files and rely entirely on in-file stats. That can be fine if files are well clustered; it is painful if they are not.

Practical design often uses **coarse directory partitions** (e.g. date) plus **in-file clustering/sort** on secondary filter columns—so pruning removes most days and statistics remove most row groups within the day.

> **Choosing partitions is like deciding how many rooms the warehouse gets.** One room for the whole company and you walk forever. A room per paperclip and you spend the day reading door signs. A room per shipping day, with sorted pallets inside, is usually the shape analytics wants.

---

## Clustering and sorting: making statistics tell the truth

Writers control whether row-group stats are useful. Sorting by a filter column before write—or repartitioning so values that belong together land in the same files—tightens min/max ranges. Unsorted append-only dumps maximize write simplicity and minimize skip efficiency.

This is why "we have Parquet, so filters should be free" is false. **Parquet enables skipping; layout earns it.** Two tables with identical schemas and identical compression can differ by orders of magnitude in bytes read for the same SQL, solely because one is laid out for the predicate and the other is not.

Related Spark ideas: range repartition before write to produce ordered files; avoiding unnecessary wide shuffles that randomize a carefully ordered stream; measuring in the UI whether scans show large "files read / rows skipped" gaps.

---

## How Spark combines both at read time

A typical selective scan looks like this:

1. **Resolve paths** for the table or glob.
2. **Partition prune** — drop directories that cannot match partition filters.
3. **Plan files / splits** — remaining files become tasks (often aligned with files or row groups).
4. **Read footers** — load row-group and column statistics (and bloom filters if present).
5. **Skip row groups** that cannot match data filters; **project** only needed column chunks.
6. **Decode** surviving pages into vectorized batches.

If step 2 fails (bad partition design or non-partition filters), step 5 must carry the load. If step 5 fails (random layout), you decode far more than the result set needs—even with perfect projection pushdown on columns.

Directory partitioning and in-file layout also interact with **writes**: partition-aware writes create the folder tree; coalesce/repartition before write set file sizes *within* each partition folder. A well-partitioned table full of 1 KB files per day folder is still a small-file lake.

---

## What this story deliberately leaves alone

Table formats build richer skipping on top of these ideas—data skipping indexes, clustering policies, compaction that rewrites layout without changing SQL. Those mechanisms still bottom out in **which Parquet files exist** and **how rows are arranged inside them**. Delta picks the file set in [Which Files Am I Reading? How Delta Resolves a Snapshot](delta_snapshot_read_path.md); this story is what each surviving file costs to scan.

---

## Bringing it together

Parquet scan performance is a **layout** story, not only a format story. **Hive-style directories** let Spark prune whole subtrees of files when filters hit partition columns. **Row-group statistics and bloom filters** let Spark skip horizontal slices inside the files that remain—but only when values are clustered enough for min/max (or blooms) to be selective. Over-partitioning creates tiny-folder chaos; under-partitioning and random in-file order push all the work onto brute-force reads. Design coarse partitions for common filters, write sensible file sizes into those folders, and cluster by the columns you filter on. Then projection and predicate pushdown—described in the anatomy story—finally deliver the I/O savings Parquet is famous for.

For how Spark turns that layout into tasks and batches, see [From Footer to Batches: How Spark Reads Parquet](spark_parquet_read_path.md). For how column definitions drift across files, see [Columns That Change: Schema, Nested Types, and Evolution in Parquet](parquet_schema_and_evolution.md).
