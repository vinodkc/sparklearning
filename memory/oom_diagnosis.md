# Out of Memory: A Field Guide to Spark OOM Errors

This is the story of how to understand and fix out-of-memory errors in Spark. OOM errors are among the most common and most frustrating Spark problems: the job dies, the error message points to a line deep in the JVM stack, and it's not obvious what ran out of memory, why, or how much more it needed. The key to fixing OOM errors is knowing *which* memory was exhausted, *why* it grew beyond its limit, and *what* to change. This story maps out the main OOM scenarios, their symptoms, and their remedies.

---

## Spark's memory model: a quick map

Before diagnosing OOMs, you need to know how Spark's memory is divided. Each executor JVM has:

- **Reserved memory**: a small fixed amount (~300 MB) held back by Spark for internal objects—the JVM class overhead, Spark's internal components. You can't use this; it's always reserved.
- **User memory**: a fraction of the remaining heap (default 40%) for user code—objects you create explicitly in closures, hash maps in custom aggregations, Python UDF results.
- **Spark memory pool**: the remainder (default 60%), shared between storage (cached blocks) and execution (sort buffers, hash maps for joins and aggregations, shuffle input). Storage and execution share this pool dynamically, with execution able to evict storage when needed.
- **Off-heap memory**: optional native memory for Tungsten operations.

The driver has its own heap, separate from executors.

---

## OOM type 1: executor heap — execution memory

**Symptom**: task fails with `java.lang.OutOfMemoryError: Java heap space` or `GC overhead limit exceeded`. The stage fails; if retries exhaust the limit, the job fails. Often appears on tasks with large inputs or during aggregations and joins on wide data.

**Cause**: the task's working memory (sort buffer, aggregation hash map, join build-side hash map) grew beyond the executor heap's capacity. The most common triggers:

- **Too few shuffle partitions**: if `spark.sql.shuffle.partitions = 200` but you're processing 2 TB of data, each partition is ~10 GB. One task must hold that in memory. Raising the partition count to 2000 reduces each partition to ~1 GB.
- **Large join build side**: a shuffled hash join or broadcast join where the build side is larger than expected. If Spark chose broadcast but the small table is actually 500 MB per executor, the executor OOMs trying to hold the hash map.
- **Unbounded aggregation**: a `groupBy` with very high cardinality (many distinct keys) causes the in-memory hash map to grow large before spilling.
- **User code allocating large objects**: a closure that builds a large Java collection for each partition.

**Fixes**:
- Increase `spark.sql.shuffle.partitions` to reduce per-task data volume.
- Increase executor memory (`--executor-memory`) to give each task more headroom.
- If a broadcast join is causing it, reduce `spark.sql.autoBroadcastJoinThreshold` to prevent broadcast of the oversized table and fall back to sort-merge join.
- Enable spill: execution memory can spill to disk when it's insufficient. Spill is slow, but it's better than an OOM. If you see spill in the UI, the fix is more memory or fewer rows per partition; if there's no spill and the task OOMs, execution memory ran out before it could spill.

---

## OOM type 2: executor heap — user memory and large closures

**Symptom**: executor OOMs, but the task input is small and there's no obvious aggregation. The stack trace points to user code or a large collection being iterated.

**Cause**: code in the closure captures or builds a large data structure. Common examples: reading the entire partition into a list, building a large HashMap from data in the task, or loading a large model file inside the task function (which should be broadcast instead).

**Fixes**:
- **Broadcast large objects** instead of capturing them in the closure. A 100 MB lookup table broadcast to every executor is loaded once; if it's captured in the closure, it's deserialized once per task.
- Move large object creation to a lazy `lazy val` or singleton inside the task so it's initialized once per executor (via a connection pool pattern or `mapPartitions`) rather than once per row.
- Use `mapPartitions` instead of `map` for operations that require per-partition initialization—one initialization per partition instead of one per row.

---

## OOM type 3: executor off-heap

**Symptom**: executor dies with `Direct buffer memory` error or the native process is killed by the OS (out-of-memory killer) even though JVM heap usage looks fine. In Kubernetes, the pod is OOMKilled.

**Cause**: memory used outside the JVM heap—Arrow buffers, off-heap Tungsten storage, or native library allocations—exceeded the container or OS memory limit. The JVM heap limit (`-Xmx`) only controls heap memory; native and direct buffer memory has a separate budget.

**Fixes**:
- Set `spark.executor.memoryOverhead` (YARN) or `spark.executor.memoryOverheadFactor` to reserve more non-heap memory for the executor container. Default is 10% of executor memory or 384 MB, whichever is larger—often insufficient for Arrow-heavy PySpark workloads.
- Reduce Arrow batch size (`spark.sql.execution.arrow.maxRecordsPerBatch`) to reduce peak Arrow buffer memory.
- If using off-heap Tungsten (`spark.memory.offHeap.enabled = true`), ensure `spark.memory.offHeap.size` is accounted for in the container memory request.

