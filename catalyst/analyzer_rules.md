# Making Sense of Names: The Analyzer's Resolution Rules

This is the story of Spark's **Analyzer**—the phase of Catalyst that transforms an unresolved logical plan into a fully resolved one. When you write `SELECT amount * 1.1 FROM orders WHERE status = 'complete'`, Spark parses this into a tree of nodes, but those nodes carry nothing but names: `orders` is just a string, `amount` is just a string, `status` is just a string. The Analyzer's job is to give those names meaning: look up `orders` in the catalog, discover that `amount` is a `DoubleType` column at position 2, check that `status` exists, find what `1.1` should be typed as, and verify the whole thing is coherent. If anything doesn't resolve—a table that doesn't exist, a column that isn't in the schema, a function with the wrong argument count—the Analyzer raises an `AnalysisException` before a single byte of data is read.

Understanding what the Analyzer does, which rules it applies, and in what order explains why certain errors happen at "analysis time" rather than at runtime, what `explain()` means when it shows `Unresolved` nodes, and how Spark validates your query before committing to execute it.

---

## The analysis phase in context

Catalyst's pipeline has four stages:

```
SQL string / DataFrame API
        ↓
   Parsed Logical Plan      ← parser output: node names are unresolved strings
        ↓
   Analyzed Logical Plan    ← Analyzer output: all names resolved to typed nodes  ← THIS STORY
        ↓
   Optimized Logical Plan   ← Optimizer output: cheaper equivalent plan
        ↓
   Physical Plan            ← Planner output: concrete execution operators
```

The Analyzed Logical Plan is what `df.explain("extended")` shows as `== Analyzed Logical Plan ==`. You can also access it programmatically:

```python
df = spark.sql("SELECT amount * 1.1 FROM orders WHERE status = 'complete'")
df.queryExecution.analyzed   # the fully analyzed plan
df.queryExecution.logical    # the unresolved parsed plan
```

> **The Analyzer is like an editor fact-checking a manuscript before it goes to print.** The writer (parser) produces a draft with names and references everywhere. The editor checks every reference: "Does this table exist? Does this column belong to this table? Does this function take these argument types?" Any reference that can't be verified causes the manuscript to be rejected before it reaches readers (optimizer).

---

## How the Analyzer works: rule batches

The Analyzer is a subclass of `RuleExecutor`, the same framework used by the optimizer. It holds a list of **rule batches**, each batch containing one or more rules applied in sequence. Each rule is a function `LogicalPlan => LogicalPlan`: it pattern-matches on plan nodes and expression nodes, replacing unresolved nodes with resolved equivalents.

The Analyzer applies its batches in a fixed-point loop: it keeps applying rules until no rule fires (the plan stops changing). Some batches are marked as single-pass (run once regardless); others are fixed-point (repeat until stable). This design allows rules to depend on earlier rules' output—e.g., after `ResolveRelations` attaches schemas, `ResolveReferences` can use those schemas to resolve column names.

The key rule batches (in approximate order):

1. **Substitution** — substitute named references with their concrete values
2. **Resolution** — resolve table names, column names, functions, aliases
3. **Post-resolution** — clean up after resolution (remove sub-aliases, finalize types)
4. **Nondeterministic** — handle non-deterministic expressions
5. **Type coercion** — insert cast nodes where types don't match
6. **Finalization** — final validations and checks

---

## Rule 1: ResolveRelations — looking up tables

**What it does**: replaces `UnresolvedRelation("orders")` nodes with the actual relation backed by the catalog. After this rule fires, the plan knows the table's schema (column names, types, nullability) and where the data lives (files, JDBC URL, etc.).

**Before** (unresolved):
```
'UnresolvedRelation [orders]
```

**After** (resolved):
```
Relation[order_id#1L, customer_id#2L, amount#3, status#4, order_date#5] parquet
```

The `#N` suffix is Spark's unique ID for each attribute—even if two columns share a name in different tables (e.g., `orders.id` and `customers.id`), they get different IDs and are never confused.

**What causes AnalysisException here**: referencing a table that doesn't exist in the catalog:
```python
spark.sql("SELECT * FROM nonexistent_table")
# AnalysisException: Table or view not found: nonexistent_table
```

