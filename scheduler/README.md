# Scheduler

Stories about how Spark's DAG Scheduler and Task Scheduler turn a job into running tasks.

## Stories

- [From One Action to Many Tasks](from_action_to_tasks.md) — DAG Scheduler, stage boundaries, TaskScheduler, task assignment
- [Locality and Delay Scheduling](locality_and_delay_scheduling.md) — data locality levels, delay scheduling, when Spark waits for a better executor
- [Scheduling Pools and Fair Sharing](scheduling_pools_and_fair_sharing.md) — FIFO vs fair scheduler, pools, minimum share, weight-based ordering

## Related stories

- [The Driver, the Executors, and How a Job Actually Runs](../execution/driver_executors_and_the_execution_model.md) — the driver hosts the DAG and Task Schedulers described in these stories
- [How Spark Survives Failure](../fault-tolerance/lineage_and_fault_tolerance.md) — what the DAG Scheduler does when a stage or task fails
- [Partitions: The Grain of Parallelism](../partitioning/partitions_coalesce_repartition_pruning.md) — each partition becomes one task; partition count determines task count
- [Elastic Executors: How Dynamic Allocation Grows and Shrinks the Cluster](../execution/dynamic_allocation.md) — the scheduler's backlog of pending tasks triggers dynamic allocation scale-up
