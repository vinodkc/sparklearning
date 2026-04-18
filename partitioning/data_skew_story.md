# When One Partition Holds Up Everyone: The Data Skew Story

This is the story of data skew—one of the most common and frustrating performance problems in Spark. Your job has 500 tasks. Four hundred and ninety-nine of them finish in 30 seconds. One runs for 45 minutes. The stage never ends. The cluster sits mostly idle while a single executor grinds through a mountain of data that should have been spread across many tasks. Understanding why skew happens, how to recognize it, and how to fix it—automatically or manually—is one of the most practical things you can learn about Spark.

---

## What skew is and why it happens

Data skew means that some partitions are much larger than others. In a perfectly distributed job, every partition has roughly the same number of rows, every task finishes in roughly the same time, and the stage completes when the last task finishes—which is about the same time as all the others. In a skewed job, a few partitions are enormous—sometimes thousands of times larger than average—and the stage's duration is dominated by the time it takes to process those giant partitions.

> **Imagine 10 cashier lanes at a supermarket.** Nine lanes have 5 people each and clear in 3 minutes. One lane has 500 people, because it is the only lane that accepts coupons. The store can't close until the last person in the coupon lane is served—the other nine cashiers stand idle for over an hour. That one lane is your skewed partition, and the store closing time is your stage completion time.

Skew almost always comes from the **key distribution in your data**. When Spark shuffles data—for a `groupBy`, `join`, `reduceByKey`, or any other wide transformation—it hashes each record's key and routes it to a partition based on that hash. If your keys are evenly distributed, partitions are roughly equal. If your data has a **hot key**—a key that appears in a large fraction of records—all those records hash to the same partition, and one task inherits a disproportionately huge workload.

Classic examples: a `groupBy(user_id)` where one user has 50 million events while typical users have a few hundred; a join on `country` where 40% of your records are `country = 'US'`; a join with null keys where nulls are all routed to the same partition. In each case, one partition swells while the rest stay small.

---

## Recognizing skew in the Spark UI

Skew is visible in the **Spark UI's stage detail view**. Click on a stage and look at the task metrics table—specifically the **Duration** column and the **Shuffle Read Size** or **Input Size** column. A skewed stage has a handful of tasks with run times or input sizes far above the median. The summary statistics panel (showing min, 25th percentile, median, 75th percentile, max) makes this obvious: if the max duration is 10x the median, you have skew.

The **event timeline** view is even more revealing: you see all tasks as horizontal bars, and the skewed tasks are the long bars that extend well past when all the other tasks finished—leaving most executor slots idle while one or two cores are still running.

---

## The cost of skew

Skew is expensive in several overlapping ways. The obvious cost is **wall-clock time**: the stage can't complete until every task finishes, so one giant task serializes what should be parallel work. The less obvious cost is **resource waste**: while the slow task runs, the rest of the executor cores are idle but still allocated, consuming cluster resources. And if the skewed partition is large enough, the task processing it may **spill to disk** (sort buffer or aggregation hash map overflows), adding disk I/O on top of the already excessive computation.

In join-heavy pipelines, skew in one stage can cascade: the skewed partition produces a large amount of output that becomes the input for the next stage, propagating skew downstream.

---

## Manual fix: salting

Before Spark 3's Adaptive Query Execution, the standard remedy for skew in aggregations was **salting**. The idea is to artificially widen the key space so that a hot key maps to multiple partitions instead of one.

For an aggregation (like `sum(amount) GROUP BY user_id`): append a random integer from 0 to N-1 to the key before the first shuffle. Each record with `user_id = 'big_user'` now has a key like `('big_user', 0)`, `('big_user', 3)`, `('big_user', 7)`, and so on—spread across N partitions. Each of those N partitions runs a partial aggregation in parallel, producing N partial sums. Then a second aggregation removes the salt and combines the partial sums into the final result.

> **It's like opening 5 extra "coupon-accepted" lanes at the supermarket.** Customers who used to all queue in the single coupon lane are now randomly assigned to lanes 1–5. Each lane processes a fifth of the coupon customers in parallel. A supervisor at the end combines the totals. Two passes, but dramatically faster overall.