> **ResolveRelations is like a librarian looking up a book by title.** "Orders" is just a name until the librarian retrieves the actual book from the shelf (catalog) and sees its contents (schema). Only after retrieval does the system know the book exists and what's inside.

---

## Rule 2: ResolveReferences — resolving column names

**What it does**: replaces `UnresolvedAttribute("amount")` nodes with typed `AttributeReference("amount", DoubleType, nullable=true, exprId=#3)` nodes, using the schemas provided by `ResolveRelations`.

**Before**:
```
Filter 'amount > 100
+- Relation[order_id#1L, customer_id#2L, amount#3, ...]
```

**After**:
```
Filter amount#3 > 100
+- Relation[order_id#1L, customer_id#2L, amount#3, ...]
```

The reference `amount` in the filter is now pointing to the exact attribute `amount#3` from the relation—not just a name string.

**What causes AnalysisException here**: referencing a column that doesn't exist:
```python
spark.sql("SELECT nonexistent_col FROM orders")
# AnalysisException: Column 'nonexistent_col' does not exist
```

Referencing an ambiguous column (same name in both sides of a join):
```python
spark.sql("SELECT id FROM orders JOIN customers ON orders.customer_id = customers.id")
# AnalysisException: Reference 'id' is ambiguous, could be: orders.id, customers.id
```

---

## Rule 3: ResolveFunctions — looking up built-in and registered functions

**What it does**: replaces `UnresolvedFunction("upper", args)` with the concrete function expression—either a built-in like `Upper(name#4)`, or a registered Python/Scala UDF.

**Before**:
```
Project ['upper('name)]
```

**After**:
```
Project [upper(name#4) AS upper(name)#10]
```

For aggregate functions: `UnresolvedFunction("sum", [amount#3])` becomes `sum(amount#3)` which is then wrapped in an `AggregateExpression`.

**What causes AnalysisException here**: calling a function that isn't registered:
```python
spark.sql("SELECT my_unregistered_udf(amount) FROM orders")
# AnalysisException: Undefined function: my_unregistered_udf
```

Calling a function with the wrong number of arguments:
```python
spark.sql("SELECT substring('hello')")  # requires 2-3 args
# AnalysisException: Invalid number of arguments for function substring
```

---

## Rule 4: ResolveAliases — propagating SELECT aliases into GROUP BY and ORDER BY

**What it does**: SQL allows using an alias defined in the SELECT clause in the GROUP BY and ORDER BY clauses. `ResolveAliases` substitutes the original expression wherever the alias appears.

**Example**:
```sql
SELECT amount * 1.1 AS adjusted_amount, customer_id
FROM orders
GROUP BY adjusted_amount, customer_id
ORDER BY adjusted_amount DESC
```

**Before analysis**: `GROUP BY adjusted_amount` contains `UnresolvedAttribute("adjusted_amount")` which doesn't exist in the input—it's defined in the output projection.

**After analysis**: `GROUP BY adjusted_amount` is replaced with `GROUP BY (amount#3 * 1.1)`, the actual expression that `adjusted_amount` refers to.

> **ResolveAliases is like a contract that lets you use a nickname for a complex term.** "Adjusted amount" is shorthand for "amount multiplied by 1.1." Wherever the contract says "adjusted amount," the analyzer substitutes the full definition so there's no ambiguity.

---

## Rule 5: ResolveSubquery — handling correlated subqueries

**What it does**: identifies and resolves **correlated subqueries**—subqueries that reference columns from the outer query. The rule finds these outer references, marks them as `OuterReference` nodes, and validates that the referenced columns actually exist in the outer scope.

**Example**:
```sql
SELECT * FROM orders o
WHERE amount > (SELECT AVG(amount) FROM orders WHERE customer_id = o.customer_id)
```

The inner `o.customer_id` reference must be resolved against the outer `orders` alias. `ResolveSubquery` threads the outer scope's attributes into the inner subquery's resolution context, turning the outer reference from unresolved to `OuterReference(customer_id#2L)`.

**What causes AnalysisException here**:
```sql
SELECT * FROM orders WHERE amount > (SELECT AVG(amount) FROM orders WHERE xyz = o.nonexistent)
-- AnalysisException: Resolved attribute(s) missing from child
```

---

## Rule 6: ImplicitTypeCasting (TypeCoercion) — inserting Cast nodes

