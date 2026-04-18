# Keeping Score: How Spark Maintains State Across Micro-Batches

This is the story of stateful operations in Structured Streaming—the mechanisms that let a streaming job remember things across time. A stateless streaming query is easy: take each micro-batch's data, apply a transformation, emit output. But most real streaming problems are inherently stateful: count events per user session, detect patterns that span multiple batches, aggregate over a sliding time window, or deduplicate records that may arrive twice. All of these require memory that persists from one micro-batch to the next. This story explains the main patterns of stateful streaming, how state is stored and managed, and the constraints that govern how long state can be kept.

---

## Why state is necessary

In batch processing, the entire dataset is available at once. A `groupBy` aggregation sees all rows, computes the result, and returns. In streaming, rows arrive batch by batch over time. A `groupBy customer_id` that counts orders must accumulate counts across many batches—the count for customer 42 seen in batch 1 must persist and be updated by batches 2, 3, and 4.

Without state, each micro-batch would compute results only from the rows in that batch—ignoring all history. The result would be a partial, incorrect answer.

> **State in streaming is like a running scoreboard at a sporting event.** Each play (micro-batch) adds new points for the relevant team (group key). The scoreboard doesn't reset after each play—it accumulates. If you erased the scoreboard after every play, the final score would only reflect the last play. State persistence is what makes the scoreboard show the correct total.

---

## Pattern 1: streaming aggregations

The simplest stateful pattern is a **streaming aggregation**: `groupBy` on one or more columns with an aggregate function like `count`, `sum`, or `avg`.

```python
from pyspark.sql.functions import count
event_counts = events.groupBy("user_id").count()
```

For each micro-batch, Spark reads the new rows, updates the aggregation state for each key that appeared, and emits the updated aggregates. The state is a map from group key (e.g., user_id) to aggregation buffer (e.g., the running count).

**Without a watermark**, the state map grows indefinitely—every unique `user_id` ever seen is kept forever. For unbounded streams, this eventually exhausts memory. The solution is to either:
- Add a watermark and use event-time windowed aggregation (state is cleaned up after windows close)
- Accept that state will grow (only viable for bounded key spaces, like a small fixed set of user IDs)
- Use the output mode carefully: `Update` mode emits results for each group that received new rows this batch; `Complete` mode emits all groups every batch (only for aggregations without watermarks)

---

## Pattern 2: event-time windowed aggregations

For event-time analytics, you group by both a key and a **time window**:

```python
from pyspark.sql.functions import window
windowed_counts = events \
    .withWatermark("event_time", "10 minutes") \
    .groupBy(window("event_time", "1 hour"), "user_id") \
    .count()
```

Each row is assigned to one or more time windows based on its event timestamp. The state map now keys on `(window, user_id)` pairs. As the watermark advances, windows that are past the watermark are closed, their results are emitted (in Append mode), and their state is deleted.

> **Windowed aggregation is like keeping separate running totals for each hour of the day.** The "2pm–3pm" counter accumulates all events that happened between 2pm and 3pm, regardless of when Spark processed them. When the watermark moves past 3pm (plus the delay buffer), you know the 2pm–3pm window is complete, emit its total, and erase that counter from the scoreboard. New hours get new counters; old hours are erased once final.

The watermark is the mechanism that bounds state growth for windowed aggregations. Without it, every window ever created persists indefinitely.

---

## Pattern 3: stream deduplication

Deduplication removes duplicate records that may arrive more than once due to retries, redelivery, or at-least-once sources:

```python
deduplicated = events.withWatermark("event_time", "1 hour") \
    .dropDuplicates(["event_id", "event_time"])
```

The state for deduplication is a set of seen event IDs (within the watermark window). For each new record, Spark checks whether its `event_id` is already in the state. If yes, drop it; if no, add it to state and emit the record.

The watermark bounds the deduplication state: once an event's timestamp is older than the watermark, Spark can safely stop remembering its ID—any duplicate arriving after the watermark expires is simply accepted (the deduplication guarantee only applies within the watermark window).

> **Deduplication with a watermark is like a ticket booth that keeps a stub of each ticket sold for the last 24 hours.** If someone tries to use a duplicate ticket within 24 hours, they're turned away (the stub is in the box). After 24 hours, stubs are discarded—the assumption is that no valid ticket from 2 days ago is still floating around. For events older than the watermark, you accept re-arrivals.

---

## Pattern 4: mapGroupsWithState — arbitrary stateful logic

For more complex stateful patterns—session detection, pattern matching, finite-state machines—Spark provides `mapGroupsWithState` (and the more flexible `flatMapGroupsWithState`). These let you write arbitrary stateful logic as a Scala or Java function:

