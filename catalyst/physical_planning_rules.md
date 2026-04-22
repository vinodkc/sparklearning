# From Logic to Execution: How Spark Picks Physical Operators

This is the story of **physical planning**—the phase of Catalyst that converts an optimized logical plan into concrete physical operators that executors can actually run. The logical plan describes *what* to compute: "filter these rows, join these tables, aggregate by this key." The physical plan describes *how* to compute it: which join algorithm to use, where to insert shuffles, how to sort data, and which groups of operators to fuse into a single compiled loop. Every decision in this phase has direct performance consequences—the wrong join algorithm can cause OOM errors; a missing shuffle corrupts results; failing to fuse operators means data moves through memory unnecessarily.

Understanding physical planning explains why `EXPLAIN` shows the operators it does, why some plans have many `Exchange` nodes and others have none, and why Spark sometimes makes different physical choices for logically identical queries.

---

## The two-phase physical planning process

Physical planning has two distinct phases:

**Phase 1 — Strategy application**: the `SparkPlanner` applies a set of **planning strategies**, each of which pattern-matches on logical nodes and produces one or more candidate physical plans. The planner selects the best candidate (typically the one with the lowest estimated cost or the first one produced by the highest-priority strategy).

**Phase 2 — Physical rule application**: after a physical plan is selected, a set of **preparation rules** refine it—inserting `Exchange` (shuffle) and `Sort` nodes wherever the plan's data flow requirements aren't met, fusing operators into codegen stages, and applying AQE rewrites after shuffle execution.

> **Physical planning is like a construction project manager converting blueprints into a work schedule.** The blueprints (logical plan) say "there should be a wall here." The project manager decides *how* to build it: which crew to use, what materials, in what order. Different choices produce the same wall with different cost and time profiles. Some choices require staging areas (shuffles); some can be done on-site (local operations).

---

## Phase 1: Planning strategies

The `SparkPlanner` applies strategies in a fixed priority order. The first strategy to produce a physical plan for a logical node wins.

### Strategy 1: FileSourceStrategy — planning scans

Converts logical `Relation` nodes (pointing to files) into physical `FileSourceScanExec` (for V1 sources) or `BatchScanExec` (for DataSource V2) operators.

**Logical node**:
```
Relation[order_id#1, amount#3, status#4] parquet (with PushedFilters)
```

**Physical operator**:
```
FileScan parquet [order_id#1, amount#3, status#4]
  Batched: true
  Format: Parquet
  Location: InMemoryFileIndex[hdfs:///data/orders]
  PartitionFilters: [order_date#5 >= 2024-01-01]
  PushedFilters: [IsNotNull(status), EqualTo(status,complete)]
  ReadSchema: struct<order_id:bigint,amount:double,status:string>
```

Key decisions made here:
- **Partition pruning**: which directory partitions to read (from `PartitionFilters`)
- **Pushed filters**: which row-level filters the format can apply internally
- **ReadSchema**: column pruning—only requested columns are in the schema

---

### Strategy 2: JoinSelection — choosing a join algorithm

This is the most consequential strategy. For each logical `Join` node, `JoinSelection` evaluates candidates in this exact priority order:

**Step 1 — Check for hints**:
```sql
SELECT /*+ BROADCAST(customers) */ * FROM orders JOIN customers ON ...
SELECT /*+ MERGE(orders, customers) */ * FROM orders JOIN customers ON ...
SELECT /*+ SHUFFLE_HASH(orders) */ * FROM orders JOIN customers ON ...
```
Hints always win over automatic selection.

**Step 2 — Broadcast Hash Join**: can either side be broadcast?
- Check if left or right side's estimated size ≤ `spark.sql.autoBroadcastJoinThreshold` (default 10MB)
- Check if the join type allows broadcasting (full outer joins cannot broadcast both sides)
- Check if the build side (broadcast side) can fit in executor memory

If yes → `BroadcastHashJoin(buildSide=Left|Right)`

