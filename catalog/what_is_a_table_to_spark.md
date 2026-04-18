# What Is a Table to Spark? The Catalog, Metadata, and the Metastore

This is the story of how Spark understands tables. When you write `spark.sql("SELECT * FROM orders")` or `spark.table("orders")`, Spark needs to know: where is the data for `orders`? What columns does it have and what are their types? Is it partitioned, and by what column? What file format is it in? The component that answers these questions is the **catalog**—Spark's metadata system. Understanding the catalog explains what happens when a table "doesn't exist," what a managed versus external table is, how Spark connects to the Hive Metastore, and how database namespaces organize tables.

---

## The catalog: Spark's metadata registry

The **catalog** is the registry of all named data objects that Spark knows about in a session: databases, tables, views, and functions. When you reference a table by name in a SQL query, Spark looks it up in the catalog to resolve its location, schema, partition information, and storage format. Without the catalog, Spark would have no way to answer "what is `orders`?"

In Spark's API, the catalog is accessible as `spark.catalog`. It exposes methods like `listTables()`, `tableExists("orders")`, `listColumns("orders")`, and `createTable()`. Most users interact with it indirectly through SQL DDL statements or through DataFrame operations.

> **The catalog is like the card catalogue in a library.** You look up "orders" in the catalogue and it tells you: the books are in the East Wing, on shelf B, filed under the Parquet format, with columns defined on the index card. Without the catalogue, you'd have to wander the entire library looking for a shelf labelled "orders." The catalog is that reference point—a single lookup that tells Spark exactly where and what every named dataset is.

The catalog operates within a **namespace hierarchy**: there is a current database (default: `default`), and unqualified table names are resolved in the current database. Fully qualified names (`catalog.database.table`) are also supported when multiple catalogs are in play.

---

## The session catalog: temporary objects

The **session catalog** is the in-memory catalog that Spark maintains for the current SparkSession. It contains temporary views and functions—objects that exist for the lifetime of the session and are invisible to other sessions.

When you call `df.createOrReplaceTempView("v_orders")`, you register a temporary view in the session catalog. The view is a named logical plan: when you query `SELECT * FROM v_orders`, Spark substitutes the view's definition and executes it. The view itself holds no data; it is a named alias for a query.

`createOrReplaceGlobalTempView("v_orders")` creates a global temporary view, accessible across sessions in the same SparkContext, under the special database `global_temp`. These are slightly more persistent than session-scoped views but still disappear when the Spark application terminates.

Temporary functions (registered via `spark.udf.register(...)`) also live in the session catalog. They are callable by SQL name but invisible outside the session.

---

## The Hive Metastore: persistent metadata

For persistent tables—tables that survive beyond a single SparkSession—Spark uses an external **metadata store**. The default is the **Hive Metastore**: a relational database that stores table definitions, column schemas, partition metadata, table properties, and statistics.

The Hive Metastore is the universal metadata layer for the Hadoop ecosystem. When you create a table in Spark that should persist, Spark writes its metadata to the Hive Metastore. When another Spark session (or Hive, or Presto, or Trino) later queries the same table, it reads the metadata from the same Metastore and finds the table.

> **The Hive Metastore is like a city's shared property registry.** When you register a building (table), the registry records the address (HDFS path), the floor plan (schema), the building type (file format), and who owns it. Any architect or contractor (compute engine) can look up the registry, find the building, and work with it according to the recorded plan. Multiple firms can use the same building without duplicating the registry entry.

The Metastore stores: the table name and database, the table's schema (column names and types), the storage format (Parquet, ORC, CSV, Delta), the data location (an HDFS path or object storage URI), partition columns and the list of partition values, and table statistics.

---

## Managed vs. external tables

The most important distinction in table metadata is between **managed tables** and **external tables**.

A **managed table** is one where Spark owns both the metadata (in the Metastore) and the data (in a location managed by Spark, typically `spark.sql.warehouse.dir`). When you `DROP TABLE` a managed table, Spark deletes both the Metastore entry and the underlying data files. Managed tables are convenient but dangerous—a `DROP TABLE` is irreversible.

An **external table** is one where Spark owns only the metadata. The data lives at a location you specify. When you `DROP TABLE` an external table, Spark deletes the Metastore entry but leaves the data files untouched.

> **The managed vs. external distinction is like renting vs. owning furniture.** With a managed table, the furniture company (Spark) owns the sofa—when your contract ends (DROP TABLE), they take it back. With an external table, the sofa is yours—the company only has your address (metadata). When you end the contract, they remove your address from their records but don't touch your sofa.

A common pattern: write Parquet files to S3, register an external table pointing at that location, and then query it by name from any engine that reads the Metastore. The data lives in S3 under your control; the Metastore provides the schema and location.

---

## Partitioned tables and the partition catalog

For tables partitioned by one or more columns (a common pattern for data lake tables: partition by `date`, `region`, or `event_type`), the Metastore also tracks the list of known partition values. When you write a new partition, Spark registers it with the Metastore. Queries that filter on partition columns can use the Metastore to skip partitions without even listing the files—the Metastore already knows which partitions exist.

For external tables on object storage, the partition discovery process (listing the storage to find new subdirectories) is often the bottleneck for tables with many partitions. Delta Lake addresses this by recording partition metadata in the transaction log rather than the Metastore.

---

## Table format: what the Metastore records

The Metastore records the **storage format** for each table. For Parquet tables, it records `STORED AS PARQUET`. For Delta tables, it records the table provider as `delta` and the location. When Spark executes a query on a named table, it reads the format from the Metastore and instantiates the appropriate reader.

---

## Pluggable catalogs in Spark 3

Spark 3 introduced the **Catalog Plugin API**, which allows external catalog implementations to be plugged into Spark. Instead of the single Hive Metastore, Spark can be configured with multiple named catalogs. For example:

- `spark_catalog` (the default catalog, backed by the Hive Metastore)
- `unity` (a Unity Catalog implementation for Databricks)
- `iceberg` (an Apache Iceberg REST catalog)
- `glue` (AWS Glue Data Catalog)

With multiple catalogs, you can reference tables with fully qualified names: `unity.finance.orders` resolves in the `unity` catalog. Different catalogs can use different metadata backends, different authorization models, and different table formats—all within the same SparkSession.

> **Pluggable catalogs are like having multiple reference directories in a city.** The main city directory (Hive Metastore) covers most buildings. A specialist directory (Unity Catalog) covers government buildings with additional access controls. A regional directory (Glue) covers the airport zone. Each directory has its own listing format, but all use the same address resolution standard (Catalog Plugin API) so any map reader (Spark) can find addresses in any directory.

---

## Bringing it together

The catalog is the layer that lets Spark refer to data by name. The **session catalog** holds temporary views and functions for the current session. The **Hive Metastore** stores persistent table definitions—schema, location, format, partition list—that survive across sessions and are shared across engines. **Managed tables** have their data owned and deleted by Spark; **external tables** have their data owned by you and leave it intact on drop. Partitioned tables track their partition directories in the Metastore, enabling partition pruning without listing files. The **Catalog Plugin API** allows multiple named catalogs (Hive, Unity, Iceberg, Glue) to coexist in one session, each backing a different namespace. Together, these mechanisms let you write `SELECT * FROM orders` and have Spark know exactly where the data is, what columns it has, and how to read it—regardless of which engine last wrote it or where it lives.
