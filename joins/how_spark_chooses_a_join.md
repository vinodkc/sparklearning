# How Spark Chooses a Join

This is the story of how Spark decides *how* to physically execute a join. The logical operation—combine rows from two datasets where a key matches—can be carried out in several very different ways, with very different costs. The choice between a broadcast hash join, a sort-merge join, a shuffled hash join, or a cross join is one of the most consequential decisions Spark's physical planner makes. Get it right and a join that touches billions of rows finishes in seconds. Get it wrong and the same join shuffles terabytes unnecessarily or blows up executors. Understanding the criteria Spark uses to make that choice lets you understand your query plans—and change them when needed.

---

## Why the choice matters

Every join requires that rows with the same key end up in the same place, because the match can only happen if both rows are available to the same task at the same time. The question is how you arrange for that to happen—and the answer determines how much data moves, how much memory each task needs, and whether the join can start before both sides are fully read.

> **Think of a join like a meeting between two groups of people.** The key question is: do you move one small group to where the large group is, or do you shuffle everyone into a central venue? If one group has 5 people and the other has 50,000, it's obviously cheaper to move the 5. If both groups have 50,000 people, you need to organise the meeting systematically. The choice of venue and logistics is the join strategy.

---

## Broadcast hash join: send the small table everywhere

The most efficient join in Spark is the **broadcast hash join**. It applies when one side of the join is small enough to fit in memory on every executor. The idea is simple: instead of moving rows of the large table to meet rows of the small table, copy the entire small table to every executor. Each executor builds a **hash map** from the small table's join key to its rows, then streams through its partition of the large table and probes the hash map for each row. No shuffling of the large table. No sorting. No synchronization between executors. The large table never moves.

> **It's like giving every employee in a company a printed copy of the company directory (small table) and asking them to look up names (join) against their own local list of customers (large table).** No one needs to go to a central office to do the lookup—they do it at their desk with their own copy. The large table (customer list) never moves; only the small one (directory) is distributed.

The threshold for "small enough" is controlled by `spark.sql.autoBroadcastJoinThreshold` (default 10 MB). If Spark has statistics for a table and its estimated size is under this threshold, the physical planner will choose a broadcast hash join automatically. You can also force it with a `broadcast()` hint.

The catch: if the small table is not actually small—if the size estimate is wrong, or if many rows match and the hash map balloons—executors can OOM. And broadcast hash joins cannot be used for certain join types (like a full outer join) where both sides must contribute rows even when there is no match.

---

## Sort-merge join: the scalable default

When neither side is small enough to broadcast, Spark falls back to **sort-merge join**. This is the most general and scalable join strategy, and it's the default for large-large joins.

Sort-merge join has two phases. In the first phase, both sides are **shuffled** by their join key: all rows with the same key end up in the same partition, on the same executor, for both the left and right sides. In the second phase, each partition runs the merge: the left side's rows and the right side's rows—both already sorted by key within the partition—are merged together in a single pass. Two pointers walk the two sorted streams simultaneously, emitting matching pairs.

> **Think of merging two phone books.** First, both books are reprinted in the same format and their pages are distributed so that the same letters go to the same desk (shuffle). Then a clerk at each desk takes the two sorted stacks for their letter range, walks through them simultaneously, and pairs up every matching name (merge). The clerk only needs to hold a few pages at a time—not the entire book.

The advantage of sort-merge is that it requires only enough memory to hold a modest amount of in-flight rows during the merge—it does not build a full hash map of either side. It scales to arbitrarily large inputs. The disadvantage is the shuffle cost: two full shuffles, sorting on both sides, and the associated I/O.

---

## Shuffled hash join: shuffle then hash, skip the sort

**Shuffled hash join** is a middle ground. Like sort-merge, it shuffles both sides by join key so that matching rows are co-located. Unlike sort-merge, it does **not** sort the rows. Instead, it builds a **hash map** from the smaller side's rows (after the shuffle) and probes that hash map with the larger side's rows—exactly like broadcast hash join, but applied per partition rather than cluster-wide.

The appeal: no sorting required, which avoids the cost of sorting large partitions. The risk: each task must hold the entire smaller side of its partition in a hash map. If partitions are very large, or data is skewed so that one partition is enormous, this hash map can exceed available memory and cause an OOM. Spark will choose shuffled hash join when sort-merge is explicitly disabled and neither side qualifies for broadcast.