**Step 3 — Shuffled Hash Join**: with `preferSortMergeJoin=false`, if the build side fits per partition:
- Both sides shuffled, one side hashed per partition
- Faster than sort-merge when partition sizes are small enough to hash

If yes → `ShuffledHashJoin`

**Step 4 — Sort-Merge Join**: the default for large-large equi-joins:
- Both sides shuffled and sorted
- Works for any size, but requires two shuffles and two sorts

If none of the above → `SortMergeJoin`

**Step 5 — Broadcast Nested Loop / Cartesian**: for non-equi joins or cross joins:
- `BroadcastNestedLoopJoin`: O(n×m) with one side broadcast
- `CartesianProduct`: no keys at all (explicit CROSS JOIN)

**Example: which plan you get for different table sizes**

```sql
SELECT * FROM orders o JOIN customers c ON o.customer_id = c.id
```

| orders size | customers size | Plan chosen |
|-------------|---------------|-------------|
| 1 TB | 5 MB | BroadcastHashJoin (customers broadcast) |
| 1 TB | 500 MB | SortMergeJoin (customers too large to broadcast) |
| 500 MB | 500 MB | SortMergeJoin |
| 10 MB | 5 MB | BroadcastHashJoin (smaller side broadcast) |

```
-- BroadcastHashJoin in EXPLAIN output:
*(2) BroadcastHashJoin [customer_id#2], [id#20], Inner, BuildRight
:- *(2) FileScan parquet orders[...]
+- BroadcastExchange HashedRelationBroadcastMode([id#20])
   +- *(1) FileScan parquet customers[...]

-- SortMergeJoin in EXPLAIN output:
*(5) SortMergeJoin [customer_id#2], [id#20], Inner
:- *(3) Sort [customer_id#2 ASC]
:  +- Exchange hashpartitioning(customer_id#2, 200)
:     +- *(1) FileScan parquet orders[...]
+- *(4) Sort [id#20 ASC]
   +- Exchange hashpartitioning(id#20, 200)
      +- *(2) FileScan parquet customers[...]
```

---

### Strategy 3: Aggregation — choosing an aggregation algorithm

For logical `Aggregate` nodes, the strategy produces a two-phase aggregation:

**HashAggregate** (the default and fastest):
- Partial phase: each task builds a local hash map (key → partial aggregate buffer), output is the partially aggregated rows
- Final phase: after shuffle, each task merges the partial results from all mappers

```
*(4) HashAggregate(keys=[customer_id#2], functions=[sum(amount#3)])
+- Exchange hashpartitioning(customer_id#2, 200)
   +- *(3) HashAggregate(keys=[customer_id#2], functions=[partial_sum(amount#3)])
      +- *(3) FileScan parquet orders[customer_id#2, amount#3]
```

**ObjectHashAggregate**: used when the aggregate function doesn't support partial aggregation in binary format (e.g., `collect_list`, `collect_set`, custom UDAFs). Slower—uses Java object heap rather than Tungsten buffers.

**SortAggregate**: fallback when hash aggregation isn't possible (no code generation support). Requires sorted input; slowest option.

> **Two-phase aggregation is like vote counting at a national election.** Each polling station (task) counts its own ballots locally (partial HashAggregate). The local counts are sent to regional centers (shuffle). The regional centers combine the local counts into a final total (final HashAggregate). You never need to ship all raw ballots to one location.

---

### Strategy 4: Window — planning window functions

Logical `Window` nodes become `WindowExec` physical operators, which require their input to be partitioned by the window's `PARTITION BY` keys and sorted by the `ORDER BY` keys.

```sql
SELECT customer_id, amount,
       RANK() OVER (PARTITION BY customer_id ORDER BY amount DESC) AS rnk
FROM orders
```

**Physical plan**:
```
*(3) Window [rank() OVER (PARTITION BY customer_id ORDER BY amount DESC)]
+- *(2) Sort [customer_id ASC, amount DESC]
   +- Exchange hashpartitioning(customer_id, 200)
      +- *(1) FileScan parquet orders[customer_id, amount]
```

