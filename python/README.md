# Python & PySpark

Stories about how PySpark bridges Python and the JVM, and how to use the Python API efficiently.

## Stories

- [Two Runtimes, One Job: How PySpark Bridges Python and the JVM](pyspark_bridge.md) — Py4J gateway, Python worker processes, pickle vs Arrow serialization
- [Pandas UDFs: How Arrow Makes Python Functions Fast Enough for Spark](pandas_udfs.md) — scalar, scalar iterator, grouped aggregate, and grouped map pandas UDFs; Arrow batching

## Related stories

- [The UDF Tax: Why User-Defined Functions Are a Black Box to the Optimizer](../udfs/udf_tax.md) — why Python UDFs are expensive and how pandas UDFs improve on them
- [One Row In, Many Rows Out: The Story of User-Defined Table Functions](../udfs/udtfs.md) — Python UDTFs for generating multiple output rows per input
- [The Columnar Fast Lane: How Apache Arrow Speeds Up PySpark](../data-sources/arrow_columnar.md) — the Arrow format that powers fast toPandas() and pandas UDF transfers
- [Bytes on the Wire: How Spark Serializes Data for Tasks and Shuffles](../serialization/bytes_on_the_wire.md) — pickle vs Arrow is the Python-specific chapter of the serialization story
