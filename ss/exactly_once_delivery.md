# Exactly Once, For Real: How Structured Streaming Guarantees No Duplicates

This is the story of the strongest guarantee in Structured Streaming: **exactly-once processing**. In a streaming system that processes data continuously, things go wrong—drivers crash, executors fail, network partitions occur. The question isn't whether failures happen but what their effect is on the output. At-most-once processing loses data on failure. At-least-once processing re-processes data on failure, potentially producing duplicates. Exactly-once processing means that even in the presence of failures, every input record is reflected exactly once in the output—no losses, no duplicates. How Structured Streaming achieves this is a story of write-ahead logs, idempotent sinks, and transactional commits.

---

## Three delivery guarantees

**At-most-once**: the system processes each record at most once. On failure, some records may be lost entirely. Easy to implement: if something fails, just skip it. Acceptable for metrics or logs where occasional data loss is tolerable.

**At-least-once**: the system processes each record at least once. On failure, records are re-processed, potentially more than once. This avoids data loss but can produce duplicates in the output. Acceptable for aggregations where duplicates can be de-duplicated downstream, or for idempotent sinks.

**Exactly-once**: each record is reflected exactly once in the output. No losses and no duplicates. The hardest to achieve, because it requires coordination between the source (tracking which records were read), the engine (tracking which records were processed), and the sink (ensuring output is not duplicated on retry).

Structured Streaming targets exactly-once for supported sources and sinks. The mechanism is different for the source side (tracking what was read) and the sink side (ensuring writes are not duplicated).

---

## Source side: the offset log as a write-ahead log

The source side of exactly-once is handled by the **offset log** in the checkpoint directory. Before a micro-batch reads any data, StreamExecution writes the batch's input offsets to the offset log. This write happens first, before the Spark job is submitted.

If the driver crashes after writing offsets but before the batch completes, the recovered driver reads the offset log and knows exactly which offsets the interrupted batch intended to process. It re-runs that batch with the same offsets. Because the offsets are written ahead of the computation—the write-ahead log pattern—the engine always knows what to process, even after a crash.

On recovery, StreamExecution reads the offset log and the commit log. The offset log says "the last batch we started was for offsets X → Y." The commit log says "the last batch we completed was for offsets A → B." If X ≠ A (meaning a batch started but didn't commit), the engine re-runs the uncommitted batch. This re-run produces the same output as the original run would have, as long as the source can replay the same data (which Kafka, rate source, and file sources can; network socket sources cannot, which is why socket is not suitable for production).

This re-run is what achieves at-least-once on the source side: no record is lost. To achieve exactly-once overall, the sink must also handle the re-run without duplicating output.

---

## Sink side: idempotent writes and transactional commits

The sink is where exactly-once gets hard. If a batch writes output and then the driver crashes before committing, the next run re-processes the same input and re-writes the same output. Without any sink-side protection, this produces duplicates.

There are two approaches to preventing sink-side duplication.

**Idempotent writes**: the sink handles duplicate writes naturally, because writing the same data twice has the same effect as writing it once. This can be achieved by including the batch ID or a deterministic record key in the output. For example, writing to Delta Lake with a merge on a natural key ensures that re-writing the same records on retry has no effect beyond what the first write achieved. HDFS file sinks in Spark use deterministic file naming: the output file for batch N, task M always has the same name, so writing it again on a retry simply overwrites the previous (identical) file. As long as the file content is deterministic (same batch ID, same task ID, same input data), the idempotent write is safe.

**Transactional commits**: the sink supports a two-phase commit protocol. The batch writes its output in a "pending" or "staged" state that is not yet visible to readers. Only after the batch job completes successfully does the engine call the sink's commit method, which atomically makes the output visible. If the driver crashes after writing but before committing, the staged (invisible) output is discarded on recovery and the batch re-runs. On the re-run, output is staged again and committed once the batch succeeds. Readers never see a partially written batch.

Delta Lake supports transactional commits via its transaction log (covered in the Delta Lake story). Kafka sinks support transactional writes using Kafka's own transaction API. The Foreach and ForeachBatch sinks require the user to implement idempotency themselves—Spark can't know what an arbitrary function does.

---

## The two-phase commit protocol in Structured Streaming

Structured Streaming formalizes the transactional sink pattern through the `StreamingWrite` API. A sink that supports exactly-once implements two methods:

**`commit(epochId)`**: called after the batch's tasks have all written their output to a staging area. The sink atomically promotes the staged output to the committed state. For Delta, this is writing a new transaction log entry. For a file sink, this might be renaming staged files to their final location.

**`abort(epochId)`**: called if the batch fails. The sink discards staged output.

The `epochId` (or batch ID) is deterministic and stable: if a batch is re-run after a failure, it has the same epoch ID. A sink that has already committed a given epoch ID can simply no-op when `commit(epochId)` is called again—it knows that epoch was already committed. This makes the protocol idempotent at the epoch level: committing the same epoch twice is safe.

---

## End-to-end exactly-once: the full picture

For end-to-end exactly-once, all three components must cooperate:

1. **The source must be replayable**: it must be able to re-deliver the exact same data for a given offset range after a failure. Kafka, Delta as a streaming source, and file sources satisfy this. Sources like network sockets or `socketTextStream` do not, because data is consumed and gone once read.

2. **The engine must track progress durably**: the offset log (write-ahead) and commit log (write-after) in the checkpoint directory ensure the engine knows exactly what was processed and what needs to be re-processed on recovery.

3. **The sink must be idempotent or transactional**: either re-writing the same data has no additional effect (idempotent), or the sink uses a two-phase commit so that only fully-committed batches are visible (transactional).

When all three conditions are met, the end-to-end guarantee is exactly-once: every input record is reflected exactly once in the output, regardless of failures.

---

## What breaks exactly-once

**Non-replayable sources**: if your source can't replay data for a given offset range, re-runs on failure will process different data or no data. The guarantee degrades to at-most-once.

**Non-idempotent external side effects in ForeachBatch**: if your `ForeachBatch` function calls an external API (sends an email, increments a counter in a database) without any deduplication logic, re-runs will duplicate those effects. Spark can't protect you here—idempotency is your responsibility.

**Deleting or corrupting the checkpoint**: the checkpoint directory is the source of truth for what has been processed. Deleting it resets the query; the next run starts from the configured starting offset (or the earliest available). Data between the last committed offset and the reset point will be re-processed (at-least-once) or lost (at-most-once), depending on source and sink properties.

**Using an unsupported sink**: not all sinks support the transactional commit API. Console, memory, and custom ForeachBatch sinks without idempotency logic provide at-least-once at best.

---

## Bringing it together

Structured Streaming achieves exactly-once end-to-end through three interlocking mechanisms. The **offset log** acts as a write-ahead log: offsets are committed before the batch runs, so on failure the engine always knows what to re-process. The **commit log** records completed batches, so the engine doesn't re-run what already succeeded. The **sink** either handles duplicate writes idempotently (same epoch ID, same output, second write is a no-op) or uses a two-phase commit (stage output, commit atomically only on success, discard on failure). With a replayable source, a durable checkpoint, and an idempotent or transactional sink, every record is reflected in the output exactly once—no data lost, no data duplicated, regardless of how many times the driver or executors fail.
