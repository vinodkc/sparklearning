# Expressions All the Way Down: How Spark Represents and Evaluates Computations

This is the story of **expressions**—the fundamental building blocks of every computation in Spark SQL. When you write `amount * 1.1 + fee` in a SQL query or a DataFrame column operation, Spark doesn't store that as a string or evaluate it immediately. It builds a **tree of expression nodes**: a Multiply node wrapping an AttributeReference and a Literal, added to another AttributeReference by an Add node. Every filter predicate, every computed column, every join condition, every aggregate function is an expression tree. Understanding how these trees are built, traversed, and compiled explains how Catalyst optimizes queries and how whole-stage codegen produces tight loops.

---

## What is an expression?

In Spark's codebase, an `Expression` is a tree node that takes zero or more child expressions as inputs and produces a value when evaluated against a row. Expressions form a recursive tree structure: leaf nodes (like `Literal(42)` or `AttributeReference("amount")`) have no children; internal nodes (like `Add`, `Multiply`, `If`) have one or more children.

Every expression has:
- A **data type**: what type of value it produces (`IntegerType`, `StringType`, `BooleanType`, etc.)
- **Nullability**: whether it can produce `null`
- A list of **children** (the sub-expressions it depends on)
- An `eval(row)` method that evaluates the expression against a given row (the interpreted path)
- A `genCode(ctx)` method that generates Java source code for this expression (the codegen path)

> **An expression tree is like a mathematical formula written as a tree diagram.** `(2 + x) * y` written as a tree has `*` at the root, `+` as the left child of `*`, and `y` as the right child. `2` and `x` are the children of `+`. Evaluating the tree means evaluating the leaf nodes first, combining results up through the tree, and reading the answer at the root.

---

## Leaf expressions: the data sources

Leaf expressions have no children. They are the raw material:

**Literal(value, dataType)**: a constant value. `Literal(42, IntegerType)` represents the number 42. `Literal("US", StringType)` represents the string "US". Constant folding optimization replaces `Literal(1) + Literal(2)` with `Literal(3)` at plan time.

**AttributeReference(name, dataType)**: a reference to a named column from the input. `AttributeReference("amount", DoubleType)` means "the column named 'amount' from the input row." After analysis, all column name strings are resolved to specific `AttributeReference` nodes with known types. The analysis phase's job is largely converting unresolved attribute strings into typed `AttributeReference` nodes.

**BoundReference(ordinal, dataType)**: after analysis and physical planning, `AttributeReference` nodes are replaced with `BoundReference` nodes that know the column's position in the row schema (its ordinal). Evaluation of a `BoundReference` is just `row.get(ordinal)`: read the value at position N. This is faster than looking up by name.

---

## Unary and binary expressions: the operations

Most expressions take one or two children:

**UnaryExpression** examples: `IsNull(child)`, `Not(child)`, `Cast(child, targetType)`, `Abs(child)`, `Upper(child)`. Each applies one operation to one input expression.

**BinaryExpression** examples: `Add(left, right)`, `Subtract`, `Multiply`, `Divide`, `EqualTo`, `GreaterThan`, `And`, `Or`, `Like`. Each combines two child expressions.

**ComplexTypeMerging expressions**: some expressions take many children. `Coalesce(exprs)` takes a list of expressions and returns the first non-null value. `In(value, list)` takes a value expression and a list of candidate expressions.

---

## Aggregate expressions: stateful evaluation

Aggregate functions are a special category—they are not evaluated against a single row but against a set of rows. In the expression tree, they appear as `AggregateExpression` nodes that wrap an aggregate function (`Sum`, `Count`, `Average`, `Max`, etc.) and specify the aggregation mode (partial, merge, final).

Aggregate expressions cannot be evaluated with a simple `eval(row)`. Instead, they maintain **aggregation buffers**—mutable state that accumulates as rows are processed. The buffer is initialized (`initialize(buffer)`), updated for each input row (`update(buffer, row)`), and merged across partitions (`merge(buffer1, buffer2)`). At the end, the final result is extracted from the buffer (`eval(buffer)`).

> **An aggregate expression is like a running tally on a whiteboard.** For each row (customer), you update the tally (add to the sum). You can't evaluate "what is the sum?" until all rows have been processed and the tally is complete. The aggregation buffer is the whiteboard; `update` is adding a number; `eval` is reading the total.

---

## How expressions are used in plans

Logical and physical plan nodes use expressions to describe their computations:

