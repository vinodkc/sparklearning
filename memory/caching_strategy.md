# Cache Wisely: When Persisting Data Helps and When It Hurts

This is the story of Spark caching—when to use it, when to avoid it, and what's actually happening when you call `.cache()` or `.persist()`. Caching is one of the most overused and misunderstood features in Spark. Used correctly, it eliminates redundant computation and dramatically speeds up iterative algorithms. Used incorrectly, it wastes memory, evicts data you actually needed, and can make performance worse. Understanding the mechanics of caching—how data is stored, when it's recomputed, and how the cache interacts with the execution plan—turns caching from a cargo-cult optimization into a deliberate tool.

---

## What caching does

When you call `df.cache()` (equivalent to `df.persist(StorageLevel.MEMORY_AND_DISK)`), you're telling Spark: "after computing this DataFrame for the first time, store the result partitions in memory (or on disk if memory is insufficient) so that subsequent accesses reuse the stored result instead of recomputing."

The first action that materializes the DataFrame triggers computation. Each partition's result is stored in the executor's BlockManager. Subsequent accesses to the same DataFrame (in the same application) use the cached partitions directly.

`.cache()` is just `.persist()` with a default storage level. `.persist(StorageLevel.DISK_ONLY)` stores on disk; `.persist(StorageLevel.MEMORY_AND_DISK_2)` stores in memory (or disk) with a 2× replication across executors.

