# Spark on YARN: ApplicationMaster, Containers, and the Queue

This is the story of how Spark runs on YARN (Yet Another Resource Negotiator)—the resource management layer of the Hadoop ecosystem. When your organization runs a shared Hadoop cluster, YARN is the traffic controller: it decides which application gets which resources on which nodes. Understanding YARN's model—its queues, the ApplicationMaster, NodeManagers, and containers—explains why your Spark job sometimes waits in a queue, why executors land on specific nodes, and how memory and CPU limits are enforced at the cluster level.

---

## YARN's resource model

YARN divides cluster resources into **containers**: allocated units of memory and CPU on a specific NodeManager (worker node). A container is a slice of a worker node: "4 GB of RAM and 2 virtual CPU cores on node worker-05." Everything in YARN runs inside containers—your driver, your executors, every process.

The ResourceManager (RM) is the cluster-wide arbiter. It receives resource requests from applications and assigns containers based on available capacity and scheduling policy. The NodeManager (NM) on each worker node runs the containers the RM assigns to it and reports health and resource usage back to the RM.

> **YARN is like a hotel chain with a central reservations desk.** The ResourceManager is the reservations desk: it allocates rooms (containers) on whichever hotel property (worker node) has vacancy. The NodeManager is the hotel manager on each property: they run the rooms, monitor occupancy, and report to headquarters. When you check in (submit a job), the reservations desk tells you which property to go to; the property manager gets you into your room.

---

## The ApplicationMaster: your job's advocate in the cluster

When Spark runs in YARN cluster mode, the first container YARN allocates is for the **ApplicationMaster (AM)**. For Spark, the ApplicationMaster is also the **driver**—the same JVM that runs your SparkContext. (In client mode, the AM is a lightweight coordinator while the driver runs externally, but this is less common for production.)