---

## OOM type 4: driver heap

**Symptom**: driver fails with `java.lang.OutOfMemoryError`. Job submission fails, or the job dies partway through with the driver OOMing.

**Cause**: the driver heap is used for: the application code, the DAG/stage metadata, task result objects, broadcast variable data (before it's sent to executors), and any data collected from executors (via `collect()`, `toPandas()`, `take()`). The most common driver OOM causes:

- **`collect()` on a large DataFrame**: `collect()` brings all rows to the driver. For a 10 GB DataFrame this puts 10 GB of data on the driver heap.
- **Large broadcast variables**: broadcasting a multi-gigabyte object puts it in the driver heap before distribution.
- **Accumulating task results**: each task's result (for small result stages) is sent to the driver. If you have millions of tiny tasks each returning a significant result, the driver accumulates them all.
- **Too many cached RDD partitions tracked**: the BlockManagerMaster on the driver tracks every cached block across all executors. Extremely large numbers of tiny cached blocks can stress driver memory.

**Fixes**:
- Replace `collect()` with `write` to storage and read back separately. Never collect a DataFrame you haven't sized first.
- Set `--driver-memory` high enough for the broadcast variables and result accumulation your job needs.
- For `toPandas()` calls, ensure Arrow optimization is enabled so data transfers in batches rather than accumulating all in the driver heap at once.

---

## OOM type 5: Python worker (PySpark)

**Symptom**: task fails with a Python process crash or `MemoryError` from Python. The error may appear as a Spark executor error wrapping a Python exception.

**Cause**: the Python worker process—a separate OS process from the executor JVM—ran out of memory. For row-at-a-time UDFs, each row is processed individually so this is rare. For `mapPartitions` with Python or `ForeachBatch` writing large objects, the entire partition or batch may be materialized in Python memory at once.

**Fixes**:
- Process data in smaller chunks within Python rather than accumulating the entire partition.
- Use `spark.python.worker.memory` to configure the Python worker's memory limit (used for accounting, not a hard OS limit).
- For pandas UDFs, reduce `spark.sql.execution.arrow.maxRecordsPerBatch` to send smaller Arrow batches to Python.

---

## Diagnosis: reading the error message and the UI

The OOM error message and stack trace tell you which heap ran out. `Java heap space` is the JVM heap—execution, storage, or user memory. `GC overhead limit exceeded` means the JVM spent more than 98% of its time in GC trying to free memory—effectively the heap is full and GC can't reclaim enough. `Direct buffer memory` is off-heap. A pod OOMKill on Kubernetes points to container-level memory (both JVM + off-heap + OS overhead).

The Spark UI's **Executors tab** shows GC time per executor. High GC fraction (> 20–30%) means memory pressure before the OOM hit. The **Stage detail** shows spill metrics: if spill is present, memory was insufficient but the job didn't die—increasing memory or partition count would reduce spill. If there's no spill and the job died, execution memory was exhausted before it could begin spilling (can happen with very large hash maps in joins or aggregations that aren't spill-friendly).

---

## The OOM prevention checklist

- Set shuffle partitions to keep per-partition data under 200–300 MB (estimate: total shuffle data in bytes / desired partition size in bytes).
- Use broadcast joins only for truly small tables; check AQE's runtime broadcast decisions in the UI.
- Never `collect()` without knowing the data size; prefer `write`.
- Set `memoryOverhead` generously for PySpark and Arrow-heavy workloads.
- Cache selectively: don't cache more data than fits in the storage fraction of the executor heap.
- Broadcast large lookup tables instead of capturing them in closures.
- For long lineage chains in iterative jobs, checkpoint periodically to bound the lineage graph size.

---

## Bringing it together

Spark OOM errors fall into five categories, each with a distinct cause and remedy. **Executor execution OOM**: per-task working memory (sort buffers, hash maps) exceeds the heap—fix with more partitions, more executor memory, or a different join strategy. **Executor user memory OOM**: large objects allocated by user code in closures—fix with broadcast variables or `mapPartitions`. **Off-heap/container OOM**: native memory (Arrow, Tungsten off-heap) exceeds the container limit—fix with `memoryOverhead` or `offHeap.size`. **Driver OOM**: `collect()`, large broadcasts, or result accumulation exhausts driver heap—fix with `--driver-memory`, write instead of collect. **Python worker OOM**: too much data materialized in the Python process—fix with smaller batches or chunked processing. The Spark UI (GC time, spill metrics, task duration) and the OOM stack trace together point to which category you're in. Each category has a different fix, and diagnosing which memory ran out is the first and most important step.
