# Cluster & Deploy

Stories about how Spark runs on cluster managers — from submitting a job to running tasks on worker nodes.

## Stories

- [From spark-submit to Running Tasks: The Resource Negotiation Story](spark_submit.md) — spark-submit launch chain, driver startup, executor acquisition, client vs cluster mode
- [Spark on YARN: ApplicationMaster, Containers, and the Queue](spark_on_yarn.md) — YARN resource model, ApplicationMaster, container lifecycle, queue scheduling
- [Spark on Kubernetes: Pods, Namespaces, and Ephemeral Executors](spark_on_kubernetes.md) — driver/executor pods, container images, ephemeral storage, RBAC

## Related stories

- [The Driver, the Executors, and How a Job Actually Runs](../execution/driver_executors_and_the_execution_model.md) — the roles of the driver and executors once they are running
- [Elastic Executors: How Dynamic Allocation Grows and Shrinks the Cluster](../execution/dynamic_allocation.md) — how Spark requests and releases executors at runtime via the cluster manager
- [SparkConf to Code: How Configuration Reaches the Component That Needs It](../config/sparkconf_configuration.md) — how configuration set at submission time propagates to executors
- [The Event Log: A Complete Record of Everything That Happened in Your Job](../ui-metrics/event_log_and_history_server.md) — how to observe completed jobs after the cluster tears them down
- [The Journey of a Shuffle Record](../shuffle/journey_of_a_shuffle_record.md) — the External Shuffle Service that makes dynamic allocation safe