The `Exchange` (shuffle by `customer_id`) and `Sort` (sort by `customer_id, amount`) are planned by `WindowExec`'s requirements, enforced in Phase 2 (see `EnsureRequirements` below).

---

### Strategy 5: BasicOperators — everything else

Simple operators have direct physical equivalents:
- `Filter` → `FilterExec`
- `Project` → `ProjectExec`
- `Sort` → `SortExec` (or `TakeOrderedAndProjectExec` for `ORDER BY ... LIMIT n`)
- `LocalLimit` / `GlobalLimit` → `LocalLimitExec` / `CollectLimitExec`
- `Expand` → `ExpandExec`
- `Generate` (explode, etc.) → `GenerateExec`

---

## Phase 2: Physical preparation rules

After a physical plan tree is assembled from strategies, **preparation rules** refine it. These rules are applied in order, not in a fixed-point loop.

### Rule 1: EnsureRequirements — inserting Exchange and Sort

This is the most important preparation rule. Every physical operator declares what it **requires** from its children (input partitioning and sort order) and what it **provides** to its parent (output partitioning and sort order).

For example:
- `SortMergeJoin` requires both children to be hash-partitioned by the join key **and** sorted by the join key
- `HashAggregate` (final phase) requires its input to be hash-partitioned by the group key
- `WindowExec` requires its input to be partitioned by PARTITION BY keys and sorted by ORDER BY keys
- `FileScan` provides no partitioning guarantee (random read order)

`EnsureRequirements` walks the physical plan tree bottom-up. For each operator, it checks whether the child's output partitioning/ordering satisfies the operator's requirements. If not, it **inserts** `Exchange` (for partitioning requirements) and/or `Sort` (for ordering requirements) between them.

**Example: adding Exchange for SortMergeJoin**

Before `EnsureRequirements`:
```
SortMergeJoin [customer_id], [id]
:- FileScan parquet orders[...]           ← arbitrary partitioning
+- FileScan parquet customers[...]        ← arbitrary partitioning
```

After `EnsureRequirements`:
```
SortMergeJoin [customer_id], [id]
:- Sort [customer_id ASC]
:  +- Exchange hashpartitioning(customer_id, 200)   ← inserted
:     +- FileScan parquet orders[...]
+- Sort [id ASC]
   +- Exchange hashpartitioning(id, 200)            ← inserted
      +- FileScan parquet customers[...]
```

**Example: no Exchange needed (already partitioned)**

If `orders` was bucketed by `customer_id` when written, `FileScan` already provides `HashPartitioning(customer_id, num_buckets)`. `EnsureRequirements` sees that the child already satisfies the join's requirement and inserts nothing.

> **EnsureRequirements is like a logistics manager who adds transit steps wherever shipments aren't in the right place.** The manager looks at each step of the supply chain: "Does the raw material arrive at this factory in the right form?" If not, they insert a distribution center (Exchange) or sorting facility (Sort) before it. If it already arrives correctly, nothing is added.

---

### Rule 2: CollapseCodegenStages — grouping into WholeStageCodegen

After `EnsureRequirements`, the plan has its full set of operators. `CollapseCodegenStages` groups consecutive operators that support code generation into a single `WholeStageCodegenExec` wrapper. The operators inside one `WholeStageCodegenExec` stage are compiled together into a single tight Java loop.

