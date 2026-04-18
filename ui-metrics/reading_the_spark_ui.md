# Reading the Spark UI: What Every Tab Is Actually Telling You

This is the story of how to read the Spark UI—not just what each tab contains, but what it is trying to tell you about your job's health and where time is being spent. The Spark UI is the primary diagnostic tool for understanding what a running or completed Spark job actually did. It can tell you whether a job is slow because of shuffle, skew, GC, serialization, data size mismatches, or stragglers. The challenge is knowing how to read it. This story walks through each major section and explains how to connect the numbers to diagnoses.

---

## The Jobs tab: the bird's-eye view

The **Jobs tab** is the first screen you see. It lists every Spark job that has been submitted, with status (running, succeeded, failed), start time, duration, and a progress bar showing tasks completed vs. total.

Each row corresponds to one call to an action (`.count()`, `.write.save()`, `.collect()`). A job that takes 10 minutes while another takes 2 seconds tells you immediately where to focus. Click into the slow job to see its stages.

The job description links back to the code that triggered it. If you see the same action triggered repeatedly (e.g., `count at MyJob.scala:42` appearing 100 times), you may have an accidental loop running a query multiple times.

> **The Jobs tab is like the arrivals and departures board at an airport.** At a glance you can see which flights are delayed, which are on time, and which gate to run to. You don't learn *why* a flight is delayed from the board—you need to go to the gate (stage detail) to find out. But the board tells you where to look first.

---

## The Stages tab: where work actually happens

The **Stages tab** is where most performance investigation happens. Every stage is listed with: number of tasks, input size, shuffle read size, shuffle write size, duration, and status.

The key numbers to look at first are **duration** and **shuffle read/write size**. A stage with large shuffle write (many GB) is an expensive map stage. A stage with large shuffle read is an expensive reduce stage. A stage with a small shuffle size but long duration is either CPU-bound, GC-bound, or has a slow user function.

Click into a stage for the stage detail view. This is where skew, GC, and straggler diagnosis happens.

---

## Stage detail: the task metrics table

The stage detail page shows metrics for every task in the stage. The most important columns:

**Duration**: how long each task ran. Sort this column descending to find the slowest tasks. A healthy stage has a tight distribution—all tasks finish within a similar timeframe. A skewed stage has a long tail: most tasks finish in 1 second, but one or two take 10 minutes. That's the data skew signature.

> **The task duration distribution is like finishing times in a 5K race.** In a well-trained group, everyone finishes within a few minutes of each other—the chart is a narrow cluster. If one runner takes 3 hours while everyone else finishes in 25 minutes, something is wrong with that runner (task). The stage can't close until the last person crosses the finish line.

**Shuffle Read Size**: how much shuffle data each reduce task read. A heavily skewed distribution (one task reads 50 GB while others read 100 MB) is the data skew diagnosis from the reducer's perspective.

**GC Time**: how much time each task spent in garbage collection. A task that spends more than 20–30% of its time in GC is memory-stressed. Look for tasks where GC time is a large fraction of task duration.

**Input Size**: for stages reading data directly from files. Uneven input sizes indicate that your data files are not evenly sized.

**Peak Execution Memory**: the maximum memory used by the task for execution (sort buffers, hash maps). Tasks approaching the task memory limit are candidates for spill.

**Spill (Memory) and Spill (Disk)**: non-zero spill means the task ran out of its memory allocation and wrote intermediate data to disk. Spill is always slow—writing and re-reading bytes from disk costs orders of magnitude more than keeping them in memory.

---

## The summary statistics panels

At the top of the stage detail view, Spark shows a **summary statistics panel** for each metric: min, 25th percentile, median, 75th percentile, max, with a box plot. This is the fastest way to spot skew.

For a healthy stage: the median and 75th percentile are close together, and the max is not much larger than the 75th. For a skewed stage: the median is 5 seconds, the 75th percentile is 7 seconds, but the max is 45 minutes. The box plot shows one outlier bar extending far to the right.

> **The summary statistics panel is like a height chart for a school class.** If all the children are between 4'5" and 4'10", the class is uniform. If one child measures 8'3", something unusual is happening. The box plot makes that outlier visually obvious—you don't have to scroll through 200 tasks to spot the one that's 1,000× slower than the rest.

The same analysis applies to shuffle read size, input size, and GC time. A max GC time of 30 seconds when the median is 0.2 seconds tells you one executor is GC-stressed.