The AM is your job's representative inside YARN:
1. It registers with the ResourceManager and declares how many executor containers it wants.
2. It negotiates with the RM for containers as the job progresses (including dynamic allocation—asking for more executors when tasks are queued, releasing executors when they're idle).
3. It communicates with NodeManagers to launch executor containers on the allocated nodes.
4. If the AM fails (driver crash), YARN considers the application failed. The RM can restart the AM if `yarn.resourcemanager.am.max-attempts` allows it.

> **The ApplicationMaster is like the on-site project manager at a construction site.** The company headquarters (ResourceManager) assigns workers (containers) to the site based on the project manager's requests. The project manager knows what workers they need today, requests them through headquarters, and tells each worker exactly what to build. If the project manager leaves mid-project (driver crash), the project stalls unless a replacement is sent.

---

## How Spark requests executor containers

The Spark `YarnSchedulerBackend` sends container requests to the RM with specific resource requirements:
- **Memory**: `spark.executor.memory` + `spark.executor.memoryOverhead` (default: max(384MB, 10% of executor memory)). YARN enforces the total memory limit—if the executor JVM exceeds the container's memory limit, YARN kills the container.
- **vCores**: `spark.executor.cores`. YARN's scheduler tracks virtual CPU (vCores) per container.
- **Locality preference**: the RM is told which nodes the data lives on (via Spark's locality-awareness). YARN tries to allocate containers on preferred nodes. If not available, it falls back to preferred racks, then any available node.

> **Container requests with locality preferences are like specifying a seat preference when booking a flight.** "I'd prefer window, row 14" (specific node). If that's taken, "window seat anywhere" (same rack). If that's also full, "any seat" (any available node). The airline (YARN) does its best but doesn't guarantee your preference.

---

## YARN queues and resource allocation

YARN organizes capacity through **queues**. A queue is a named partition of cluster capacity: the `default` queue might get 50% of total cluster resources; an `engineering` queue gets 30%; a `data-science` queue gets 20%. Applications submitted to a queue compete for that queue's capacity.

Two scheduler types:

**Capacity Scheduler**: each queue has a guaranteed capacity (e.g., 30% of cluster RAM). If queue A is using less than its capacity, jobs in queue B can "borrow" the slack. If queue A then has jobs waiting, borrowed capacity is gradually reclaimed. Within a queue, jobs are scheduled FIFO by default (or with pre-emption and priority).

**Fair Scheduler**: resources are shared fairly among running applications. If only one job runs, it gets everything. If two jobs run simultaneously, they split resources 50/50 (by default). Fairness can be tuned with weights.

> **Queues are like reserved seating sections in a stadium.** Section A is reserved for VIPs (priority queue: 60% of seats guaranteed). Section B is for general admission (default queue: 40% guaranteed). If Section A is empty, general admission fans can spill into those seats. But when VIPs arrive, they get their seats back—general admission fans are moved (pre-empted). Submitting to the right queue is choosing the right section.

Submit to a specific queue with `--queue queue_name` in spark-submit.

---

## Executor container lifecycle

After the AM requests containers, YARN's workflow:

1. **RM allocates containers**: the RM's scheduler finds worker nodes with available capacity that match the memory/core request (and locality preference if possible). It returns container allocations to the AM.

2. **AM tells NodeManagers to launch containers**: the AM contacts each NodeManager directly (via NM's heartbeat/launch API) and sends the container launch context—the command to run, environment variables, and the Spark executor JAR.

3. **NM launches the container**: the NodeManager starts a JVM process in the container (the executor), bounded by the container's cgroup memory and CPU limits.

4. **Executor registers with the driver**: the executor JVM contacts the driver's RPC endpoint and says "I'm ready." The AM (driver) adds this executor to its pool.

When a container finishes or is released, the NM cleans it up and reports back to the RM. The capacity is returned to the pool for future allocation.

---

## Memory limits: executor vs. overhead

YARN enforces memory limits strictly (via cgroups on Linux). If your executor JVM exceeds the container's memory limit, the NM kills the container immediately—no graceful shutdown, no spark.task.maxFailures benefit. This appears in logs as a YARN container killed due to memory limit exceeded, often surfacing in Spark as a lost executor.

The total container memory is:
```
container_memory = spark.executor.memory + spark.executor.memoryOverhead
```

`memoryOverhead` covers JVM metaspace, direct memory buffers, the Netty buffer pool, and other off-heap allocations. If you see `Container killed by YARN for exceeding memory limits`, the fix is either to increase `memoryOverhead` or to reduce executor memory usage (more shuffle partitions, fewer cached datasets).

---

## YARN and the Spark UI / History Server

While a Spark job runs on YARN, the Spark UI is accessible via the driver's HTTP port (default 4040). YARN's ResourceManager UI links to the Spark UI through the application's tracking URL.

After the job completes, the Spark UI is gone (the driver container is killed). To review the job after completion, configure the **History Server**: set `spark.eventLog.enabled=true` and `spark.eventLog.dir=hdfs:///spark-events`, and run `spark-history-server.sh`. The History Server reads the event logs from HDFS and serves the Spark UI for completed jobs indefinitely.

---

## Bringing it together

Spark on YARN delegates resource management to YARN's two-tier system: the **ResourceManager** arbitrates cluster capacity and assigns containers to applications; **NodeManagers** run the containers on each worker node. Spark's **ApplicationMaster** (the driver in cluster mode) is your job's agent inside YARN—it requests executor containers, monitors their health, and releases them when done. **Queues** divide cluster capacity between competing applications; the Capacity Scheduler guarantees baseline capacity and allows borrowing; the Fair Scheduler divides resources dynamically among running jobs. **Memory limits** are strictly enforced by YARN cgroups—the sum of `executor.memory` and `memoryOverhead` is the container's hard ceiling. The journey from spark-submit to running tasks on YARN is: **submit to RM → AM launched → AM requests executor containers → RM allocates → NMs launch → executors register with driver → tasks flow.** YARN is the infrastructure that makes a Spark application a cooperative tenant on a shared cluster.
