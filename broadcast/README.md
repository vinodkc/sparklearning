# Broadcast Variables & Accumulators

Stories about how Spark shares read-only data and collects metrics across distributed tasks.

## Stories

- [Shared State in a Distributed Job](broadcast_variables_and_accumulators.md) — broadcast variables, tree-based distribution, accumulators, and their limitations

## Related stories

- [How Spark Chooses a Join](../joins/how_spark_chooses_a_join.md) — broadcast hash join uses broadcast variables to distribute the small side
- [Bytes on the Wire: How Spark Serializes Data for Tasks and Shuffles](../serialization/bytes_on_the_wire.md) — how broadcast data is serialized before being sent to executors
- [The Driver, the Executors, and How a Job Actually Runs](../execution/driver_executors_and_the_execution_model.md) — the driver/executor model that broadcast and accumulators sit on top of
