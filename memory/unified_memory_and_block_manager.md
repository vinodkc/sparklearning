# The Two Lives of Spark's Memory

This is the story of how Spark decides where to put things—your cached RDDs, your shuffle buffers, your aggregation maps, and the raw bytes coming off the wire. Understanding that story explains why caching sometimes evicts data you needed, why your job OOMs even when you've given it plenty of memory, and what the BlockManager actually does all day.

---

## One heap, two jobs

Every executor has one JVM heap (and, optionally, off-heap memory). That memory must serve two very different jobs at once.

The first job is **execution**: holding the intermediate state produced while tasks are running—the in-memory sort buffers that sort-based shuffle uses, the hash tables that aggregations build, the rows that a join accumulates on one side before it starts probing. This memory is *transient*: it appears during a task, grows as the task works, and is released when the task finishes or spills to disk.

The second job is **storage**: holding data that you've asked Spark to keep around—cached RDDs, broadcast variables, and cached DataFrame results. This memory is *persistent*: it was explicitly saved and is expected to stay until evicted or unpersisted.

For a long time, Spark divided the heap into fixed fractions: a fixed fraction for execution, a fixed fraction for storage, and a small reserved region. If your job used too little execution memory and too much caching (or vice versa), the fixed boundary meant one side sat empty while the other spilled or evicted unnecessarily.

---

## Unified memory: one shared pool

The answer was **unified memory management**, introduced in Spark 1.6. Instead of two fixed buckets, there is one shared pool (after subtracting a reserved region for the JVM itself, class loading, and Spark internals). Within that pool, execution and storage start out with soft boundaries: storage can use up to a certain fraction of the pool by default, and execution can use the rest—but either side can borrow from the other when its side is idle.

The interesting case is when they conflict. If execution needs more memory and storage is using some of the pool, execution can **evict** cached blocks to reclaim space. Storage cannot evict execution's working memory—that would kill running tasks. So execution has the stronger claim. If storage needs more memory and execution is using some of the pool, storage simply doesn't get it; it either waits (for execution to release), uses less, or (if the data must be stored somewhere) spills to disk.

In practice this means: if your job is actively running tasks, the hash tables and sort buffers can grow into the fraction that your cached RDDs would otherwise occupy—up to the point where caching evictions occur. When tasks finish and release their working memory, that space becomes available for storage again (or for the next wave of tasks). The shared pool lets memory follow the work rather than sitting idle in the wrong bucket.

---

## The BlockManager: everything is a block

Under this memory model sits the **BlockManager**, one of the most central components in a Spark executor. The BlockManager is Spark's local storage system: it manages every chunk of data that an executor holds and tracks where each chunk lives. Everything that Spark stores—an RDD partition, a shuffle map output, a broadcast variable's data, a streamed shuffle block—is a **block**. Each block has a **BlockId** (an identifier that encodes what kind of thing it is—RDD block, shuffle block, broadcast block) and a **StorageLevel** (where it should live: memory only, disk only, memory-with-disk-fallback, serialized or deserialized, replicated or not).

When a task finishes computing a partition and Spark is told to cache it, the executor hands the data to the BlockManager: *store this RDD block at this storage level*. The BlockManager decides whether to deserialize it and store it as live objects in the JVM heap (fast to read, expensive on memory), serialize it into a byte array in memory (slower to read, cheaper on memory), or write it to disk. If the block is too big to fit in the memory fraction available to storage, the BlockManager may first **evict** an existing cached block to make room—choosing the victim by LRU or by how recently it was accessed.

---

## Eviction: the price of caching

