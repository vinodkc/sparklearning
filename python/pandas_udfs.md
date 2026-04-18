# Pandas UDFs: How Arrow Makes Python Functions Fast Enough for Spark

This is the story of how Spark bridged the performance gap between Python UDFs and built-in functions. The original Python UDF (covered in the UDF tax story) processes one row at a time: serialize each row to pickle, cross the JVM-Python process boundary, call the Python function on one value, serialize the result back, cross the boundary again. For a hundred million rows this overhead dominates. **Pandas UDFs**—also called vectorized UDFs—replace that row-at-a-time protocol with a batch protocol: data crosses the boundary once per batch of thousands of rows, in Apache Arrow format, and your Python function operates on a pandas Series using native NumPy operations. The performance difference is dramatic, often 10–100× faster than row-at-a-time Python UDFs.

---

## The core idea: batches instead of rows

A row-at-a-time Python UDF is called once per row. Spark serializes row N, sends it, calls the function, receives the result, deserializes it, then repeats for row N+1. The JVM–Python process boundary is crossed 2 × (number of rows) times.

A pandas UDF is called once per **batch** of rows (default batch size: 10,000 rows). Spark serializes an entire batch of values for the input column(s) into a single Arrow buffer, sends the buffer to the Python worker, calls your function with a pandas Series, receives a pandas Series back, and deserializes the results back into Spark rows. The JVM–Python boundary is crossed 2 times per batch—not 2 times per row.

> **Row-at-a-time UDFs are like a teller processing one customer at a time at a bank—greet, ID check, transaction, farewell, next.** For 10,000 customers, this ceremony is repeated 10,000 times. A pandas UDF is like a batch processing system: 10,000 forms are bundled together, sent to the back office at once, processed in bulk, and returned together. The ceremony happens once; the actual work happens in bulk.

For a 10,000-row batch, the per-row overhead is amortized across 10,000 rows. The Python function itself also benefits: instead of being called 10,000 times with scalar values, it is called once with a pandas Series of 10,000 values, and pandas operations on Series call into NumPy's C-implemented routines, which are far faster than Python-level loops.

---

## Apache Arrow: the fast lane

The efficiency of pandas UDFs rests on **Apache Arrow**, a language-independent columnar in-memory format. Arrow defines how to lay out a column of values as a contiguous buffer with a null bitmap, offsets (for variable-length types like strings), and type metadata.

The key properties of Arrow for PySpark:

**Zero-copy reads**: a pandas Series backed by an Arrow buffer can be read by the Python process without copying the data. When the JVM writes an Arrow batch to the pipe connecting it to the Python worker, Python receives it as a memory view; pandas wraps it into a Series without copying.

**Columnar layout**: Arrow stores one column at a time. To transfer the `amount` column for 10,000 rows, Arrow writes 10,000 doubles contiguously. A pandas Series is naturally columnar—the correspondence between Arrow column and pandas Series is direct.

**Type awareness**: Arrow has a rich type system that maps cleanly to both Spark's SQL types and pandas/NumPy dtypes. The JVM-to-Python transfer preserves types without conversion.

> **Arrow is like sending a typed spreadsheet instead of a pile of Post-it notes.** Row-at-a-time pickle is one Post-it per value—each requires individual handling, reading, and stacking. Arrow is a spreadsheet with all 10,000 values in one column, already formatted and typed. Python picks it up as-is; there's no re-formatting.

---

## Types of pandas UDFs

There are three main pandas UDF types, each suited to different use cases:

**Scalar pandas UDF**: maps one or more columns to a single output column. Input is one pandas Series per input column; output is one pandas Series. Called once per batch. The output Series must have the same length as the input. This is the direct replacement for a row-at-a-time scalar UDF.

**Scalar iterator pandas UDF**: like the scalar UDF but the function receives an iterator of Series batches and must yield an iterator of result batches. Useful when your function has expensive initialization (loading a model, opening a connection) that you want to do once per task rather than once per batch. The initialization happens before the first `next()` call; all batches in the task are then processed with the already-initialized state.

