# UI & Metrics

Stories about observing Spark jobs — the live Spark UI and the event log for completed applications.

## Stories

- [Reading the Spark UI: What Every Tab Is Actually Telling You](reading_the_spark_ui.md) — Jobs, Stages, SQL, Executors, Storage, and Environment tabs; task metrics and what they reveal
- [The Event Log: A Complete Record of Everything That Happened in Your Job](event_log_and_history_server.md) — event log structure, enabling logging, History Server, task metrics, programmatic parsing

## Related stories

- [EXPLAIN Yourself: How to Read a Spark Physical Plan](../catalyst/explain_output.md) — the SQL tab in the UI shows the physical plan; this story teaches you how to read it
- [Taming the Shuffle: Partition Count, Spill, and the Right Shuffle for Your Job](../shuffle/shuffle_tuning.md) — the Stages tab's shuffle metrics are the primary diagnostic for shuffle problems
- [Out of Memory: A Field Guide to Spark OOM Errors](../memory/oom_diagnosis.md) — the Executors tab and task metrics help locate which executor or task ran out of memory
- [When One Partition Holds Up Everyone: The Data Skew Story](../partitioning/data_skew_story.md) — task duration distribution in the Stages tab is how you spot data skew
- [SparkConf to Code: How Configuration Reaches the Component That Needs It](../config/sparkconf_configuration.md) — the Environment tab shows the actual merged configuration a job is running with
