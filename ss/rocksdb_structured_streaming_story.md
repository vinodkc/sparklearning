# RocksDB in Structured Streaming: The Story

## The problem: streaming never forgets

In batch, you see all data at once. In streaming, data arrives in small chunks over time. If you want to do things like "count per key," "drop duplicates," or "join two streams," you can't only look at the current chunk. You have to **remember** what you saw in earlier chunks.

So streaming needs **state**: a place that survives from one micro-batch to the next and that can be restored after a crash. That place is the **state store**.

> **Think of the state store as a running scoreboard at a sporting event.** Each time new game data arrives (a new micro-batch), you update the scoreboard. The scoreboard persists between halves, between days. When the stadium has a power outage (driver crash), you need to restore the scoreboard from a backup. The state store is that scoreboard, and the checkpoint is the backup.

---

## The default store: everything in memory

Spark's first state store kept everything in the JVM: a big in-memory map on each executor. When a micro-batch finished, Spark wrote a copy of that map to your checkpoint directory (e.g. HDFS) so you could recover.

That's simple and fast when the state is small. But when you have **millions of keys**—e.g. one row per user, or per session—that map gets huge. The JVM has to hold it all, and the garbage collector has to walk it. You get long GC pauses and micro-batches that sometimes take seconds instead of milliseconds.

> **It's like trying to run a customer service desk where the agent has to memorise every customer's entire history.** With 10 customers, fine. With 10 million customers, the agent's mental overhead collapses the operation. You need a filing cabinet (RocksDB) rather than trying to hold it all in the agent's head (JVM memory).

---

## Enter RocksDB: state outside the JVM

RocksDB is an embedded key-value store. Spark can use it as **another implementation** of the same state store idea: same API (get, put, remove, commit), but a different way of storing data.

Instead of one giant map in the JVM, RocksDB keeps data in **its own memory and on local disk** on the executor. It's designed for large datasets and heavy write traffic. So when you have a lot of state, the JVM stays lighter, GC is happier, and micro-batch latency is more stable.

> **RocksDB is like moving the customer history from the agent's head into a laptop at the desk.** The agent doesn't memorise everything—they look it up when needed. The laptop is fast for the current customer (hot data in memory) and can page to its SSD for older records. The agent's mental load drops; the filing speed stays high.

---

## Where does the state actually live?

Think of it in two layers.

**On the executor (where the work happens)**  
RocksDB runs there. It has a **working directory** on local disk (and uses memory for hot data). All reads and writes during a micro-batch hit this local RocksDB. So **state is always used locally**; that's where the "story" of your streaming job runs.

**In the checkpoint (your safety net)**  
You give the query a checkpoint location (e.g. on HDFS or S3). After each successful micro-batch, Spark **copies** what's needed from that local RocksDB into the checkpoint: either a full snapshot of the store or a log of changes (changelog). So the checkpoint is a **backup** of the state, not the place where the engine runs. It's there for recovery and for moving state to another executor if the scheduler sends the next batch somewhere else.

So: **live state = local RocksDB; durability and recovery = checkpoint (e.g. HDFS).**

---

## The rhythm of each micro-batch

Every micro-batch follows the same pattern.

**Before the batch**  
The task needs the latest state. So it **loads** it: it goes to the checkpoint, finds the right version (last committed one), and either restores a snapshot or restores an older snapshot and replays changelogs. Now the task has the same state the previous batch left behind.

**During the batch**  
The streaming operator (aggregation, dedup, join, etc.) runs. For each input row it might look up a key, update a value, or add a new key. All of that goes to the **local** RocksDB. No network, no distributed store—just the executor and its local RocksDB.

**After the batch**  
The task **commits**. RocksDB flushes its in-memory changes to local disk. Then Spark creates a checkpoint from that: it either uploads a new snapshot or writes a changelog. That new version becomes the starting point for the next batch.

> **It's like a cashier taking over a shared till.** At the start of their shift (load from checkpoint), they bring the till to the counter with the last balance already in it. During the shift (during the batch), they process all transactions locally. At the end of the shift (commit), they write up the closing balance report and file it in the safe (checkpoint). The next cashier restores from that report.

So the story of each batch is: **load last version → do work on local RocksDB → commit a new version to the checkpoint.**

---

## Snapshots and changelogs (two ways to remember)

When committing, Spark can persist state in two ways.

**Snapshot**  
A point-in-time copy of the whole RocksDB state. Restoring is simple: download that snapshot and open RocksDB on it. But creating and uploading it can be heavy when state is large.

**Changelog**  
Instead of uploading the full state every time, Spark can write only the **changes** made in that batch (puts and deletes). That's small and fast. To restore a version, you load the latest snapshot before that version and replay changelogs up to the desired version.

> **Snapshots are like a full photograph of your desk every day. Changelogs are like a daily diary entry: "moved the stapler, added three new Post-it notes, removed last week's memos."** Restoring from a diary is: find the most recent photo, then apply each diary entry forward. It's more steps but the diary entries are tiny compared to daily photographs.

---

## Cleanup: not everything is kept forever

If you kept every snapshot and every changelog, the checkpoint would grow without bound. So a **maintenance** process runs periodically: it deletes old snapshot and changelog files that are older than what you need to retain. That way the checkpoint only keeps a bounded history of versions.

---

## Bringing it together

RocksDB in Structured Streaming is the **engine that holds the story** of your stream: all the keys and values that aggregation, deduplication, and joins need across time. It lives on the executor, so each batch runs against local, fast storage. The checkpoint is the **backup** of that story, so you can recover or move to another executor. RocksDB is there so that when the story gets long—millions of keys, lots of updates—the system can still tell it without choking the JVM, and without turning your stream into a slow, GC-bound job.
