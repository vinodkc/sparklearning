# Locality and Delay Scheduling

Why do tasks sometimes sit “pending” for a few seconds before they launch? And why does changing a single setting like the locality wait change how fast your job runs? This story is about how Spark decides *where* to run each task: **preferred locations** (so tasks run near their data), **locality levels** (from “same process” to “anywhere”), and **delay scheduling** (waiting a bit for a good executor instead of taking the first offer).

---

## Why location matters

A task usually needs to **read data**—a partition that might be cached on an executor, sitting in shuffle output on a host, or in a file with a preferred host (e.g. an HDFS block). If the task runs on an executor that already has that data in memory or on local disk, it avoids network transfer and runs faster. If it runs elsewhere, the data has to be fetched over the network. So Spark tries to **schedule tasks close to their data**. That’s **data locality**.

The scheduler doesn’t push tasks out on its own. It only gets **resource offers**: “Executor X on host H has N free cores.” For each offer it must decide: *do I run one of my pending tasks here, or do I say no and wait for a better offer?* Preferred locations and delay scheduling are how it makes that choice.

---

## Where “preferred” comes from

When a stage is submitted, the **driver** figures out, for each partition, *where it would be best to run*. Those preferred locations are attached to every task and handed to the Task Scheduler.

How does the driver know?

- **Cache.** If that partition is already cached, it uses where the block lives—which executor or host holds it.
- **Input placement.** If the RDD comes from input (e.g. HDFS), the input layer often knows where blocks are (e.g. which host has the block). Those hosts become preferences.
- **Narrow dependencies.** If the RDD was built with narrow steps (e.g. a *map* over a cached RDD), the driver follows the lineage: the partition depends on specific parent partitions, so it reuses their preferred locations. Locality flows along the lineage.

If none of that yields a location, the partition has **no preference**—the task can run anywhere. By the time the Task Scheduler sees the stage, every task already has a list of preferred places (or an empty list).

---

## Locality levels: from “same process” to “anywhere”

The scheduler doesn’t schedule tasks by itself. It only answers: *given an offer from executor E on host H, do I have a task that fits here, and how “local” would it be?*

Locality is thought of in levels, from best to least local:

- **Process-local:** The task’s data is on *this exact executor* (e.g. a cached block there). Best case.
- **Node-local:** The task’s data is on *this host*—maybe another executor on the same machine, or an HDFS block on that host.
- **No preference:** The task has no locality info; it can run anywhere without giving up a preference.
- **Rack-local:** The task’s data is on the same rack (if the cluster exposes rack info).
- **Any:** No locality; the task is fine running on any executor.

When the stage’s tasks are queued, they’re grouped by these levels—by executor, by host, by rack, and “no preference.” So for an offer from executor E on host H, the scheduler can quickly see: “Do I have a task that wants this executor? This host? This rack? Or only a task with no preference?” It tries in that order and picks the first match that’s *allowed* by the current **delay scheduling** rule (below).

---

## Delay scheduling: waiting for a good offer

If the cluster is busy, the “best” executor for a task might not be free yet. Spark can **decline** an offer—launch nothing on it—and wait for a better one. That’s **delay scheduling**: trade a short wait for better locality.

The scheduler keeps, per stage:

- The **locality levels** that actually apply (only levels that have pending tasks and live executors or hosts).
- A **current allowed level**: “I’m only willing to launch tasks at this level or better (more local).”
- A **timer** per level: when the wait time for the current level has elapsed, the allowed level is relaxed to the next one.

So at any moment there is a strictest level Spark is still willing to use. If the offered executor isn’t good enough for any task at that level, the scheduler returns “no task” for this offer—a **delay scheduling reject**. Nothing is launched on that executor in this round.

You can tune how long Spark waits at each level. The main knob is the locality wait (default a few seconds); you can set separate waits for process, node, and rack. By default, Spark might wait a few seconds at process-local, then at node-local, and so on, before allowing “any.” There’s also an optimization: if there are **no more pending tasks** that can run at the current level (e.g. no process-local tasks left for any live executor), the scheduler skips to the next level immediately instead of waiting for the full timeout.

When do new offers appear? The backend runs a thread that periodically asks for offers again (about once per second). So after a reject, the next round of offers comes soon; by then the wait may have expired and the scheduler will accept a less-local offer. Task completions also trigger new offers (a core freed on an executor), so sometimes the “right” executor frees up and the pending task gets scheduled there.

---

## Giving every executor a chance at local work

The Task Scheduler gets a batch of offers (one per executor). To give each stage a fair chance to place **local** tasks on the **right** executors, it goes through locality levels in order: first it offers every executor at the strictest level (e.g. process-local), so every stage can try to place its most local tasks; then at the next level, and so on. So the same executor can be “offered” several times in one round at different levels, and the scheduler can place a local task when the level allows it.

---

## Summary

**Preferred locations** are computed on the driver from cache, input placement, and narrow-dependency lineage, and attached to each task. The **Task Scheduler** groups pending tasks by locality (executor, host, rack, no-pref) and, for each resource offer, picks a task that matches at the **allowed** locality level. **Delay scheduling** is the rule that keeps a current “allowed” level and per-level wait times; until the wait expires (or there are no tasks left at that level), it only accepts offers that are good enough. Otherwise it declines. Periodic offer revival and task completions bring new offers, so after waiting, Spark can relax the level and schedule the task on a less-local executor. So: the **DAG Scheduler** is responsible for preferred locations; the **Task Scheduler** is responsible for using them and for delay scheduling. The backend’s periodic “revive offers” is what gives the scheduler repeated chances to place tasks with better locality—and why you sometimes see those short, deliberate pauses before tasks launch.
