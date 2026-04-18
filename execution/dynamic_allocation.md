# Elastic Executors: How Dynamic Allocation Grows and Shrinks the Cluster

This is the story of how a Spark application can grow and shrink its pool of executors while it runs, rather than reserving a fixed set of resources for the entire job. **Dynamic resource allocation** (DRA) is the mechanism behind this elasticity. Understanding how it works explains why a short ETL job doesn't waste cluster resources during quiet periods, why a batch job that spikes in the middle can acquire extra capacity without restarting, and what the External Shuffle Service has to do with giving executors back safely.

---

## The problem with fixed allocation

In static allocation—the original Spark model—you declare how many executors you want when you launch the application (`--num-executors N`), and those executors are reserved for the application's entire lifetime. This is simple but wasteful. A typical Spark job is not uniformly busy: it reads data (tasks running on many cores), then hits a wide shuffle (most cores idle while map output accumulates), then reduces (cores busy again), then writes output (tasks running), then the job ends. Between each active phase, many executors sit idle, consuming cluster memory and CPU that another job could be using.

On a shared cluster—a YARN or Kubernetes environment serving dozens of teams—idle reserved executors can prevent other jobs from acquiring resources. And if you under-provision (to be a good cluster citizen), your own job runs slower than it could during its busy phases.

Dynamic allocation solves both: the application holds only the executors it currently needs, releasing them when idle and requesting more when the task queue grows.

---

## The two halves: requesting and releasing

Dynamic allocation operates through two independent timers running on the driver.

The **scale-up timer** watches the **pending task queue**: tasks that are ready to run but have no executor slot available. If there are pending tasks and the application has fewer executors than the configured maximum, the driver requests additional executors from the cluster manager. Requests are made in a multiplicative ramp: if you have N executors and need more, Spark requests up to 2N additional executors in the next round (doubling up to the maximum). This aggressive ramp-up means a job that suddenly has 500 tasks to run doesn't wait through many small incremental requests before getting the capacity it needs.

The **scale-down timer** watches for **idle executors**: executors that have had no running tasks for a configurable duration (default: 60 seconds via `spark.dynamicAllocation.executorIdleTimeout`). When an executor has been idle long enough, the driver tells the cluster manager to remove it. The executor is decommissioned and its resources are returned to the pool for other jobs.

There is also a separate, shorter timeout (`spark.dynamicAllocation.cachedExecutorIdleTimeout`, default: infinity) for executors that hold **cached data**. An executor with a cached RDD partition is more valuable to release late, because releasing it loses that cached data. By default Spark never evicts cached executors unless you explicitly set this timeout.

---

## The shuffle service problem

The original design of dynamic allocation hit a fundamental obstacle: shuffle data. When a map stage completes, its output lives in local files on the executors that ran the map tasks. Reduce tasks read those files later. If a map-stage executor is released (returned to the cluster) before the reduce stage runs, its shuffle files are gone—the reduce tasks fail with fetch failures, the map stage must re-run, and you've paid a steep cost to save a little memory.

The solution is the **External Shuffle Service** (ESS): a long-lived daemon process that runs on each worker node, separate from the executor JVM. When the ESS is running, map-stage shuffle files are written so the ESS can serve them. If the executor is later released, the files remain on the node's local disk and the ESS continues to serve them to reducers. The reduce stage reads its shuffle data from the ESS rather than directly from the executor, so the executor can be safely returned to the cluster between the map and reduce stages.

Dynamic allocation is only safe to enable (`spark.dynamicAllocation.enabled = true`) when either (a) the External Shuffle Service is running (`spark.shuffle.service.enabled = true`), or (b) you are using **shuffle-tracking** (a Spark 3 feature, `spark.dynamicAllocation.shuffleTracking.enabled`, which tracks which executors hold shuffle data and keeps them alive until their shuffle output has been consumed, without requiring a separate service). On Kubernetes, where the ESS is not always available, shuffle tracking is the preferred option.

---

## Executor lifecycle under dynamic allocation

The executor lifecycle becomes more dynamic and observable. At application start, Spark requests `spark.dynamicAllocation.initialExecutors` executors (default: `spark.dynamicAllocation.minExecutors`, which defaults to 0). As the first stage's tasks are submitted, the task queue fills and the scale-up timer fires, requesting more executors. The cluster manager allocates them as nodes become available.

As stages complete and the task queue drains, idle executors accumulate. After their idle timeout, they are released. If the next stage is another shuffle consumer with many reduce tasks, the scale-up timer fires again and executors are requested again—potentially on different nodes than before.

This means that under dynamic allocation, the set of executors is **not stable** across stages. Cached data on a released executor is lost (unless replication was used). If your job caches an RDD in stage 2 and reads it in stage 5, and the caching executors were released between those stages (because stage 3 and 4 were idle), stage 5 will see cache misses and recompute. The `spark.dynamicAllocation.cachedExecutorIdleTimeout` setting controls how long cached executors are protected; setting it to a finite value risks losing cache; leaving it at infinity means cached executors are never released, undermining some of the resource-saving benefit.

---

## Minimum, maximum, and initial executor counts

Three configuration values define the bounds:

- `spark.dynamicAllocation.minExecutors` (default 0): the floor. The application will never have fewer than this many executors, even if all are idle. Setting this above zero ensures a warm pool of executors is always available for new tasks, at the cost of holding resources even when idle.
- `spark.dynamicAllocation.maxExecutors` (default infinity): the ceiling. The application will never request more than this many executors, no matter how large the task queue grows. This is the primary way to limit cluster impact.
- `spark.dynamicAllocation.initialExecutors` (default = minExecutors): how many executors to request at startup, before any tasks are submitted.

A common production pattern: `minExecutors = 1` (keep at least one warm executor to avoid cold-start latency for small queries), `maxExecutors = 50` (cap cluster impact), and let DRA handle everything in between.

---

## Dynamic allocation and Kubernetes

On Kubernetes, executors are pods. Dynamic allocation works by creating new executor pods (via the Kubernetes API) when scale-up is needed and deleting them when idle. Because Kubernetes pods are ephemeral, there is no persistent ESS equivalent. Shuffle tracking is the default mechanism: Spark records which executor holds which shuffle partition's output and avoids releasing that executor until the shuffle data has been fetched by reducers. Once the data is no longer needed, the executor pod is deleted and its resources are freed.

Kubernetes-native Spark also benefits from executor decommissioning: rather than abruptly killing an executor pod, Spark sends a decommission signal. The executor finishes its currently running tasks, migrates any shuffle or cache blocks it can, and then exits cleanly. This reduces the likelihood of fetch failures during scale-down.

---

## Bringing it together

Dynamic allocation lets a Spark application hold only the executors it currently needs, scaling up when the task queue grows and scaling down when executors go idle. It operates through two timers on the driver: a scale-up timer that requests new executors (doubling per round) when tasks are pending, and a scale-down timer that releases executors that have been idle beyond a threshold. The External Shuffle Service (or shuffle tracking in Spark 3) makes scale-down safe by ensuring that shuffle output remains accessible after the executor that wrote it has been released. Cached-data executors are protected by a separate, longer idle timeout. Minimum and maximum executor counts bound the cluster footprint. So the story is: **tasks pile up → driver requests executors → cluster allocates them → tasks drain → executors go idle → driver releases them → shuffle data stays accessible via ESS or shuffle tracking.** The result is a Spark application that scales with its workload rather than reserving peak capacity for the entire job.