Salting trades one expensive stage for two cheaper stages. The overhead is the second pass, but for a badly skewed key the two-pass cost is far less than the single slow task. The downside: you have to write the salting logic yourself, you need to choose N (too small and you don't spread enough; too large and you add unnecessary overhead), and it only works cleanly for associative aggregations where partial results can be merged.

For skewed joins, salting is trickier: you salt one side and replicate the matching rows of the other side N times. This is more invasive and increases the amount of data processed on the non-skewed side.

---

## AQE skew join: automatic salting at runtime

Spark 3's **Adaptive Query Execution** (AQE) automates skew handling for joins. With AQE enabled (`spark.sql.adaptive.enabled = true`) and skew join optimization on (`spark.sql.adaptive.skewJoin.enabled = true`), Spark detects skewed partitions at runtime—after the shuffle—and automatically splits them.

Here is how it works. AQE runs the map stage, observes the actual sizes of the shuffle output partitions, and identifies any partition that is significantly larger than the median. For each skewed partition, AQE splits it into multiple sub-tasks, each responsible for a contiguous range of the partition's data. On the other side of the join, the matching partition is **replicated** so that each sub-task has a complete copy of what it needs to join against. Multiple sub-tasks then process the skewed partition in parallel instead of one task processing the whole thing.

This is conceptually the same as manual salting, but done automatically, without changing your query, and applied only to the partitions that actually need it. The non-skewed partitions proceed as usual.

> **AQE is like a smart store manager watching the queue lengths in real time.** When the coupon lane backs up, the manager opens extra lanes and redirects the waiting customers—without any customer needing to know the plan changed. The lanes that were already moving normally are not touched.

The threshold defaults mean AQE will split a partition if it is at least 5x the median size and larger than 256 MB. You can tune these thresholds for your workload. The key thing to observe in the Spark UI is whether a stage has been split: you'll see more tasks than shuffle partitions, and the task sizes will be more uniform.

---

## AQE coalescing: fixing the other end

Skew is one side of the partition size problem. The other side is too many tiny partitions—the result of using a large `spark.sql.shuffle.partitions` (default 200) for a small dataset, which produces hundreds of near-empty partitions and hundreds of near-instant tasks with more overhead than work. AQE's **partition coalescing** (`spark.sql.adaptive.coalescePartitions.enabled`) runs after the shuffle and merges adjacent small partitions into larger ones, targeting a configurable minimum partition size. It reduces task count, reduces scheduling overhead, and avoids writing many tiny shuffle files. Skew handling and coalescing work together: AQE splits the big ones and merges the small ones, pushing the whole distribution toward uniformity.

---

## Other skew sources and mitigations

**Null keys**: Null values all hash to the same partition in many hash functions. If your join key has many nulls, they can pile up in one partition. The fix: filter out nulls before the join if they shouldn't match, or handle them separately.

**Bucketing**: If a table is bucketed on the join key, Spark can join without a shuffle—each bucket's file is read directly by the corresponding task. Bucketing pre-distributes data evenly at write time, so join-time skew never occurs. The cost is that you must write the table with bucketing enabled and both sides must have compatible bucket counts.

**Broadcast hints**: If the skewed side of a join is actually small (the hot key inflates one partition but the distinct key count is small), broadcasting the smaller table eliminates the shuffle entirely and the skew problem with it.

**Repartitioning before writing**: If skew persists into a write stage and produces giant output files, an explicit `repartition(n)` before the write redistributes data evenly across output files, even if key-based skew exists.

---

## Bringing it together

Data skew happens when key distribution is uneven: hot keys concentrate rows into a few giant partitions while the rest are small. The result is a slow stage where one task outlasts hundreds of others, wasting cluster resources and delaying the job. Skew is visible in the Spark UI's task duration and size distributions. The manual remedy is **salting**—widening the key space to spread hot keys across multiple partitions, at the cost of a two-pass aggregation. Spark 3's **AQE skew join** automates this: it detects oversized partitions after the shuffle, splits them into sub-tasks, and replicates the matching side—without any changes to your query. AQE also coalesces tiny partitions at the other end of the size spectrum. For joins, bucketing and broadcasting are structural alternatives that eliminate the shuffle—and the skew—entirely. So the skew story is: **uneven keys → uneven partitions → one slow task → wasted cluster → fix with salting, AQE, bucketing, or broadcast.**
