# Scheduler

Stories about how Spark schedules work: the DAG Scheduler and the Task Scheduler, and how stages and tasks get submitted and run on executors.

**Planned stories:**
- DAG Scheduler: from job to stages (why shuffle is a boundary)
- Task Scheduler: from stages to tasks, and how tasks are offered to executors
- Locality: preferred locations, delay scheduling, and when Spark waits for a “good” executor
- Scheduling pool and fair sharing (when multiple jobs run)

Stories will be added here gradually.
