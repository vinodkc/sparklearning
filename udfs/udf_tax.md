# The UDF Tax: Why User-Defined Functions Are a Black Box to the Optimizer

This is the story of what happens when you write a UDF—a user-defined function—and what Spark can and cannot do with it. UDFs are one of the most used features in Spark: whenever built-in functions don't cover your logic, you reach for a UDF. But UDFs come with a cost that isn't always obvious, and understanding that cost explains why performance can drop dramatically, why certain optimizations stop working, and what alternatives exist for the cases where performance really matters.

---

## What a UDF is and why it exists

Spark's built-in function library is large—hundreds of functions covering string manipulation, date arithmetic, math, array operations, JSON parsing, and more. But it can't cover everything. Sometimes your logic involves a custom algorithm, a lookup into a non-Spark data structure, a business rule too complex for SQL expressions, or a call to a third-party library. A **user-defined function** lets you register a custom function that Spark can call during query execution, as if it were a built-in.

The function is defined in Scala, Java, or Python, registered with the SparkSession or declared inline, and then used in DataFrame operations or SQL queries. From the query's perspective it looks like any other function call.

---

## The black box problem

The reason UDFs are expensive is simple: **Catalyst cannot see inside them**. Catalyst, Spark's query optimizer, works by analyzing and transforming the logical plan. It can push a filter down because it understands what the filter does—it can reason about the expression `amount > 1000`. It can reorder a join because it understands join semantics. It can eliminate a column from the scan because it can trace which columns are actually used.

A UDF is opaque to all of this. Catalyst sees the UDF as a black box: "call this function; it takes these inputs and returns this output; I know nothing else about it." Catalyst cannot:

- Push a predicate that depends on a UDF's result down into the data source (file scan can't filter on a UDF result before reading).
- Determine whether a UDF is pure (deterministic) or has side effects—so it must be conservative about when and how many times to call it.
- Inline, simplify, or reorder UDF calls the way it can with native expressions.
- Generate tight bytecode via whole-stage codegen that spans the UDF—the UDF is always a boundary in the codegen pipeline.

The moment a query touches a UDF, a chunk of the plan becomes opaque, and optimization stops at the UDF's boundary.

---

## The deserialization/serialization cost for Python UDFs

For Scala and Java UDFs, the function runs in the same JVM as the executor and data is passed as JVM objects. The overhead is mostly the inability to codegen across the UDF boundary: the UDF is called via reflection, each row is deserialized from Spark's binary UnsafeRow format into Java objects (one object per field of the row), passed to the function, and the result is serialized back. The deserialization cost is real but moderate—it's the same JVM, just without the tight-loop efficiency of codegen.

For **Python UDFs**, the cost is far higher. The function runs in a separate Python process. Data must cross a process boundary. Each row is serialized (pickled) by the JVM, sent over a pipe to the Python worker, deserialized (unpickled) in Python, passed to the Python function, the result is pickled back, sent over the pipe, and deserialized in the JVM. This happens **for every row**. For a 100 million-row table, this is 100 million serialize–transfer–deserialize–call–serialize–transfer–deserialize cycles.

In practice, Python UDFs add 10–100× overhead compared to equivalent built-in functions for row-at-a-time operations. For simple string operations, the serialization time can be more than 90% of the total UDF cost—the actual function logic is negligible by comparison.

---

## The codegen break

Spark's whole-stage codegen (covered in the Catalyst story) collapses a pipeline of operators—filter, project, aggregate—into a single tight loop with no virtual dispatch and no intermediate object allocation. UDFs break this pipeline. Codegen can operate on the operators before the UDF and the operators after it, but the UDF itself is a function call that returns to the regular, object-passing execution model. This boundary creates:

- **Forced deserialization**: values must be converted from the compact UnsafeRow binary format into Java objects before being passed to the UDF, and back after.
- **Per-row method call overhead**: no loop unrolling, no SIMD vectorization, no inlining.
- **Object allocation per row**: every UDF call may allocate Java objects for the arguments and result, increasing GC pressure.

Even for a trivial UDF that just adds two numbers, the codegen break means that surrounding operators can't be fused across it, and the whole-stage speedup that the rest of the query benefits from is lost for the portion that touches the UDF.

---

## Determinism and the "call it twice" problem

Catalyst assumes built-in functions are deterministic (given the same inputs, they always return the same output). This lets it, for example, evaluate an expression once and cache the result for reuse. For UDFs, Catalyst must assume the function might be non-deterministic (it could call an external service, read from a file, use random numbers, or maintain internal state). Unless you explicitly declare a UDF as deterministic, Catalyst will **not** cache its result and may call it more times than you expect—once for each reference to it in the plan.

