# Garbage Collection in Spark: Why the JVM Pauses and How to Make It Stop

This is the story of JVM garbage collection in Spark—the invisible janitor that periodically halts executor threads to clean up memory, causing the mysterious task slowdowns and "GC overhead limit exceeded" errors that plague many Spark applications. Understanding why GC happens in Spark, which phases are dangerous, and how to reduce GC pressure transforms GC tuning from voodoo into a principled exercise. This story covers the JVM memory model, the types of GC events, Spark-specific GC patterns, and the concrete settings that make a difference.

---

## Why GC matters in Spark

Spark runs on the JVM, and the JVM manages memory through garbage collection: periodically identifying objects that are no longer reachable from the running program and reclaiming their memory. The problem is that GC can **pause all threads** while it runs—a "stop-the-world" pause during which no tasks execute. For a 100ms GC pause on every executor, every 10 seconds, tasks that should take 30 seconds might take 40+ seconds. For a full GC that takes 2 seconds, an executor appears unresponsive to the driver (heartbeat misses), tasks stall, and in extreme cases, executors are considered lost.

> **GC pauses are like the building janitor who, to clean the lobby, must ask everyone to stop walking through.** Everyone freezes while the janitor mops. If the janitor takes 10 seconds, everyone is 10 seconds late for their meeting. Frequent short pauses accumulate; one long pause can miss a meeting entirely. The goal is to minimize how often the janitor needs to freeze traffic and how long each freeze lasts.

The Spark UI shows GC time per task in the Stages tab task metrics. If GC time is a significant fraction of task duration (more than 5–10%), GC is a problem worth addressing.

---

## JVM heap structure: generations

The JVM heap is divided into generations based on the principle that most objects die young:

**Young generation (Eden + Survivor spaces)**: where new objects are allocated. Every object starts here. When Eden fills up, a **minor GC** runs: live objects are moved to a Survivor space; dead objects are collected. Minor GC is usually fast (10–100ms) and stop-the-world.

**Old generation (Tenured space)**: objects that survive multiple minor GCs are promoted here. The old generation fills more slowly but requires a **major GC** (or Full GC) to collect—which is much slower (hundreds of milliseconds to seconds). Full GC is stop-the-world and the main cause of serious GC problems.

**Metaspace** (replaces PermGen in Java 8+): class metadata (class definitions, method bytecode). Usually not a GC concern unless many classes are dynamically loaded (e.g., very complex Spark job with thousands of generated code classes).

> **Young and old generation are like a hotel with a lobby (young) and permanent suites (old).** Most guests (objects) check in to the lobby, have a short stay, and leave (die young). Some stay long enough to get a suite (promoted to old generation). Cleaning the lobby (minor GC) is quick—few guests and short-lived. Cleaning the suites (major GC) takes all day.

---

## Spark-specific GC patterns

Several Spark patterns generate excessive garbage:

**Many short-lived JVM objects per row**: a query that processes 100 million rows, creating a new `Row` object (or Scala case class) per row, generates 100 million objects. The young generation fills rapidly, triggering frequent minor GCs. Tungsten and codegen reduce this by working with off-heap binary buffers rather than JVM objects.

**Deserialization of task closures and shuffle data**: when tasks receive results or shuffle data from the network, they deserialize bytes into JVM objects. For large shuffles with many keys, this creates many objects simultaneously, pressuring the old generation.

**Caching with MEMORY_ONLY (deserialized)**: cached partitions stored as deserialized JVM objects are long-lived. They are promoted to the old generation and persist there until uncached. A large cache of deserialized objects fills the old generation, leaving little room for task execution objects, which then trigger frequent old-gen GCs.

**Python UDFs**: every row processed by a Python UDF crosses the JVM–Python boundary—serialized, sent to Python, result deserialized. The serialized bytes and deserialized result objects are short-lived JVM objects, flooding the young generation.

---

## GC tuning: the levers

**1. Use G1GC (recommended for modern JVMs)**

The default GC algorithm changed from Parallel GC to G1GC in Java 9, but Spark's documentation and many Spark clusters still default to Parallel GC. G1GC is better for Spark's workload because:
- It divides the heap into equal-sized regions rather than strict generational spaces, allowing more flexible allocation.
- It aims for pause time goals (configurable) rather than maximum throughput, which aligns with Spark's preference for short, predictable pauses.
- It better handles large heaps (8GB+ executor memory) where Parallel GC's Full GC pauses are very long.

Set G1GC:
```
spark.executor.extraJavaOptions = -XX:+UseG1GC -XX:G1HeapRegionSize=16M -XX:InitiatingHeapOccupancyPercent=35
```

`G1HeapRegionSize=16M` is a good default for executors with 4GB+ of heap. `InitiatingHeapOccupancyPercent=35` tells G1 to start concurrent GC at 35% heap occupancy rather than the default 45%—more frequent, shorter GCs, avoiding large stop-the-world events.

