# The Event Log: A Complete Record of Everything That Happened in Your Job

This is the story of the Spark event log—the detailed, time-stamped record of every significant thing that happened during a Spark application's execution. The event log is the foundation of the History Server, the post-completion Spark UI, and any offline analysis of application behavior. Understanding what goes into the event log, how it's written, and how to use it explains how to diagnose issues in jobs that have already finished, how to track performance trends over time, and how the Spark UI knows what to show you.

---

## What the event log is

The event log is a file (or set of files) written by the Spark driver during the application's execution. It is a sequence of **SparkListenerEvent** objects serialized as JSON lines—one JSON object per line, one line per event. The events are emitted by the `LiveListenerBus` (the driver's internal event bus) and written to disk (or HDFS/S3) by the `EventLoggingListener`.

Events include:
- `SparkListenerApplicationStart` / `ApplicationEnd`: application lifecycle
- `SparkListenerJobStart` / `JobEnd`: when each job (triggered by an action) starts and ends
- `SparkListenerStageSubmitted` / `StageCompleted`: stage lifecycle, including the DAG, number of tasks, and final status
- `SparkListenerTaskStart` / `TaskEnd`: per-task start and finish with full metrics
- `SparkListenerExecutorAdded` / `ExecutorRemoved`: executor lifecycle
- `SparkListenerBlockUpdated`: cache and storage events
- `SparkListenerEnvironmentUpdate`: the application's configuration
- `SparkListenerSQLExecutionStart` / `SQLExecutionEnd`: SQL query plan and execution

> **The event log is like a ship's voyage recorder (black box).** Everything that happened during the voyage—every change in speed, every course correction, every crew action—is recorded with a timestamp. After the voyage (job completion), investigators (engineers debugging a failure) can reconstruct exactly what happened, in what order, and how long each event took. Without the black box, all you have is "the ship arrived late" (or sank).

---

## Enabling and configuring event logging

By default, event logging is disabled. Enable it with:

```
spark.eventLog.enabled = true
spark.eventLog.dir = hdfs:///spark-events   (or s3a://bucket/spark-events, or local path)
```

Each application writes a single event log file named by the application ID: `application_1701234567890_0001`. For rolling event logs (very long-running applications), set:

```
spark.eventLog.rolling.enabled = true
spark.eventLog.rolling.maxFileSize = 128m
```

This splits the event log into multiple 128MB files, preventing single huge files and allowing the History Server to read partial logs for in-progress applications.

**Log compression** reduces storage costs:
```
spark.eventLog.compress = true
spark.eventLog.compression.codec = zstd
```
Compressed logs are smaller and cheaper to store, but the History Server must decompress them before serving the UI.

---

## The task metrics: what's inside each TaskEnd event

The most useful information in the event log is the per-task metrics in `SparkListenerTaskEnd`. For each task, Spark records:

- **Task duration**: wall-clock time from task start to completion
- **Executor deserialize time**: time to deserialize the task closure
- **Executor run time**: time actually running the task function
- **Shuffle read bytes**: bytes read from remote executors during shuffle
- **Shuffle write bytes**: bytes written to shuffle storage
- **Shuffle read fetch wait time**: time waiting for remote shuffle blocks (network bottleneck indicator)
- **Records read/written**: input/output row counts
- **Memory bytes spilled / disk bytes spilled**: indicates memory pressure
- **JVM GC time**: time spent in garbage collection during this task
- **Result serialization time**: time to serialize the task result back to the driver
- **Locality**: which locality level was achieved (PROCESS_LOCAL, NODE_LOCAL, etc.)

> **Task metrics are like a detailed time-and-motion study for each worker.** For each task, you know not just "it took 10 seconds" but "2 seconds were wasted waiting for shuffle data from network, 1 second was GC pauses, 6 seconds was actual work, and 1 second was overhead." You can identify whether a task is slow because of bad data, network congestion, GC pressure, or serialization overhead.

---

## The History Server: serving completed applications

The **History Server** is a standalone web server that reads event logs from the configured directory and serves a Spark UI for any completed (or running) application whose event log is available.

Start it with:
```bash
$SPARK_HOME/sbin/start-history-server.sh
```
Configuration (`spark-defaults.conf`):
```
spark.history.fs.logDirectory = hdfs:///spark-events
spark.history.ui.port = 18080
```

The History Server scans the log directory for event log files, parses them, and rebuilds the application's job/stage/task data structures. The resulting UI is identical to the live Spark UI during execution—you see the Jobs tab, Stages tab, SQL tab, Executors tab, and all task metrics—but for a job that may have completed hours or days ago.

> **The History Server is like a flight simulator that reconstructs a past flight from the flight data recorder.** You can't change what happened, but you can replay the execution, pause at any stage, zoom in on any task, and understand exactly what the job did and how long each part took. It's post-mortem analysis as a fully interactive UI.

**Performance tip**: large event logs (long jobs with many tasks) can take the History Server a long time to parse. Enable the event log caching (`spark.history.store.path`) so parsed application data is kept in a local LevelDB cache. Subsequent loads of the same application are nearly instant.

---

## Practical uses of the event log

**1. Diagnosing slow jobs after the fact**: find the slowest stage (longest bar in the job timeline), look at the task duration distribution (did a few tasks take 10× longer than the median?), check shuffle metrics (is fetch wait time high?), and check spill (are tasks spilling to disk?).

**2. Tracking performance over time**: event logs can be parsed programmatically. Tools like Spark-Log-Analyzer, DR. Elephant (LinkedIn), and custom scripts read event logs to track job duration, stage durations, and metrics across many runs. This enables trend analysis: "has this job been getting slower over the past month?"

**3. Auditing**: the event log records every application that ran, when it ran, what configuration it used, and how long it took. This is valuable for capacity planning and cost attribution.

**4. Debugging Structured Streaming**: streaming queries write continuous event logs. Each micro-batch generates `SparkListenerJobStart/End` events. You can reconstruct the history of every micro-batch—its duration, its input rows, its output rows—from the event log.

---

## Parsing the event log programmatically

The event log is newline-delimited JSON, readable with any JSON library:

```python
import json

with open("application_1701234567890_0001") as f:
    for line in f:
        event = json.loads(line)
        if event.get("Event") == "SparkListenerTaskEnd":
            metrics = event.get("Task Metrics", {})
            duration = event.get("Task Info", {}).get("Finish Time", 0) - \
                       event.get("Task Info", {}).get("Launch Time", 0)
            spill = metrics.get("Memory Bytes Spilled", 0)
            print(f"Task {event['Task Info']['Task ID']}: {duration}ms, spill={spill}")
```

> **The event log is JSON, which means it's a first-class data source.** You can read it with pandas, query it with Spark itself (recursive: use Spark to analyze Spark's own logs), or load it into any analytics platform. Some teams store all event logs in S3, catalog them with a Hive metastore entry pointing to the JSON, and run SQL queries over the entire history: "which job had the most total shuffle bytes last week?"

---

## Bringing it together

The event log is Spark's complete execution diary—every job, stage, task, executor, and SQL query is recorded as a JSON event with timestamps and metrics. It's enabled with `spark.eventLog.enabled=true` and written to HDFS/S3/local storage during execution. The **History Server** reads these logs and serves an interactive Spark UI for completed applications. **Task metrics** (duration, shuffle bytes, spill, GC time) are the diagnostic gold inside each TaskEnd event—they explain not just how long a task took but why. The event log can be parsed programmatically for trend analysis, capacity planning, and cost attribution. Without the event log, debugging a slow job that already finished is guesswork. With it, you have a complete, replayable record of everything Spark did.
