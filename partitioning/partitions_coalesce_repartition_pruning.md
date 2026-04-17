# Partitions: The Grain of Parallelism

This is the story of how Spark divides its work. Every dataset in Spark—every RDD, every DataFrame, every table—is split into **partitions**, and every task processes exactly one partition. Partitions are the grain at which parallelism happens: more partitions means more potential tasks running in parallel; fewer partitions means less overhead but less parallelism. Understanding partitions—how many you have, how data is distributed across them, how to change that distribution, and how the query planner can skip partitions entirely—is one of the most practical skills in Spark tuning.

---

## What a partition is

A partition is a contiguous, independent chunk of data. For an RDD built from an HDFS file, a partition usually corresponds to one HDFS block (typically 128 MB). For a Parquet table, a partition might correspond to one Parquet file or one row group. For an in-memory RDD created from a collection, partitions are determined by how many slices you specify. For the result of a shuffle, partitions correspond to one reduce bucket—determined by the number of shuffle partitions (`spark.sql.shuffle.partitions`, default 200).

Each partition is processed by one task, on one executor core, at a time. That's the fundamental constraint: partitions and tasks are one-to-one. If you have 1000 partitions and 100 executor cores, Spark runs 100 tasks at once and processes all 1000 partitions in 10 waves. If you have 100 partitions and 100 cores, one wave—each core gets one task and you're done. If you have 10 partitions and 100 cores, 90 cores sit idle.

---

## The partition count problem: too few vs. too many

Partition count is a dial with costs on both sides.

**Too few partitions** means under-utilization. If you have 2 partitions and 200 executor cores, 198 cores are idle while 2 tasks run. A single large partition also means a single task must process all that data, which can be slow and memory-hungry. Skewed partitions—where a few partitions are much larger than others—are a variant of the same problem: the job is as slow as its slowest partition.

**Too many partitions** means overhead. Spark must schedule, serialize, launch, and track each task. If each task processes only a few kilobytes of data, the overhead of launching the task dominates its runtime. Thousands of tiny tasks also stress the driver's scheduler and can create many small shuffle files, which are expensive to open and read.

A common rule of thumb: aim for partitions that are between 100 MB and a few hundred MB each when processing large data. For shuffles, `spark.sql.shuffle.partitions = 200` is the default, but for a job processing 1 GB you might want 10 partitions and for a job processing 10 TB you might want 2000.

---

## coalesce: reducing partitions without a shuffle

**`coalesce(n)`** reduces the number of partitions to `n`. Its key property is that it does this **without a full shuffle** when `n` is smaller than the current partition count. Instead of redistributing data across the network, coalesce merges existing partitions by assigning multiple old partitions to single new partitions. Tasks for the coalesced stage read from multiple input partitions locally—no network transfer.

The tradeoff: because coalesce doesn't redistribute data, the new partitions may be uneven. If you coalesce 100 partitions down to 10, each new partition reads 10 old partitions, but those 10 old partitions might not be the same size. Coalesce is best used after a filter that has significantly reduced the data and you want to consolidate the remaining small partitions—for example, before writing to storage to avoid creating many tiny files. It is not appropriate when you need evenly-sized partitions or when you're increasing the partition count.

Coalesce cannot increase the partition count. Asking for more partitions than you currently have with `coalesce` is a no-op.

---

## repartition: full redistribution via shuffle

**`repartition(n)`** changes the partition count to exactly `n` by performing a **full shuffle**: all data is hashed (or round-robined) and redistributed across the network to `n` new partitions. The result is as even as the hash function can make it—each partition gets roughly `total_rows / n` rows.