**2. Increase executor memory to reduce GC frequency**

GC runs when generations fill up. More heap space means generations fill more slowly, and GC runs less often. Doubling executor memory often halves GC frequency (at the cost of needing larger worker nodes or fewer executors per node).

```
spark.executor.memory = 8g   (instead of 4g)
```

**3. Adjust the Spark memory fraction for execution vs. storage**

`spark.memory.fraction` (default 0.6) is the fraction of executor heap available to Spark's managed memory pool (used for shuffle buffers, sort buffers, and cached data). `1 - spark.memory.fraction` (default 0.4) is reserved for user code objects (RDD transformations, deserialization buffers).

If GC is high during task execution (not caching), increasing `spark.memory.storageFraction` (fraction of the managed pool used for caching) may help—it reserves more of the managed pool for caching, leaving less for execution buffers, which forces smaller shuffle buffers but may reduce churn. More often, reducing the cache (unpersisting DataFrames that are no longer needed) is better.

**4. Tune GC logging to diagnose**

Enable GC logging to see exactly what GC events are happening:
```
spark.executor.extraJavaOptions = -XX:+UseG1GC -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:/tmp/gc.log
```

GC log files on each executor show every GC event: type (minor/major/full), duration, heap before and after, and cause. Analyzing these logs pinpoints whether you have frequent minor GCs (too many short-lived objects) or infrequent but long major GCs (old generation filling up).

**5. Use serialized caching for large caches**

`MEMORY_ONLY` (deserialized) caches store JVM objects—they're fast to access but occupy the old generation. `MEMORY_ONLY_SER` or `MEMORY_AND_DISK_SER` stores compressed, serialized bytes in direct memory buffers, which don't burden the JVM garbage collector.

```python
df.persist(StorageLevel.MEMORY_AND_DISK_SER)
```

The tradeoff: accessing serialized cache requires deserialization on each access. But for large datasets that would otherwise cause old-gen pressure, serialized caching dramatically reduces GC load.

> **Serialized caching is like keeping your files in compressed ZIP archives instead of open folders on your desk.** Opening a ZIP is slightly slower (deserialization), but you can fit 5× more files in the same drawer (memory). Your desk (JVM old gen) stays tidy.

**6. Reduce object creation in transformations**

Codegen (Whole-Stage Code Generation) reduces JVM object creation by operating directly on primitive values and off-heap binary buffers instead of wrapping every value in a Boxed Integer or Row object. Most DataFrames automatically use codegen. The risk areas are:

- **Scala/Java UDFs**: opaque to codegen; every call creates objects
- **Python UDFs**: every row involves JVM object creation
- **`collect()` on large DataFrames**: brings millions of objects to the driver's JVM heap

For these patterns, reducing the amount of data processed (filter early, aggregate before collecting) and using pandas UDFs (Arrow batches instead of row objects) are the primary mitigations.

---

## Diagnosing GC problems

**Spark UI Stages tab → task metrics**: the "GC Time" column per task. If average GC time per task is >10% of average task duration, GC is impacting performance.

**Executor logs**: GC log output (if enabled) shows individual GC events. High frequency minor GCs (every second) indicate too many short-lived objects. Occasional major GCs of long duration indicate the old generation filling up.

**"GC overhead limit exceeded" error**: the JVM throws this error when more than 98% of time is spent in GC and less than 2% of heap is being freed. This is effectively OOM—the JVM is spending all its time collecting but can't free enough memory. Solutions: increase executor memory, reduce data size per task (more partitions), or reduce object creation.

**Long executor heartbeat gaps**: the driver receives heartbeats from executors every `spark.executor.heartbeatInterval` (default 10s). If a GC pause is longer than `spark.network.timeout` (default 120s, effectively), the driver assumes the executor is lost. Long full GC pauses are a major source of "executor lost" events that appear to be network problems.

---

## Bringing it together

JVM garbage collection is Spark's invisible tax on object-heavy workloads. The JVM's generational heap (young + old generation) garbage-collects in cycles; when the old generation fills, a long stop-the-world pause halts all executor threads. Spark-specific GC pressure comes from: per-row JVM objects in non-codegen paths, deserialization during shuffle, deserialized caching occupying the old generation, and Python UDF overhead. The key tuning levers are: **use G1GC** (better pause time behavior than Parallel GC), **increase executor memory** (fewer GC triggers), **switch to serialized caching** (reduces old-gen pressure), **use codegen paths** (fewer JVM objects), and **tune memory fractions** (balancing execution vs. storage). GC time in the Spark UI and GC log files are the diagnostic tools. So the story of GC in Spark is: **objects born in young gen → short-lived ones die fast (minor GC) → long-lived ones fill old gen → old gen fills → major GC stops the world → resume.** Reduce the flow of objects into the old generation, and the pauses shrink.
