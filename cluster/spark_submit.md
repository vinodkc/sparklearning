# From spark-submit to Running Tasks: The Resource Negotiation Story

This is the story of what happens between typing `spark-submit` and the first task actually executing on a worker node. This journey involves the Spark application packaging, driver launch, cluster resource negotiation, executor acquisition, and task scheduling—a chain of events spanning the client, the cluster manager, and the distributed worker nodes. Understanding this chain explains what each component of the infrastructure is responsible for, why some failures happen before any data is read, and how to diagnose the gap between "submit" and "running."

---

## The actors

Before tracing the journey, the cast of characters:

**spark-submit**: the command-line script that packages your application, resolves dependencies, and launches the driver (or submits to a cluster manager that will launch the driver).

**Driver**: the JVM process that hosts your SparkContext, runs your main program, and coordinates everything. It communicates with the cluster manager to acquire resources and with executors to schedule work.

**Cluster manager**: the resource scheduling layer—YARN ResourceManager, Kubernetes API Server, or Spark's standalone master. It manages worker nodes and allocates containers/pods/workers to applications.

**Executor**: a JVM process on a worker node that runs tasks and stores cached data. Each executor is a container/pod/process allocated by the cluster manager to your application.

> **Think of spark-submit like a hiring agency submission.** You submit your CV (the application JAR), describe what resources you need (cores, memory), and the agency (cluster manager) decides whether to place you in an available office (worker node). The cluster manager first assigns a manager role (driver), who then requests staff (executors). No work starts until staff are in their seats.

---

## Step 1: spark-submit — packaging and launch

`spark-submit` is a shell script that:
1. **Assembles the classpath**: adds the Spark JARs, your application JAR (`--jars`), and any files (`--files`) to the classpath.
2. **Resolves configuration**: merges `SPARK_HOME/conf/spark-defaults.conf`, environment variables, and command-line arguments (`--conf key=value`). Command-line arguments take highest precedence.
3. **Determines deploy mode**: `--deploy-mode client` (driver runs on the machine where `spark-submit` was run) or `--deploy-mode cluster` (driver is launched on a worker node by the cluster manager).
4. **Launches the driver** (for client mode) or **submits the driver launch request** (for cluster mode) to the cluster manager.

The `--master` argument determines which cluster manager to use: `yarn`, `k8s://https://...`, `spark://host:7077` (standalone), or `local[*]` (local mode).

---

## Step 2: driver starts, SparkContext initializes

When the driver JVM starts and runs `new SparkContext(conf)` (or `SparkSession.builder.getOrCreate()`):

1. **Configuration is finalized**: the SparkConf is locked down (no further changes to core properties).
2. **The cluster manager client is created**: a YARN `YarnClient`, Kubernetes client, or standalone client, depending on the master URL.
3. **A `SchedulerBackend` and `TaskScheduler` are created**: the scheduler backend talks to the cluster manager to acquire executors; the task scheduler assigns tasks to available executor slots.
4. **The DAG Scheduler is created**: the high-level planner that converts RDD/DataFrame actions into stages and tasks.

---

## Step 3: requesting executors from the cluster manager

The `SchedulerBackend` contacts the cluster manager to request executor resources. The request specifies:
- `spark.executor.memory`: memory per executor (e.g., `4g`)
- `spark.executor.cores`: CPU cores per executor (e.g., `4`)
- `spark.executor.instances`: total number of executors (or min/max for dynamic allocation)
- `spark.executor.memoryOverhead`: off-heap overhead (JVM overhead, off-heap buffers)

> **Requesting executors is like a project manager calling the HR department (cluster manager) to request office space and staff.** "I need 10 desks, each with 4 computers and 8GB of RAM." HR checks availability across the office building (worker nodes) and assigns desks where space is available. Until desks are assigned, the project manager waits.

For YARN: the `SchedulerBackend` requests containers from the ResourceManager. Each container satisfies `executor.memory + memoryOverhead` in RAM and `executor.cores` in virtual CPU. YARN's scheduler places containers on NodeManagers (worker nodes) subject to queue policies and node capacity.

