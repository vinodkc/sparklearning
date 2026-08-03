# Which Files Am I Reading? How Delta Resolves a Snapshot

This is the story of what happens *before* Spark opens a single Parquet footer on a Delta table. The transaction-log story explained *why* Delta can offer ACID on object storage. This story explains *how a read becomes a concrete file list*: pick a version, reconstruct the active set from checkpoint plus JSON commits, apply partition and data skipping, then hand those files to the same Parquet scan path you already know. Understanding snapshot resolution explains time travel, why listing every object in the table directory is the wrong mental model, and where Delta's metadata ends and Parquet's I/O begins.

---

## Delta chooses files; Parquet reads them

A raw Parquet lake and a Delta table can share the same cloud bucket layout—Parquet objects under a root prefix—but they disagree about **truth**.

On a naked Parquet directory, "the table" is whatever files the listing returns. On Delta, **the table at version V** is exactly the set of data files implied by replaying `_delta_log` up to V. Files sitting on disk that are not in that set—failed staging leftovers, vacuum-pending removes, older versions—are invisible to a correct reader.

> **Delta is the guest list; Parquet files are the venue.** Anyone can stand in the building. Only names on the list for tonight's event are part of the party. Snapshot isolation is handing every guest the same printed list when they arrive, then ignoring door traffic that happens afterward.

Once that list exists, each surviving file is ordinary Parquet: footers, row-group skips, vectorized batches—the read-path story. Delta does not replace Parquet scanning; it **selects** what enters the scanner.

---

## Picking the snapshot version

Every query starts by choosing a **version**:

- **Latest** — the highest committed commit the reader can see when the scan is planned
- **Time travel by version** — `versionAsOf`
- **Time travel by timestamp** — map a timestamp to the newest commit at or before that time

That version number freezes the reader's world for the scan. Concurrent commits that land later do not join the file set mid-query. That is snapshot isolation from the reader's point of view: one integer, then a deterministic reconstruction.

---

## Reconstructing the active file set

Naively replaying every JSON commit from version 0 works and does not scale. Delta therefore uses **checkpoints**: a Parquet snapshot of table state (active files, metadata, and related actions) at some version C. The reader:

1. Finds a usable checkpoint at or before the target version (and protocol/features it understands)
2. Loads the checkpoint's file list as the baseline
3. Replays only the JSON commits **after** the checkpoint up through the target version—applying `Add`, `Remove`, and metadata actions in order

The result is the **active data files** for that snapshot, each with path, partition values, size, and **file-level statistics** recorded on the `Add` action when the file was committed.

> **Checkpoints are save points on a long trail.** You do not hike from the trailhead every visit; you drive to the last shelter (checkpoint) and walk the remaining miles (recent JSON commits). Time travel to an old version is choosing an older shelter—or walking farther back when history is still retained.

Protocol versions and table features constrain what a reader may open. A newer writer may produce log actions an older Spark/Delta client cannot interpret; the client fails fast rather than silently ignoring unknown meaning. That negotiation is part of "can I resolve this snapshot?"—not part of Parquet decode.

---

## Skipping files before opening them

With the active list in hand, Delta applies **metadata-level pruning** before Parquet I/O:

**Partition pruning.** Partition values are stored on each `Add` (and in path conventions). Filters on partition columns drop files whose partition values cannot match—same idea as Hive-style directories, driven by the log rather than hoping the metastore listing is complete.

**Data skipping.** Min/max (and related) stats on `Add` actions let Delta drop whole files whose ranges cannot satisfy data filters—even inside a partition. This is the table-format cousin of Parquet row-group skipping: coarser (file grain), cheaper (no footer open yet), and only as good as the stats writers recorded and the clustering of values into files.

Files that survive both layers become the input to Spark's file scan tasks. Then **Parquet** may still skip row groups inside those files. The two layers stack: Delta removes files; Parquet removes row groups and columns.

> **Data skipping is checking the label on the sealed box in the aisle.** If the label says contents are dated June and you need March, you never cut the tape. Opening the box (Parquet footer) is only for boxes that might still hold what you need.

Poor layout—random keys spread across every file—makes file-level stats as weak as unsorted row-group stats. Compaction and clustering policies exist to rewrite files so skipping works; those are maintenance stories. On the read path, you simply inherit whatever the log's `Add` stats claim.

---

## What you do *not* do on a healthy read

A correct Delta read does **not** treat "list all `*.parquet` under the table root" as the source of truth. Directory listing may show:

- Data files not yet committed (or never committed after a failed job)
- Data files already `Remove`d but not yet vacuumed
- Historical files still retained for time travel

Relying on listing bypasses the log and breaks isolation. Spark's Delta source uses the log; ad-hoc tools that glob Parquet under a Delta root are a common footgun.

---

## Hand-off to the Parquet read path

After snapshot resolution, the engine has:

- A schema (from Delta metadata, with column-mapping rules when enabled)
- A list of file paths (and splits)
- Pushed filters/projections for the scan

From there, behavior matches [From Footer to Batches: How Spark Reads Parquet](spark_parquet_read_path.md): open footers, skip row groups, project column chunks, decode vectorized batches. UI metrics that show files read are counting **this** list—not every object in the bucket.

Time travel is the same pipeline with a different version integer. The expensive part of ancient time travel is often **retention and checkpoint availability**, not a different scan algorithm: if the log and data files for that version were vacuumed away, the snapshot cannot be rebuilt.

---

## Bringing it together

A Delta read is a two-act play. **Act one** resolves a **snapshot version** via checkpoint plus JSON replay into an **active Parquet file set**, then prunes that set with **partition values and file-level stats**. **Act two** is the ordinary Parquet scan over whatever survived. Snapshot isolation is the frozen version; time travel is choosing another version; data skipping is dropping files before footers. When scans look wrong, ask which act failed: bad file set (log, vacuum, wrong version) or bad per-file I/O (Parquet layout and pushdown)—Delta owns the first, Parquet the second.