Eviction is the moment the BlockManager must choose a block to remove to make space for a new one. It picks the **least-recently used** cached block that is not currently being read or used by a running task. It then either drops it entirely (if it's an `MEMORY_ONLY` RDD that can be recomputed from lineage) or moves it to disk (if its storage level includes disk). Evicting to disk is cheap to observe but expensive to pay: that block, when next read, must be deserialized off disk, which is much slower than reading from memory.

If a block must be evicted but cannot be moved to disk (no disk in its StorageLevel) and there is still not enough room for the new block, the incoming block simply **won't be cached**. The task continues without caching that partition; the next job that needs that partition will recompute it. Spark will not OOM trying to honor a cache request it can't satisfy—it just skips the cache for that block.

---

## The MemoryStore and DiskStore

Internally the BlockManager delegates to two stores. The **MemoryStore** manages the memory region. It keeps a map from BlockId to the in-memory representation of that block (either a deserialized iterator/array of values or a serialized ByteBuffer). It knows how many bytes each block occupies and maintains the accounting that lets it decide whether a new block fits. When it must evict, it picks an LRU candidate, removes it from the map, and—if the StorageLevel says disk is allowed—hands the bytes to the DiskStore.

The **DiskStore** manages a configurable set of local directories (set by `spark.local.dir`). It serializes and writes blocks as files, named by their BlockId. It can read them back as streams of bytes. Shuffle files also live in these directories, though shuffle output is managed separately by the shuffle system rather than the BlockManager's storage-level machinery.

---

## The BlockManagerMaster: the driver's view

Each executor has its own BlockManager. The **driver** has a special BlockManager as well, and it also runs the **BlockManagerMaster**—a component that keeps a global registry of which blocks exist on which executors. When you call `rdd.cache()` and a task runs on executor A and caches partition 3 there, executor A's BlockManager reports to the BlockManagerMaster: "block RDD 42 partition 3 is on me, in memory, 128 MB." The master records this.

When another task—on executor B, say—needs partition 3 of that RDD, it first checks: is this block somewhere? It asks the master. The master says: "executor A has it." Executor B can then fetch it from A (a remote block read via the network) rather than recomputing it from scratch. So the BlockManagerMaster is what makes caching useful across the whole cluster rather than just on the executor that cached the data. It's also how broadcast variables work: the driver sends broadcast data to one executor, which reports it to the master; other executors can then fetch it peer-to-peer rather than all going back to the driver.

---

## Off-heap memory: escaping the GC

Everything so far is **on-heap**: objects and byte arrays in the JVM heap, subject to garbage collection. For workloads that cache a lot of data—especially as serialized bytes—GC pressure can become the bottleneck. Every time the JVM collects, it has to walk many live objects. Spark supports **off-heap storage**: using `sun.misc.Unsafe` (or, more recently, `java.lang.foreign`) to allocate memory outside the JVM heap entirely. Blocks stored off-heap are serialized byte arrays in native memory. The JVM has no objects to track for them; GC sees nothing. The tradeoff is that reading off-heap blocks still requires deserialization, and managing the off-heap region requires Spark to do its own memory accounting carefully.

Tungsten (Spark's internal binary execution engine) also uses off-heap memory for execution—sort buffers and hash maps built from binary rows that the JVM doesn't see as ordinary Java objects. That story lives in its own chapter; what matters here is that the memory subsystem has to track both on-heap and off-heap usage together to make sensible decisions about eviction and spills.

---

## Bringing it together

Spark's memory is one unified pool divided softly between storage (for cached data) and execution (for task working memory). Execution has the stronger claim and can push storage back when it needs room. Everything cached or stored is managed by the **BlockManager**: it takes blocks, assigns them to the MemoryStore or DiskStore based on their storage level, evicts LRU blocks when storage runs low, and reports to the **BlockManagerMaster** on the driver so the whole cluster knows where everything lives. When a task needs a cached block, it looks up the master, finds the executor that has it, and fetches it—avoiding a recompute. When a block can't fit in memory and disk isn't allowed, Spark skips the cache rather than OOMing. So the journey of a cached partition is: **compute → hand to BlockManager → fit in memory (maybe evicting another block) → report to master → serve to any executor that asks.** Memory in Spark is not a passive resource—it is actively managed, borrowed, returned, and reclaimed in step with the computation.
