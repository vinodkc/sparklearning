# Spark Meets Kafka: How Offsets, Partitions, and Backpressure Work Together

This is the story of how Structured Streaming reads from Apache Kafka—the most widely used streaming source in production Spark deployments. Kafka's design (immutable, partitioned, offset-tracked topics) aligns beautifully with Structured Streaming's micro-batch model. But making them work efficiently together—handling partition assignment, offset management, rate limiting, and fault tolerance—involves a detailed contract between Spark and the Kafka source connector. Understanding this contract explains why Kafka is a natural fit for exactly-once streaming, how Spark avoids reading too fast or too slow, and what happens when partitions are added to a Kafka topic.

---

## Kafka fundamentals for Spark users

A Kafka **topic** is a named, ordered, immutable log. Producers write records to topics; consumers read them. Topics are divided into **partitions**: each partition is an independent ordered log, and records within a partition have a sequential **offset** (a non-negative integer that monotonically increases). Records are identified globally by the triple `(topic, partition, offset)`.

Consumer groups track their progress by storing the last committed offset for each partition. For Structured Streaming, Spark itself tracks these offsets in the checkpoint directory (the offset log) rather than letting Kafka's consumer group mechanism handle it.

> **Think of a Kafka topic like a multi-lane highway.** Each partition is one lane; cars (records) in each lane are numbered sequentially (offsets). Cars never change lanes once on the highway; the only way to change a record's partition is at publish time. A consumer (Spark) reads one lane at a time and remembers how far along each lane they've driven. To resume after a break, they check their notes ("I left off at car 4,821 in lane 3") and start from there.

---

## The Kafka source connector

Structured Streaming reads Kafka via the `kafka` format:

```python
df = spark.readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "broker:9092") \
    .option("subscribe", "orders") \
    .option("startingOffsets", "latest") \
    .load()
```

The resulting DataFrame has fixed Kafka-metadata columns: `key`, `value` (both binary), `topic`, `partition`, `offset`, `timestamp`, `timestampType`. The actual message content is in `value` (a binary blob); you decode it with `from_json`, `CAST(value AS STRING)`, or a custom deserializer.

The Kafka source is **replayable**: given a partition and offset range, it can always re-deliver the exact same records. This is a fundamental property of Kafka (records are immutable and retained for a configurable period), and it is what enables exactly-once guarantees in Structured Streaming.

---

## How offsets are managed per micro-batch

At the start of each micro-batch, the Kafka source connector does the following:

1. **Fetch the latest available offsets** from the Kafka broker: for each subscribed partition, query the broker for the latest offset (the offset of the next record to be written, i.e., the end of the log).

2. **Determine the range to process**: the starting offset is the last committed offset for each partition (from the checkpoint offset log). The ending offset is either the latest available or a rate-limited end (see below). The range `[start_offset, end_offset)` defines exactly which records this batch will process.

3. **Write the end offsets to the offset log** (before reading data—write-ahead).

4. **Create a RDD/DataFrame** that reads records in the committed range: for each Kafka partition, one or more Spark tasks read their assigned range of records from the broker.

5. **Commit the end offsets to the commit log** after the batch succeeds.

> **The offset management is like a postal worker with a route notebook.** Before starting the round, the worker writes down "today I'll deliver mail up to house 4,821 on each street" (end offsets to offset log). During the round, they deliver all mail from where they last stopped (start offsets) to house 4,821. Afterwards, they record "done up to house 4,821" in the delivery log (commit log). If they get sick mid-route, a substitute picks up the notebook and knows exactly where to resume.

---

## Rate limiting and backpressure

Kafka topics can grow very fast. Without rate limiting, a single micro-batch might try to read millions of records, taking very long and potentially causing executor OOMs or blocking the streaming query for minutes. Structured Streaming with Kafka supports two key rate-limiting options:

**`maxOffsetsPerTrigger`**: the maximum total number of records to read across all partitions in one micro-batch. If the Kafka lag (the gap between committed offsets and latest offsets) is large, Spark reads up to this limit and leaves the rest for future batches.

**`maxRatePerPartition`** (in older Spark Streaming, not Structured Streaming): per-partition rate limit.

With `maxOffsetsPerTrigger = 100000`, Spark reads at most 100,000 records per batch regardless of how much data is available. This bounds batch latency, memory usage, and processing time at the cost of falling further behind if the source is very fast.

> **`maxOffsetsPerTrigger` is like a restaurant that only accepts 50 new orders per hour**, even if 200 customers are waiting. The kitchen (executor memory and CPU) can handle 50 orders well. Accepting 200 at once would overwhelm the kitchen and slow every order down. Customers (records) wait in the queue (Kafka topic) and are served in subsequent hours (batches).

