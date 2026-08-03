# Delta in Motion: Streaming Reads and Exactly-Once Sinks

This is the story of Delta Lake as a participant in Structured Streaming—not only a batch table format. As a **source**, Delta exposes an ordered stream of table versions: each micro-batch advances through committed versions and reads the data those versions imply—typically the contents of newly **Add**ed files. Plain file-based streaming does not by itself emit proper change rows for copy-on-write updates/deletes; for that you enable **Change Data Feed (CDF)** so the stream can surface insert/update/delete change rows. As a **sink**, each micro-batch stages Parquet data and commits one atomic transaction-log entry, which is how streaming achieves **exactly-once** output into the lake. Understanding both sides explains CDC-style pipelines, why checkpointing still matters, and how streaming creates the small-file pressure that OPTIMIZE must later cure.

---

## Delta as a streaming source: versions as offsets

File sources often track which files were seen. Delta can do better: the **transaction log is already a versioned timeline**. A streaming read on a Delta table treats versions (or the files implied by version ranges) as the progress cursor.

Roughly:

1. The stream starts from a starting version (latest, or a specified position / timestamp policy)
2. Each micro-batch selects the next range of committed versions
3. The batch's data is the net data files to read for that range—commonly files **added** by those commits (with semantics for updates/deletes depending on read mode and options)
4. Progress is recorded in the streaming **checkpoint** so recovery reprocessing the same version range does not invent a new cursor

> **Streaming Delta is like reading a magazine by issue number.** Issue 42, then 43, then 44—each issue is a committed version. Your bookmark (offset/checkpoint) is the last issue you finished. If you fall asleep and wake up, you resume at 45; you do not randomly re-open pages from the recycling bin.

Because the log defines order, concurrent batch writers and streaming readers coexist under snapshot rules: the stream only advances through **committed** versions.

**Change Data Feed (CDF).** When enabled on the table (`delta.enableChangeDataFeed`), commits can record row-level changes and readers/streams can consume insert/update/delete images (including from a `_change_data` area) instead of only "new Add file contents." That is still log-driven: the transaction history is the changelog; CDF is how Spark exposes it as typed change rows.

---

## Delta as a streaming sink: one commit per batch

On the write path, a micro-batch that ends in Delta typically:

1. Writes Parquet files for that batch's output (task outputs staged as data files)
2. On successful batch completion, the driver **commits a single new Delta version** listing those `Add` actions (and any `Remove`s for sink patterns that rewrite)

That commit is the same atomic log PUT described in the transaction-log story. Structured Streaming's sink contract—commit only after the batch succeeds, recover using the offset log—pairs with Delta's atomic publish:

- If the batch fails before the Delta commit, staged files are not part of any snapshot (later vacuum/cleanup may remove orphans)
- If the batch retries after a commit already landed, the engine and sink design aim to avoid double-committing the same batch's logical output—the **exactly-once** story's transactional sink pattern

> **Each micro-batch is a sealed shipping container with one customs stamp.** Tasks pack the container (Parquet files). Customs (the Delta commit) either accepts the whole container into inventory or rejects it. There is no "half a container is live in the warehouse."

Append sinks are the straightforward case. Upsert-style sinks use **foreachBatch** + `MERGE` (or related APIs): each batch runs a transactional merge, which is copy-on-write DML under the same OCC rules—more conflict and amplification risk, same atomic visibility.

---

## Checkpoints: streaming memory vs table history

Two histories must not be confused:

- **Streaming checkpoint** (offset log, commit log under the checkpoint path) — where *this query* left off
- **Delta `_delta_log`** — the table's global version history

Losing the streaming checkpoint can cause a query to restart from a wrong starting version (reprocess or skip), even if the Delta table is perfect. Conversely, vacuuming Delta history does not replace streaming checkpoint hygiene. Operational runbooks need both.

---

## The small-file feedback loop

Streaming sinks often commit **frequently** with **modest data per commit**. That is correctness-friendly and layout-hostile: hundreds of versions a day, each adding small files. Snapshot reads and downstream streams then inherit the small-file problem.

Mitigations sit across stories:

- Tune trigger interval / batch size vs latency needs
- Coalesce inside `foreachBatch` before write when safe
- Schedule **OPTIMIZE** (and clustering) on hot tables
- Prefer fewer, larger commits when business latency allows

Delta does not free you from physics; it makes the mess **transactionally visible** and compactable.

---

## Exactly-once in one paragraph

Structured Streaming records intended offsets first; tasks write output; the Delta sink commits one log version for the batch. On failure and retry, the same offsets are processed again, but a correctly coordinated Delta commit does not publish duplicates into the table's latest snapshot. Details and failure edge cases live in the exactly-once story; Delta is the sink that makes the "transactional commit" bullet real on object storage.

**SetTransaction and application ids.** Some exactly-once writers also commit a **SetTransaction** action: an application id plus a batch/version number saying "this app already committed batch N." On retry, Delta can recognize the same `(appId, version)` and avoid applying the batch's data effects twice even if the streaming layer re-attempts the sink commit. Streaming checkpoints remain the query's bookmark; SetTransaction is an extra idempotency receipt **inside the table log**—useful when multiple writers or replay paths need the *table* itself to remember what was done.

---

## Bringing it together

**Delta as a source** turns table versions into a stream cursor—micro-batches walk the log and read the files (or change rows) each version range implies. **Delta as a sink** turns each micro-batch into staged Parquet plus **one atomic `_delta_log` commit**, which is why streaming can land exactly-once into the lake. Checkpoints track query progress; the transaction log tracks table truth; OPTIMIZE repairs the small files frequent commits create. Once you see streaming as "versions in, versions out," Delta stops being only a batch ACID format and becomes the backbone of incremental pipelines—still sitting on the same Parquet files and the same snapshot rules as every other story in this chapter.
