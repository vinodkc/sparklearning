# Shared State in a Distributed Job: Broadcast Variables and Accumulators

This is the story of how Spark shares data and collects information across a distributed computation without turning every task into a round-trip conversation with the driver. In a Spark job, tasks run on executors scattered across the cluster. They don't share memory; they don't share a file system by default; they communicate with the driver through a narrow channel. Yet sometimes every task needs access to the same lookup table, and sometimes you want to count events that happen inside tasks. Broadcast variables and accumulators are the two mechanisms Spark provides for these needs—one for sending data *out* to executors efficiently, the other for aggregating data *back* to the driver safely.

---

## The problem broadcast solves

Imagine your job joins a large table with a small lookup table—say, a table of 100 country codes and their names. Without any special mechanism, Spark would shuffle both tables by join key, which means the 100-row lookup table gets redistributed across every reduce partition. That's 100 rows shuffled, which is trivial—but if the lookup table is 50 MB (say, a dimension table with many columns), you might not qualify for Spark's auto-broadcast threshold, and even if you do, the default mechanism sends the table from the driver to executors one-at-a-time.

More generally: any time you have data that every task needs—a machine learning model for inference, a list of blocked IDs, a set of configuration values computed once—naively including it in every task's closure means the driver **serializes** that data once per task and sends it to each executor over the task-dispatch channel. If 500 tasks need a 100 MB model and you have 50 executors each running 10 tasks, the driver sends 500 copies of that 100 MB object: 50 GB of network traffic from one machine.

> **Imagine a teacher photocopying a 100-page handout for every student in every class, all year long.** If there are 500 students, the teacher makes 500 copies—individually, one at a time. A smarter approach is to print one copy per classroom, post it on the wall, and let students in that classroom read it whenever they need it. That's the broadcast model: one copy per executor, read locally by all tasks.

---

## Broadcast variables: send once, use everywhere

A **broadcast variable** is a wrapper that tells Spark: distribute this data to every executor *once*, cache it there, and let all tasks on that executor read it from the local cache rather than receiving it with each task.

When you broadcast a value, the driver uses **torrent-style distribution** (via BitTorrent-like chunked peer-to-peer transfer in TorrentBroadcast, the default implementation): it splits the data into chunks, sends the chunks to executors, and lets executors serve chunks to each other rather than all fetching from the driver simultaneously. This means the driver's upload bandwidth is not the bottleneck; the distribution scales with the number of executors.

