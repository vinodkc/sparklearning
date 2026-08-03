# Catalog & Tables

Stories about how Spark tracks tables, databases, and metadata — from the session catalog to organization-wide governance.

## Stories

- [What Is a Table to Spark? The Catalog, Metadata, and the Metastore](what_is_a_table_to_spark.md) — session catalog, Hive Metastore, managed vs external tables, pluggable catalogs
- [Beyond the Session Catalog: Unity Catalog and the Governed Lakehouse](unity_catalog.md) — three-level namespace, fine-grained access control, data lineage, external locations

## Related stories

- [Inside a Parquet File: Row Groups, Column Chunks, and Why Spark Loves It](../data-sources/inside_a_parquet_file.md) — the physical format of files that catalog tables point to
- [Folders, Filters, and Skipping: How Layout Makes or Breaks Parquet Scans](../data-sources/parquet_layout_and_pruning.md) — how the directory layout behind a table decides what a scan can prune
- [The Transaction Log: How Delta Lake Brings ACID to Object Storage](../data-sources/delta_lake_transaction_log.md) — how Delta Lake tables maintain ACID guarantees on top of the catalog
- [Which Files Am I Reading? How Delta Resolves a Snapshot](../data-sources/delta_snapshot_read_path.md) — turning a catalog table reference into a concrete, versioned file set
- [Housekeeping the Lake: OPTIMIZE, Z-Order, and VACUUM](../data-sources/delta_optimize_and_vacuum.md) — the maintenance that keeps a catalog-registered lake table fast
- [The DataSource V2 API: How Spark Talks to Storage Systems](../data-sources/datasource_v2.md) — the connector API that pluggable catalogs and external tables use
- [From SQL to a Running Plan: The Catalyst Story](../catalyst/from_sql_to_physical_plan.md) — how the catalog is consulted during analysis to resolve table names and schemas
