# The Driver, the Executors, and How a Job Actually Runs

This is the story of how a Spark application is structured and how it turns a user's intent into computation spread across a cluster. It covers the two roles every Spark application has—the **driver** and the **executors**—what each one does, and how they collaborate to execute a job from the moment an action is called to the moment results are returned. Understanding this model is the foundation for understanding everything else: scheduling, fault tolerance, memory management, and performance.

---

## Two roles: coordinator and worker

Every Spark application has exactly one **driver** process and one or more **executor** processes. They play fundamentally different roles, and understanding the division is essential.

The **driver** is the brain. It is where your application code runs—the `main()` function, the SparkSession creation, all the DataFrame transformations you write, and the actions that trigger execution. When you write `df.filter(...).groupBy(...).count()`, those calls run on the driver and build up a logical plan in memory. The driver also hosts the DAG Scheduler, the Task Scheduler, the BlockManagerMaster, the MapOutputTracker, and the broadcast coordinator—all the components that orchestrate what the cluster does. The driver never processes your actual data rows; it processes metadata about them.

The **executors** are the hands. Each executor is a long-lived JVM process running on a worker node in the cluster. Its job is to run tasks—the actual units of computation—and to hold cached data in memory or on disk. Executors are assigned CPU cores (slots) and memory. Each slot can run one task at a time. An executor with 4 cores runs up to 4 tasks in parallel. Executors don't make planning decisions; they receive tasks from the driver, run them, and report results back.

---

## Application startup: claiming resources

Before any computation can happen, the driver must negotiate resources with the **cluster manager**—YARN, Kubernetes, Mesos, or Spark Standalone. The driver registers with the cluster manager and says: "I need N executors, each with M cores and P GB of memory." The cluster manager finds available nodes, starts executor JVM processes on them, and registers those executors with the driver. The driver now knows about all its executors: their addresses, their available cores, and their memory.

This negotiation happens once at startup (for static allocation) or continuously throughout the application's lifetime (for dynamic allocation, where Spark can request more executors when the task queue is long and release them when idle). Dynamic allocation means the cluster doesn't have to reserve resources for a Spark job that's spending 80% of its time waiting for the next batch.

Once at least one executor has registered, the driver can start submitting tasks. It doesn't need all executors to be ready—it starts as soon as it has enough to make progress.

---

## The SparkContext and SparkSession: the application handle

On the driver side, the **SparkContext** (or `SparkSession` in modern Spark, which wraps it) is the application's handle to everything. Creating a `SparkSession` starts the driver's internal services: the DAG Scheduler, the Task Scheduler, the Block Manager Master, the event log writer. It also triggers the cluster manager negotiation. Once `SparkSession` is created, the application is live: executors will be (or are being) allocated, and the driver is ready to accept jobs.

The `SparkSession` also provides the unified entry point for RDD operations, DataFrame/Dataset operations, and SQL queries. All three translate to the same underlying machinery: an RDD lineage or a logical plan, submitted as a job to the DAG Scheduler.

---

## From action to job: the trigger

Spark is lazy. DataFrame transformations (`filter`, `select`, `join`, `groupBy`) only build a description—a logical plan on the driver. Nothing runs on the executors. This laziness is intentional: it lets Catalyst see the whole pipeline and optimize it before choosing how to execute.

An **action** breaks the laziness. Calling `.count()`, `.collect()`, `.show()`, `.write.save(...)`, or `.foreach(...)` says: I need results now. The driver takes the logical plan accumulated so far, runs it through Catalyst (parsing, analysis, optimization, physical planning, codegen—covered in the Catalyst story), and hands the resulting physical plan to the **DAG Scheduler**. The DAG Scheduler sees the physical plan as an RDD DAG: a graph of RDD transformations with shuffle boundaries. It cuts the graph at shuffle boundaries to form **stages** and submits those stages (parents first) to the **Task Scheduler**.

A single action creates exactly one **job** from the Spark scheduler's perspective. A job has a unique job ID, a set of stages, and a result. Multiple actions in sequence produce multiple jobs.

---

## Tasks: the unit of work

A stage is made of **tasks**—one task per partition. If a stage reads 200 partitions of data, it has 200 tasks. Each task runs the same code but on a different slice of the data. Tasks are the atoms of Spark execution: the scheduler assigns them to executor slots, executors run them, and the results flow back.

