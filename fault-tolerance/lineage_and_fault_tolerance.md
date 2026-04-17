# How Spark Survives Failure

This is the story of how Spark keeps going when something goes wrong. Executors crash, nodes disappear, tasks stall, and shuffle output vanishes—yet jobs complete. The answer is not magic but a set of interlocking mechanisms: **lineage**, **recomputation**, **checkpointing**, and **speculation**. Each solves a different part of the failure problem, and together they define what "fault tolerance" means in Spark.

---

## The foundation: lineage

Spark's most basic resilience mechanism is not checkpointing or replication—it is **lineage**. Every RDD knows where it came from: which parent RDD it was derived from and by what transformation. An RDD that was built by filtering another RDD holds a reference to that parent. The filtered RDD was built by a map over a file—so it holds a reference to the file read. Chain these references together and you have the full **lineage graph**: a recipe for recomputing any partition from scratch.

This is what makes RDDs "resilient." If a partition is lost—because the executor that held it crashed—Spark doesn't give up. It looks at the lineage of that partition and says: I know how to rebuild this. It finds the parent partitions (on other executors, or re-read from disk), applies the transformation, and produces the lost partition again. Recomputation is the default response to data loss, and it works as long as the lineage chain reaches something stable—a file in HDFS, a Kafka topic, another cached partition—that can be re-read.

---

## Narrow and wide dependencies: how much to redo

Not all losses are equally cheap to recover from. The cost depends on the **type of dependency** between the lost partition and its parents.

A **narrow dependency** means each child partition depends on at most one parent partition (think *map*, *filter*, *flatMap*). Losing a child partition means recomputing one parent partition, maybe one more above that—a small chain. You redo only the work that produced the lost partition.

A **wide dependency** is different. In a wide dependency (produced by a shuffle: *groupByKey*, *reduceByKey*, *join*), each child partition can depend on *every* parent partition. Losing a child partition after a shuffle means the driver may have to re-run the entire map stage to regenerate the shuffle output that the child's partition needs—not just one partition but potentially all of them. That is why shuffle boundaries are both stage boundaries and fault-recovery boundaries: losing shuffle data is expensive to recover from because it forces a whole-stage re-run.

This asymmetry shapes how Spark decides what to checkpoint (more on that shortly).

---

## Task retries: the first line of defense

Before lineage recomputation even enters the picture, Spark tries something simpler: **retrying the failed task**. When a task fails—an exception in user code, a network blip causing a heartbeat timeout, an executor running out of memory for a specific task—the executor reports the failure to the driver. The **Task Scheduler** can then resubmit that same task (same partition, new attempt) on a different or the same executor when resources become available. By default Spark allows up to four attempts per task before it considers the task truly stuck and fails the stage.

Task retries handle transient failures: a brief GC pause that looked like an executor death, a short network partition, a task that hit a rare data-dependent exception on one executor but works fine elsewhere. They cost little (just one task's worth of work) and resolve most failures silently.

---

## Stage retries and fetch failures: the shuffle problem

There is one failure type that task retries cannot fix: **fetch failures**. A reduce task needs shuffle data written by a map task on another executor. If that executor has died, its shuffle output is gone. Retrying the reduce task won't help—the data it needs doesn't exist anymore.

The **DAG Scheduler** handles this. When a task fails with a fetch failure, the DAG Scheduler does not count it against the per-task retry limit. Instead it looks at the lineage: which map stage produced the missing output? It then marks that map stage as **needing to be re-run** and resubmits it. All the map tasks in that stage run again (not just the one whose output was lost, because the driver may not know exactly which map output files still exist versus which are gone). Once the map stage completes, the reduce stage is resubmitted and can now fetch its data.

There is a limit on how many times a stage can be retried (controlled by `spark.stage.maxConsecutiveAttempts`). If a stage keeps failing—due to a persistent bad node, for example—Spark eventually gives up and fails the job.

---

## Checkpointing: cutting the lineage chain

Lineage-based recomputation is powerful, but it has a cost: the longer the lineage chain, the more work a recovery requires. An iterative algorithm (like PageRank or ALS) that runs for 20 iterations builds a lineage chain 20 transformations deep. Losing a partition near the end means replaying 20 steps. Worse, if that partition is an RDD used as the input to the *next* iteration, the chain may grow unboundedly.

**Checkpointing** solves this by writing an RDD's data to a **reliable external storage** (typically HDFS) and **cutting the lineage**. After checkpointing, the RDD no longer remembers its parent chain—its lineage starts fresh from the checkpoint file. Recovering a lost partition now means reading from HDFS, not replaying the full computation. The cost of checkpointing is that writing to HDFS takes time and space; the benefit is that recovery is fast and the lineage doesn't blow up.

There is also **local checkpointing**, which writes to the executor's local disk instead of HDFS. It is faster and cheaper, but it does not survive executor death—the files are lost with the executor. Local checkpointing is useful for shortening very long lineage chains when reliability is handled by other means (like having the application re-run from the beginning if the executor dies).

The standard advice for iterative jobs: cache the RDD at each iteration (so recomputation is cheap within the iteration) and checkpoint it every few iterations (so that a late failure doesn't trigger a full replay from iteration 0).

---

## Speculation: dealing with stragglers

A very different kind of failure is the **straggler**: a task that is running, not crashed, but is simply much slower than everyone else in its stage. One slow disk, one misconfigured node, one particularly skewed partition—and the whole stage is blocked waiting for a single task to finish. All the other executors sit idle.

**Speculation** is Spark's answer. A background thread periodically examines all running tasks in a stage. If a task has been running significantly longer than the median of the already-completed tasks in that stage (and the stage has enough completed tasks to make a meaningful comparison), Spark launches a **speculative duplicate** of that task on a different executor. Both the original and the duplicate run in parallel. Whichever finishes first wins; its result is used, and the other is killed.

Speculation is disabled by default because it isn't always safe: if the task has side effects (writing to an external system, for instance), running two copies could corrupt the output. For pure Spark transformations—where task output is a partition result or a shuffle write—speculation is safe and can significantly reduce job latency when stragglers are common. For writing to external sinks, you need to think carefully.

---

## Replication: redundancy without recomputation

For data that is cached at a high storage level (e.g. `MEMORY_AND_DISK_2`), Spark will **replicate** the cached block to a second executor. If the primary executor dies, the replica is immediately available on another executor—no recomputation required. Replication is the fastest possible recovery for cached data, at the cost of using double the memory and network bandwidth during the cache write. It is most useful when recomputing the lost partition is expensive (a long lineage, or data read from a slow source) and the cached data is frequently accessed.

---

## Bringing it together

Spark's fault tolerance is layered. At the bottom is **lineage**: every RDD knows its parents, and a lost partition can be recomputed by replaying the transformation chain from stable storage. Above that is **task retry**: transient failures are recovered by simply running the task again. When shuffle output is lost, the **DAG Scheduler** steps in and re-runs the entire map stage that produced it, then re-runs the reduce stage. For iterative jobs where the lineage chain would make recovery prohibitively expensive, **checkpointing** cuts the chain and stores a snapshot in reliable storage. **Speculation** addresses the different problem of stragglers by launching duplicate tasks for anything that looks too slow. And **replication** makes recovery instantaneous for cached data, at the cost of extra storage. Together, these mechanisms mean that Spark can complete a job even when executors disappear, nodes fail, and shuffle data evaporates—as long as the original data source is still reachable and the failures aren't so pervasive that the entire cluster is gone.
