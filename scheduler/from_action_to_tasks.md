# From One Action to Many Tasks

One call—*count*, *save*, or the run of a SQL query—triggers a cascade: a **job**, then a graph of **stages**, then hundreds or thousands of **tasks** on executors. Two actors drive this: the **DAG Scheduler**, which decides *what* to run and in what order, and the **Task Scheduler**, which decides *where* each task runs. This story walks that path so you can see why your job has the stages it has, and why Spark sometimes pauses before launching tasks.

---

## The action: something must happen

Transformations like *map* or *filter* are lazy. They only describe a lineage of RDDs. Nothing runs until an **action** runs: *count*, *collect*, *saveAsTextFile*, or a DataFrame/SQL query that ends in a write or collect. That action, on the driver, creates a **job**: "run this function over these partitions and give me the result (or write it out)." The job is the unit of work that gets turned into stages and tasks.

> **Think of it like writing a recipe vs. cooking a meal.** Each transformation (`filter`, `map`, `groupBy`) is a step in the recipe—you are writing down the instructions, not turning on the stove. The action (`count()`, `save()`) is the moment you decide to actually cook. Only then does Spark read the recipe from top to bottom and start working.

---

## The DAG: from lineage to stages

The RDD you're acting on was built from other RDDs. Some steps are **narrow**: each output partition depends only on one input partition (e.g. *map*). Others are **wide**: each output partition can depend on many input partitions (e.g. *groupByKey*, *reduceByKey*). Those wide steps are **shuffles**—data must be repartitioned. So the lineage is a DAG of RDDs, and **shuffle boundaries** are where the driver must split work: you can't run the "reduce" side until the "map" side has produced its output.

The **DAG Scheduler** takes the job and walks backward through that lineage. At every shuffle it draws a boundary: everything from that shuffle back to the previous shuffle (or the source) becomes one **stage**. You get map stages that produce shuffle output and a final result stage that runs the action. So the RDD DAG becomes a DAG of stages; parents must finish before children run.

> **Think of it like a car assembly line.** Stage 1 stamps the body panels, Stage 2 welds them together, Stage 3 paints them. The welding station can't start until the stamping station has produced its parts—that handoff is the shuffle boundary. Each station's work is one stage; the handoff between stations is where data must be physically moved.

---

## Submitting stages: parents first

Stages aren't all submitted at once. The scheduler starts from the **final stage** (the one that runs the action). To submit a stage it asks: *are my parents done?* If not, it submits the parents first and puts the current stage in a waiting list. When a parent finishes, it checks the list and submits any stage whose parents are now all done. So stages run in **dependency order**: map stages first, then the reduce stage that consumes their output.

Submitting a stage means creating one **task** per partition (or the subset needed for actions like *first*), attaching **preferred locations** (where that partition's data lives or is best read from), and handing that set of tasks to the **Task Scheduler**.

---

## The Task Scheduler: matching tasks to executors

The Task Scheduler doesn't push tasks out on its own. The cluster sends **resource offers**: "Executor X has N free cores." For each offer, the scheduler asks: *given this executor, which of my pending tasks do I want to run here?* It chooses with **locality** in mind: prefer the executor that already has the data (same process, then same node, then same rack, then anywhere). So the executor that held a partition in the previous stage often runs the task that consumes it next—data stays put when it can.

If no task fits that executor well enough, the scheduler can **decline** the offer and wait. That's **delay scheduling**: Spark waits a bit (configurable) for a "better" executor before relaxing and taking any. That's why you sometimes see a short pause before tasks launch.

> **Think of it like a taxi dispatcher.** Drivers radio in "available"—those are resource offers. The dispatcher assigns each driver the closest waiting passenger (data-local task) rather than sending them across town. If no nearby passenger exists, the dispatcher waits a moment for one to appear before finally assigning the far-away fare. That brief wait is delay scheduling.

The chosen tasks are sent to the executors. Executors run them, report back success or failure, and the DAG Scheduler updates stage state. When every task in a stage has succeeded, the stage is done. If it was a map stage, its shuffle output is now available and any waiting child stage can be submitted. If it was the result stage, the job is finished.

---

## Failure and speculation

### Who retries what

**Task retries** are handled by the **Task Scheduler**. When a task fails, the executor reports it; the scheduler can resubmit that same task (same partition, new attempt) when it gets new offers. It retries up to a limit per task (default: 4 attempts). If a task hits that limit, the scheduler gives up on the whole stage and the job fails. There is no "stage retry" for that case.

**Stage retries** are handled by the **DAG Scheduler**. When a task fails because **shuffle output was lost**—for example a reducer couldn't fetch a map output, or the executor that had it died—the failure isn't counted toward the per-task limit. Instead, the DAG Scheduler figures out which **stages** must be re-run (the map stage that produced the lost data and any stages that depend on it), and after a short delay it resubmits those stages. So the whole stage runs again with fresh tasks. That's stage retry. The DAG Scheduler also enforces how many times a stage can be retried.

**Speculation** lives in the Task Scheduler. A background process periodically checks whether any running task is much slower than others in the same stage. If so, it can launch a **duplicate** of that task on another executor. Whichever copy finishes first wins; the other is cancelled. So stragglers don't hold up the stage.

> **Speculation is like assigning a backup runner at a relay race.** If one runner is visibly lagging far behind the others on the same leg, you send a fresh runner from the bench to run the same leg in parallel. Whoever reaches the handoff point first passes the baton; the other stops. The race doesn't slow down for a single tired runner.

---

## Bringing it together

One action becomes one job. The DAG Scheduler turns the job's RDD lineage into a DAG of stages by cutting at every shuffle. It submits stages in dependency order—parents first, then children when parents complete. Each stage becomes a set of tasks (one per partition). The Task Scheduler takes resource offers, picks tasks (preferring locality, using delay scheduling when it helps), and sends them to executors. When all tasks of a stage complete, the next stage can run. So the journey is: **action → job → DAG of stages → sets of tasks → tasks matched to executors → execution and completion.** The shuffle is the boundary between stages; the two schedulers turn that boundary into the execution you see.
