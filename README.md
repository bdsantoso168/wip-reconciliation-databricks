# WIP Reconciliation on Databricks

SQL-driven reconciliation of a professional services firm's work-in-progress (WIP) balance against the transactional system of record, built on Databricks Delta Lake.

## The problem

A staging report of unbilled WIP by project stopped tying out to the underlying transaction ledger for a reporting period. Finance flagged three symptoms: the WIP total looked overstated as of period end, billed amounts on several projects looked wrong, and a handful of clients were showing up twice with different regions attached.

This project traces each symptom to its root cause and builds a variance bridge that reconciles the staging total to the source system.

## What was built

- Three source files loaded into managed Delta tables on Databricks Free Edition serverless compute, staged through a volume landing zone since outbound network access is restricted on Free Edition
- Raw tables kept unmodified through the whole pipeline, so any defect found downstream can be shown to originate in the source data rather than being introduced by a transformation
- A profiling pass that establishes grain and key uniqueness on all three tables before any assumption gets applied, since a table that doesn't hold the grain it claims produces wrong answers to every question asked of it afterward
- A clean layer that classifies records instead of deleting them, so every excluded row stays traceable back to the line in the bridge that excludes it
- Two independent control tests that prove a billing formula defect rather than infer one
- A MERGE-based restoration for a project the staging pipeline silently dropped, verified against Delta's own operation history rather than asserted

## Architecture

```
raw layer         3 managed Delta tables, loaded unmodified from CSV
profiling          grain and key uniqueness checks before any transform
clean layer        classification and validation, no rows dropped
reconciliation      billing control tests + variance bridge
restoration         MERGE recovering a project the staging join dropped
```

Full three-part naming (`catalog.schema.table`) throughout. `DESCRIBE HISTORY` used at the load and again after the restoration merge, so the pipeline's lineage is checkable, not just asserted correct.

## Key findings

- **Duplicate transaction rows.** A handful of transaction IDs appeared twice, byte-identical across every source column. Confirmed via full attribute comparison, not just an ID count, so the dedup is lossless with no row-selection judgment call buried in it.
- **Billing formula defect, proven rather than assumed.** One control test showed the staging billed figure only ties when computed as gross minus adjustment, never gross plus adjustment. A second, independent test confirmed the adjustment column itself ties exactly from source to staging. Together they isolate the defect to a single sign in one transformation, not a vague "the numbers look off."
- **Duplicate clients traced to a join defect, not random duplication.** The dimension table uses slowly changing history, and the staging join was matching any dimension version whose validity window overlapped the reporting period, instead of the single version valid at period end. Only the two projects with an in-period version change fanned out. Filtering on "current version" would have fixed the duplication but stamped today's attributes onto a historical report. The fix has to be a point-in-time predicate, not a broader filter.
- **A project invisible to the report.** One project had real, ongoing transaction activity but no row in the dimension table, and the staging pipeline inner-joins to that dimension, so it dropped silently with no error and no flag. Restored with a MERGE keyed to any missing project, not a hardcoded fix for the one found this time, since an inner join with no master data record will drop the next one just as quietly.
- **A scoping question kept out of the fix list.** Some transactions post after the reporting period ends. That's a real population, but whether it belongs in or out of a period-end balance is a policy call for Finance to confirm, not something to decide inside the pipeline.

## Variance bridge

Structure of the bridge, in order applied:

```
  Staging total (starting balance)
- Held WIP excluded
- Late postings excluded
- Duplicate transactions removed
+ Project absent from staging, restored
= Reconciled total
```

[Dollar amounts per line pending confirmation this is clear to publish alongside the exercise it came from. Notebook has the full figures if you're ready to add them.]

## Delta Lake lineage

`DESCRIBE HISTORY` run at two points rather than sprinkled throughout: once on the initial load, once after the restoration MERGE. The merge is the one worth reading, since it's an operation with real metrics rather than a bare create. It records one row inserted and zero rows updated, confirming the restoration added the missing project without silently altering any existing balance. The pre-restoration version stays queryable through time travel, so the report as originally produced can still be compared against the corrected version without a second pipeline.

## Stack

Databricks Free Edition (serverless) · Unity Catalog · Delta Lake · SQL · managed tables with `DESCRIBE HISTORY` / time travel

## Notebook

Full pipeline, including every query referenced above, is in [`notebook/wip_reconciliation.ipynb`](notebook/wip_reconciliation.ipynb). The analysis without the code is in [`writeup/findings.md`](writeup/findings.md).