> **The scalar iterator type is like a chef who prepares the mise en place once per shift, then cooks many dishes.** A plain scalar UDF re-does the mise en place for every single dish (every batch). If your model takes 30 seconds to load, a scalar UDF that processes 1,000 batches per task spends 8 hours loading models. The scalar iterator loads the model once and runs all 1,000 batches on the warm model.

**Grouped aggregate pandas UDF**: used as a custom aggregation function in a `groupBy().agg()` call. Receives a pandas Series for all rows in one group, and returns a single scalar value. Useful for complex custom aggregations that can't be expressed with built-in aggregate functions.

There is also the **grouped map pandas UDF** (now called `applyInPandas`), which receives a pandas DataFrame for each group and returns a pandas DataFrame. Useful for group-level transformations where the output has a different schema or different number of rows than the input.

---

## The execution path

When a pandas UDF is evaluated as part of a task:

1. The task's executor evaluates all upstream expressions for the batch of rows (using regular Spark codegen).
2. The input columns needed by the UDF are converted to Arrow format—one Arrow buffer per column, containing the batch's values.
3. The Arrow buffers are written to the pipe connecting the executor JVM to the Python worker.
4. The Python worker reads the Arrow buffers, wraps them into pandas Series (zero-copy), and calls the UDF function.
5. The function returns a pandas Series.
6. The Python worker converts the result Series to an Arrow buffer and writes it to the pipe.
7. The executor JVM reads the result Arrow buffer and converts it back to Spark rows.

Steps 2 and 7 (Arrow conversion) involve some CPU work for type mapping and null handling, but no data copy for numeric types. Steps 3 and 6 (pipe transfer) involve writing and reading bytes. Steps 4 and 5 (Python execution) involve the UDF logic.

---

## When to use pandas UDFs vs. built-in functions

**Always try built-in functions first**. Built-in Spark functions run entirely in the JVM with codegen; they have no process-crossing cost and can be fused into tight loops with surrounding operators. Pandas UDFs, despite being much faster than row-at-a-time UDFs, still cross a process boundary once per batch and create a codegen break.

**Use pandas UDFs when**:
- Your logic genuinely requires Python libraries (scikit-learn, scipy, custom C extensions) that have no Spark equivalent.
- You need the exact behavior of a specific Python function and can't express it in Spark SQL or the Column API.
- You are doing inference with a Python ML model (scalar iterator pandas UDF is ideal here—load the model once per task, apply it to all batches).

**Prefer scalar iterator for expensive initialization**: if your UDF loads a 500 MB model from disk, a scalar iterator UDF loads it once per task instead of once per batch. For tasks that process many batches, this can be a large speedup.

---

## The Arrow enablement knob

Pandas UDFs always use Arrow—there is no row-at-a-time fallback. Arrow must be available (PyArrow installed in the Python environment). If PyArrow is not installed, pandas UDFs fail at registration time.

Arrow optimization for `toPandas()` and `createDataFrame()` is a separate setting (`spark.sql.execution.arrow.pyspark.enabled`, default true in modern Spark). Enabling it for `toPandas()` can make data collection 5–50× faster for large DataFrames.

---

## Bringing it together

Pandas UDFs replace row-at-a-time Python UDF execution with batch execution using Apache Arrow. Instead of crossing the JVM-Python process boundary once per row, data crosses once per batch—typically 10,000 rows at a time—as a compact Arrow columnar buffer. The Python function receives a pandas Series (zero-copy from Arrow) and returns one, operated on by NumPy-backed pandas routines rather than Python-level loops. Three pandas UDF types cover the main use cases: **scalar** (column-to-column transform), **scalar iterator** (amortize expensive init over all batches in a task), and **grouped aggregate** (custom aggregation per group). The result is Python UDF performance that is often 10–100× faster than row-at-a-time UDFs—fast enough for production ML inference and complex custom logic—while still being slower than native Spark built-in functions, which remain the first choice when the logic can be expressed in SQL.
