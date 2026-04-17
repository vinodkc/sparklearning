# Watermarks: How Structured Streaming Decides When to Stop Waiting

This is the story of how Structured Streaming handles the messiness of time. In an ideal world, events arrive in the order they happened. In the real world, a mobile device buffers events for two hours before reconnecting, a network partition delays a batch of log lines by 45 minutes, and retry logic replays events that were already counted. Structured Streaming uses **watermarks** to reason about event time under these conditions—deciding when enough time has passed that it's safe to produce a result for a time window, and when a late-arriving record can still be incorporated versus when it must be discarded. Understanding watermarks is understanding how streaming systems balance correctness against latency.

---

## Event time vs. processing time

Every record in a stream has two timestamps: the time it was **produced** (embedded in the record's data—say, a sensor reading with a timestamp field) and the time Spark **processed** it (the wall-clock time when Spark ingested the record). These two times diverge whenever events are delayed in transit.

Processing time is easy: it's just the system clock when the batch ran. But results based on processing time don't reflect when things actually happened. A session computed by processing time is a session of "when Spark saw the events," not "when the user actually did things." For most analytics—"how many orders were placed per hour?"—you want event-time windows.

**Event-time windowing** groups records by a timestamp column in the data itself. A record with timestamp `14:32:17` falls into the `14:00–15:00` window regardless of when Spark processed it. This gives accurate results—but creates a problem: when is the `14:00–15:00` window done? How long do you wait before computing its result? If you wait too long, your output is stale; if you wait too short, you miss late-arriving records.

---

## The late data problem

"Late data" means records whose event time is in the past relative to the current processing time. A record with event time `14:32` that arrives when Spark is processing events from `16:45` is 2 hours and 13 minutes late. Whether you should incorporate it into the `14:00–15:00` window or discard it depends on how long you're willing to wait.

Without a mechanism to bound this, a streaming aggregation would need to keep every window open forever: you can never know whether a record 10 years late might arrive. That's not practical. State must be bounded; memory is finite. The watermark is the mechanism that bounds how long a window stays open.

---

## What a watermark is

A **watermark** is a declaration: "I believe that all events with event time earlier than T have arrived." Any event that arrives with an event time before T is **late beyond the watermark** and may be dropped (or handled specially, depending on the output mode).

Spark computes the watermark dynamically from the data itself, using this formula:

```
watermark = max(event_time seen so far) − watermark_delay
```

You declare the watermark by calling `.withWatermark("timestamp_col", "2 hours")` on your streaming DataFrame. The `"2 hours"` is the **watermark delay**—your estimate of the maximum lateness you expect. Spark continuously tracks the maximum event time it has seen across all records in all batches. The current watermark is that maximum minus your declared delay.

If the latest event time Spark has seen is `16:45`, and your watermark delay is 2 hours, the current watermark is `14:45`. Windows that end before `14:45` are considered complete: no more records can arrive that legitimately belong to them. Those windows can be finalized and their state cleaned up.

---

## How the watermark advances

The watermark is updated at the start of each micro-batch, based on the maximum event time observed in all previous batches. It is **monotonically non-decreasing**: it only moves forward, never back. A single very late record (with an old event time) does not move the watermark backward—it contributes nothing to the max and is simply late.

The watermark advances when records with newer event times arrive. If your stream goes quiet for 10 minutes with no new events, the max event time doesn't change, and the watermark doesn't advance. This can be important for monitoring: if your watermark isn't advancing, either the stream is idle or all incoming events have old timestamps.

In practice, the watermark lags behind real time by approximately the declared delay. If events typically arrive within 30 minutes of their event time and you declare a 2-hour delay, your watermark will be roughly 2 hours behind the leading edge of your data—2 hours of state is kept open at any given time.

---

## Windows, state, and when state is cleaned up

When you write a windowed aggregation—say, counting events per 1-hour window with a 15-minute slide—Structured Streaming maintains an **aggregation state** for each active window. For each micro-batch, incoming records are routed to the windows they fall into, and the aggregation state is updated. Every active window is held in the **state store** (in-memory, backed by RocksDB or HDFS).

When the watermark advances past the end of a window, that window is **closed**. Spark knows no more records can arrive that belong to it (modulo any records late beyond the watermark delay, which will be dropped). Spark emits the final result for that window and **removes its state** from the state store. This is the mechanism that bounds memory: state is cleared for windows that are definitively in the past.

Without a watermark, no state is ever cleaned up—the state store grows without bound and the job eventually fails with an OOM.

---

## Late records and what happens to them

A record arrives late beyond the watermark when its event time is earlier than the current watermark. What happens to it depends on the **output mode**:

In **Append mode** (the default for windowed aggregations with watermarks), a window's result is emitted once—only when the window is closed by the watermark. Records that arrive before the window closes are incorporated normally. Records that arrive after the window closes are **silently dropped**: they don't update any state (the state for that window was already cleaned up) and they don't produce any output. Append mode guarantees that once a result is emitted, it is final and won't be revised.

In **Update mode**, state is updated and output is emitted on every batch, for every window that received new records. Late records within the watermark window do update state and produce revised output. Records beyond the watermark are still dropped (because the state is gone). Update mode is useful for dashboards or sinks that can handle updates, but the output is not final until the watermark passes the window end.

**Complete mode** emits the entire result table on every batch and doesn't clean up state. It doesn't use watermarks for state cleanup and is only appropriate for queries without windowing (full aggregations that must emit everything every batch).

---

## Watermarks and joins

Watermarks also apply to **stream-stream joins**—joins between two streaming DataFrames. For a stream-stream join on event time, Spark must buffer records from both sides, waiting for records from the other side that might arrive late. Without a watermark on both sides, Spark would buffer every record forever, waiting for a potential match.

With watermarks declared on both sides, Spark can bound the buffer: once the watermark on side A advances past a certain point, records from side A that haven't found a match from side B are dropped (or emitted as unmatched, for outer joins). The state cleanup logic is symmetric: Spark keeps records from both sides only within the watermark window of the other side.

Stream-static joins (joining a stream with a fixed DataFrame or table) don't need watermarks—the static side is always fully available.

---

## Choosing the watermark delay

The watermark delay is a business decision disguised as a technical parameter. It answers: "What is the maximum time after an event occurs that I'm willing to wait for a record reporting that event to arrive?" Set it too short and legitimate late records are dropped, producing under-counted results. Set it too long and you hold more state in memory, your outputs are delayed (in Append mode), and your watermark lags further behind real time.

Common approaches: instrument your pipeline to measure actual event-time lag (the difference between event time and ingestion time, by percentile), then set the watermark delay to the 99th or 99.9th percentile of that distribution. Some late records will still be dropped (beyond the 99th percentile), but the vast majority are captured while state is bounded.

---

## Bringing it together

Watermarks let Structured Streaming work with event time while keeping memory bounded. The watermark is a moving threshold—always the maximum event time seen minus a declared delay—that advances monotonically through the stream. Windows that end before the watermark are closed: their results are finalized and emitted (in Append mode), and their state is cleaned up from the state store. Records that arrive after the watermark has passed their window are late beyond the threshold and are dropped. Without a watermark, state grows without bound. With a too-short watermark, late records are dropped prematurely. With a well-chosen watermark, Structured Streaming delivers correct event-time aggregations with bounded state and predictable latency: **event-time window → state accumulates as records arrive → watermark advances → window closes → result emitted → state freed → late records dropped.**