Repartition is the right tool when you need a specific, even distribution: before a join whose other side has a different partition count, before a sort that must produce a fixed number of output files, or when increasing the number of partitions to use more parallelism. It is also the only way to increase partition count (coalesce can't).

The cost is a full shuffle: all your data crosses the network (or at least gets sorted and reassigned). This is equivalent to the cost of a `groupBy` or `join`—a stage boundary, disk and network I/O for all rows. Don't repartition unless you need to, and don't repartition more than once in a pipeline.

---

## repartitionByRange: for sorted output

**`repartitionByRange(n, col)`** is a variant that shuffles data and assigns rows to partitions such that partition 0 has the smallest values of `col`, partition 1 has the next range, and so on. This produces **range-partitioned** data: the output partitions are ordered by key and contain contiguous ranges of values.

Range-partitioned data is useful when writing a table that will be queried with range filters on the partition key—the data is physically sorted, so a query for `col BETWEEN 100 AND 200` only needs to scan the relevant partitions. It's also useful as input to a merge-sort: if you then sort within each partition, the whole dataset is globally sorted. The cost is the same as `repartition`: a full shuffle plus the overhead of sampling to determine range boundaries.

---

## Partitioning by key: the HashPartitioner and RangePartitioner

When a shuffle happens (for `groupByKey`, `reduceByKey`, `join`, etc.), Spark uses a **partitioner** to decide which partition each record belongs to. The default is the **HashPartitioner**: it computes `hash(key) % numPartitions` to assign each record to a partition. This is fast and produces an even distribution for most keys, but it can produce severe skew if your key distribution is uneven—if 80% of your records share the same key, 80% of the data ends up in one partition.

The **RangePartitioner** (used by `sortByKey` and `repartitionByRange`) samples the data to estimate the distribution of keys, then sets range boundaries so that roughly equal amounts of data go to each partition. It requires an initial sampling pass but produces much more even partitions for skewed key distributions.

For DataFrames and SQL, the partitioner is managed automatically—you don't specify it directly. But you can influence it through `repartition(n, col)`, which partitions by the hash of `col`.

---

## Partition pruning: skipping data you don't need

Partition pruning is a different kind of partitioning story—not about task parallelism but about avoiding reading data that can't match a query's filters.

When a table on disk is **physically partitioned** by a column (using Hive-style partitioning: `/orders/year=2024/month=01/...`), each directory is a partition containing only rows with that value of the partitioning column. Spark's file scan can read the directory metadata without opening any files. If your query has a filter `WHERE year = 2024 AND month = 01`, the file scan only opens the directories that match—it **prunes** all other partitions. No files are opened; no rows are read; the partitioned data simply doesn't exist from the query's perspective.

Partition pruning is one of the highest-leverage optimizations in data lake workloads. A table partitioned by `date` that receives a query for a single day's data reads only 1/365th of the files. For petabyte-scale tables, this is the difference between a query that finishes in seconds and one that runs for hours.

Catalyst handles static partition pruning automatically: if the filter predicate involves only constants and partition columns, Spark evaluates it at plan time and passes the list of matching partitions to the file scan. **Dynamic partition pruning** (a Spark 3.x feature, covered in the AQE story) extends this to cases where the filter value is determined by the result of another query—for example, joining a large fact table with a small dimension table filtered by a condition; Spark can use the result of the dimension-table scan to prune partitions of the fact table at runtime.

---

## Skew: when partition distribution goes wrong

Even with a good partitioner, data skew can make partition counts meaningless. If one key appears 10 million times and all others appear a few hundred times, one partition will be 100x larger than average after a `groupBy`. The task for that partition takes 100x longer than all others; the stage is blocked waiting for it; the job is as slow as that one task.

Spark 3's AQE can detect skewed partitions after a shuffle and automatically **split them** into multiple sub-tasks, each processing a fraction of the skewed partition. This requires no changes to your code. For pre-3.x Spark, the manual solution is to add a **salt**: append a random number (0–9, say) to the key before the first aggregation, aggregate on the salted key in parallel, then remove the salt and aggregate again to combine the partial results. Salting spreads one hot key across many partitions at the cost of a second aggregation step.

---

## Bringing it together

A partition is the grain of Spark's parallelism: one partition, one task, one executor core. The number and size of partitions determines how well the cluster is utilized and how fast each task runs. **`coalesce`** merges partitions locally without a shuffle—cheap but uneven, good for consolidating after filtering. **`repartition`** redistributes all data with a full shuffle—more expensive but perfectly even, good before joins, sorts, or writes requiring a specific layout. **Partitioners** control where each record lands after a shuffle: hash for speed and evenness, range for ordered output. **Partition pruning** turns physical table layout into a query accelerator: filter by partition column and Spark never opens files that can't match. Skew—uneven partition sizes caused by uneven key distribution—is the enemy of partition count tuning, and AQE's skew handling (or manual salting) is the remedy. So the story of partitions is: **data is divided → tasks map to partitions one-to-one → the right number and distribution of partitions determines utilization, latency, and whether the cluster can skip data entirely.**
