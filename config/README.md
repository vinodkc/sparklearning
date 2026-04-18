# Configuration

Stories about how Spark configuration is assembled, propagated, and applied.

## Stories

- [SparkConf to Code: How Configuration Reaches the Component That Needs It](sparkconf_configuration.md) — config sources and precedence, frozen vs dynamic properties, executor distribution, Environment tab

## Related stories

- [From spark-submit to Running Tasks: The Resource Negotiation Story](../cluster/spark_submit.md) — where command-line `--conf` flags fit in the submission flow
- [The Driver, the Executors, and How a Job Actually Runs](../execution/driver_executors_and_the_execution_model.md) — the components that consume configuration at runtime
- [Reading the Spark UI: What Every Tab Is Actually Telling You](../ui-metrics/reading_the_spark_ui.md) — the Environment tab shows the final merged configuration for a running job
- [AQE: How Spark Rewrites Plans After the Shuffle](../adaptive/aqe_rewriting_plans.md) — AQE is controlled by config flags that can be set dynamically at runtime
