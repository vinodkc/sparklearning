# Inside a Parquet File: Row Groups, Column Chunks, and Why Spark Loves It

This is the story of how Parquet stores data and why that storage format is so well-suited to the kinds of queries Spark runs. Parquet is the default file format for Spark SQL, Delta Lake, and most modern data lake architectures—not by accident, but because its internal design aligns precisely with how analytical queries read data: a few columns from many rows, filtered by predicates on a handful of fields. Understanding Parquet's structure explains why Spark can scan a 500 GB table in seconds when the query touches two out of fifty columns, and why adding the right filter can cut I/O by 99%.

---

## Row-oriented vs. columnar: the fundamental difference

Traditional row-oriented storage (think CSV, Avro, JSON) stores each record contiguously: all the fields of row 1, then all the fields of row 2, and so on. This is great for workloads that read complete rows (OLTP: fetch me the record for user 42), but terrible for analytics: if you want the `amount` column from 100 million rows, you have to read every byte of every row—including all the fields you don't need—just to extract that one column.

**Columnar storage** flips the layout: all values of column 1 (contiguously), then all values of column 2, and so on. To read the `amount` column from 100 million rows, you read only the bytes belonging to `amount`. The other 49 columns are not touched.

> **Row-oriented storage is like a filing cabinet full of employee folders.** Each folder has the employee's name, address, salary, job title, start date, and 47 other fields. To find everyone's salary, you open every single folder, flip to the salary field, and then close it. Columnar storage is like having a separate folder that contains *only* salaries for all employees—you open one folder and read every salary in sequence without touching any other data.

Parquet is a **hybrid**: it combines columnar layout with row-group chunking so that both column projection and predicate-based row skipping are efficient.

---

## The structure of a Parquet file

A Parquet file is organized in four layers, from outermost to innermost: **file → row groups → column chunks → pages**.

**Row groups** are the top-level division. A Parquet file is split into row groups, each containing a horizontal slice of the rows—say, 128 MB of data. A file with 1 billion rows might have 100 row groups of 10 million rows each. Row groups are the unit of parallelism: each Spark task typically reads one row group.

**Column chunks** are the columnar division within a row group. For a row group containing rows 0–9,999,999, the column chunk for `amount` contains the values of `amount` for all those rows, stored contiguously. Each column chunk has its own metadata: the minimum and maximum value in that chunk, the count of nulls, and optional statistics.

**Pages** are the smallest subdivision within a column chunk. Each page is independently compressed and encoded. Pages are the unit of decompression.

At the end of the file sits the **file footer**: a metadata section that lists all row groups, their byte offsets in the file, the column chunks within each row group, and the statistics for each column chunk. Reading the footer—a few kilobytes—gives Spark a complete map of the file's layout without reading any data pages.

> **The file footer is like the table of contents plus the index of a textbook.** You read those two pages first and learn exactly where "Chapter 7: Column Chunks" starts and what pages cover "amount > 1000." You don't read the whole book to find out; the index tells you, and you jump directly to the relevant pages.

---

## Projection pushdown: reading only the columns you need

When Spark executes a query that touches only certain columns, it reads only the column chunks for those columns. This is **projection pushdown**: the column selection is pushed down into the file reader, so bytes belonging to unselected columns are never read from disk.

The mechanics are straightforward: the footer tells Spark the byte offset and length of each column chunk. To read column `amount` from row group 3, Spark seeks to that column chunk's byte range and reads it, skipping all other column chunks in that row group entirely. For a table with 50 columns where your query uses 3, projection pushdown reduces I/O by roughly 94%.

This is why `SELECT *` is expensive in Spark SQL. `SELECT col1, col2, col3` lets the reader skip 47 column chunks per row group. The difference is not just I/O—it also reduces the memory needed to hold decoded data and the CPU cost of decoding columns you'll never use.

---

## Predicate pushdown: skipping row groups you don't need

The column chunk statistics—min, max, null count—stored in the file footer enable a second major optimization: **predicate pushdown** (also called **row group filtering**).

