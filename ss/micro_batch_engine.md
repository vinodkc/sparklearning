# Batch by Batch: Inside the Structured Streaming Micro-Batch Engine

This is the story of how Structured Streaming processes a continuous stream of data by breaking it into a rapid sequence of small batch jobs. The micro-batch engine is the core execution model of Structured Streaming—the mechanism that turns an unbounded stream into a series of bounded computations, each producing a committed result. Understanding how each micro-batch is planned, executed, and committed explains the latency characteristics of streaming jobs, the meaning of checkpoints, and what happens when a batch fails or the driver restarts.

---

## The streaming abstraction: an infinite query that keeps running

When you define a Structured Streaming query—a DataFrame read from Kafka, transformed and written to a Delta table—you are defining a query that will run continuously. But Spark doesn't run it continuously in the sense of a long-lived operator pipeline. Instead, it runs the query in a loop, one micro-batch at a time, forever.

Each iteration of the loop—each micro-batch—is a complete Spark job. It reads a bounded slice of the input stream, applies all your transformations, and writes the output. When it finishes, the loop immediately starts the next micro-batch. From the outside it looks like streaming; inside, it is batch computation applied repeatedly.

This approach is powerful because it reuses all of Spark's batch execution infrastructure: the DAG Scheduler, Catalyst optimizer, fault tolerance, shuffle, and so on. A streaming query is just a batch query that keeps rerunning with new input data.

---

## The StreamExecution thread: the conductor

Each streaming query runs in its own **StreamExecution** thread on the driver. This thread is the conductor of the micro-batch loop. It owns the query's lifecycle: starting and stopping the query, deciding when to run the next micro-batch, managing offsets, interacting with sources and sinks, and committing progress.

The StreamExecution thread runs the micro-batch loop indefinitely until the query is stopped (by calling `query.stop()`) or a terminal error occurs. If a micro-batch fails with a recoverable error (a transient network issue, a temporary source unavailability), the loop catches the exception, waits briefly, and retries. For unrecoverable errors, the query terminates.

---

## Phase 1: determining the input — offsets

Each micro-batch starts with a question: **what new data is available?** The StreamExecution thread asks each source for the latest offsets—the position up to which data is available in that source. For a Kafka source, an offset is a per-partition topic offset (e.g., partition 0: offset 1000, partition 1: offset 2000). For a file source, it is a list of new files.

The thread **commits** the new offsets to the **offset log**—a write-ahead log stored in the checkpoint directory. Writing the offsets before reading the data is critical for fault tolerance: if the driver fails after writing offsets but before finishing the batch, on recovery it knows exactly what data the next batch should process. The offsets are durable before any data moves.

The range of data to process in this micro-batch is the difference between the previous committed offsets (stored in the commit log) and the new offsets. This range is the micro-batch's "input."

---

## Phase 2: building and running the batch job

With the input range determined, StreamExecution constructs a **DataFrame** representing that slice of data from each source. This DataFrame is then combined with all the transformations in the user's query definition. The result is a logical plan for this micro-batch—exactly as if you had written a batch query reading a bounded dataset.

This logical plan goes through the full Catalyst pipeline: analysis, optimization, physical planning, codegen. The plan is submitted to the DAG Scheduler as a regular Spark job. Tasks run on executors just like any batch job.

For **stateful operations** (windowed aggregations, `mapGroupsWithState`, stream-stream joins), the batch job reads and updates state from the **state store**. The state store is a per-partition key-value store that persists across micro-batches. Each batch reads the relevant state, computes updates, writes the new state back, and the state store snapshots its contents to the checkpoint directory at each micro-batch boundary.

The output of the job—the new rows computed in this micro-batch—is passed to the **sink**: a Delta table, a Kafka topic, a console, or any other output. How the sink handles the output depends on the output mode (Append, Update, Complete) and the sink's own commit protocol.

---

## Phase 3: committing — making progress durable

After the batch job completes successfully, StreamExecution writes the processed offsets to the **commit log** (a separate log from the offset log, also in the checkpoint directory). The commit log entry for this micro-batch says: "I have successfully processed data up to these offsets; the next micro-batch should start from here." Writing to the commit log is the moment of **progress commitment**: if the driver fails after this write, a recovered driver knows this batch was completed and should not be re-processed.

The checkpoint directory therefore contains two logs: the **offset log** (what we decided to process) and the **commit log** (what we finished processing). On recovery, the driver replays the last committed offset and the next unprocessed offset range to determine where to resume. If an offset was written but not committed (the batch started but didn't finish), the batch is re-run from that offset.

---

## Triggers: when does the next batch run?

The trigger controls when each micro-batch starts. There are four options:

**ProcessingTime("interval")** — the most common trigger. Batches run on a fixed schedule: every second, every 10 seconds, every minute. If a batch completes before the interval is up, StreamExecution waits until the next interval boundary before starting the next batch. If a batch takes longer than the interval, the next batch starts immediately after the current one finishes (no skipping). This is the right trigger for near-real-time processing with bounded latency.

**Once** — a single batch is run consuming all available data, then the query stops. Useful for incremental batch processing ("run once when new data arrives") rather than continuous streaming.

**AvailableNow** — like Once but may run multiple batches until all available data is consumed, then stops. Useful for backfill scenarios where you want to process a large backlog efficiently.

**Continuous(checkpoint_interval)** — an experimental engine that processes records continuously rather than in micro-batches, with very low latency. It uses a different execution model (truly continuous operators) and has limited operator support.

---

## Watermark advancement and state cleanup

At the start of each micro-batch, StreamExecution also advances the **watermark** based on the maximum event time seen in the previous batch. This is used by windowed aggregations and stream-stream joins to determine which state can be cleaned up. State entries for windows or joins whose event time has passed the watermark are removed from the state store at the end of the batch where they become eligible. This bounds state store growth.

The watermark is computed, advanced, and recorded in the checkpoint as part of each micro-batch cycle.

---

## Failure and recovery: the checkpoint contract

If the driver crashes mid-batch—after writing the offset log entry but before writing the commit log entry—the query re-runs that batch on restart. The re-run reads the same input range (same offsets) and applies the same transformations. For idempotent sinks (like Delta tables with merge-on-read), this re-run produces the same output, which the sink accepts without duplication. For non-idempotent sinks (like a raw Kafka write that just appends), the re-run may produce duplicates.

The checkpoint directory is the entire persistent state of a streaming query. Deleting it resets the query to the beginning (or wherever the source starts). Pointing a new query instance at the same checkpoint directory lets it resume exactly where the previous instance left off.

---

## Bringing it together

The micro-batch engine turns a continuous stream into a rapid loop of small batch jobs. Each iteration is three phases: **determine input** (ask sources for new offsets, write them to the offset log), **run the batch** (build a logical plan from the input range and the query's transformations, execute it as a Spark job, read and write state via the state store, write output to the sink), and **commit progress** (write processed offsets to the commit log). The trigger controls when each batch starts; the checkpoint directory contains the offset log, commit log, and state store snapshots needed to recover after a failure. The StreamExecution thread on the driver is the conductor of the entire loop. So the story of a micro-batch is: **new offsets → plan a batch job → execute → update state → commit → repeat.** Every batch is exactly the same machinery as a regular Spark job, applied to a new slice of the stream.
