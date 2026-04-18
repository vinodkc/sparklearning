# When Should the Next Batch Run? The Story of Trigger Types

This is the story of triggers—the mechanism in Structured Streaming that controls when each micro-batch starts. The trigger answers a simple but consequential question: "how often should Spark check for new data and process it?" The answer shapes everything about a streaming query's behavior: its latency, its throughput, its cost, and its interaction with sources and sinks. Different trigger types are designed for different operational goals, from near-real-time processing to controlled batch processing. Understanding what each trigger does—and what happens inside Spark when it fires—lets you choose the right one for your workload.

---

## The role of the trigger in the micro-batch loop

In Structured Streaming's micro-batch engine, the **StreamExecution** thread runs a continuous loop: determine input → run the batch → commit progress → wait → repeat. The trigger governs the "wait" step: how long does the loop pause between completing one batch and starting the next?

The trigger is set when you start a streaming query:

```python
query = df.writeStream \
    .format("delta") \
    .trigger(processingTime="10 seconds") \
    .start(path)
```

If no trigger is specified, the default is `processingTime="0 seconds"`—which means: start the next batch immediately after the previous one commits, with no intentional wait.

> **The trigger is like the pace setter for a relay race.** A fast trigger says "hand off the baton immediately after each leg." A slow trigger says "wait at the handoff zone for 30 seconds before the next runner starts, even if you arrived early." The pace setter doesn't do the running—it just controls the timing between legs.

---

## Trigger 1: ProcessingTime (interval-based)

```python
.trigger(processingTime="30 seconds")  # or Trigger.ProcessingTime("30 seconds")
```

This is the most common trigger for production streaming jobs. It sets a fixed interval between batch starts.

**How it works:**
- The first batch starts immediately when the query is started.
- After each batch completes, StreamExecution waits until the next interval boundary before starting the next batch.
- If a batch takes longer than the interval (e.g., batch takes 45 seconds with a 30-second trigger), the next batch starts immediately after the current one completes—no batches are skipped.
- If a batch completes early (e.g., batch takes 5 seconds with a 30-second trigger), StreamExecution sleeps until the 30-second mark, then starts the next batch.

**When to use it:** near-real-time processing where you need a predictable latency bound. A `processingTime="10 seconds"` trigger means output is at most 10 seconds behind the source (plus processing time). Good for dashboards, alerting, and continuous ETL.

> **ProcessingTime is like a TV news channel that updates the ticker every 30 seconds.** If the latest news came in at second 8, it appears at second 30. If preparing the update takes 40 seconds (longer than the interval), the next update starts immediately after—the ticker doesn't wait for the next exact 30-second boundary. But it never publishes faster than the interval if data is sparse.

**Key properties:**
- Output latency: approximately `max(processing_time, trigger_interval)`
- Resource usage: predictable—the cluster processes one batch per interval
- Good for: continuous streams with regular data arrival

---

## Trigger 2: default (no trigger / processingTime="0 seconds")

```python
.trigger(processingTime="0 seconds")  # or no .trigger() call at all
```

This is `ProcessingTime` with a zero interval: as soon as one batch commits, the next starts immediately, with no pause. The streaming loop runs as fast as possible.

**When to use it:** maximum throughput scenarios where you want to minimize lag at the cost of continuous cluster resource consumption. If your source delivers data continuously and you want to process it as fast as Spark can go, this is the right choice.

**Caution:** at maximum speed, a streaming query with zero trigger interval never pauses. The cluster is continuously occupied. Small batches of data (e.g., a few records per second from Kafka) result in many tiny Spark jobs, each with scheduling overhead that dominates the actual processing work. In this case, a small positive interval (e.g., 1 second) produces better overall throughput by batching data between launches.

---

## Trigger 3: Once

```python
.trigger(once=True)  # or Trigger.Once()
```

The query processes all available data as a single batch, then stops automatically.

**How it works:**
- StreamExecution runs exactly one micro-batch, consuming all data available from the source at the time the batch starts.
- After the batch completes and progress is committed, the query terminates.
- The next run (when scheduled again) picks up from where the last run committed—exactly like a continuous streaming query resuming from its checkpoint.

**When to use it:** **incremental batch processing**. You have a Kafka topic, Delta table, or file directory as a source, and you want to process it periodically (e.g., once per hour via a job scheduler) rather than continuously. Each invocation processes the backlog since the last run, then exits. The cost is pay-per-run rather than continuous cluster occupancy.

> **`Trigger.Once` turns a streaming query into an incremental ETL job.** It's like a mail carrier who only delivers when called—they pick up all mail that has accumulated since their last visit, deliver it, and go home. The next call starts where they left off. Contrast with `ProcessingTime`, where the carrier lives in the mailbox and delivers continuously.