A task is essentially a closure: the transformation function (compiled into bytecode, potentially via whole-stage codegen) plus a description of which partition to read. The Task Scheduler serializes this closure and sends it to the assigned executor. The executor deserializes it, finds (or reads) the input partition, runs the function, and either returns the result to the driver (for result stages) or writes shuffle output to local storage (for map stages). Task duration includes everything: reading the input, deserializing the task, running user code, writing output, and serializing and sending the result.

---

## The heartbeat and executor health

Executors send **heartbeats** to the driver periodically (every few seconds). Each heartbeat carries two things: "I'm still alive" and accumulator updates (partial values from running tasks that contribute to accumulators defined in the driver). If the driver doesn't receive a heartbeat from an executor within a timeout, it assumes the executor is dead and marks all its in-progress tasks as failed. The DAG Scheduler then re-queues those tasks for other executors, and—if shuffle data was on the dead executor—schedules re-execution of the map stages that produced it.

The heartbeat mechanism is how Spark detects executor failures without requiring the cluster manager to proactively notify it. In practice, a dead executor is noticed within one heartbeat timeout (default 120 seconds), which sets a floor on fault-recovery latency.

---

## Driver failure: the whole application is lost

The driver is a single point of failure in a standard Spark application. If the driver process dies, the application is lost: all executors are released, all cached data is gone, all in-flight tasks are abandoned. There is no automatic driver restart unless the cluster manager is configured to support it (YARN has a `--driver-cores` and `--supervise` option for Spark Standalone; Kubernetes can restart driver pods). For long-running Spark Streaming or Structured Streaming applications, driver recovery is critical and is implemented by checkpointing the driver's state to durable storage so a new driver can pick up where the old one left off.

---

## Data locality: keeping computation near data

The Task Scheduler, when assigning tasks to executor slots, strongly prefers to run a task on the executor that already holds its input partition. This preference is called **data locality**. A task that reads HDFS data prefers the executor on the same node as the HDFS DataNode holding that block. A task that reads a cached RDD partition prefers the executor that cached it. Running a task locally avoids a network transfer of the input data—often the most expensive part of a task.

The Scheduler implements locality preferences as a priority order: `PROCESS_LOCAL` (same JVM, data in memory), `NODE_LOCAL` (same physical node), `RACK_LOCAL` (same network rack), `ANY` (anywhere). When a resource offer arrives from an executor, the Scheduler checks whether any pending task has a locality preference for that executor. If not, it can wait briefly (delay scheduling) before relaxing the locality requirement—accepting the offer even though the data will have to travel. The delay is configurable and represents a bet: "wait a bit for a better executor rather than run now on a bad one."

---

## Result collection and large results

When a result-stage task finishes, it has a result to return—the output of an action like `count()` or `collect()`. Small results (a single count, the first few rows of `take()`) are returned inline: serialized and sent directly to the driver via the task result channel. Large results (a full `collect()` of many rows) are first stored as a block in the executor's BlockManager, and the driver fetches the block separately via the BlockManager protocol. This two-path design avoids overwhelming the driver's network buffer with a single huge result.

For `collect()` on very large DataFrames—millions of rows—the driver must hold all results in memory. This is a common source of driver OOMs. The idiomatic alternative is to write results to storage (Parquet, Delta, object store) and read them back separately rather than routing them through the driver.

---

## Bringing it together

A Spark application has one **driver**—which runs user code, hosts all scheduler and coordination components, and never touches data rows—and one or more **executors**—which run tasks and hold cached data. At startup, the driver negotiates executor resources from the cluster manager. Transformations on the driver build a lazy logical plan; an **action** triggers its compilation into a physical plan and submission to the DAG Scheduler as a **job**. The DAG Scheduler cuts the job into **stages** at shuffle boundaries and submits them in dependency order. The Task Scheduler assigns each stage's tasks to executor slots, preferring data-local placement. Executors run tasks, report results and heartbeats back to the driver, and release task memory when done. So the journey from user code to computation is: **action → job → stages → tasks → executor slots → results back to driver.** The driver is the brain; the executors are the hands; data locality and the scheduler are the connective tissue that makes them efficient together.
