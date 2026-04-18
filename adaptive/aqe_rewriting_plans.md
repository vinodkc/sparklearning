# AQE: How Spark Rewrites Plans After the Shuffle

This is the story of Adaptive Query Execution—Spark's mechanism for changing its mind at runtime. The central problem AQE solves is that query plans are made before data is read, using size estimates that may be wrong. AQE breaks that constraint by using the actual statistics collected at shuffle boundaries to re-plan the parts of the job that haven't run yet. Understanding how AQE works explains why Spark 3 jobs often run faster with no code changes, why a sort-merge join might silently become a broadcast hash join mid-job, and why your shuffle produces fewer tasks than you set in `spark.sql.shuffle.partitions`.

---

## The fundamental problem: planning with guesses

Spark's query planner runs before any data moves. It uses two sources of information to make decisions: statistics stored in the catalog (from `ANALYZE TABLE`) and heuristics (fixed rules about what "small" means, what join types are safe). Both are imprecise. Statistics can be stale, absent, or wrong after a filter dramatically reduces a table's size mid-query. Heuristics are tuned for average cases.

The result is that a query plan chosen before execution may turn out to be suboptimal once the data is in motion. A table the planner estimated at 500 MB might be 4 MB after filtering—perfect for a broadcast join, but the planner had already committed to a sort-merge. Two hundred shuffle partitions might produce 195 nearly-empty partitions and 5 medium ones—most tasks finish in milliseconds while shuffle overhead dominates.

> **It's like planning a dinner party for 200 people based on the RSVPs you sent out, only to find that 190 people cancelled.** You ordered 200 place settings, hired 10 waiters, and reserved the grand ballroom—all for a table of 10. AQE is the caterer who checks at the door how many guests actually showed up and adjusts the room layout, staffing, and serving strategy accordingly. The menu (logical query) stays the same; the execution changes to fit reality.

AQE's insight: shuffle boundaries are natural checkpoints. Both sides of a shuffle must be fully written before any reducer starts reading. That means the driver already knows, at that boundary, the exact size of every reducer's input. AQE exploits this information to re-plan what comes next.

---

## How AQE runs: the materialized shuffle stage

AQE divides query execution into **query stages**, each terminated by a shuffle (or a broadcast). A query stage is a set of tasks that produces shuffle output; its successors can only run after it completes. When a query stage finishes, the driver knows the number of map output partitions and the exact byte size of each one. With this information, three categories of re-planning become possible.

---

## Optimization 1: coalescing shuffle partitions

`spark.sql.shuffle.partitions` sets the number of reduce partitions for every shuffle in the job—default 200. For a job that shuffles 10 GB, 200 partitions of ~50 MB each is reasonable. For a job that shuffles 50 MB (after aggressive filtering), 200 partitions of ~250 KB each means 200 near-empty tasks, each with scheduling overhead that dwarfs the actual work.

> **It's like assembling a 1,000-piece puzzle but starting with 200 people—one person per 5 pieces.** The coordination overhead of assigning tasks, checking who's finished, and collecting results dwarfs the actual work of placing 5 pieces. If AQE can see that only 12 of the 200 sections have any pieces at all, it merges the empty sections and assigns the work to 12 people instead of 200.

AQE's coalescing optimizer looks at the actual sizes of the shuffle output partitions and merges adjacent small partitions into fewer larger ones, targeting a configurable target size. The merged partitions become a single task. The practical effect: a job configured for 200 shuffle partitions might end up with 12 actual reduce tasks if the data was small.

---

## Optimization 2: converting sort-merge joins to broadcast hash joins

The physical planner chooses a join strategy based on estimated table sizes before the query runs. If statistics are missing or wrong, it may choose sort-merge join for a table that, after filtering or joining, is actually tiny.

AQE's join conversion optimizer re-evaluates the join strategy after upstream stages complete. If one side of a sort-merge join produced a shuffle output smaller than the runtime broadcast threshold, AQE cancels the sort-merge plan and replaces it with a broadcast hash join.

