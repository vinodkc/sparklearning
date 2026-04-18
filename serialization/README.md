# Serialization

Stories about how Spark converts objects to bytes — for tasks, shuffles, broadcasts, and cross-language transfer.

## Stories

- [Bytes on the Wire: How Spark Serializes Data for Tasks and Shuffles](bytes_on_the_wire.md) — Java serialization, Kryo, Tungsten binary format, task closure serialization, broadcast, Python pickle vs Arrow

## Related stories

- [The Journey of a Shuffle Record](../shuffle/journey_of_a_shuffle_record.md) — shuffle data is serialized before being written to disk and deserialized on the read side
- [Tungsten: How Spark Stopped Trusting the JVM](../tungsten/tungsten_and_binary_rows.md) — Tungsten's binary row format is Spark's answer to the overhead of Java object serialization
- [Two Runtimes, One Job: How PySpark Bridges Python and the JVM](../python/pyspark_bridge.md) — Python adds an extra serialization layer (pickle or Arrow) on top of the JVM path
- [Shared State in a Distributed Job](../broadcast/broadcast_variables_and_accumulators.md) — broadcast data is serialized once on the driver and deserialized on every executor
