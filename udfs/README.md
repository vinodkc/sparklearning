# UDFs & Table Functions

Stories about user-defined functions — their cost, their limitations, and more powerful alternatives.

## Stories

- [The UDF Tax: Why User-Defined Functions Are a Black Box to the Optimizer](udf_tax.md) — Catalyst opacity, Python UDF overhead, deserialization cost, when to use built-ins instead
- [One Row In, Many Rows Out: The Story of User-Defined Table Functions](udtfs.md) — explode and built-in generators, custom Python UDTFs, LATERAL VIEW, table arguments

## Related stories

- [Two Runtimes, One Job: How PySpark Bridges Python and the JVM](../python/pyspark_bridge.md) — Python UDFs pay the full Py4J bridge overhead described here for every row
- [Pandas UDFs: How Arrow Makes Python Functions Fast Enough for Spark](../python/pandas_udfs.md) — the Arrow-based alternative to row-at-a-time Python UDFs
- [From SQL to a Running Plan: The Catalyst Story](../catalyst/from_sql_to_physical_plan.md) — why UDFs are opaque to Catalyst's optimization rules
- [Expressions All the Way Down: How Spark Represents and Evaluates Computations](../catalyst/expression_tree.md) — built-in functions are expression tree nodes; UDFs are not, which is why they can't be optimized