> **Think of it like distributing a new company policy document.** Instead of the CEO emailing a 20 MB PDF to all 500 employees at once (flooding the CEO's inbox), the CEO sends it to the team leads, who forward it to their reports, who forward it to their teams. Everyone ends up with a copy, but the load is distributed across the organisation. That's the peer-to-peer broadcast.

Once an executor receives all chunks of a broadcast variable, it stores the deserialized value in its local memory (managed by the BlockManager). Every task running on that executor that accesses the broadcast variable reads from that in-memory copy—a local memory read, no network involved.

The practical implication: broadcasting a 100 MB lookup table to 50 executors costs 100 MB of network transfer per executor (distributed peer-to-peer), not 500 × 100 MB. Tasks read from local memory; no per-task serialization; no repeated driver-to-executor transfers.

---

## Broadcast variables and joins

Spark SQL uses broadcast variables implicitly for broadcast hash joins. When the query planner decides to broadcast a table, it wraps the table in a broadcast variable, distributes it to all executors, and each executor uses its local copy to probe the hash map for each row of the larger table. You can also broadcast values explicitly in RDD-based jobs: call `sc.broadcast(myValue)`, then access `myValue.value` inside your task function. The `.value` call reads from the executor-local cache.

Broadcast variables are **read-only** from the perspective of tasks. There is no mechanism to update a broadcast variable from inside a task—it's a one-way distribution of immutable data. If you need to update shared state from tasks, that's the accumulator's job.

---

## Unpersisting broadcasts

A broadcast variable lives in executor memory until it is explicitly unpersisted. For long-running applications that broadcast different data over time (a streaming job that re-broadcasts a model periodically), always call `broadcastVar.unpersist()` or `broadcastVar.destroy()` when the old value is no longer needed. `unpersist()` removes it from executor memory asynchronously; `destroy()` removes it from executor memory and also deletes the driver's metadata.

---

## The problem accumulators solve

Tasks run in isolation. Each task has its own local variables, its own stack, its own memory. If you want to count how many records matched a filter, you can't increment a counter in the driver from inside a task—tasks don't share memory with the driver. You could collect all matching records and count them on the driver, but that may mean moving a lot of data.

**Accumulators** are Spark's solution: variables that tasks can only **add to** (increment, append), whose aggregated value is only readable on the driver. They flow in the opposite direction from broadcast variables—broadcast sends data out to tasks; accumulators send aggregated data back to the driver.

> **Think of accumulators like vote tallying in an election.** Each polling station (task) counts its own ballots locally throughout the day. At the end of the day, each station reports its count to the central returning office (driver). The returning office adds up all the counts to get the national total. Voters never call the returning office mid-day; the office never calls the polling stations. The tally only flows one direction, at the end.

---

## How accumulators work

When you create an accumulator on the driver (`sc.longAccumulator("records_skipped")`, for example), Spark registers it and gives it a unique ID. When a task runs, it gets a local copy of the accumulator starting at zero (or the identity value). The task increments this local copy as it runs. When the task finishes, the executor reports the final local value to the driver as part of the task result. The driver merges (adds) all the task-local values into the accumulator's total. When the job is done, you read the total on the driver: `myAccumulator.value`.

This design—local copies, report at task end, merge on driver—has two important properties. First, tasks never wait for each other or talk to the driver mid-task; accumulator updates are purely local until the task finishes. Second, the accumulator value is only correct after all tasks in the job complete; reading it mid-job gives you a partial sum.

---

## The exactly-once problem: accumulators and task retries

There is a subtle but important caveat: **accumulators are not exactly-once**. If a task fails and is retried, both the failed task's update and the successful retry's update may be counted. Spark mitigates this by only merging accumulator updates from *successful* task attempts—if a task fails, its partial update is discarded. But speculative execution can cause double-counting: when speculation launches a duplicate of a slow task, both the original and the duplicate may finish successfully, and both updates may be counted.

For metrics that just need to be approximate—counting records processed, events skipped, errors encountered—this is usually fine. For metrics where exact counts matter (billing, auditing), don't use accumulators; use a reliable aggregation in the job output instead.

---

## Built-in accumulator types and custom accumulators

Spark provides built-in accumulator types for the most common cases: `LongAccumulator`, `DoubleAccumulator`, and `CollectionAccumulator` (which accumulates a list of values). For custom aggregation logic—tracking a histogram, the maximum of a custom object, a set of distinct values—you can extend `AccumulatorV2` and implement the `add` (local increment) and `merge` (combine two accumulators) methods. The only requirement is that `merge` is commutative and associative: the driver can merge task updates in any order.

Be cautious with `CollectionAccumulator` and any custom accumulator that grows without bound. The accumulator's merged value is held on the driver in memory. Accumulating one value per row in a job with billions of rows will OOM the driver.

---

## Accumulators in the Spark UI

Accumulator values appear in the Spark UI's **stage** view: for each stage, the UI shows each named accumulator and its current value across all completed tasks. This makes accumulators a lightweight observability tool—a way to track domain-level counters (records filtered, nulls encountered, schema mismatches) without writing to an external system or pulling the full dataset to the driver.

---

## Bringing it together

**Broadcast variables** and **accumulators** are Spark's two mechanisms for shared state between the driver and tasks, flowing in opposite directions.

Broadcast variables flow **driver → executors**: immutable data that every task needs is distributed once using peer-to-peer transfer, cached on each executor, and read locally by all tasks on that executor—no per-task serialization, no repeated network transfers, no coupling to the driver during task execution. They are the right tool for lookup tables, model weights, configuration maps, and any other read-only data that tasks must share.

Accumulators flow **tasks → driver**: tasks increment local copies; the driver merges updates at task completion; the aggregated total is readable on the driver after the job. They are the right tool for metrics, counters, and lightweight observability. They are not exactly-once (speculative retries can double-count), so use them for approximate or diagnostic purposes rather than precision accounting.

Together, these two mechanisms let Spark jobs read shared data and report aggregated results without any task-to-task communication or driver round-trips during execution—keeping the system's coordination overhead low even as the number of tasks grows into the thousands.
