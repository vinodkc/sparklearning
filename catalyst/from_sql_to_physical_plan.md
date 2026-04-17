# From SQL to a Running Plan: The Catalyst Story

This is the story of what happens between the moment you write a SQL query (or a DataFrame transformation) and the moment Spark's executors start moving bytes. The gap between those two moments is bridged by **Catalyst**, Spark's query optimizer. Catalyst takes your intent—expressed as SQL text or a chain of DataFrame operations—and turns it into an efficient, executable plan. The journey has four acts: parsing, analysis, optimization, and physical planning. Understanding each act explains why Spark often runs queries much faster than you'd expect, and why changing a few lines of SQL can dramatically change what actually executes.

---

## Act 1: Parsing — from text to a tree

If you write a SQL string, the first thing Catalyst does is **parse** it. A parser reads the SQL text and builds an **unresolved logical plan**: a tree of nodes where each node represents an operation (Select, Filter, Join, Aggregate) and each expression represents a computation (a column reference, a function call, an arithmetic expression). At this point, the tree is syntactically valid but **unresolved**: column names are just strings; there are no types; the planner doesn't yet know which table "orders" refers to or what type the column "amount" is.

If you write a DataFrame program instead of SQL, you skip the text-parsing step. Each DataFrame transformation you call directly constructs the same kind of logical plan node. Whether you write `SELECT amount FROM orders WHERE amount > 100` or `orders.filter($"amount" > 100).select($"amount")`, by the end of this step you have the same unresolved tree.

---

## Act 2: Analysis — making the tree meaningful

The analyzer takes the unresolved logical plan and resolves every name and type using the **catalog** (Spark's metadata repository, which knows about registered tables, views, and databases) and the schemas of the DataFrames in scope.

Column references are replaced with typed attributes—not just the string `"amount"` but "the column amount from the table orders, of type double." Function names are resolved to actual function implementations. Ambiguous references are detected and rejected with an error. Types are propagated: if you multiply a column of type `int` and a column of type `double`, the analyzer inserts an implicit cast. At the end of analysis, you have a **resolved logical plan**—a tree where every node and every expression has a concrete type, and every name points to a specific column or function.

If anything can't be resolved—an unknown column name, a type mismatch with no safe cast, an ambiguous reference—Spark throws an `AnalysisException` here, before any optimization or execution starts. This is why Spark's error messages for schema problems appear early, at plan construction time.

---

## Act 3: Optimization — making the plan cheaper

The resolved logical plan is logically correct but not necessarily efficient. Act 3 is where Catalyst earns its reputation: the **optimizer** applies a large set of **transformation rules** to rewrite the plan into an equivalent but cheaper one. Rules are applied repeatedly, in passes, until no rule changes the plan anymore (or until a fixed-point limit is reached).

Some rules eliminate wasted work. **Predicate pushdown** moves filter conditions as early as possible in the plan—closer to the data source. If you filter on `country = 'US'` after a join, the optimizer may push that filter below the join so that less data enters the join in the first place. When the data source is a Parquet file, Spark can push the filter even further—into the file reader—so that rows that don't match are never deserialized at all.

**Column pruning** removes columns from earlier steps that are never used by later steps. If you select only two columns out of twenty at the end of a complex query, the optimizer may propagate that information backward and avoid reading or computing the other eighteen columns.

**Constant folding** replaces expressions that can be evaluated at plan time with their results: `1 + 1` becomes `2`, a `CASE WHEN false THEN ...` arm is eliminated. **Null propagation** simplifies expressions involving nulls. Boolean simplification collapses redundant predicates.

**Join reordering** in the cost-based optimizer (CBO) uses statistics—row counts, column cardinality, null counts, min/max values—to reorder a sequence of joins so that smaller tables are joined first. This is the most impactful optimization for complex multi-table queries, and it is only available when you've run `ANALYZE TABLE` (or `computeStatistics` on a DataFrame) to collect statistics.

By the end of the optimization phase you have an **optimized logical plan**—still abstract (it says "join these two things" without saying *how*), but much leaner than what you started with.

---

## Act 4: Physical planning — choosing how to execute

The optimized logical plan is still a description of *what* to compute, not *how*. The physical planner turns it into a **physical plan** by choosing concrete execution strategies for each logical operation.

The most visible choice is for **joins**: should this join be a broadcast hash join, a sort-merge join, or a shuffled hash join? The planner looks at the sizes of the inputs (from statistics), the join keys, and configuration thresholds to decide. A small table joined with a large one becomes a broadcast hash join: the small table is sent to every executor so that the large table's partitions don't need to move. Two large tables become a sort-merge join: both sides are sorted on the join key and merged. The join story has its own chapter; the point here is that physical planning is where those decisions are made.

For aggregations, the planner chooses between hash-based aggregation (build a hash map of group keys → aggregated values) and sort-based aggregation (sort by key, then stream through merging equal keys). For data sources, it decides which scan strategy to use. For exchanges (shuffles), it inserts **Exchange** nodes that represent the data movement.

The planner may generate **multiple candidate physical plans** and score them, picking the cheapest. This is where cost-based optimization fully applies: if the CBO has statistics, it can compare the estimated cost of two physical plans and pick the winner.

---

## Whole-stage codegen: collapsing the plan into tight loops

There is one more transformation after physical planning: **whole-stage code generation (codegen)**. Spark's execution model was originally **Volcano-style**: each operator in the plan implemented a `next()` method that called `next()` on its child, pulled one row, processed it, and returned it. This is clean and composable but slow: every row involves many virtual function calls, and each operator has to box/unbox values and check types.

Whole-stage codegen collapses a pipeline of operators into a single Java function that is generated at runtime and JIT-compiled. A chain of Filter → Project → Aggregate might generate a tight loop that, for each input row, checks the filter condition, extracts the needed columns, and updates the aggregation state—all in one contiguous block of code with no virtual dispatch, no intermediate row objects, and no type checks at each step. The JVM's JIT compiler then has a single simple loop to optimize, which it can vectorize, unroll, and register-allocate aggressively.

Not every operator supports codegen—complex aggregations, sorts, and exchanges are "codegen boundaries" that break the pipeline and fall back to the Volcano model for that step. But for the long chains of filters, projections, and simple aggregations that appear in analytic queries, codegen is the reason Spark approaches the performance of hand-written code.

---

## The EXPLAIN plan: reading the map

You can see all of this at any time by calling `explain(extended=True)` on a DataFrame or running `EXPLAIN EXTENDED` in SQL. You get four sections: the parsed logical plan, the analyzed logical plan, the optimized logical plan, and the physical plan. Reading from bottom (closest to data) to top (the final output), you can see exactly what operators Spark will run, what filters were pushed down, what exchanges were inserted, and which join strategy was chosen. When a query is slow, the EXPLAIN output is usually the first place to look.

---

## Bringing it together

A SQL query or DataFrame program enters Catalyst as unresolved text (or an unresolved tree of transformations). The **analyzer** resolves names, types, and schemas using the catalog. The **optimizer** applies dozens of rewrite rules—predicate pushdown, column pruning, constant folding, join reordering—to produce an optimized logical plan that describes the same computation at lower cost. The **physical planner** chooses concrete strategies for joins, aggregations, and scans, generating one or more candidate physical plans and picking the cheapest. Finally, **whole-stage codegen** collapses pipelines of operators into single tight loops that the JVM can compile and run efficiently. So the journey is: **SQL text → unresolved tree → resolved tree → optimized logical plan → physical plan → generated code → execution.** Each step removes ambiguity or inefficiency, so that by the time data starts moving, Spark is doing much less work than a naïve reading of your query would suggest.