**What it does**: when an expression mixes types that don't naturally match—an integer literal compared to a long column, a string concatenated with an integer, a string column compared to a numeric literal—the type coercion rules insert explicit `Cast` nodes to make the types consistent.

**Before**:
```sql
SELECT * FROM orders WHERE order_id = '12345'
-- order_id is LongType, '12345' is StringType
```

**After analysis**:
```
Filter (order_id#1L = cast('12345' as bigint))
```

Spark casts the string literal to `bigint` to match the column type. The cast is done once (the literal is constant), so it's efficient. But if the cast is on the column rather than the literal (e.g., `cast(order_id as string) = '12345'`), it blocks predicate pushdown into Parquet—every row must be read and cast before the filter can apply.

Common type coercion rules:
- `WidenSetOperationTypes`: ensures UNION branches have matching types
- `PromoteStrings`: promotes string literals in comparisons to match the column type
- `DecimalPrecision`: normalizes decimal arithmetic to avoid overflow
- `FunctionArgumentImplicitCasting`: inserts casts where function argument types don't match

> **Type coercion is like a translator at a multilingual meeting.** When someone says "12345" in English and the database speaks Long, a translator (Cast node) converts the message before it's delivered. The conversation can happen; it just passes through the translator first.

---

## Rule 7: VerifyAnalysis — the final check

After all resolution rules have run, `VerifyAnalysis` makes a final pass to confirm that:
- No `UnresolvedAttribute` or `UnresolvedRelation` nodes remain in the plan
- All aggregate expressions are in valid positions (no aggregates in WHERE clauses without a GROUP BY)
- Window functions are not nested inside other window functions
- Subquery return types are correct (scalar subqueries return exactly one column)

If any check fails, `VerifyAnalysis` raises an `AnalysisException` with a specific message.

**Common AnalysisExceptions from this phase**:
```python
# Aggregate in WHERE (should be HAVING)
spark.sql("SELECT customer_id FROM orders WHERE SUM(amount) > 100 GROUP BY customer_id")
# AnalysisException: Filter 'sum(amount) > 100' contains aggregate function

# Non-aggregate column in SELECT that is not in GROUP BY
spark.sql("SELECT customer_id, amount FROM orders GROUP BY customer_id")
# AnalysisException: expression 'amount' is neither present in the group by,
# nor is it an aggregate function. Add to group by or wrap in first() if you
# don't care which value you get.
```

---

## Seeing the analysis in action

To observe the Analyzer's work directly:

```python
from pyspark.sql import SparkSession
spark = SparkSession.builder.getOrCreate()

df = spark.sql("SELECT upper(status), SUM(amount) FROM orders GROUP BY status")

# Unresolved plan — just parsed tokens
print(df.queryExecution.logical)

# Analyzed plan — all names resolved to typed AttributeReferences
print(df.queryExecution.analyzed)

# Full extended explain — shows all four stages
df.explain("extended")
```

In the analyzed plan, you'll see attributes like `status#4` instead of just `status`, `upper(status#4)` instead of `upper(status)`, and `sum(amount#3)` as an `AggregateExpression`. The `#N` IDs are how Spark tracks attribute identity throughout the rest of the pipeline.

---

## Bringing it together

The Analyzer is Catalyst's fact-checker. It runs **batches of resolution rules** in sequence on the parsed logical plan, transforming unresolved name strings into typed, identified nodes:

- **ResolveRelations** — `"orders"` → full schema from the catalog
- **ResolveReferences** — `"amount"` → `AttributeReference("amount", DoubleType, exprId=#3)`
- **ResolveFunctions** — `"upper"` → the `Upper` expression class
- **ResolveAliases** — SELECT aliases substituted into GROUP BY and ORDER BY
- **ResolveSubquery** — outer references in correlated subqueries marked and validated
- **TypeCoercion** — `Cast` nodes inserted wherever types need reconciling
- **VerifyAnalysis** — final validation; any remaining unresolved node raises `AnalysisException`

Every `AnalysisException` you've ever seen—"table not found," "column does not exist," "ambiguous reference," "aggregate not allowed in WHERE"—comes from one of these rules failing. The Analyzer ensures that by the time the optimizer and planner see the plan, every name has a meaning, every type is known, and the query is structurally valid.
