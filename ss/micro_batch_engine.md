# Batch by Batch: Inside the Structured Streaming Micro-Batch Engine

This is the story of how Structured Streaming processes a continuous stream of data by breaking it into a rapid sequence of small batch jobs. The micro-batch engine is the core execution model of Structured Streaming—the mechanism that turns an unbounded stream into a series of bounded computations, each producing a committed result. Understanding how each micro-batch is planned, executed, and committed explains the latency characteristics of streaming jobs, the meaning of checkpoints, and what happens when a batch fails or the driver restarts.

---

## The streaming abstraction: an infinite query that keeps running

When you define a Structured Streaming query—a DataFrame read from Kafka, transformed and written to a Delta table—you are defining a query that will run continuously. But Spark doesn't run it continuously in the sense of a long-lived operator pipeline. Instead, it runs the query in a loop, one micro-batch at a time, forever.

Each iteration of the loop—each micro-batch—is a complete Spark job. It reads a bounded slice of the input stream, applies all your transformations, and writes the output. When it finishes, the loop immediately starts the next micro-batch. From the outside it looks like streaming; inside, it is batch computation applied repeatedly.

> **Think of a newspaper printing press that runs hourly.** Each hour's edition is a complete batch: gather the news that arrived in the last hour, typeset it, print it, distribute it. The press runs continuously—but each run produces one bounded, complete edition. Readers see a continuous flow of news; the printing process is a sequence of bounded jobs. Structured Streaming's micro-batches are those hourly editions, just shrunk to seconds.

This approach is powerful because it reuses all of Spark's batch execution infrastructure: the DAG Scheduler, Catalyst optimizer, fault tolerance, shuffle, and so on. A streaming query is just a batch query that keeps rerunning with new input data.

---

## The StreamExecution thread: the conductor

Each streaming query runs in its own **StreamExecution** thread on the driver. This thread is the conductor of the micro-batch loop. It owns the query's lifecycle: starting and stopping the query, deciding when to run the next micro-batch, managing offsets, interacting with sources and sinks, and committing progress.

The StreamExecution thread runs the micro-batch loop indefinitely until the query is stopped or a terminal error occurs. If a micro-batch fails with a recoverable error, the loop catches the exception, waits briefly, and retries. For unrecoverable errors, the query terminates.

---

## Phase 1: determining the input — offsets

Each micro-batch starts with a question: **what new data is available?** The StreamExecution thread asks each source for the latest offsets—the position up to which data is available in that source. For a Kafka source, an offset is a per-partition topic offset. For a file source, it is a list of new files.

The thread **commits** the new offsets to the **offset log**—a write-ahead log stored in the checkpoint directory. Writing the offsets before reading the data is critical for fault tolerance: if the driver fails after writing offsets but before finishing the batch, on recovery it knows exactly what data the next batch should process. The offsets are durable before any data moves.

> **The offset log is like a library checkout slip.** Before you take the books from the shelf (process the data), you fill out the checkout form (write the offsets). If you drop the books on the way out, the librarian can see from the checkout slip exactly which books you were carrying—and give them back to you to re-carry. Without the slip, no one knows what you had.

---

## Phase 2: building and running the batch job

With the input range determined, StreamExecution constructs a **DataFrame** representing that slice of data from each source. This DataFrame is then combined with all the transformations in the user's query definition. The result is a logical plan for this micro-batch—exactly as if you had written a batch query reading a bounded dataset.

This logical plan goes through the full Catalyst pipeline: analysis, optimization, physical planning, codegen. The plan is submitted to the DAG Scheduler as a regular Spark job. Tasks run on executors just like any batch job.

For **stateful operations** (windowed aggregations, stream-stream joins), the batch job reads and updates state from the **state store**. The state store is a per-partition key-value store that persists across micro-batches. Each batch reads the relevant state, computes updates, writes the new state back, and the state store snapshots its contents to the checkpoint directory.

---

## Phase 3: committing — making progress durable

After the batch job completes successfully, StreamExecution writes the processed offsets to the **commit log**. The commit log entry says: "I have successfully processed data up to these offsets; the next micro-batch should start from here." Writing to the commit log is the moment of **progress commitment**.

The checkpoint directory therefore contains two logs: the **offset log** (what we decided to process) and the **commit log** (what we finished processing). On recovery, the driver replays the last committed offset and the next unprocessed offset range to determine where to resume.

> **The offset log and commit log are like a two-part recipe journal.** Before starting to cook a dish, you write "I'm making Dish 37" (offset log). After finishing it, you write "Dish 37: done" (commit log). If the kitchen burns down halfway through, when you rebuild you know to remake Dish 37—because it's in the offset log but not the commit log. You never lose track of which dishes are done and which aren't.

---

## Triggers: when does the next batch run?

The trigger controls when each micro-batch starts. There are four options:

**ProcessingTime("interval")** — the most common trigger. Batches run on a fixed schedule: every second, every 10 seconds, every minute. If a batch completes before the interval is up, StreamExecution waits. If a batch takes longer than the interval, the next batch starts immediately after. This is the right trigger for near-real-time processing with bounded latency.

**Once** — a single batch is run consuming all available data, then the query stops. Useful for incremental batch processing rather than continuous streaming.

**AvailableNow** — like Once but may run multiple batches until all available data is consumed, then stops. Useful for backfill scenarios.

**Continuous(checkpoint_interval)** — an experimental engine that processes records continuously rather than in micro-batches, with very low latency. It uses a different execution model and has limited operator support.

---

## Watermark advancement and state cleanup

At the start of each micro-batch, StreamExecution also advances the **watermark** based on the maximum event time seen in the previous batch. State entries for windows or joins whose event time has passed the watermark are removed from the state store at the end of the batch where they become eligible. This bounds state store growth.

Without watermarks, state grows without bound—the streaming query eventually OOMs. Watermarks are the release valve: they tell the state store "you can forget everything before this timestamp."

---

## Failure and recovery: the checkpoint contract

If the driver crashes mid-batch—after writing the offset log entry but before writing the commit log entry—the query re-runs that batch on restart. The re-run reads the same input range and applies the same transformations. For idempotent sinks (like Delta tables), this re-run produces the same output, which the sink accepts without duplication.

The checkpoint directory is the entire persistent state of a streaming query. Deleting it resets the query to the beginning. Pointing a new query instance at the same checkpoint directory lets it resume exactly where the previous instance left off.

---

## Bringing it together

The micro-batch engine turns a continuous stream into a rapid loop of small batch jobs. Each iteration is three phases: **determine input** (ask sources for new offsets, write them to the offset log), **run the batch** (build a logical plan from the input range and the query's transformations, execute it as a Spark job, read and write state via the state store, write output to the sink), and **commit progress** (write processed offsets to the commit log). The trigger controls when each batch starts; the checkpoint directory contains the offset log, commit log, and state store snapshots needed to recover after a failure. The StreamExecution thread on the driver is the conductor of the entire loop. So the story of a micro-batch is: **new offsets → plan a batch job → execute → update state → commit → repeat.** Every batch is exactly the same machinery as a regular Spark job, applied to a new slice of the stream.
