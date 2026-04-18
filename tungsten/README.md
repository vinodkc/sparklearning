# Tungsten & JVM Internals

Stories about Spark's low-level memory and execution optimizations — and the JVM garbage collector.

## Stories

- [Tungsten: How Spark Stopped Trusting the JVM](tungsten_and_binary_rows.md) — UnsafeRow binary format, off-heap memory, cache-friendly layout, binary comparisons
- [Garbage Collection in Spark: Why the JVM Pauses and How to Make It Stop](gc_tuning.md) — generational GC, G1GC tuning, Spark-specific GC patterns, diagnosing GC pauses

## Related stories

- [The Two Lives of Spark's Memory](../memory/unified_memory_and_block_manager.md) — the unified memory manager that controls how much heap Tungsten's execution pool gets
- [From SQL to a Running Plan: The Catalyst Story](../catalyst/from_sql_to_physical_plan.md) — whole-stage codegen generates the tight loops that Tungsten's binary format is designed for
- [Bytes on the Wire: How Spark Serializes Data for Tasks and Shuffles](../serialization/bytes_on_the_wire.md) — Tungsten's binary format is also used for shuffle serialization
- [The Encoder Contract: How Spark Converts Between JVM Objects and Binary Rows](../catalyst/encoders_and_datasets.md) — encoders translate between JVM objects and Tungsten's UnsafeRow binary format
- [Cache Wisely: When Persisting Data Helps and When It Hurts](../memory/caching_strategy.md) — serialized caching stores data in Tungsten-compatible byte buffers, reducing GC pressure
