# The Journey of a Shuffle Record

This is the story of how one record travels from the task that produced it to the task that consumes it—across the boundary between stages that we call **shuffle**. Understanding this journey explains why your job has multiple stages, where shuffle time goes, and what happens when an executor disappears.

---

## Why shuffle exists

In Spark, a job is split into **stages**. Within a stage, each partition is processed independently: no data passes between tasks. Between stages, data *must* move: every record produced in one stage and needed in the next has to find its way to the right place. That movement is **shuffle**.

So shuffle is not an accident—it’s the bridge. Whenever you do a `groupBy`, `join`, or `repartition`, Spark draws a line: everything before the line is one stage (map side), everything after is another (reduce side). The map tasks write; the reduce tasks read. The journey of a shuffle record is the story of that write and that read.

---

## The map side: “Where should this record go?”

Every record has a **key** (or is assigned a partition). The **partitioner** decides which reduce partition will need it. So the first thing that happens to a shuffle record on the map side is: *which partition number is mine?*

The map task doesn’t send records one-by-one to the reducers. Instead, it **writes** them into local storage in a way that the reduce tasks can later **read** by partition. Today, that almost always means **sort-based shuffle**.

---

## Sort-based shuffle: one output per map task

In sort-based shuffle, each map task produces **one logical output**: a single file (or set of files) that contains all the records emitted by that task, organized so that reduce partition 0’s records are in one stretch, partition 1’s in another, and so on. That way, a reduce task only has to read the stretches that belong to its partition(s).

To build that output, the map task doesn’t write directly to the final file. It **buffers** records in memory. As records arrive, they are sorted (or grouped) by partition id—and sometimes by key within partition—so that when the buffer is flushed, it’s already in the right order. If the buffer gets too big, the excess is **spilled to disk**: sorted chunks that will later be merged. So the record you’re following might live in memory for a while, then end up in a spill file, and finally in the merged map output. Serialization happens early (often as soon as the record is produced) so that what’s buffered and spilled is bytes, not Java objects—that keeps memory and GC under control.

When the map task finishes, it flushes any remaining buffer, merges all spills (and in-memory segments) into one contiguous layout per partition, and writes that out. It also writes a small **index** that says “partition 0 starts here and is this long, partition 1 starts there and is that long, …” so that reduce tasks know exactly where to read. The task then reports to the driver: *I’m done; my output is at this executor, in this file; here are the sizes of each partition’s slice.* That report is what the driver stores in the **MapOutputTracker**: the driver is the single place that knows, for every map task, where its output lives and how big each partition is.

So by the end of the map side, your record sits in a file on one executor’s disk (or in that executor’s block manager), and the driver knows it’s there.

---

## The reduce side: “Where did my records go?”

A reduce task is responsible for one (or a few) reduce partitions. To get *all* records for its partition(s), it must read the corresponding slice from **every** map task’s output. So the reduce task first asks the driver (via MapOutputTracker): *For this shuffle, give me the list of (executor, block id, size) for every map output that contains my partition.* The driver answers with a list of blocks—some on the same executor as the reduce task (local), some on others (remote).

The reduce task then **fetches** those blocks. Local blocks are read from the local block manager. Remote blocks are requested over the network from the executors (or from the External Shuffle Service, if it’s running) that hold them. Fetches are throttled so that the reducer doesn’t pull too much data into memory at once: there’s a limit on how many bytes (or blocks) can be in flight. As each block arrives, it’s decompressed and deserialized, and the records are passed to the rest of the reduce task (e.g. into an aggregator or a sorter). So your record, which was serialized and perhaps compressed on the map side, is now read from a remote (or local) block, decompressed, deserialized, and handed to the operator that runs the reduce logic.

If the reducer supports it, contiguous blocks from the same executor can be fetched in one request (batch fetch), which reduces overhead. Either way, the reducer sees a single logical stream of records for its partition, assembled from many map outputs.

---

## Streaming, not waiting: the reducer starts as soon as the first block arrives

The reducer does **not** wait for all shuffle blocks to be present before it starts applying reduce logic. It works in a **streaming, pipelined** way.

The shuffle read is implemented as an **iterator**: when the reduce logic (e.g. an aggregator or the next operator) asks for the next record, that iterator may be reading from a block that’s already been fetched. If the current block is exhausted, the iterator moves to the next block—which might already be in memory (prefetched) or might trigger a wait for that block to be fetched. So the reducer **starts consuming records as soon as the first block is available** and keeps consuming while more blocks are still in flight. Fetches are throttled (e.g. by a limit on how many bytes can be in flight at once), but several blocks can be fetched in parallel. So while the reducer is processing records from block one, the fetcher can already be bringing in blocks two, three, and so on. Fetch and process overlap.

If the reduce side needs to **sort by key**, all records are fed into an external sorter—but they are still **fed in as they arrive**. The sorter receives a stream of records and may spill to disk as it goes. We don’t wait for every block to be fetched before starting to insert into the sorter; we stream into the sorter while more blocks are still being fetched.

So: the reducer starts applying reduce logic as soon as the first block is available and continues in a stream. It does not wait for all required shuffle data to be local first.

---

## When the executor that wrote the data is gone

Map output lives on the executor that ran the map task. If that executor is still running when the reduce task runs, the block is fetched from that executor. But if the executor has died (crashed, preempted, or been removed), its in-memory blocks are gone. Its **shuffle files** might still be on disk—if Spark (or the cluster) wrote them to local disk that survives the process, or if an **External Shuffle Service** is running.

The External Shuffle Service is a separate process (often on the same node) that can serve shuffle files written by executors on that node. When the service is enabled, map output is written so that the service can read it. When a reduce task asks for a block and the original executor is gone, the fetcher can ask the External Shuffle Service on that host instead. So your record can still be read—the journey continues—as long as the files are on disk and the service is there to serve them. If the node itself is gone, or the service isn’t running, the fetch fails. The reduce task then reports a **fetch failure**. Spark responds by re-running the *map* stage (and any earlier stages) so that the lost output is regenerated; then the reduce stage runs again. So shuffle is also the moment where Spark’s fault tolerance becomes visible: losing an executor often means redoing the map stage so that shuffle data can be re-read.

---

## Why sort-based shuffle (and not hash shuffle)

Older Spark used a “hash” shuffle: each map task wrote one file per reduce partition. With many reduce partitions, that meant many open files and a lot of random I/O. Sort-based shuffle replaced that: one (logical) output per map task, with records sorted by partition id so that each partition’s slice is a contiguous segment. Reducers then read contiguous segments instead of many tiny files. That reduces open-file pressure, improves I/O patterns, and scales better when the number of partitions is large. So the “sort” in sort-based shuffle is primarily about **organization**—by partition, and optionally by key—to make the reduce side efficient and to avoid the limitations of the old hash shuffle.

---

## Bringing it together

A shuffle record is produced in a map task, assigned a partition, buffered (and possibly spilled) in sorted form, then written into that map task’s output file with an index. The driver records where that output lives. On the reduce side, the reducer asks the driver for every block that contains its partition, then fetches those blocks—local or remote, or from the External Shuffle Service if the original executor is gone. It does **not** wait for all blocks to arrive: it consumes in a streaming way, processing records as blocks become available (with prefetch so fetch and process overlap). The record is decompressed, deserialized, and fed into the reduce logic. If a fetch fails and the data can’t be recovered, the map stage is re-run. So the journey of a shuffle record is: **produce → buffer and sort → spill if needed → merge and write → register with the driver → fetch on the reduce side → deserialize and consume.** That journey is why shuffle is expensive—and why it’s the boundary between stages.