For Kubernetes: the `SchedulerBackend` creates Pod specs and submits them to the Kubernetes API Server. The K8s scheduler places pods on nodes subject to node selector, affinity, resource limits, and namespace quotas.

---

## Step 4: executors start and register with the driver

Once the cluster manager allocates resources:
1. A new JVM process starts on the worker node (inside the YARN container or Kubernetes pod).
2. The executor JVM runs `CoarseGrainedExecutorBackend`, which immediately contacts the driver's RPC endpoint and **registers itself**: "I'm Executor 1, I have 4 cores and 4GB of memory."
3. The driver acknowledges the registration and adds this executor to its pool of available resources.

As executors register, task slots become available. With 10 executors × 4 cores each, the driver has 40 task slots. Tasks are sent to executors as soon as slots are available.

> **Executor registration is like new employees showing up for their first day and checking in at the front desk.** The project manager (driver) now knows how many people are ready to work. Until at least one person checks in, no work assignments can be sent out.

---

## Step 5: driver sends tasks to executors

When an action is called (e.g., `df.count()`), the DAG Scheduler creates stages and the Task Scheduler assigns tasks to executor slots. For each task:

1. The driver serializes the task (including the closure—the function to run) into bytes.
2. The serialized task is sent to the selected executor's slot via the Spark RPC framework.
3. The executor deserializes the task, starts a thread from its thread pool to run it, and executes.

The complete time from `spark-submit` to first task running is:
- `spark-submit` startup + JAR upload (a few seconds to tens of seconds)
- Driver JVM startup + SparkContext initialization (a few seconds)
- Cluster manager container/pod scheduling (seconds to minutes, depending on cluster load)
- Executor JVM startup + registration (10–30 seconds per executor, though they overlap)
- First batch of tasks dispatched

For large clusters (100+ executors), the first tasks may start running while other executors are still starting.

---

## Deploy modes: client vs. cluster

**Client mode** (`--deploy-mode client`): the driver runs on the machine where `spark-submit` was invoked. This is convenient for development (driver logs appear in the terminal) but means the driver is not managed by the cluster. If you lose the terminal or the submit machine restarts, the driver dies and the job fails.

**Cluster mode** (`--deploy-mode cluster`): the cluster manager launches the driver as a container/pod on a worker node. The driver is managed by the cluster—if the driver container fails, the cluster manager may restart it. For production jobs, cluster mode is preferred because the driver's lifecycle is tied to the cluster, not to the user's laptop.

> **Client mode is working from your laptop in the office (convenient, fragile). Cluster mode is working from a dedicated workstation locked in the server room (less convenient to interact with, more resilient).** In cluster mode, you don't get real-time console output on your machine—you view logs through the Spark UI or the cluster's logging system.

---

## Diagnosing startup failures

Many failures happen before any tasks run:
- **"Application not found"** or YARN/K8s authentication errors: the cluster manager rejected the submission. Check credentials, queue names, and namespace permissions.
- **Executors never register**: containers/pods allocated but JVM doesn't start. Check executor logs in YARN NodeManager or Kubernetes pod logs. Common causes: wrong container image, classpath errors, OOM at executor startup.
- **Long gap between submit and first task**: executor allocation is slow (cluster under heavy load, resource quotas) or executor JVM startup is slow (large JARs, slow storage for JAR distribution).
- **Tasks fail immediately with ClassNotFoundException**: JAR not distributed to executors. Check `--jars` or `spark.jars` configuration.

---

## Bringing it together

The journey from `spark-submit` to running tasks has five acts: **package and launch** (spark-submit assembles classpath and starts/submits the driver), **driver initializes** (SparkContext starts, scheduler components are created), **executors are requested** (SchedulerBackend asks the cluster manager for containers/pods with the specified cores and memory), **executors start and register** (worker JVMs come up and check in with the driver), and **tasks are dispatched** (the first action triggers DAG/Task Scheduler to send work to executor slots). Every step is a potential failure point—from authentication errors at submission to classpath errors at executor startup. Understanding the chain gives you the mental model to locate failures quickly: is the problem before the driver starts? Between driver and cluster manager? Between cluster manager and executors? Or after executors are up, in task execution itself?