A query like `SELECT my_udf(col) AS x FROM t WHERE my_udf(col) > 5` may call the UDF twice per row if Catalyst doesn't recognize it as deterministic: once to compute `x` and once to evaluate the filter. Declaring a UDF deterministic (`udf(..., deterministic=True)` in PySpark) allows Catalyst to reuse the result, but you must be correct—a non-deterministic UDF declared as deterministic can produce inconsistent results in plans that evaluate it multiple times.

---

## When the optimizer skips pushdown

Catalyst's predicate pushdown moves filter conditions closer to the data source, ideally pushing them into the file scan so that entire row groups are skipped before any data is decoded. Pushdown requires that the filter expression is one Catalyst understands and can pass to the data source connector.

A UDF in a filter expression blocks pushdown entirely. If you write `.filter(my_udf(col) > 0)`, Spark cannot push that filter into a Parquet scan because the connector doesn't know what `my_udf` does. All rows must be scanned and decoded, then passed to the UDF, then filtered. Compare this to `.filter(col > 0)` where Spark can pass `col > 0` to the Parquet reader, which uses column statistics to skip whole row groups and never decodes rows that can't match.

For tables where predicate pushdown skips 90% of the data, replacing that filter's logic with built-in expressions instead of a UDF could reduce I/O and runtime by 10×.

---

## Alternatives to row-at-a-time UDFs

**Use built-in functions first.** The built-in function library is richer than most people realize. Functions for JSON/XML parsing, complex type manipulation, string operations, window functions, and math are all built in, all fully supported by Catalyst, and all capable of being codegen'd. The first question before writing a UDF should always be: can this be expressed with built-ins?

**Use `expr()` and SQL expressions.** Even complex conditional logic can often be expressed as a SQL expression string, which Catalyst parses and optimizes like any other expression.

**Use pandas UDFs (vectorized UDFs) in PySpark.** Instead of calling a Python function once per row, a pandas UDF calls it once per batch of rows—typically 4096 at a time—passing a pandas Series and receiving a pandas Series back. Data is transferred in Apache Arrow format (a columnar batch) rather than row-by-row pickle. The overhead per row drops dramatically. Pandas UDFs still create a codegen boundary, but the process-crossing and serialization cost is amortized over thousands of rows instead of paid per row.

**Use Scala/Java UDFs instead of Python UDFs.** If you must use a UDF in a performance-critical path and can implement the logic in Scala or Java, the JVM-to-JVM call eliminates the process-crossing and pickle overhead. The codegen boundary still exists, but the per-row overhead is much lower.

**Rewrite logic as a native expression using `Column` API.** Sometimes a UDF wraps logic that could be expressed as a chain of native column operations. This is worth exploring because native column operations are transparent to Catalyst and can be codegen'd, filtered, and optimized end-to-end.

---

## When UDFs are the right tool

None of this means you should never use UDFs. For logic that genuinely can't be expressed with built-ins—calling a custom ML model, integrating with a third-party library, applying a domain-specific algorithm—UDFs are the right choice. The goal is to understand the tradeoffs so you can make the decision deliberately:

- Put the UDF as late in the query as possible, so built-in filters and projections can reduce data volume before the UDF sees it.
- Don't use a UDF as a filter if you can separate the filter into a built-in predicate that runs before the UDF.
- For Python UDFs on large datasets, evaluate whether a pandas UDF or a Scala rewrite would be acceptable.
- Register UDFs as deterministic if they are, to avoid redundant evaluations.

---

## Bringing it together

A UDF is a black box from Catalyst's perspective. The optimizer cannot look inside it, cannot push predicates through it, cannot codegen across it, and cannot safely assume it's deterministic. The consequences are: **predicate pushdown stops** at the UDF's filter; **whole-stage codegen breaks** at the UDF's boundary; **data must be deserialized** from binary row format into objects before the UDF sees it and serialized again after; for Python UDFs, **every row crosses a process boundary** with full pickle overhead. The UDF tax is paid in I/O (no pushdown), CPU (no codegen, per-row overhead), and memory (object allocation, GC). The remedy is to prefer built-in functions, use pandas UDFs when row-at-a-time Python is unavoidable, and place UDFs as late in the pipeline as possible so earlier operators can reduce data volume before the UDF runs.