> **Think of it like planning a meeting.** You scheduled a meeting room for 50 people because you expected 50 participants. When only 3 show up, you downgrade to a small conference room instead—the outcome is the same (the meeting happens), but you're not wasting a 50-person room. AQE downgrading a sort-merge join to a broadcast join is the same adjustment: same result, much smaller room (less network traffic and no sorting of the large side).

This optimization is often the single biggest AQE win for workloads with complex multi-table queries where filters along the pipeline reduce one table to a tiny fraction of its original size.

---

## Optimization 3: skewed join handling

When a shuffle finishes and AQE examines the partition sizes, it may find that a few partitions are dramatically larger than the others—the signature of data skew. AQE's skew join optimizer detects these and splits them.

A partition is considered skewed if its size exceeds both a minimum threshold and a multiple of the median partition size. If partition 42 is 5× larger than the median and above 256 MB, AQE will split it into sub-ranges, each processed by a separate task. For each sub-range of the left side, the corresponding complete right-side partition is replicated so that the sub-task has everything it needs.

> **It's like discovering mid-marathon that one lane of the road has all the runners piled up—the other lanes are nearly empty.** Instead of forcing everyone to run through the bottleneck, a marshal opens up extra temporary lanes for the congested section and redirects runners into them. Each group finishes the bottleneck stretch in parallel, and the race moves on at normal speed. AQE does this automatically at runtime; you don't have to pre-plan the extra lanes.

---

## Dynamic partition pruning: a close cousin

Closely related to AQE but conceptually distinct is **Dynamic Partition Pruning** (DPP). DPP applies when you join a large fact table (partitioned on disk by, say, `date`) with a small dimension table that has a filter on it. At plan time, Spark doesn't know which dates the dimension table will select. With DPP, Spark executes the dimension-table scan first, collects the set of matching values, and uses that set as a runtime filter on the fact table's file scan—skipping entire partitions on disk that can't possibly match.

> **DPP is like calling ahead to a library to ask which shelves contain the books you want.** Rather than walking every aisle, you get a list of aisle numbers from the librarian and go directly to those aisles. The dimension table is the phone call; the fact table is the library. Only the relevant aisles (partitions) are visited.

DPP is enabled by default in Spark 3. It is most powerful when the fact table is large and heavily partitioned, the dimension table is small (eligible for broadcast), and the join key is the same as the partition column.

---

## Reading AQE plans in EXPLAIN

When AQE is active, `explain()` output shows two plan sections: the **initial plan** (what was planned before execution) and the **final plan** (what actually ran after re-planning). You may see `BroadcastHashJoin` where the initial plan had `SortMergeJoin`, or a different number of exchange partitions than `spark.sql.shuffle.partitions` suggests.

The Spark UI's SQL tab also shows both plans side-by-side for a completed query, with nodes annotated with their actual row counts and sizes—exactly the statistics that drove the re-planning decisions.

---

## When AQE doesn't help

AQE is powerful but not universal. It only operates at **shuffle and broadcast boundaries**—it can't re-plan within a single stage. If your query has no shuffles, AQE has no checkpoints at which to collect statistics and re-plan. DPP similarly requires a partition column that matches the join key and a broadcastable dimension table.

AQE also cannot fix problems rooted in the initial read: if a file scan produces a skewed partition because one input file is enormous, AQE doesn't split files. And the runtime broadcast conversion requires that the small side's data actually fits in memory when broadcast.

---

## Bringing it together

AQE makes Spark's query planning adaptive rather than static. At every shuffle boundary, it collects the actual sizes of all output partitions and uses them to re-plan the stages ahead. Three optimizations fire based on what it sees: **coalescing** merges adjacent tiny partitions into fewer, right-sized tasks; **join conversion** promotes sort-merge joins to broadcast hash joins when one side turns out to be small; **skew join handling** splits oversized partitions and replicates the matching side to parallelize what would otherwise be a single slow task. Dynamic partition pruning complements AQE by pushing a join's filter result back into the file scan, skipping entire partitions before any data is read. Together, these mechanisms mean Spark can compensate at runtime for the stale statistics, wrong estimates, and unexpected data shapes that would otherwise produce slow, wasteful jobs.
