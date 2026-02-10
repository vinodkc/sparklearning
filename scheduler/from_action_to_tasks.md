# From One Action to Many Tasks

This is the story of how a single call—like `count()` or `save()`—turns into a directed acyclic graph (DAG) of **stages**, and then into hundreds or thousands of **tasks** running on executors. Two pieces do most of the work: the **DAG Scheduler**, which decides *what* to run (stages and their order), and the **Task Scheduler**, which decides *where* to run it (which task on which executor). Understanding this flow explains why your job has the stages it has, and why Spark sometimes waits before running a task.

---

## The action: something must happen

In Spark, transformations (e.g. `map`, `filter`) are lazy. They only describe a lineage of RDDs. Nothing runs until an **action** is called: `count()`, `collect()`, `saveAsTextFile()`, or the execution of a DataFrame or SQL query that ends in a write or a collect. That action, on the driver, triggers a **job**: “run this function over these partitions of this RDD and give me the result (or write it out).” The job is the unit of work that the scheduler turns into stages and tasks.

---

## The DAG: from RDD lineage to stages

The target RDD of the job does not exist in isolation. It was built from other RDDs by a chain of transformations. Some of those transformations are **narrow**: each partition of the child depends only on one partition of the parent (e.g. `map`). Others are **wide**: each partition of the child can depend on many partitions of the parent (e.g. `groupByKey`, `reduceByKey`). That wide dependency is a **shuffle**: data must be repartitioned. So the RDD lineage is a DAG of RDDs, and the **shuffle dependencies** are the natural boundaries where the driver must break the work into separate steps. You cannot run the “reduce” side of a shuffle until the “map” side has produced its output.

The **DAG Scheduler** takes the job (target RDD, the function to run, which partitions) and walks backward through the RDD lineage. Whenever it hits a shuffle dependency, it draws a boundary: everything from that dependency’s RDD back to the next shuffle (or to the source) becomes one **stage**. So you get **ShuffleMapStages** (they produce shuffle output) and a final **ResultStage** (they run the action and produce the result). The DAG of RDDs becomes a DAG of stages. Parents of a stage must complete before the stage can run.

---

## Submitting stages: parents first

The DAG Scheduler does not submit all stages at once. It starts from the **final stage** (the ResultStage for the job). When it tries to submit a stage, it first checks: *are all my parent stages done?* If not, it recursively submits the missing parents and puts the current stage in a waiting list. When a parent stage completes, the DAG Scheduler checks the waiting list: any stage whose parents are now all done becomes eligible and is submitted. So stages run in **dependency order**: all map stages that feed a shuffle run before the reduce stage that consumes it.

Submitting a stage means: create one **task** per partition of the stage’s RDD (or the relevant subset for actions like `first()`), attach preferred locations (from the RDD’s metadata—e.g. which block is where), wrap them in a **TaskSet**, and hand that TaskSet to the **Task Scheduler**.

---

## The Task Scheduler: matching tasks to executors

The Task Scheduler’s job is to take pending tasks and run them on executors. It does not push tasks out on its own. The **SchedulerBackend** (which talks to the cluster manager and the executors) receives **resource offers**: “Executor X has N free cores.” When an offer arrives, the Task Scheduler asks the TaskSetManager: *given this executor and these cores, which of your pending tasks do you want to run here?* The TaskSetManager chooses tasks, respecting **locality**: it prefers to run a task on an executor that already has the data the task needs (process-local, then node-local, then rack-local, then any). So the same executor that held a partition in the previous stage often runs the task that consumes it in the next—data stays put when possible.

If no task can use the offered executor with acceptable locality, the TaskSetManager may **decline** the offer and wait. That’s **delay scheduling**: Spark is willing to wait a bit (configurable) for a “better” offer so that more tasks run with data local to them. After the wait, it relaxes locality and takes any executor. So you sometimes see a short pause before tasks launch—Spark is holding out for locality.

The backend then serializes the chosen tasks and sends them to the executors. Each executor runs its tasks in a thread pool, reports success or failure back to the driver, and the DAG Scheduler updates the stage’s state. When every task in a stage has finished successfully, the stage is done. If it was a ShuffleMapStage, its output is now available (the driver knows where each map output is); any child stage that was waiting on it can now be submitted. If it was a ResultStage, the job’s result (or write) is complete and the job finishes.

---

## Failure and speculation

### Who retries what

- **Failed task retry** is done by the **Task Scheduler** (and its **TaskSetManager**). When a task fails, the executor sends a status update to the driver; the TaskScheduler marks the task failed and can **resubmit that same task** (a new attempt for the same partition) when it gets new resource offers. It retries each task up to a per-stage limit (`spark.task.maxFailures`, default 4). If a given task hits that limit, the TaskSetManager **aborts the whole TaskSet** and notifies the DAGScheduler via `taskSetFailed`; the DAGScheduler then aborts that stage and the job fails—there is no stage retry for “too many task failures.”

- **Failed stage retry** is done by the **DAG Scheduler**. When a task fails because **shuffle output was lost** (e.g. `FetchFailed`—a reducer could not fetch a map output—or the executor that held the output was lost), the TaskSetManager does *not* count that failure toward the task-failure limit; it marks the TaskSet as zombie and the completion event still goes to the DAGScheduler. The DAGScheduler then figures out which **stage(s)** must be re-run (the map stage that produced the lost output, and any dependent stages), adds them to `failedStages`, and after a short delay posts **ResubmitFailedStages**. When that runs, it calls `submitStage(stage)` again for each failed stage, which creates a **new stage attempt** and a new TaskSet with fresh tasks. So: **TaskScheduler** retries **tasks** (same task, new attempt); **DAGScheduler** retries **stages** (whole stage resubmitted) when the failure is due to lost shuffle (or similar) data. The DAGScheduler also enforces stage-level retry limits (e.g. `spark.stage.maxConsecutiveAttempts`).

### Speculation

**Speculation** is also on the Task Scheduler side. A periodic thread in **TaskSchedulerImpl** calls `checkSpeculatableTasks()`, which asks each **TaskSetManager** whether any running task is slow compared to others in the same stage (e.g. beyond a multiple of the median duration). If so, the TaskSetManager marks that task as speculatable and, when it gets a resource offer, can launch a **duplicate** copy of the same task on another executor. Whichever copy finishes first wins; the other is cancelled. The DAGScheduler is only notified (`speculativeTaskSubmitted`) for logging and listeners; it does not decide or run speculation.

---

## Bringing it together

One action becomes one job. The DAG Scheduler turns the job’s RDD lineage into a DAG of stages by cutting at every shuffle dependency. It submits stages in dependency order: parents first, then children when parents complete. Each stage becomes a TaskSet of one task per partition. The Task Scheduler takes resource offers from the cluster, picks tasks (preferring locality, using delay scheduling when useful), and sends those tasks to executors. When all tasks of a stage complete, the next stage can run. So the journey is: **action → job → DAG of stages → TaskSets → tasks matched to executors → execution and completion.** The shuffle is the boundary between stages; the DAG Scheduler and Task Scheduler are what turn that boundary into the execution you see.