---

## Partition-to-task assignment

The Kafka source creates one Spark task per Kafka partition (or one task per partition range if multiple partitions are grouped). Each task reads records from one Kafka partition using the Kafka consumer client. Tasks run in parallel on executor cores.

The parallelism of Kafka ingestion is therefore bounded by the number of Kafka partitions: a topic with 10 partitions can be read by at most 10 Spark tasks in parallel. If you need more throughput, increase the number of Kafka partitions (a Kafka administrative operation). If you have more Spark cores than Kafka partitions, the extra cores sit idle during the Kafka read stage.

> **Kafka partitions are the minimum unit of Spark parallelism for reads.** If a highway has 10 lanes, at most 10 inspection teams can work simultaneously. Adding more teams beyond 10 doesn't speed up the inspection—extra teams have no lane to work on. Adding more Kafka partitions is adding more lanes.

---

## Starting offsets and replaying

When a streaming query starts for the first time (no checkpoint), the `startingOffsets` option controls where in the topic to begin:

- **`"latest"`**: start from the current end of the topic—only process records written after the query starts. Records written before the query started are ignored.
- **`"earliest"`**: start from the beginning of the topic—process all retained records. Good for backfill scenarios.
- **JSON string**: specify exact per-partition offsets: `{"orders": {"0": 1000, "1": 2000}}` starts partition 0 from offset 1000 and partition 1 from offset 2000.

Once a checkpoint exists, `startingOffsets` is ignored—the checkpoint's committed offsets take precedence. This ensures exactly-once: the query always resumes from where it left off.

---

## Dynamic partition discovery

Kafka topics can have partitions added at any time (a Kafka administrative operation to increase throughput). By default, the Kafka source discovers the current list of partitions when the query starts and uses only those. New partitions added after the query starts are not seen until the query is restarted.

With `option("kafka.partition.assignment.strategy", ...)` and related settings, you can enable automatic detection of new partitions. When new partitions are detected, the source begins reading from their earliest available offset (or from `startingOffsets` if configured). This allows Kafka topic scaling without restarting the Spark application.

---

## Fault tolerance and exactly-once

Exactly-once with Kafka requires:

1. **Replayable source**: Kafka is replayable—given any (partition, offset) range, it can replay the same records. ✓

2. **Durable offset tracking**: Spark's checkpoint offset log persists the last processed offsets to HDFS/S3. On recovery, the query reads from the checkpoint, not from Kafka's consumer group. This avoids the problem of Kafka's consumer group offsets being committed when the batch didn't complete. ✓

3. **Idempotent or transactional sink**: covered in the exactly-once delivery story—the output side must not duplicate records on re-runs. ✓

Notably, Spark does NOT commit offsets to Kafka's consumer group by default (there is no `commitSync` or `commitAsync`). The checkpoint directory is the sole source of truth for progress tracking. This gives Spark complete control and avoids the complexity of Kafka's offset management interacting with Spark's recovery logic.

---

## Kafka as a sink: writing back to Kafka

Structured Streaming can also write output to Kafka topics:

```python
query = result \
    .writeStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "broker:9092") \
    .option("topic", "enriched-orders") \
    .option("checkpointLocation", "/checkpoint/path") \
    .start()
```

The Kafka sink writes records to a Kafka topic. For exactly-once writes, use the `kafka.transaction.timeout.ms` and enable Kafka's transactional producer API. Each batch uses a deterministic transaction ID (derived from the Spark query ID and batch ID), so retried batches don't produce duplicates.

---

## Bringing it together

The Kafka source is Structured Streaming's most capable production source, built on Kafka's fundamental guarantees (immutable, partitioned, offset-tracked, replayable). Each micro-batch reads a specific `[start, end)` offset range per partition, determined by the checkpoint's committed offsets and bounded by `maxOffsetsPerTrigger`. Spark tasks map one-to-one with Kafka partitions (parallelism is bounded by partition count). Offsets are tracked exclusively in Spark's checkpoint directory—not in Kafka's consumer group—giving Spark full control over exactly-once recovery. Rate limiting through `maxOffsetsPerTrigger` prevents batch overload when the source is faster than the sink. Dynamic partition discovery allows Kafka topic scaling without query restarts. So the story of Kafka + Structured Streaming is: **subscribe to partitions → determine the safe range to read this batch → read in parallel (one task per partition) → process and write output → commit offsets to checkpoint.** Kafka's replayability and Spark's checkpoint-based offset tracking together form the foundation of exactly-once streaming at scale.