- A **Filter** node contains a `condition: Expression` (a Boolean expression that, when evaluated to true, keeps the row).
- A **Project** node contains a `projectList: Seq[NamedExpression]` (the set of expressions defining the output columns).
- A **Join** node contains a `condition: Option[Expression]` (the join predicate).
- An **Aggregate** node contains `groupingExpressions: Seq[Expression]` (the GROUP BY keys) and `aggregateExpressions: Seq[NamedExpression]` (the aggregate functions and projections).

When the optimizer applies rules to the plan, it often transforms expressions within plan nodes. **Constant folding** replaces `Add(Literal(1), Literal(2))` with `Literal(3)` inside Filter nodes. **Predicate pushdown** moves a Filter node's condition expression closer to the scan. **Column pruning** removes `AttributeReference` nodes from Project lists that are not needed downstream.

---

## The interpreted evaluation path: eval()

Every expression implements `eval(row: InternalRow): Any`, which evaluates the expression given a row and returns a value (or null). This path is used in tests, in cases where codegen is disabled, and as a fallback.

Interpreted evaluation works recursively: a `Multiply(left, right)` calls `left.eval(row)` and `right.eval(row)`, then multiplies the results. The recursion mirrors the tree structure.

This recursive path is correct but slow. For each row:
- Multiple method calls (one per node in the tree)
- Virtual dispatch (JVM doesn't know which Expression subclass it's calling)
- Boxing/unboxing of primitives (JVM methods return `Any`/`Object`, not primitive types)
- Null checking at each node

For a query processing 100 million rows with a complex expression tree, the per-row overhead of interpreted evaluation is significant.

---

## The codegen path: genCode()

The fast path uses **whole-stage code generation**. Each expression implements `doGenCode(ctx, ev)`, which generates Java source code (as strings) for evaluating the expression. The generated code for a `Multiply(AttributeReference("amount"), Literal(1.1))` might look like:

```java
double result_1;
boolean result_1_isNull = false;
// evaluate left child (amount at ordinal 2)
double result_left = row.getDouble(2);
boolean result_left_isNull = row.isNullAt(2);
// evaluate right child (literal 1.1)
double result_right = 1.1;
// multiply
if (result_left_isNull) {
  result_1_isNull = true;
} else {
  result_1 = result_left * result_right;
}
```

The generated code for all expressions in a pipeline is **inlined together** by whole-stage codegen into a single loop. Instead of recursive method calls, the resulting code is a flat sequence of primitive operations that the JVM's JIT compiler can optimize aggressively.

> **The difference between `eval()` and codegen is the difference between a human translating a document word-by-word in real time and a machine-translated document printed before the meeting.** Interpreted evaluation translates each expression one at a time while processing each row. Codegen produces the entire translation upfront, prints it, and then reads it at full speed. The upfront cost (compilation) is paid once; the per-row cost is near zero.

---

## Expression trees and Catalyst rules

Catalyst's optimization rules operate by **pattern-matching on expression trees** and **replacing matched subtrees with equivalent, cheaper subtrees**. Each rule is a transformation `Expression => Expression`.

Examples:
- `ConstantFolding`: `if (e.isInstanceOf[Literal]) evaluate e else e.withNewChildren(e.children.map(transform))`
- `NullPropagation`: `if (child.isNullable && isAlwaysNull(child)) Literal.create(null, expr.dataType)`
- `BooleanSimplification`: `And(Literal(true), right)` → `right`

Rules are applied recursively to the entire expression tree in every plan node. The optimizer applies rules in batch passes, repeating until no rule fires. The expression tree is immutable: rules produce new trees rather than modifying existing ones. This makes the optimization process correct (no mutation surprises) and enables full tree traversal.

---

## Bringing it together

Expressions are the atoms of Spark SQL's computation model. Every filter condition, computed column, join predicate, and aggregate function is an **expression tree**: leaf nodes (`Literal`, `AttributeReference`, `BoundReference`) provide data; internal nodes (`Add`, `Multiply`, `EqualTo`, `And`) combine child expressions into richer computations; aggregate expressions (`Sum`, `Count`) maintain mutable state across rows. Plan nodes hold expressions in their fields—filters hold condition expressions, projects hold output expressions. The optimizer **pattern-matches and rewrites** expression trees to produce cheaper equivalents (constant folding, null propagation, boolean simplification). At execution time, expressions are either **interpreted** (recursive `eval()` calls, flexible but slow) or **code-generated** (flat Java code inlined by whole-stage codegen, fast but requires a compilation step). The expression tree model is what makes Spark's optimizer composable, extensible, and powerful: every computation is a tree, and every tree can be inspected, transformed, and compiled.
