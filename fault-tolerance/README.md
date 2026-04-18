# Fault Tolerance

Stories about how Spark recovers from failures — through lineage, recomputation, checkpointing, and speculation.

## Stories

- [How Spark Survives Failure](lineage_and_fault_tolerance.md) — lineage graph, narrow vs wide dependencies, task/stage retries, checkpointing, speculative execution

## Related stories

- [The Driver, the Executors, and How a Job Actually Runs](../execution/driver_executors_and_the_execution_model.md) — the execution model that fault tolerance is built on top of
- [From One Action to Many Tasks](../scheduler/from_action_to_tasks.md) — how the DAG Scheduler detects failures and retries stages
- [The Journey of a Shuffle Record](../shuffle/journey_of_a_shuffle_record.md) — fetch failures during shuffle are a common trigger for stage retries
- [Batch by Batch: Inside the Structured Streaming Micro-Batch Engine](../ss/micro_batch_engine.md) — how Structured Streaming extends fault tolerance to continuous queries via checkpointing
- [Keeping Score: How Spark Maintains State Across Micro-Batches](../ss/stateful_operations.md) — how stateful streaming operations survive failures through the state store