**Rules for what can be in the same codegen stage**:
- Operators that consume rows one at a time (pipeline-able): `FilterExec`, `ProjectExec`, `HashAggregateExec`, `SortMergeJoinExec`, `SortExec`
- A stage boundary exists at every `Exchange` (shuffle can't be fused with surrounding operators), and at operators that don't support codegen (Python UDFs, some legacy operators)

**Example**:
```
== Physical Plan ==
*(3) HashAggregate(...)                         ← stage 3
+- Exchange hashpartitioning(customer_id, 200)  ← boundary
   +- *(2) HashAggregate(...)                   ← stage 2
      +- *(2) Filter (amount > 100)
         +- *(2) FileScan parquet orders[...]   ← stages 2 fuses scan + filter + partial agg
```

Stage `*(2)` fuses scan, filter, and partial aggregation into one loop: for each row read from Parquet, the filter is applied and, if it passes, the row is fed into the hash aggregate buffer—all without materializing intermediate rows.

The `*(N)` prefix in `explain()` output is the codegen stage number. Every `Exchange` resets the stage counter. Counting `*(N)` boundaries tells you how many compiled loops the query uses.

> **CollapseCodegenStages is like combining several assembly line steps into one workstation.** Instead of a worker doing one thing and passing the part to the next worker (materializing intermediate rows), one workstation does all three steps on each part before moving on. Less handling, less time.

---

### Rule 3: ReuseExchange — deduplicating identical shuffles

If the same subplan appears in two places in the physical plan (e.g., a self-join, or a query where the same filtered table feeds two branches), `ReuseExchange` detects the duplicate `Exchange` nodes and replaces the second one with a reference to the first. The shuffle is executed once; both consumers read from the same shuffle output.

**Example** (self-join for period-over-period comparison):
```sql
SELECT a.customer_id, a.amount - b.amount AS change
FROM orders a JOIN orders b
  ON a.customer_id = b.customer_id
  AND a.order_date = b.order_date - INTERVAL 1 DAY
```

Without `ReuseExchange`, `orders` is shuffled twice (once for each side of the join). With `ReuseExchange`, the shuffle happens once and both sides of the join read from it.

---

### Rule 4: InsertAdaptiveSparkPlan — wrapping for AQE

When `spark.sql.adaptive.enabled=true`, the entire physical plan is wrapped in an `AdaptiveSparkPlanExec` operator. This wrapper:
1. Executes the plan stage by stage
2. After each shuffle stage completes, collects statistics (partition sizes, row counts)
3. Re-runs physical planning on downstream stages using those real statistics
4. Applies AQE-specific rules: `CoalesceShufflePartitions`, `OptimizeSkewedJoin`, `OptimizeLocalShuffleReader`

**AQE physical rules**:

| Rule | What it does | Trigger |
|------|-------------|---------|
| `CoalesceShufflePartitions` | Merges small adjacent shuffle partitions into one task | Post-shuffle: many partitions are tiny |
| `OptimizeSkewedJoin` | Splits one large partition into sub-partitions, duplicates the matching small-side partition | Post-shuffle: one partition is 5× the median size |
| `OptimizeLocalShuffleReader` | Reads shuffle data locally (no network) when the consumer runs on the same executor as the producer | Post-shuffle: BroadcastHashJoin conversion |
| `DynamicJoinSelection` | Converts `SortMergeJoin` to `BroadcastHashJoin` if the actual shuffle output is small enough | Post-shuffle: measured size < broadcast threshold |

```
== Physical Plan ==
AdaptiveSparkPlan isFinalPlan=true
+- == Final Plan ==                      ← plan after AQE adapted it
   *(2) BroadcastHashJoin [...]          ← was SortMergeJoin in initial plan
   :- *(2) FileScan parquet orders[...]
   +- BroadcastExchange [...]
      +- *(1) FileScan parquet customers[...]
+- == Initial Plan ==                    ← what was planned before runtime stats
   SortMergeJoin [...]
   ...
```

---

## From physical plan to RDD: the bridge to execution

The physical plan tree is a tree of `SparkPlan` nodes. Every `SparkPlan` implements one method that connects Catalyst's planning world to Spark's execution engine:

```scala
def execute(): RDD[InternalRow]
```

Calling `execute()` on the **root** node of the physical plan triggers a recursive chain of `execute()` calls down the tree. Each operator wraps its children's RDDs in a new RDD that applies the operator's logic. The result is an RDD DAG—the same kind of DAG the DAG Scheduler has always understood. **The physical plan is just a typed, optimized recipe for constructing an RDD DAG.**

> **The physical plan is the blueprint; the RDD DAG is the building.** The blueprint (SparkPlan tree) describes what to build in structured, readable form. `execute()` is the act of construction—it walks the blueprint and produces the actual building (RDD DAG) that the execution engine can run. Once `execute()` returns, Catalyst is done; Spark's scheduler takes over.

---

### How each operator type maps to an RDD

**FileScan / BatchScan**

The leaf of the tree. `execute()` creates an `RDD[InternalRow]` backed by the file system. Each partition of the RDD corresponds to one or more input file splits. For Parquet, each partition reads a subset of row groups; for CSV, a byte range. The RDD's compute function does the actual I/O, applies pushed-down filters, and deserializes bytes into `InternalRow` objects (or, with the vectorized reader, into columnar `ColumnarBatch` objects).

```
FileScan parquet → HadoopFileScanRDD (one partition per file split)
```

**FilterExec / ProjectExec**

Wraps the child's RDD with a `mapPartitionsWithIndexInternal` that applies the filter condition or projection expression to each row. No shuffle. No new partitioning.

```
FilterExec(child)  →  child.execute().mapPartitionsWithIndexInternal(applyFilter)
ProjectExec(child) →  child.execute().mapPartitionsWithIndexInternal(applyProjection)
```

**Exchange (shuffle)**

`Exchange` is the operator that splits the RDD DAG into **stages**. `execute()` on an `Exchange`:
1. Calls `child.execute()` to get the map-side RDD.
2. Registers a `ShuffleDependency` on that RDD — this is the shuffle boundary the DAG Scheduler recognises.
3. Returns a `ShuffledRowRDD` — reading shuffle blocks written by the map stage.

The DAG Scheduler sees the `ShuffleDependency` as a stage boundary and schedules the map stage (everything below the `Exchange`) before the reduce stage (everything above it).

```
Exchange hashpartitioning(key, 200)
  → child.execute()                         ← map stage RDD
  → ShuffleDependency(partitioner)          ← stage boundary
  → ShuffledRowRDD                          ← reduce stage RDD (reads shuffle blocks)
```

> **Exchange is the toll booth on the RDD highway.** Everything before the toll booth (map stage) must complete before anything after it (reduce stage) can proceed. The DAG Scheduler enforces this sequencing. Every `Exchange` in the physical plan equals one stage boundary in the execution timeline.

**BroadcastExchange**

Similar to `Exchange` but instead of writing shuffle files, it calls `collect()` on the child's RDD (gathering data to the driver) and then broadcasts the result to all executors via `SparkContext.broadcast()`. The result is a `Broadcast[HashedRelation]`—a hash map available in every executor's memory.

```
BroadcastExchange
  → child.execute().collect()             ← one small job to gather data to driver
  → SparkContext.broadcast(hashedRelation) ← distribute to all executors
  → Future[Broadcast[HashedRelation]]
```

**BroadcastHashJoinExec**

`execute()` on the probe side calls the child's `execute()` RDD and applies a `mapPartitionsWithIndexInternal` that, for each row, probes the broadcast `HashedRelation` hash map. No shuffle on the probe side.

```
BroadcastHashJoinExec
  → probeChild.execute()
      .mapPartitionsWithIndexInternal { row =>
          val relation = broadcastRelation.value   // from executor local copy
          relation.get(row.joinKey)                // O(1) hash map lookup
      }
```

**SortMergeJoinExec**

Two child RDDs (both already shuffled and sorted by `EnsureRequirements`). `execute()` zips the two RDDs partition-by-partition and runs a merge algorithm on each pair of partitions.

```
SortMergeJoinExec
  → ZippedPartitionsRDD2(leftChild.execute(), rightChild.execute())
      .mapPartitions { (leftIter, rightIter) => mergeJoin(leftIter, rightIter) }
```

**HashAggregateExec**

Two instances in the plan (partial + final). Each calls `child.execute()` and applies a `mapPartitionsWithIndexInternal` that processes rows through the aggregation buffer (a hash map of key → buffer).

```
HashAggregateExec(Partial)  → child RDD → mapPartitions(buildHashMap, emitPartialAgg)
HashAggregateExec(Final)    → shuffled RDD → mapPartitions(mergeBuffers, emitResult)
```

**WholeStageCodegenExec**

When `CollapseCodegenStages` fuses multiple operators into one stage, the root of that stage is wrapped in `WholeStageCodegenExec`. Its `execute()`:
1. Generates Java source code for the entire fused pipeline (e.g., scan + filter + partial agg in one loop).
2. Compiles it at runtime using Janino (a Java compiler embedded in the JVM).
3. Returns an RDD whose compute function calls the compiled code.

```
WholeStageCodegenExec(FileScan → Filter → HashAggregate)
  → generates Java code:
      for each row in fileSplit:
        if (row.amount > 100):           // filter inlined
          aggBuffer[row.customerId] += row.amount  // agg inlined
  → compiles to bytecode
  → RDD whose partitions run the compiled bytecode
```

This is why `*(N)` stages in `explain()` output are so much faster than non-fused pipelines: each partition runs one tight compiled loop, not a chain of virtual `next()` calls through the operator tree.

---

### The complete execution chain for a SQL query

```
spark.sql("SELECT country, SUM(amount) FROM orders JOIN customers ...")
         ↓
  queryExecution.executedPlan          ← the final SparkPlan tree
         ↓
  executedPlan.execute()               ← recursive execute() calls build the RDD DAG
         ↓
  RDD DAG with ShuffleDependencies     ← submitted to DAG Scheduler
         ↓
  DAG Scheduler splits at ShuffleDependencies → Stage 1, Stage 2, Stage 3 ...
         ↓
  Task Scheduler submits tasks per partition per stage to executors
         ↓
  Executor runs the compiled bytecode (or interpreted eval) for each partition
         ↓
  Results collected / written to sink
```

**Key insight**: there is no SQL-specific execution engine. Once `execute()` is called, everything is RDDs, stages, tasks, and the DAG Scheduler—the same infrastructure that runs `rdd.map().filter().reduce()`. SQL is just a structured, optimized way of building the RDD DAG.

---

### Worked example: tracing one SQL query from plan to RDD DAG

Take this query and follow it all the way to executor tasks:

```python
df = spark.sql("""
    SELECT region, SUM(sales) AS total_sales
    FROM transactions
    WHERE year = 2024
    GROUP BY region
""")
df.show()
```

**Step 1 — Physical plan produced by Catalyst**

```
== Physical Plan ==
*(3) HashAggregate(keys=[region#10], functions=[sum(sales#11)])
+- Exchange hashpartitioning(region#10, 200), ENSURE_REQUIREMENTS
   +- *(2) HashAggregate(keys=[region#10], functions=[partial_sum(sales#11)])
      +- *(2) Filter (isnotnull(year#12) AND (year#12 = 2024))
         +- *(2) FileScan parquet transactions[region#10, sales#11, year#12]
               PartitionFilters: [year#12 = 2024]
               PushedFilters:    [IsNotNull(year#12)]
               ReadSchema:       struct<region:string, sales:double>
```

**Step 2 — `execute()` call tree builds the RDD DAG**

When `df.show()` triggers execution, Spark calls `executedPlan.execute()`. This fans out recursively:

```
HashAggregate(Final).execute()
  └─ Exchange.execute()
       ├─ [registers ShuffleDependency]          ← Stage boundary here
       └─ HashAggregate(Partial).execute()
            └─ Filter.execute()
                 └─ FileScan.execute()
                      └─ returns HadoopFileScanRDD   ← leaf; starts I/O
```

Each `execute()` call does not run any computation yet — it just **wraps** the child's RDD in a new RDD, adding one transformation layer. This is still lazy.

**Step 3 — RDD DAG that results**

```
HadoopFileScanRDD          (8 partitions — 8 Parquet files in year=2024/)
  └─ mapPartitionsRDD       (Filter: year = 2024 inlined by codegen)
       └─ mapPartitionsRDD  (Partial HashAggregate: builds per-partition hash map)
            └─ ShuffledRowRDD  ← ShuffleDependency inserted by Exchange
                 └─ mapPartitionsRDD  (Final HashAggregate: merges partial results)
```

> **The RDD DAG is the physical plan rewritten in Spark's native language.** Every operator became an `mapPartitions` transformation. The `Exchange` became a `ShuffledRowRDD`. The `FileScan` became a `HadoopFileScanRDD`. The structure is identical — just expressed as RDD lineage rather than a plan tree.

**Step 4 — DAG Scheduler splits at the ShuffleDependency**

The DAG Scheduler walks the RDD DAG and finds the `ShuffleDependency` inserted by `Exchange`. It creates two stages:

```
Stage 1 (map stage):   HadoopFileScanRDD → Filter → Partial HashAggregate
                       8 tasks (one per partition)
                       Each task: reads one Parquet file, filters year=2024,
                                  builds a partial sum per region,
                                  writes shuffle output to local disk

Stage 2 (reduce stage): ShuffledRowRDD → Final HashAggregate
                        200 tasks (one per shuffle partition — spark.sql.shuffle.partitions)
                        Each task: reads shuffle blocks for one region bucket,
                                   merges partial sums,
                                   emits final SUM(sales) per region
```

**Step 5 — WholeStageCodegen: what each task actually runs**

The `*(2)` label in the plan means `FileScan + Filter + Partial HashAggregate` are fused. Stage 1's tasks do not run three separate operators. Janino compiled them into a single Java method that looks roughly like:

```java
// Generated code for Stage 1 — all three operators fused into one loop
while (parquetReader.hasNext()) {
    InternalRow row = parquetReader.next();
    // Filter inlined:
    if (row.getInt(2) != 2024) continue;          // year = 2024
    // Partial agg inlined:
    String region = row.getString(0);
    double sales  = row.getDouble(1);
    aggHashMap.merge(region, sales, Double::sum);  // build partial sum
}
// emit partial aggregates to shuffle output
for (Map.Entry<String, Double> e : aggHashMap.entrySet()) {
    shuffleWriter.write(e.getKey(), e.getValue());
}
```

One loop, no virtual dispatch, no row-by-row operator chaining. This is the payoff of `CollapseCodegenStages`.

**Step 6 — Stage 2 tasks read shuffle blocks and produce the result**

Each of the 200 Stage 2 tasks:
1. Fetches shuffle blocks (partial sums for its assigned region bucket) from all Stage 1 executors via the `BlockManager` network protocol.
2. Runs the Final `HashAggregate` compiled loop, merging partial sums into a final sum.
3. Emits `(region, total_sales)` rows.

`df.show()` collects all 200 partitions to the driver and prints the first 20 rows.

**Summary of what RDDs contribute**

| What RDD provides | How it's used in SQL execution |
|---|---|
| Lazy evaluation | `execute()` builds the RDD DAG without running anything; the job runs only when an action is called |
| Partitioning | Each Parquet file split → one RDD partition → one task |
| `ShuffleDependency` | `Exchange` registers a shuffle; the DAG Scheduler uses this to draw the stage boundary |
| `mapPartitions` | Every plan operator (Filter, Project, Aggregate) becomes a `mapPartitions` transformation |
| `NarrowDependency` | Within a codegen stage, operators are fused — no inter-operator data movement |
| Fault tolerance | If a task fails, Spark re-runs just that partition's `mapPartitions` chain from the last shuffle boundary |

---

### Accessing the executed plan and RDD programmatically

```python
df = spark.sql("SELECT country, SUM(amount) FROM orders GROUP BY country")

# The final physical plan after all preparation rules
print(df.queryExecution.executedPlan)

# The RDD that executedPlan.execute() would return
rdd = df.queryExecution.toRdd   # returns RDD[InternalRow]

# How many partitions / stages result from this plan
print(rdd.getNumPartitions())

# Full pipeline: what Spark will actually run
df.explain("formatted")
```

`queryExecution.toRdd` triggers `executedPlan.execute()` and returns the root RDD without running any computation (it's still lazy). Calling an action on that RDD (or on `df` directly) submits the job to the DAG Scheduler.

---

## Reading the full plan: putting it all together

```python
df = spark.sql("""
    SELECT c.country, SUM(o.amount) AS total
    FROM orders o JOIN customers c ON o.customer_id = c.id
    WHERE o.amount > 100
    GROUP BY c.country
""")
df.explain("formatted")
```

**Annotated physical plan**:
```
== Physical Plan ==
*(5) HashAggregate(keys=[country#20], functions=[sum(amount#3)])      ← final agg (stage 5)
+- Exchange hashpartitioning(country#20, 200)                        ← shuffle for final agg
   +- *(4) HashAggregate(keys=[country#20], functions=[partial_sum]) ← partial agg (stage 4)
      +- *(4) BroadcastHashJoin [customer_id#2],[id#21], BuildRight   ← join (stage 4)
         :- *(4) Filter (isnotnull(customer_id#2) AND amount#3 > 100) ← filter (fused in stage 4)
         :  +- *(4) FileScan parquet orders[customer_id#2, amount#3]  ← scan (fused in stage 4)
         +- BroadcastExchange [id#21, country#20]                     ← broadcast customers
            +- *(3) Filter (isnotnull(id#21))                         ← filter (stage 3)
               +- *(3) FileScan parquet customers[id#21, country#20]  ← scan (fused in stage 3)
```

**Tracing the decisions**:
1. `FileSourceStrategy` planned two `FileScan` nodes (with pruned columns and pushed filters)
2. `JoinSelection` chose `BroadcastHashJoin` (customers is small)
3. `AggregationStrategy` planned two-phase `HashAggregate`
4. `EnsureRequirements` inserted `Exchange hashpartitioning(country)` for the final aggregate
5. `CollapseCodegenStages` fused scan + filter + join + partial_agg into stage `*(4)`; scan + filter for customers into `*(3)`; final agg into `*(5)`

---

## Bringing it together

Physical planning converts an optimized logical plan into runnable operators through two phases:

**Phase 1 — Strategy application** (logical → physical operators):
- `FileSourceStrategy` → `FileScan` with pushed filters and column pruning
- `JoinSelection` → `BroadcastHashJoin` / `SortMergeJoin` / `ShuffledHashJoin` based on size and hints
- `Aggregation` → two-phase `HashAggregate` (partial → shuffle → final)
- `Window` → `WindowExec` requiring shuffle + sort
- `BasicOperators` → direct physical equivalents for Filter, Project, Sort, Limit

**Phase 2 — Preparation rules** (refining the physical plan):
- `EnsureRequirements` → inserts `Exchange` (shuffle) and `Sort` wherever children don't satisfy operator requirements
- `CollapseCodegenStages` → fuses consecutive operators into `WholeStageCodegenExec` compiled loops
- `ReuseExchange` → deduplicates identical shuffle nodes
- `InsertAdaptiveSparkPlan` → wraps the plan for AQE runtime re-planning

Every operator in the final plan reflects a deliberate decision about data movement, memory use, and CPU efficiency. The `EXPLAIN` output is the complete record of those decisions — and understanding the strategies and preparation rules that produced it is how you diagnose why a plan is slow and how to make it faster.

**The RDD bridge — how the plan actually runs**: every `SparkPlan` node implements `execute(): RDD[InternalRow]`. Calling `execute()` on the root recursively builds an RDD DAG where `Exchange` nodes introduce `ShuffleDependency` stage boundaries. `WholeStageCodegenExec` nodes compile fused pipelines into bytecode so each partition runs a tight loop. Once `execute()` returns, Catalyst is finished. The resulting RDD DAG is handed to the DAG Scheduler, which splits it into stages at the shuffle boundaries, then to the Task Scheduler, which dispatches tasks to executors. SQL execution is just a structured, optimized way of constructing an RDD DAG — the same engine that has always powered Spark.