---

## Broadcast nested loop join and Cartesian product: when there is no key

What if there is no join key—a cross join, or a join with only inequality conditions? Spark uses a **broadcast nested loop join**: it broadcasts one side and, for each row in the other side, loops over all rows of the broadcast side looking for matches. This is O(n × m)—the dreaded nested loop—and is only acceptable when at least one side is genuinely tiny.

> **This is like asking every employee to personally check whether they know everyone on a guest list.** If the guest list has 10 people, it's quick. If it has 10 million, every employee spends a day. Nested loop joins only work when at least one side is small enough to make the brute-force search tractable.

---

## How Spark decides: the physical planning rules

Spark's physical planner evaluates the join candidates in roughly this order:

1. **Broadcast hash join** — if either side has a known size under the broadcast threshold, and the join type supports broadcasting. Hints always win.

2. **Shuffled hash join** — if `preferSortMergeJoin` is false, or the join key doesn't support sorting, and the build side is estimated to fit per partition.

3. **Sort-merge join** — the default for everything else where keys support ordering. Both sides are shuffled and sorted.

4. **Broadcast nested loop join / Cartesian** — for cross joins or joins where no equi-condition exists.

Statistics matter enormously here. Without statistics (no `ANALYZE TABLE`, no catalog size hints), Spark can't know whether a table is 10 MB or 10 GB. It may use a sort-merge join for a table that would have qualified for broadcast. Collecting statistics on tables, especially dimension tables that are joined frequently, is one of the most impactful tuning actions for SQL workloads.

---

## Adaptive Query Execution: changing the plan at runtime

Static planning has a fundamental problem: estimates are guesses. A table's size estimate might be wrong; data might be skewed; a filter that was supposed to reduce one side to 8 MB might reduce it to 800 MB instead. **Adaptive Query Execution (AQE)**, enabled by default in Spark 3.x, addresses this by re-planning joins at runtime, after shuffles.

> **AQE is like a GPS that re-routes you based on live traffic.** The original route (static plan) was based on last night's traffic estimates. When you're actually driving and a motorway is closed (data was larger than expected), the GPS re-routes on the fly to take a better path. The destination (query result) is the same; only the route changes.

When AQE is on, Spark executes the shuffle, collects statistics about the actual sizes of the shuffle output, and then re-evaluates the join plan with those real numbers. If one side of a sort-merge join turned out to be 6 MB after filtering, AQE can **switch it to a broadcast hash join on the fly**—saving the sort and the network transfer of the large side.

---

## Reading the physical plan for join decisions

When you call `explain()`, the physical plan shows you exactly which strategy was chosen. Look for the node names: `BroadcastHashJoin`, `SortMergeJoin`, `ShuffledHashJoin`, `BroadcastNestedLoopJoin`. Below a `SortMergeJoin`, you'll see `Exchange` nodes (the shuffles) and `Sort` nodes on each side. Below a `BroadcastHashJoin`, you'll see a `BroadcastExchange` on one side (the small table being broadcast). The absence of an `Exchange` on the large side confirms that the large table wasn't shuffled.

---

## Bringing it together

Spark has four physical join strategies, each suited to different size ratios and data properties. **Broadcast hash join** is fastest but requires one side to be small: it copies that side everywhere and avoids any shuffle of the large table. **Sort-merge join** is the scalable default: shuffle both sides, sort them, and merge—any size, but two shuffles. **Shuffled hash join** shuffles both sides but skips the sort, building a per-partition hash map—faster than sort-merge when partitions are small enough to hash, risky when they're not. **Broadcast nested loop** handles non-equi joins by brute-force looping, only feasible when one side is tiny. The physical planner picks a strategy based on size estimates from statistics and hints, evaluating broadcast first. With AQE enabled, the plan can be revised after the shuffle using actual measured sizes rather than estimates. So the story of a join is: **choose a strategy → shuffle (if needed) → match rows → adapt if reality differed from the estimate.** The choice is everything: the same logical join, executed with the right strategy, can be ten or a hundred times faster than with the wrong one.
