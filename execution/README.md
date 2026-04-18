# Execution & Scheduling

Stories about how Spark turns an action into a DAG of stages and tasks, and how the driver and executors coordinate.

## Stories

- [The Driver, the Executors, and How a Job Actually Runs](driver_executors_and_the_execution_model.md) — driver role, executor model, job/stage/task hierarchy, data locality, heartbeats
- [Elastic Executors: How Dynamic Allocation Grows and Shrinks the Cluster](dynamic_allocation.md) — scale-up/scale-down timers, External Shuffle Service, executor decommissioning

## Related stories

- [From One Action to Many Tasks](../scheduler/from_action_to_tasks.md) — the DAG Scheduler and Task Scheduler that turn jobs into running tasks
- [The Journey of a Shuffle Record](../shuffle/journey_of_a_shuffle_record.md) — what happens between stages when tasks exchange data
- [The Two Lives of Spark's Memory](../memory/unified_memory_and_block_manager.md) — how executor memory is managed once tasks are running
- [How Spark Survives Failure](../fault-tolerance/lineage_and_fault_tolerance.md) — what happens when an executor or task fails mid-job
- [From spark-submit to Running Tasks: The Resource Negotiation Story](../cluster/spark_submit.md) — how executors are provisioned before they can run any tasks
