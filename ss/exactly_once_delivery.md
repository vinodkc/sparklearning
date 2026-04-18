# Exactly Once, For Real: How Structured Streaming Guarantees No Duplicates

This is the story of the strongest guarantee in Structured Streaming: **exactly-once processing**. In a streaming system that processes data continuously, things go wrong—drivers crash, executors fail, network partitions occur. The question isn't whether failures happen but what their effect is on the output. At-most-once processing loses data on failure. At-least-once processing re-processes data on failure, potentially producing duplicates. Exactly-once processing means that even in the presence of failures, every input record is reflected exactly once in the output—no losses, no duplicates. How Structured Streaming achieves this is a story of write-ahead logs, idempotent sinks, and transactional commits.

---

## Three delivery guarantees

**At-most-once**: the system processes each record at most once. On failure, some records may be lost entirely. Easy to implement: if something fails, just skip it. Acceptable for metrics or logs where occasional data loss is tolerable.

**At-least-once**: the system processes each record at least once. On failure, records are re-processed, potentially more than once. This avoids data loss but can produce duplicates in the output.

**Exactly-once**: each record is reflected exactly once in the output. No losses and no duplicates. The hardest to achieve, because it requires coordination between the source (tracking which records were read), the engine (tracking which records were processed), and the sink (ensuring output is not duplicated on retry).

> **Think of it like receiving a package delivery.** At-most-once: if the courier drops the package in a puddle, the order is marked "delivered" anyway—you might get nothing. At-least-once: if the delivery is uncertain, the courier brings it again the next day to be sure—you might get two packages. Exactly-once: the courier uses a tracked, signed delivery system. If the first attempt fails, they retry, but the warehouse is smart enough to know the item was already dispatched and not ship a second copy.

---

## Source side: the offset log as a write-ahead log

The source side of exactly-once is handled by the **offset log** in the checkpoint directory. Before a micro-batch reads any data, StreamExecution writes the batch's input offsets to the offset log. This write happens first, before the Spark job is submitted.

If the driver crashes after writing offsets but before the batch completes, the recovered driver reads the offset log and knows exactly which offsets the interrupted batch intended to process. It re-runs that batch with the same offsets. Because the offsets are written ahead of the computation—the write-ahead log pattern—the engine always knows what to process, even after a crash.

> **The offset log is like writing your grocery list before you go to the store.** If you get halfway through the shopping and have to go home sick, you still have the list. When you come back, you start from the list—not from memory of what you might have bought. You know exactly what the "batch" was supposed to contain, and you re-do it completely.

---

## Sink side: idempotent writes and transactional commits

The sink is where exactly-once gets hard. If a batch writes output and then the driver crashes before committing, the next run re-processes the same input and re-writes the same output. Without any sink-side protection, this produces duplicates.

There are two approaches to preventing sink-side duplication.

**Idempotent writes**: the sink handles duplicate writes naturally, because writing the same data twice has the same effect as writing it once. For example, writing to Delta Lake with a merge on a natural key ensures that re-writing the same records on retry has no effect beyond what the first write achieved. HDFS file sinks in Spark use deterministic file naming: the output file for batch N, task M always has the same name, so writing it again on a retry simply overwrites the previous (identical) file.

> **Idempotent writes are like submitting a form with a duplicate-prevention code.** The form has a unique ID based on the batch and task. If the form is submitted twice (due to a retry), the system says "we already processed ID 42—ignoring the duplicate." The second submission is acknowledged but has no additional effect.

**Transactional commits**: the sink supports a two-phase commit protocol. The batch writes its output in a "pending" state that is not yet visible to readers. Only after the batch job completes successfully does the engine call the sink's commit method, which atomically makes the output visible. If the driver crashes after writing but before committing, the staged (invisible) output is discarded on recovery.

> **Transactional commits are like a sealed envelope in an escrow.** The letter (output) is written and placed in escrow (staging), but not delivered to the recipient until both parties sign off (the batch commits). If one party disappears mid-process, the escrow agent destroys the letter and the process starts over. The recipient never sees a half-delivered letter.

---

## The two-phase commit protocol in Structured Streaming

Structured Streaming formalizes the transactional sink pattern through the `StreamingWrite` API. A sink that supports exactly-once implements two methods:

**`commit(epochId)`**: called after the batch's tasks have all written their output to a staging area. The sink atomically promotes the staged output to the committed state. For Delta, this is writing a new transaction log entry. For a file sink, this might be renaming staged files to their final location.

**`abort(epochId)`**: called if the batch fails. The sink discards staged output.

The `epochId` (or batch ID) is deterministic and stable: if a batch is re-run after a failure, it has the same epoch ID. A sink that has already committed a given epoch ID can simply no-op when `commit(epochId)` is called again. This makes the protocol idempotent at the epoch level: committing the same epoch twice is safe.

---

## End-to-end exactly-once: the full picture

For end-to-end exactly-once, all three components must cooperate:

1. **The source must be replayable**: it must be able to re-deliver the exact same data for a given offset range after a failure. Kafka, Delta as a streaming source, and file sources satisfy this. Sources like network sockets do not.

2. **The engine must track progress durably**: the offset log (write-ahead) and commit log (write-after) in the checkpoint directory ensure the engine knows exactly what was processed and what needs to be re-processed on recovery.

3. **The sink must be idempotent or transactional**: either re-writing the same data has no additional effect (idempotent), or the sink uses a two-phase commit so that only fully-committed batches are visible (transactional).

When all three conditions are met, the end-to-end guarantee is exactly-once: every input record is reflected exactly once in the output, regardless of failures.

---

## What breaks exactly-once

**Non-replayable sources**: if your source can't replay data for a given offset range, re-runs on failure will process different data or no data. The guarantee degrades to at-most-once.

**Non-idempotent external side effects in ForeachBatch**: if your `ForeachBatch` function calls an external API (sends an email, increments a counter in a database) without any deduplication logic, re-runs will duplicate those effects. Spark can't protect you here—idempotency is your responsibility.

**Deleting or corrupting the checkpoint**: the checkpoint directory is the source of truth for what has been processed. Deleting it resets the query; data between the last committed offset and the reset point will be re-processed or lost.

**Using an unsupported sink**: not all sinks support the transactional commit API. Console, memory, and custom ForeachBatch sinks without idempotency logic provide at-least-once at best.

---

## Bringing it together

Structured Streaming achieves exactly-once end-to-end through three interlocking mechanisms. The **offset log** acts as a write-ahead log: offsets are committed before the batch runs, so on failure the engine always knows what to re-process. The **commit log** records completed batches, so the engine doesn't re-run what already succeeded. The **sink** either handles duplicate writes idempotently (same epoch ID, same output, second write is a no-op) or uses a two-phase commit (stage output, commit atomically only on success, discard on failure). With a replayable source, a durable checkpoint, and an idempotent or transactional sink, every record is reflected in the output exactly once—no data lost, no data duplicated, regardless of how many times the driver or executors fail.
