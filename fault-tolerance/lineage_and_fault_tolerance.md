# How Spark Survives Failure

This is the story of how Spark keeps going when something goes wrong. Executors crash, nodes disappear, tasks stall, and shuffle output vanishes—yet jobs complete. The answer is not magic but a set of interlocking mechanisms: **lineage**, **recomputation**, **checkpointing**, and **speculation**. Each solves a different part of the failure problem, and together they define what "fault tolerance" means in Spark.

---

## The foundation: lineage

Spark's most basic resilience mechanism is not checkpointing or replication—it is **lineage**. Every RDD knows where it came from: which parent RDD it was derived from and by what transformation. An RDD that was built by filtering another RDD holds a reference to that parent. The filtered RDD was built by a map over a file—so it holds a reference to the file read. Chain these references together and you have the full **lineage graph**: a recipe for recomputing any partition from scratch.

> **Think of an RDD as a recipe, not a dish.** If you drop the dish (lose a partition), you don't go to the restaurant to get another one—you go back to the recipe and cook it again from the ingredients. As long as the recipe (lineage) and the raw ingredients (source data) exist, you can always recreate the dish. Spark never stores the dish permanently; it just remembers the recipe.

This is what makes RDDs "resilient." If a partition is lost—because the executor that held it crashed—Spark doesn't give up. It looks at the lineage of that partition and says: I know how to rebuild this. It finds the parent partitions (on other executors, or re-read from disk), applies the transformation, and produces the lost partition again. Recomputation is the default response to data loss, and it works as long as the lineage chain reaches something stable—a file in HDFS, a Kafka topic, another cached partition—that can be re-read.

---

## Narrow and wide dependencies: how much to redo

Not all losses are equally cheap to recover from. The cost depends on the **type of dependency** between the lost partition and its parents.

A **narrow dependency** means each child partition depends on at most one parent partition (think *map*, *filter*, *flatMap*). Losing a child partition means recomputing one parent partition, maybe one more above that—a small chain. You redo only the work that produced the lost partition.

> **Narrow dependency recovery is like remaking one page of a document.** You lost page 12? Retype page 12. You don't have to retype the whole book.

A **wide dependency** is different. In a wide dependency (produced by a shuffle: *groupByKey*, *reduceByKey*, *join*), each child partition can depend on *every* parent partition. Losing a child partition after a shuffle means the driver may have to re-run the entire map stage to regenerate the shuffle output that the child's partition needs—not just one partition but potentially all of them.

> **Wide dependency recovery is like losing one chapter of a book that was assembled by cutting and rearranging pages from every other chapter.** To recover that one chapter, you need to go back to all the source chapters—you can't just redo one page. That is why shuffle boundaries are the expensive fault-recovery boundaries.

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

> **It's like saving your progress in a video game.** Without saves, dying sends you all the way back to level 1. With a checkpoint every 5 levels, the worst case is replaying 5 levels—not the entire game. The save file cuts the lineage: after loading from the save, the game doesn't remember how you got there, only where you are now.

**Checkpointing** solves this by writing an RDD's data to a **reliable external storage** (typically HDFS) and **cutting the lineage**. After checkpointing, the RDD no longer remembers its parent chain—its lineage starts fresh from the checkpoint file. Recovering a lost partition now means reading from HDFS, not replaying the full computation. The cost of checkpointing is that writing to HDFS takes time and space; the benefit is that recovery is fast and the lineage doesn't blow up.

There is also **local checkpointing**, which writes to the executor's local disk instead of HDFS. It is faster and cheaper, but it does not survive executor death—the files are lost with the executor. Local checkpointing is useful for shortening very long lineage chains when reliability is handled by other means.

The standard advice for iterative jobs: cache the RDD at each iteration (so recomputation is cheap within the iteration) and checkpoint it every few iterations (so that a late failure doesn't trigger a full replay from iteration 0).

---

## Speculation: dealing with stragglers

A very different kind of failure is the **straggler**: a task that is running, not crashed, but is simply much slower than everyone else in its stage. One slow disk, one misconfigured node, one particularly skewed partition—and the whole stage is blocked waiting for a single task to finish. All the other executors sit idle.

**Speculation** is Spark's answer. A background thread periodically examines all running tasks in a stage. If a task has been running significantly longer than the median of the already-completed tasks in that stage, Spark launches a **speculative duplicate** of that task on a different executor. Both the original and the duplicate run in parallel. Whichever finishes first wins; its result is used, and the other is killed.

> **It's like a relay race where one runner is clearly lagging.** Rather than waiting for the tired runner to finally reach the handoff point, the coach sends a fresh runner from the bench to run the same leg on a parallel track. Whoever reaches the finish line first passes the baton; the other stops. The race isn't held up by one bad leg.

Speculation is disabled by default because it isn't always safe: if the task has side effects (writing to an external system, for instance), running two copies could corrupt the output. For pure Spark transformations, speculation is safe and can significantly reduce job latency when stragglers are common.

---

## Replication: redundancy without recomputation

For data that is cached at a high storage level (e.g. `MEMORY_AND_DISK_2`), Spark will **replicate** the cached block to a second executor. If the primary executor dies, the replica is immediately available on another executor—no recomputation required.

> **Replication is like keeping a photocopy of an important document in two different drawers.** If one drawer is lost in a fire, you still have the copy in the other drawer—no need to reconstruct the document from scratch. The cost is using two drawers instead of one.

Replication is the fastest possible recovery for cached data, at the cost of double the memory and network bandwidth during the cache write. It is most useful when recomputing the lost partition is expensive (a long lineage, or data read from a slow source) and the cached data is frequently accessed.

---

## Bringing it together

Spark's fault tolerance is layered. At the bottom is **lineage**: every RDD knows its parents, and a lost partition can be recomputed by replaying the transformation chain from stable storage—like rebuilding a dish from its recipe. Above that is **task retry**: transient failures are recovered by simply running the task again. When shuffle output is lost, the **DAG Scheduler** steps in and re-runs the entire map stage. For iterative jobs, **checkpointing** cuts the lineage chain and stores a snapshot in reliable storage—like a game save that prevents replaying from level 1. **Speculation** addresses stragglers by launching duplicate tasks in parallel, like a backup relay runner. And **replication** makes recovery instantaneous for cached data by keeping two copies. Together, these mechanisms mean that Spark can complete a job even when executors disappear, nodes fail, and shuffle data evaporates—as long as the original data source is still reachable and the failures aren't so pervasive that the entire cluster is gone.
