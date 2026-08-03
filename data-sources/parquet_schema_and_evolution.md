# Columns That Change: Schema, Nested Types, and Evolution in Parquet

This is the story of what Parquet *means*—the schema that turns bytes into columns—and what happens when that meaning drifts over time. Anatomy and layout explain how data is packed and skipped; schema explains why two files in the same directory can disagree about types, why nested structs need their own projection story, and why a "simple" `mergeSchema` read can suddenly stall on a metadata storm of footer opens. Understanding schema evolution is how you keep a growing lake readable without surprising every job that opens it.

---

## Schema lives in the file

Every Parquet file carries its own schema in the **footer**: field names, types, nullability, and nested structure. Spark does not invent the columns from thin air when it opens a file; it reconciles the file schema with what the reader asked for—table metadata, an explicit schema, or a schema inferred from a sample of files.

That design is powerful: files are self-describing and can be read outside Spark. It is also the source of evolution pain. A directory of "the same table" may contain files written months apart with **different footers**. Readers must decide how to unify those footers into one logical schema for the query.

> **A Parquet footer schema is like the ingredients list printed on each jar in a pantry.** Most jars agree. Occasionally someone restocks with a new recipe—an extra spice, a renamed label, a different bottle size. Opening the pantry as one meal means deciding whether those jars still count as the same dish.

Catalogs and table formats add a **declared** table schema on top. Raw Parquet directories lean harder on what is actually in the files. Either way, physical files remain the ground truth of what can be decoded.

---

## Nested types: structs, arrays, and maps

Parquet is not limited to flat columns. It can store **structs** (nested fields), **arrays** (repeated values), and **maps** (key/value repeated groups). Nested data is still columnar: a struct's fields are stored as separate column paths, so projection can reach into a nested field without reading siblings.

That matters for wide event payloads. A query that needs `user.id` and `user.country` should not pay to decode `user.preferences.*` if the reader supports nested projection. Arrays and maps are heavier: repetition levels and definition levels track which values belong to which row and which nested slots are null or missing. Decoding nested columns costs more CPU than flat primitives, and some optimizations (statistics, bloom filters, vectorized paths) are richer for top-level primitives than for deep nesting.

> **Nested Parquet is like a filing cabinet where each drawer has folders, and each folder has subfolders.** Columnar layout still lets you pull only the "address → city" slips from every folder without emptying the whole drawer—but the labels and nesting depth decide how fiddly that pull is.

Practical guidance: prefer **flattening hot filter and join keys** to top-level columns when those fields drive pruning and joins; keep deep nesting for payload that is rarely filtered and mostly projected as a blob or a few subfields.

---

## Compatible evolution: what usually works

Writers and readers generally tolerate a few evolutionary moves when older files must remain readable:

**Adding columns.** New files include the field; old files omit it. Readers treat missing fields as **null** (or a default, depending on context). This is the safest everyday change.

**Widening nullability.** Making a field nullable is usually fine for readers that already handle nulls.

**Type widening in limited cases.** Some promotions are accepted by Spark's Parquet reader (for example, certain integer widenings), but relying on silent type changes is fragile—especially across Spark versions and when statistics/pushdown expect one physical type.

**Reordering fields.** Parquet binds by name (and nested path), not by position in the way a CSV might. Reordering fields in the writer schema is typically safe for named reads.

These moves keep a directory readable under a unified schema **without rewriting history**, as long as every reader agrees on the merge rules.

---

## Breaking or surprising changes

Other changes look small in application code and large on disk:

**Renames.** Parquet has no first-class "rename" in raw files. A rename looks like **drop old name + add new name**. Old files still have `customer_name`; new files have `client_name`. A merged schema may show both, with nulls on opposite sides—unless you rewrite files or use a table format with column mapping.

**Type changes that are not widenings.** String to int, int to string, timestamp representation changes—readers fail, silently mis-read, or disable optimizations. Prefer add-column + backfill over in-place type mutation.

**Tightening nullability.** Declaring a field non-nullable when older files contain nulls invites failures or incorrect assumptions.

**Nested reshapes.** Moving a field from top-level into a struct (or the reverse) is a rename-plus-reparent for the physical column path. Projection and pushdown references must follow the new path.

> **Renaming a column without rewriting files is like changing the street name on new maps only.** Delivery drivers using old addresses still look for the old street. You either reprint every map (rewrite files) or keep both names on the mailbox (dual fields / mapping layer).

---

## mergeSchema and the cost of inference

When Spark reads a directory of Parquet files, it may **infer** schema from a subset of files for speed. If later files added columns, inference can miss them—unless you enable **schema merging** (`mergeSchema`), which inspects more (or all) footers and unifies fields.

Merging is correct and expensive at scale: listing and opening footers across hundreds of thousands of files turns "just read the table" into a metadata storm on the driver or planning path. Teams discover this when a lake grows quietly for a year and one day every job starts with a long pause before the first scan task runs.

Better patterns:

- Maintain a **stable declared schema** (catalog / table format) and make writers conform.
- Prefer **additive** evolution; avoid relying on directory-wide merge for hot tables.
- Compact or rewrite when renames and type fixes must become physical reality.

---

## Pushdown and types: silent performance traps

Even when a read "works," schema mismatches can **disable predicate or projection pushdown**. Filters that cast a column to another type, or compare a partition field through a function Spark cannot push, fall back to reading more bytes and filtering in the engine. Timestamp storage quirks (historical INT96 vs modern INT64 timestamp types) are a classic source of "the filter is in the SQL but the scan still touches everything."

So schema is not only correctness—it is part of the **performance contract** with the layout and write stories. Clean types and stable names make partition pruning and row-group skipping actually attach to the columns you think you are filtering.

---

## Bringing it together

Parquet files are self-describing: each footer carries a schema that may **evolve** as writers add fields, nest payloads, or drift from older conventions. **Nested types** remain columnar but complicate projection and CPU cost; hot keys often deserve top-level columns. **Additive** changes are the safe default; **renames and type rewrites** are physical events, not label swaps. **`mergeSchema`** can unify drifted directories at a steep metadata cost—table-level declared schemas scale better. Keep evolution boring, and the anatomy, write, and layout machinery keep delivering cheap scans; let footers diverge unchecked, and every query pays for archaeology before it reads a single useful page.

For how Spark turns schemas and files into batches, continue with [From Footer to Batches: How Spark Reads Parquet](spark_parquet_read_path.md). For renames without rewriting history at table-format level, see Delta [column mapping](delta_format_features.md).