**Key properties:**
- Runs exactly one batch per invocation
- Exits automatically when done—no need to call `query.stop()`
- Checkpoint is updated after each run; the next run continues from there
- Good for: hourly/daily incremental loads, cost-sensitive environments, scheduled pipelines

**Limitation:** one single batch for all available data. If the source has accumulated a very large backlog, this one batch may be very large and slow. For large backlogs, `AvailableNow` is better.

---

## Trigger 4: AvailableNow

```python
.trigger(availableNow=True)  # or Trigger.AvailableNow()
```

Like `Once`, but may run multiple micro-batches until all currently available data is consumed, then stops.

**How it works:**
- StreamExecution queries the source for all available data at the time the query starts.
- It runs multiple micro-batches (each rate-limited by `maxOffsetsPerTrigger` or similar settings), processing the backlog in manageable chunks.
- After all backlog data has been processed and committed, the query terminates automatically.
- The next invocation starts from the last committed offset, processing the new backlog.

> **`AvailableNow` is like a mail carrier who processes a large accumulated pile of mail in multiple trips.** If 5,000 letters have piled up, they don't try to carry all 5,000 in one go (that would be `Trigger.Once`). They take 500 per trip, make multiple trips until the pile is gone, then go home. The pace is controlled (rate limiting); the result is the same (all letters delivered); the approach is more manageable.

**When to use it:** incremental batch processing with rate limiting. If you schedule a job every hour but in some hours the source produces 10 million records, `Trigger.Once` would try to process 10 million records in one giant batch. `AvailableNow` processes them in smaller, rate-limited micro-batches, keeping each batch's resource usage predictable.

**Key properties:**
- Processes all available data at query start time, in multiple controlled batches
- Exits when the backlog is fully processed
- Respects rate limits (`maxOffsetsPerTrigger`)
- Good for: large-backlog incremental loads, backfill jobs

---

## Trigger 5: Continuous (experimental)

```python
.trigger(continuous="1 second")  # or Trigger.Continuous("1 second")
```

The experimental Continuous processing engine breaks the micro-batch model entirely. Instead of a batch of tasks launched per interval, Continuous mode runs **long-lived tasks** that continuously read from sources and write to sinks, checkpointing progress every `checkpoint_interval` seconds.

**How it works:**
- Each executor runs a permanent task that reads records from its assigned source partition in a tight loop.
- Records are processed immediately, one at a time, rather than waiting for a batch boundary.
- Progress (offsets) is checkpointed to the driver at the specified interval (not per record).

**Latency:** millisecond-scale end-to-end latency, vs. seconds for micro-batch mode.

**Limitations (why it's still experimental):**
- Very limited operator support: only stateless operations (`map`, `filter`, `flatMap`, `select`, `where`). No aggregations, no joins, no stateful operations.
- Fault tolerance is weaker: a task failure requires restarting from the last checkpoint, which may replay up to `checkpoint_interval` seconds of data.
- Less mature in terms of monitoring, AQE support, and recovery tooling.

> **Continuous processing is like a live phone interpreter who translates each sentence as it's spoken, rather than taking notes during the whole meeting and translating everything at the end.** Ultra-low latency, but much harder to maintain state (you can't summarize across sentences), and if the interpreter loses the connection for a moment, you lose a few seconds of translation.

**When to use it:** only when millisecond latency is a hard requirement and your query is simple (stateless filtering and transformation). For most production workloads, micro-batch with a short `ProcessingTime` trigger (1–5 seconds) achieves sufficient latency with much better reliability.

---

## Choosing the right trigger

| Goal | Trigger |
|------|---------|
| Near-real-time continuous processing | `processingTime("5 seconds")` |
| Minimum possible latency (stateless) | `continuous("1 second")` (experimental) |
| Maximum throughput, don't care about per-batch latency | `processingTime("0 seconds")` |
| Hourly incremental load, moderate data volume | `once=True` |
| Hourly incremental load, large backlog possible | `availableNow=True` |
| Scheduled backfill job | `availableNow=True` |

---

## Bringing it together

Triggers control when each micro-batch starts in Structured Streaming's execution loop. **ProcessingTime** runs batches on a fixed schedule—start immediately, complete, wait until the next interval, repeat. **Zero interval / default** runs batches back-to-back with no pause—maximum throughput, continuous resource use. **Once** processes all available data in a single batch and exits—ideal for incremental scheduled pipelines. **AvailableNow** processes all available data in multiple rate-limited batches and exits—the safer version of Once for large backlogs. **Continuous** (experimental) abandons micro-batches entirely for millisecond latency but only supports stateless operations. The choice of trigger is a fundamental design decision that determines latency, throughput, cost, and operational complexity. Most production workloads should use `ProcessingTime` with a short interval for continuous queries, or `AvailableNow` for scheduled incremental loads.