> **Caching is like photocopying a reference document.** The first time you need it, you retrieve the original (compute the DataFrame). You make a copy (cache it) and keep it at your desk (executor's BlockManager). The next time you need it, you use your desk copy (cached partition) instead of going to the filing room again (recomputing from scratch). If your desk has limited space (memory), some copies may get moved to a storage room (disk) or discarded (evicted).

---

## When caching helps

**Multiple accesses to the same DataFrame**: the clearest signal. If a DataFrame appears multiple times in your execution plan—because you use it in two separate joins, or because you apply a split and use both branches—caching avoids recomputing it each time.

```python
# Without caching: filtered_orders is computed 3 times
filtered_orders = orders.filter(col("status") == "completed")
count = filtered_orders.count()
top_customers = filtered_orders.groupBy("customer_id").count()
avg_amount = filtered_orders.agg(avg("amount"))

# With caching: computed once, reused 3 times
filtered_orders.cache()
count = filtered_orders.count()  # triggers computation and caches
top_customers = filtered_orders.groupBy("customer_id").count()  # uses cache
avg_amount = filtered_orders.agg(avg("amount"))  # uses cache
filtered_orders.unpersist()  # release when done
```

**Iterative algorithms**: algorithms like k-means, PageRank, and graph algorithms iterate over the same dataset many times. Without caching, each iteration recomputes the entire lineage from the source. With caching, the base dataset is read from storage once; all iterations access the cached version.

**Expensive transformations that feed multiple downstream operations**: if a DataFrame involves a complex multi-stage pipeline (10 joins, 5 aggregations, 3 window functions) and its result is used in multiple places, caching the result prevents rerunning that pipeline for each use.

---

## When caching hurts

**Large datasets with few reuses**: caching 500GB of data to avoid recomputing it once is often a bad trade. The caching write takes time, the memory is consumed (potentially evicting other useful data), and the one-time benefit doesn't justify the cost.

**Scans from cheap sources**: reading Parquet files from S3 with column pruning and predicate pushdown is already fast. If a DataFrame is a simple filtered scan of a Parquet file and used only twice, the scan might be cheaper than the memory cost of caching.

**Interfering with AQE and plan changes**: caching pins the DataFrame's partitioning and data to a specific point in the plan. If AQE would have coalesced shuffle partitions or converted a join strategy after the cache point, caching may prevent those optimizations from applying to downstream stages.

**Sequential pipelines**: if a DataFrame is only accessed once (a sequential pipeline where each transformation builds on the previous without any fan-out), caching it wastes memory—the data is computed once and discarded anyway.

> **Caching a dataset you only use once is like photocopying a document, putting it in a filing cabinet, and never opening the cabinet.** You paid the cost (memory, time to cache) and got no benefit. The desk space (executor memory) you used for the copy is space that could have been used for something else.

---

## Storage levels: choosing the right one

`StorageLevel` controls where cached data lives:

| Storage Level | Memory | Disk | Serialized | Replicas | Use Case |
|--------------|--------|------|------------|---------|----------|
| `MEMORY_ONLY` | ✓ | ✗ | No | 1 | Fast JVM objects; fails if doesn't fit |
| `MEMORY_AND_DISK` | ✓ | ✓ | No | 1 | Most common: memory first, spills to disk |
| `MEMORY_ONLY_SER` | ✓ | ✗ | Yes | 1 | Compact; JVM GC-friendly; slower access |
| `MEMORY_AND_DISK_SER` | ✓ | ✓ | Yes | 1 | Compact + disk fallback |
| `DISK_ONLY` | ✗ | ✓ | Yes | 1 | Very large datasets; slower than memory |
| `MEMORY_AND_DISK_2` | ✓ | ✓ | No | 2 | Fault-tolerant cache; double the storage |
| `OFF_HEAP` | off-heap | ✗ | Yes | 1 | Reduces GC pressure; requires Tungsten |

**`MEMORY_ONLY`** (the default for RDDs via `.cache()`) stores deserialized JVM objects. Accessing them is fast (no deserialization needed) but they consume the most memory (JVM object overhead). If partitions don't fit, they are **not stored at all**—Spark recomputes missing partitions on the fly.

**`MEMORY_AND_DISK`** (the default for DataFrames via `.cache()`) is more forgiving: if memory is insufficient, partitions spill to disk. Accessing disk-cached partitions is slower than memory, but better than full recomputation.

**Serialized levels** (`_SER`) store partitions as byte arrays using Kryo or Java serialization. More compact than deserialized objects (typically 2–5× smaller), easier on the garbage collector, but slower to access (deserialization on each read).

> **Storage level is like choosing between a quick-access desk, a filing cabinet in the room, and a storage unit down the street.** `MEMORY_ONLY` is your desk: instant access but limited space. `MEMORY_AND_DISK` is desk first, then filing cabinet if the desk is full. `DISK_ONLY` is always the storage unit—always available, always slower.

---

## The cache invalidation problem

Cached DataFrames are lazily materialized (cached on first action) and remain valid until:
1. You explicitly call `df.unpersist()`
2. The executor that holds the partition crashes (the BlockManager on that executor is gone)
3. Memory pressure causes eviction (LRU-based: least-recently-used partitions are evicted first)

**There is no automatic invalidation when the source data changes.** If you cache a DataFrame read from a Delta table, then another process updates the Delta table, your cached DataFrame reflects the old data—Spark doesn't know the source changed. This is a significant correctness concern in production pipelines that read external data.

For this reason, caching is safest for:
- Immutable datasets (no updates)
- In-session computations where you control all writes
- Iterative algorithms where the base data doesn't change between iterations

---

## Diagnosing cache effectiveness

The **Storage tab** in the Spark UI shows all currently cached DataFrames/RDDs: their names, storage levels, how many partitions are cached vs. not cached, and what fraction is in memory vs. on disk. This is the ground truth for cache occupancy.

If you cached a DataFrame but see it being recomputed in the DAG (stages showing the full plan from source to transformation), it means:
- The cache was invalidated (executor failure, eviction)
- The cache was never materialized (the first action that should have triggered caching was actually a `.count()` on a different DataFrame branch—caching is only triggered when the specific DataFrame is accessed)
- The DataFrame identity changed (you called `.cache()` but then applied another transformation, creating a new DataFrame that's not the cached one)

---

## A practical caching recipe

```python
# Step 1: cache the DataFrame before the first action
df_filtered = raw_df.filter(expensive_condition).select(needed_cols)
df_filtered.persist(StorageLevel.MEMORY_AND_DISK)

# Step 2: trigger caching with the first action
count = df_filtered.count()  # this materializes and caches all partitions

# Step 3: do the rest of your work reusing the cached DataFrame
result1 = df_filtered.groupBy("key").agg(...)
result2 = df_filtered.join(other_df, "key")
result3 = df_filtered.filter(additional_condition).agg(...)

# Step 4: release memory when done
df_filtered.unpersist()
```

Always call `.unpersist()` when you're done with a cached DataFrame in a long-running application. Without it, the cached data stays in memory (or on disk) indefinitely, wasting resources for data you no longer need.

---

## Bringing it together

Caching is a tool for eliminating redundant computation—it's valuable when a DataFrame is accessed multiple times (fan-out in the plan, iterative algorithms) and expensive enough to justify the memory cost. It's counterproductive for large datasets with few reuses, simple scans from fast sources, or sequential pipelines where data flows through once. **Storage levels** control the tradeoff between speed (memory) and safety (disk fallback, replication). **There is no automatic invalidation**—cached data becomes stale if the source changes. The **Storage tab** in the Spark UI shows what's actually cached and how much memory it occupies. The recipe is simple: cache → trigger with an action → use → unpersist. So the story of caching is: **identify fan-out in the plan → cache the repeated DataFrame → trigger materialization → reuse from cache → release when done.** Cache deliberately, not reflexively.