---

## The SQL tab: query plans and timing

For DataFrame and SQL queries, the **SQL tab** provides the most detailed view. It shows the query plan as a DAG with each physical operator (Scan, Filter, Exchange, HashAggregate, SortMergeJoin, etc.) and—critically—**actual row counts and timing** for each operator.

The row count next to each operator is the number of rows that flowed through it. This is extremely useful for plan debugging:

- A `Filter` that reads 1 billion rows but passes 5 through means your predicate is very selective—good, but also means the scan read 1 billion rows unnecessarily if pushdown didn't fire.
- An `Exchange` (shuffle) node shows the total data size shuffled. Large shuffle sizes are expensive.
- A `SortMergeJoin` where one side has 1 billion rows and the other has 5 rows suggests a broadcast join was missed.
- A `HashAggregate` followed immediately by another `HashAggregate` is the two-phase aggregation pattern—normal and expected.

> **The SQL tab with row counts is like a water flow diagram for a plumbing system.** Each pipe shows how many litres flow through it. If you see 1 billion litres entering a filter but only 5 litres coming out, you know the filter is doing real work—but you also wonder why the source pipe is carrying 1 billion litres if only 5 are needed downstream. That gap is where predicate pushdown would have helped.

With AQE enabled, the SQL tab shows both the original plan and the adapted plan.

---

## The Executors tab: cluster resource view

The **Executors tab** shows every executor with its memory usage, disk usage, task counts, GC time, and shuffle read/write bytes. This is where you diagnose executor-level problems.

**Memory usage**: look for executors close to their memory limit. High "Storage Memory Used" relative to total means a lot of cached data.

**GC time as a fraction of task time**: the "GC Time" column divided by "Task Time" gives the GC fraction. Any executor consistently above 20–30% GC fraction is a candidate for memory pressure.

**Failed tasks**: the "Failed Tasks" column shows how many tasks have been retried on each executor. A specific executor with many failed tasks is suspicious—that machine may have a hardware issue, a slow disk, or a network problem.

**Shuffle write and read**: uneven shuffle write sizes across executors can indicate partition skew or uneven data distribution at the map side.

---

## The Storage tab: cached data

The **Storage tab** shows every RDD or DataFrame that has been cached, with its storage level, fraction of partitions cached, size in memory, and size on disk.

If you see a large dataset cached with many "Cached Partitions" but low "Fraction Cached," some partitions were evicted (not enough memory). Those partitions will be recomputed on next access—the caching is not fully effective.

If the storage tab shows no cached data but your code calls `.cache()`, either the cache hasn't been materialized yet (lazy evaluation—an action must trigger caching), or the cached blocks were all evicted.

---

## The Environment tab: configuration and classpath

The **Environment tab** lists all active SparkConf properties, JVM properties, system properties, and classpath entries. This is where you verify that configuration you set is actually in effect. A common mistake: setting a property with the wrong key name or in the wrong config file—the Environment tab shows exactly what Spark sees.

---

## Reading a slow job: a diagnostic workflow

1. **Jobs tab**: identify the slowest job by duration. Click into it.
2. **Stages tab**: identify the slowest stage. Is it a shuffle read or write stage? Note the sizes.
3. **Stage detail**: look at the task duration distribution. Is there a long tail (skew)? Check GC time (memory pressure). Check spill (insufficient memory or too few partitions). Check input size distribution (data file size imbalance).
4. **SQL tab**: find the corresponding query. Look at row counts. Is a filter not pushing down? Is a join using sort-merge when broadcast should apply? Is shuffle size large?
5. **Executors tab**: is one executor responsible for most of the slow tasks? Is GC time high on specific executors?
6. Correlate what you find with the physical plan, data layout, and job configuration.

---

## Bringing it together

The Spark UI is organized around four main diagnostic surfaces: the **Jobs tab** (which job and stage is slow), the **Stages tab with task metrics** (where in the stage is time spent—skew, GC, spill, or imbalanced input), the **SQL tab** (how the plan was executed, with actual row counts and operator timing), and the **Executors tab** (resource usage and failure counts per executor). The summary statistics panels on the stage detail page are the fastest path to spotting skew: a wide gap between median and max duration or shuffle size reveals the bottleneck partition. Row counts in the SQL plan reveal missed filter pushdowns and wrong join strategies. Together, these surfaces give a complete picture of where time goes—and the information needed to fix it.