When your query has a filter like `WHERE amount > 1000`, Spark's Parquet reader reads the footer and examines the min/max statistics for `amount` in every row group. If a row group's maximum `amount` is 800, then no row in that row group can satisfy `amount > 1000`—the entire row group can be **skipped**. Spark never reads those column chunks at all.

> **Predicate pushdown is like checking each chapter's summary before deciding whether to read the chapter.** If the chapter is about 18th-century French architecture and you're researching 20th-century modernism, the summary tells you to skip it. You don't read the chapter—you flip to the next one. The row group statistics are those chapter summaries.

For predicate pushdown to be effective, the data should be **sorted or clustered** by the filter column—so that row groups have non-overlapping value ranges. A table sorted by `date` will have row groups with tight, non-overlapping date ranges, and a filter on `date` will skip almost all row groups.

---

## Encoding: making column data smaller and faster

Within a column chunk, Parquet applies **encoding** to reduce size before compression.

**Plain encoding** stores values as-is.

**Dictionary encoding** is the most impactful for low-cardinality columns (like `status`, `country`, `category`). A dictionary page at the start lists all distinct values. Each data value is stored as a small integer index into the dictionary instead of the full string. A column with 10 million rows but only 50 distinct strings stores 10 million small integers instead of 10 million strings.

> **Dictionary encoding is like replacing every occurrence of a person's full name in a document with their employee number.** "Maria Rodriguez Martinez" (20 bytes) becomes "42" (2 bytes) everywhere in the document. A lookup table at the front maps 42 → "Maria Rodriguez Martinez." For a document that mentions her 10,000 times, the savings are enormous.

**Run-length encoding (RLE)** compresses runs of repeated values. **Delta encoding** stores the differences between consecutive values. Encoding is applied before compression—the combination of encoding and compression means a Parquet file is often 5–10× smaller than an equivalent CSV.

---

## Spark's vectorized Parquet reader

Spark includes a **vectorized Parquet reader** that reads column data in **batches of rows** (typically 4096) rather than one row at a time. Instead of decoding row 1, passing it to the operator above, then decoding row 2, the vectorized reader fills a column vector for 4096 rows at once, then passes the whole batch up.

This aligns with how CPUs work: operating on arrays of the same type is cache-friendly, SIMD-vectorizable, and avoids the per-row method call overhead of the Volcano model. Filtering can be applied to a whole batch at once—check all 4096 values of `amount` against `> 1000` in a single tight loop.

---

## Bloom filters: precise row-group skipping for point lookups

Min/max statistics work well for range filters and sorted data but not for point lookups on unsorted columns. If you're looking for `user_id = 'abc123'` and the column isn't sorted, every row group's min/max range will include all possible user IDs—statistics can't help.

Parquet supports **bloom filters** at the column chunk level: a probabilistic data structure that answers "is this value definitely not in this column chunk?" with no false negatives. For a point lookup, Spark reads the bloom filter for each row group, checks whether the target value could be present, and skips row groups where the bloom filter says "definitely not present."

> **A bloom filter is like a nightclub bouncer with a list.** The bouncer can say with certainty "that name is NOT on the list" (skip the row group). But the bouncer occasionally says "that name might be on the list" even when it isn't (a false positive)—in that case you still go inside to check. False negatives are impossible; occasional false positives are acceptable.

---

## Bringing it together

Parquet is a columnar, row-group-partitioned format built for the access patterns of analytic queries. The file is organized into **row groups** (horizontal slices of rows), each containing **column chunks** (vertical slices by column), each split into **pages** (the unit of compression). The **file footer** holds the complete map of all row groups and column chunk statistics, readable in one small I/O. **Projection pushdown** lets Spark read only the columns the query uses, skipping all others at the byte level. **Predicate pushdown** uses min/max statistics to skip entire row groups that can't contain matching rows. **Dictionary and delta encoding** shrink data before compression. The **vectorized reader** processes data in column-oriented batches for CPU efficiency. **Bloom filters** extend row-group skipping to point lookups on unsorted high-cardinality columns. Together, these mechanisms mean that a Parquet-backed query reading 2 columns from a 500-column table with a selective filter may read less than 1% of the file's bytes—making Parquet the format of choice for fast analytics on large datasets.
