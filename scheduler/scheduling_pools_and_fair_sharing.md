# Scheduling Pools and Fair Sharing

When several jobs run in the same Spark application—for example, multiple users sharing a session, or a long-running service that submits many small jobs—who gets the executors first? By default, Spark schedules in **FIFO** order: the first job’s stages get resources until they block or finish, then the next. If you want to share resources more evenly, you can switch to **fair scheduling** and put jobs into **pools** with minimum shares and weights. This story explains how that works and why it matters when multiple jobs run at once.

---

## One application, many jobs

A single Spark application can run many **jobs**. Each job is triggered by an action (e.g. a *count*, a *save*, or a SQL query). Jobs may come from different threads, different users (e.g. in a shared Thrift server), or the same driver submitting one job after another. While one job is running, another can be submitted. So at any moment the Task Scheduler may have pending tasks from several jobs, and it must decide **which job’s tasks get the next resource offer**.

That decision is made by a **scheduling tree**: a root and, in fair mode, **pools** (groups), with each pool containing either more pools or the actual **task sets** (one per stage). The scheduler doesn’t pick tasks at random. It **orders** the task sets according to a policy, then walks that order and offers resources to each set in turn (subject to locality and delay scheduling). So “who gets resources first” is determined by how that ordering is built.

---

## Default: FIFO

Out of the box, Spark uses **FIFO** (first in, first out). There is a single logical pool: every stage’s task set goes into the same queue. The queue is ordered by **job and stage**: earlier stages come first. So the first job submitted gets to run its stages; when a stage blocks (e.g. waiting on a shuffle) or finishes, the next stage in line gets a chance. If a second job was submitted later, its stages sit behind the first job’s stages until those make progress or complete.

That’s simple and predictable, but it can be unfair. One long or heavy job can starve others. If you’re sharing an application—for example, many users in a single Thrift server—you usually want some form of **fair sharing** so that no single job or user monopolizes the cluster.

---

## Fair scheduling and pools

Spark can run the Task Scheduler in **fair** mode instead of FIFO. In fair mode, the root of the scheduling tree can have multiple **pools**. Each pool has:

- A **name** (e.g. “default”, “analytics”, “users”).
- A **scheduling mode** for ordering the pool’s own children: FIFO or FAIR (so you can have fair sharing between pools but FIFO within a pool).
- A **minimum share**: a number of slots (e.g. tasks) that the pool is guaranteed, if it needs them, before excess is shared.
- A **weight**: how much of the *extra* resources (beyond minimums) this pool gets relative to others.

When a job is submitted, it is assigned to a **pool**. That assignment comes from a property (e.g. “use pool X”) set on the thread that submits the job—so different threads or users can set different pool names and their jobs go into different pools. If the named pool doesn’t exist yet, Spark can create it with default settings. So you can have one pool per user, one per team, or one per priority level.

---

## How the fair ordering works

When the scheduler has to decide which task set gets the next chance at resource offers, it doesn’t look at a single flat queue. It walks the **tree**: root → pools → task sets. At each level, it **sorts** the children using the pool’s policy (FIFO or fair).

Under the **fair** policy, the idea is to balance who is “behind”:

- **Needy** means a pool (or task set) is running fewer tasks than its **minimum share**. Needy pools and task sets get priority over non-needy ones. So every pool can reach at least its minimum share before others get more.
- Among **needy** candidates, the one with the *lowest* ratio (running tasks ÷ minimum share) goes first. So the most under-served gets the next slot.
- Among **non-needy** candidates (everyone already at or above their minimum share), the one with the *lowest* ratio (running tasks ÷ weight) goes first. So extra capacity is shared in proportion to **weights**: a pool with weight 2 gets roughly twice as many extra slots as a pool with weight 1.

Within a pool, if the mode is FIFO, stages are ordered by submission order (and stage id). If the mode is FAIR, the same fair rule applies to the task sets inside that pool. So you get a consistent ordering from root down to the actual task sets, and that ordering decides which stage gets to place tasks when offers arrive.

---

## How jobs get into a pool

A job carries **properties** (e.g. a pool name, or a job group). Those properties are usually set on the **thread** that will submit the job—for example, “all jobs from this thread use pool *analytics*.” So before running an action, the application (or the Thrift server, or your framework) sets the pool name for the current thread; when the job is submitted, the scheduler reads that property and adds the job’s task sets to the corresponding pool. If the pool doesn’t exist, fair mode can create it with default minimum share and weight. Pools can also be **predefined** in a config file (e.g. an XML file), with explicit minimum shares and weights per pool, so that important groups get guaranteed capacity.

In FIFO mode, the pool property is ignored: everything goes into the single root queue. So fair sharing only applies when the scheduler is configured for **fair** mode.

---

## What you see in practice

With FIFO, one long job can keep others waiting. With fair scheduling and pools, multiple jobs (or users) share the cluster: each pool can get at least its minimum share, and beyond that, capacity is split by weight. So you can give a “premium” pool a higher weight and a minimum share, and a “batch” pool a lower weight, and both make progress. The exact order in which tasks launch still depends on locality and delay scheduling—the pool ordering only decides *which* task set gets to choose from the current resource offers first. But over time, fair scheduling prevents one job from monopolizing the executors and keeps the workload balanced across pools.

---

## Summary

When multiple jobs run in one application, the Task Scheduler orders their task sets using a **scheduling tree**. By default (**FIFO**), there is a single queue: first-submitted job’s stages go first. In **fair** mode, the tree has **pools**; each pool has a minimum share and a weight. Jobs are assigned to pools (e.g. via a thread-local property). The scheduler orders pools and task sets by a **fair** rule: needy ones (below minimum share) get priority, then extra capacity is shared by weight. So fair scheduling gives you controlled sharing when many jobs or users compete for the same executors—and prevents one job from starving the rest.
