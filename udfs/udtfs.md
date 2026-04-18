# One Row In, Many Rows Out: The Story of User-Defined Table Functions

This is the story of **User-Defined Table Functions (UDTFs)**—functions that take input and return multiple rows rather than a single value. Most functions in SQL produce one value per input row (scalar functions) or aggregate many rows into one (aggregate functions). UDTFs break both patterns: they accept arguments (which may reference input columns) and return a set of rows for each call. They're the mechanism behind `explode()`, `posexplode()`, `json_tuple()`, and any custom generator function you write. Understanding UDTFs explains why `LATERAL VIEW` exists in Hive SQL, how `explode` and `inline` work under the hood, and how to write a custom UDTF for advanced row-generation logic.

---

## The problem: one-to-many transformations

Relational databases are fundamentally one-row-in, one-row-out for most operations. But data often has nested or repeated structure that needs to be "flattened":

```python
# Orders table has an items column that's an array of products
# [order_1, ["product_A", "product_B"]]
# [order_2, ["product_C"]]

# You want one row per (order, product) pair:
# [order_1, "product_A"]
# [order_1, "product_B"]
# [order_2, "product_C"]
```

A scalar UDF can't do this: it returns one value per row. An aggregate function can't do this: it collapses rows. You need a function that, given the array `["product_A", "product_B"]`, emits two rows: one containing `"product_A"` and one containing `"product_B"`. This is the job of a UDTF (or a generator function, in Spark's terminology).

> **A UDTF is like a receipt printer that can issue one or many receipts per transaction.** Most functions are like a calculator: one input, one output. A UDTF is like a cloning machine: it takes one input (an array, a string to split, a JSON blob) and produces as many output copies (rows) as the data demands.

---

## Built-in generator functions: Spark's first-party UDTFs

Spark ships with several built-in generator functions that serve as UDTFs:

**`explode(array_or_map_col)`**: emits one row per element of an array column, or one row per key-value pair of a map column. Rows from the outer table are duplicated once per element.

```python
from pyspark.sql.functions import explode
df.select("order_id", explode("items").alias("item"))
```

**`posexplode(array_col)`**: like `explode` but also emits a position index (0, 1, 2, ...) for each element.

**`explode_outer(array_col)`**: like `explode` but preserves rows with null or empty arrays (emitting a single row with null for the exploded value), rather than dropping them.

**`inline(array_of_structs_col)`**: expands an array of struct values into multiple rows, with each struct's fields as separate columns.

**`json_tuple(json_col, field1, field2, ...)`**: parses a JSON string and returns specified fields as columns in a single output row (technically a UDTF returning one row per input, but multiple output columns).

**`stack(n, col1, col2, ..., col2n)`**: turns n pairs of columns into n rows.

These built-in generators are first-class citizens in Catalyst's optimizer—they participate in predicate pushdown and other optimization rules.

---

## How generator functions fit in the execution model

Generator functions (built-in UDTFs) are represented in Catalyst's logical plan as `Generate` nodes. A `Generate` node wraps a generator expression and emits its output rows:

```
Generate explode(items), outer=false, qualifier=None, generatorOutput=[item#5]
+- Relation[order_id#1, items#2]
```

During execution, a `GenerateExec` physical operator processes each input row through the generator expression and emits the resulting rows downstream. For `explode`, each input row with an array of N elements becomes N output rows; the other columns from the input are replicated for each emitted row.

> **A `Generate` node is like a paper shredder run in reverse—a paper multiplier.** The input row is the original document; the UDTF is the multiplier rule ("print one copy per item in the items list"). The `GenerateExec` operator runs the machine: one document in, multiple copies out.

---

## SQL syntax: LATERAL VIEW

In SQL, UDTFs are invoked with `LATERAL VIEW` syntax (Hive-compatible):

```sql
SELECT o.order_id, item
FROM orders o
LATERAL VIEW explode(o.items) t AS item
```

`LATERAL VIEW` creates a virtual table `t` from the UDTF output, joined (laterally) to the input row. The result is the same as the DataFrame API's `select(..., explode(...))`. With multiple `LATERAL VIEW`s, you can apply multiple UDTFs, creating a cross-product of all their outputs.

In Spark 3.x SQL, the `TABLE` function syntax is also supported for registered UDTFs:

```sql
SELECT * FROM my_udtf(SELECT * FROM input_table)
```

---

## Writing custom UDTFs in Python (Spark 3.3+)

Spark 3.3 introduced Python UDTFs registered via the `@udtf` decorator:

```python
from pyspark.sql.functions import udtf
from pyspark.sql.types import Row

@udtf(returnType="word: string, count: int")
class WordCount:
    def eval(self, text: str):
        for word, cnt in Counter(text.split()).items():
            yield word, cnt

# Register and use
spark.udf.register("word_count", WordCount)
```

Or use inline:

```python
from pyspark.sql.functions import lit
WordCount(lit("hello world hello"))
```

The class implements:
- `eval(self, *args)`: called for each input row. It `yield`s one or more tuples, each representing one output row.
- Optionally, `analyze(self, *args)`: a classmethod that computes the return type dynamically based on the input types/values (useful for UDTFs whose output schema depends on the input).

> **A Python UDTF's `eval` method is like a machine that accepts one raw material and produces as many finished parts as the material yields.** A woodworker's router (the `eval` method) takes one board (input row) and can produce 3 shelves, 5 frames, or 1 table depending on the board's size and the routing pattern. The number and shape of outputs is determined by the function's logic, not declared upfront.

---

## Spark 3.5: table arguments (polymorphic UDTFs)

Spark 3.5 added support for **table-valued functions with table arguments**: a UDTF that receives an entire table (or a subquery result) as its input, rather than scalar values:

```python
@udtf(returnType="id: int, score: double")
class Normalize:
    def eval(self, input: Row):
        # receives rows from the input table, one at a time
        ...
```

```sql
SELECT * FROM normalize(TABLE(SELECT id, value FROM data))
```

This enables UDTFs that process a set of rows as a group—similar to a grouped map pandas UDF, but expressible as a table function in SQL. It's particularly useful for custom analytical functions that need to see the full dataset (or a sorted portion of it) to produce their output.

---

## Performance considerations

Python UDTFs suffer from the same overheads as Python UDFs: the JVM–Python bridge (Py4J or Arrow), Python GIL, and the overhead of Python function calls per row. For `eval` functions that yield many rows, the net overhead per output row may be lower than for scalar UDFs (amortized over the yield loop), but the per-input-row cost of crossing the JVM–Python bridge still applies.

**When to use Python UDTFs:**
- Custom JSON/XML parsing that yields multiple records per input
- Generating synthetic rows (e.g., date range explosion)
- Custom tokenization or text splitting

**When to consider Scala/Java UDTFs instead:**
- Performance-critical applications where the JVM–Python bridge overhead is unacceptable
- Large-scale production pipelines with millions of rows per second

**Built-in alternatives first:** before writing a UDTF, check whether `explode`, `posexplode`, `inline`, `split`, `from_json`, or a combination of `select` + `explode` covers your use case. Built-in generator functions participate in Catalyst optimization; Python UDTFs are opaque to the optimizer.

---

## Bringing it together

UDTFs (generator functions) solve the one-to-many transformation problem in Spark SQL: one input row can produce zero, one, or many output rows. **Built-in generators** like `explode`, `posexplode`, `inline`, and `stack` handle the most common cases and are fully integrated with Catalyst's optimizer. They appear as `Generate` nodes in the logical plan and `GenerateExec` in the physical plan. **Custom Python UDTFs** (via `@udtf` decorator, Spark 3.3+) allow arbitrary row-generation logic with `yield`-based syntax. **Table arguments** (Spark 3.5+) allow UDTFs to receive and process sets of rows, not just scalar values. SQL access uses `LATERAL VIEW` or the `TABLE()` function syntax. Performance-wise, Python UDTFs carry the JVM–Python bridge overhead; built-in generators and Scala/Java UDTFs do not. So the story of UDTFs is: **receive input → apply generator logic → yield multiple output rows per input → replicate the input row's other columns for each yielded row → emit to the downstream plan.** They're the bridge between Spark's relational model and the fundamentally one-to-many structure of much real-world data.