```scala
def updateState(
  key: String,
  rows: Iterator[Event],
  state: GroupState[MySessionState]
): SessionOutput = {

  val current = if (state.exists) state.get else MySessionState.empty
  val updated = rows.foldLeft(current)(processEvent)

  if (updated.isComplete) {
    state.remove()
    SessionOutput(key, updated)
  } else {
    state.update(updated)
    SessionOutput(key, updated)
  }
}

events.groupByKey(_.userId).mapGroupsWithState(updateState)
```

For each micro-batch, for each key that received new rows, Spark calls your function with:
- The key (e.g., user ID)
- An iterator of all new rows for that key in this batch
- A `GroupState` object: a handle to the persistent state for this key across batches

Your function reads the current state, updates it based on new rows, and either updates the state for next time or removes it (if the session is complete).

**`flatMapGroupsWithState`** is the more flexible variant: instead of returning one output per group, it returns an iterator, allowing zero, one, or many output rows per group per batch. This is useful for state machines that emit different numbers of events at different transitions.

> **`mapGroupsWithState` is like a customer service agent who has access to a customer's entire file.** For each new call (micro-batch), the agent retrieves the customer's folder (state), reviews their history, processes the new issue (new rows), updates the notes (new state), and files everything back. The agent doesn't start from scratch on each call—the accumulated history is always available.

---

## Pattern 5: stream-stream joins

Two streaming DataFrames can be joined together—both sides are unbounded streams that arrive over time. A join between `orders` and `payments` by `order_id` requires buffering rows from both sides: an order that arrives without a matching payment must be held until the payment arrives (or until the watermark declares it won't).

```python
matched = orders.join(payments, "order_id", "inner")
```

With watermarks declared on both sides, Spark bounds the buffer: rows from `orders` are held for at most `watermark_delay(orders)` time after their event timestamp, waiting for a matching `payments` row. Once the watermark passes, unmatched orders are either dropped (inner join) or emitted as unmatched (outer join).

The state for stream-stream joins is two-sided: buffers for both the left and right side. The watermarks determine when buffered rows are too old to find a match and can be removed.

---

## The state store: where state lives

All of these patterns store their state in the **state store**—a per-partition, per-operator key-value store that persists across micro-batches. Spark's default state store keeps everything in JVM memory, checkpointing snapshots to HDFS/S3. For large state (millions of keys), the **RocksDB state store** stores state on local disk with RocksDB, reducing JVM heap pressure and GC pauses (covered in the RocksDB story).

The state store is partition-local: each task reads and writes only its own partition's state. For a streaming job with 100 shuffle partitions, there are 100 independent state store instances. Scaling the number of partitions changes state locality—increasing partitions means existing state must be migrated, which requires stopping the job or using state migration tools.

---

## State expiry and timeouts

For custom state in `mapGroupsWithState`, you can set **timeouts**: if a key has not received any new rows for a period, Spark automatically calls your function with an empty row iterator and a `timedOut = true` flag. Your function can then remove stale state.

Two timeout types:
- **ProcessingTimeTimeout**: fires after a wall-clock duration of inactivity. Simpler but not tied to event time.
- **EventTimeTimeout**: fires when the watermark advances past a specified event timestamp. Tied to event time; more accurate for event-time semantics.

> **Timeouts are like automatic archiving for inactive files.** A folder that hasn't been touched in 30 days (timeout period) is automatically moved to cold storage. Your function is the archivist: when called with `timedOut = true`, you decide whether to truly delete the state, emit a summary, or do something else. Without timeouts, state for inactive keys accumulates forever.

---

## Bringing it together

Stateful streaming is what makes Spark Structured Streaming useful for real analytical problems. The main patterns are: **streaming aggregations** (accumulate group results across batches), **windowed aggregations** (accumulate within event-time windows, closed by the watermark), **deduplication** (track seen IDs within the watermark window), **`mapGroupsWithState`** (arbitrary state machines with per-key state objects), and **stream-stream joins** (buffer both sides and match when possible). All state lives in the **state store**, backed by memory (default) or RocksDB (for large state), checkpointed to durable storage for fault tolerance. **Watermarks** and **timeouts** are the mechanisms that bound state growth—without them, state accumulates indefinitely. So the story of stateful streaming is: **persist state per key across batches → update on new data → expire via watermarks or timeouts → checkpoint for recovery.** The state store is the memory that turns a sequence of isolated micro-batches into a coherent, continuously-updated view of the world.
