# WIP Reconciliation on Databricks

SQL-driven reconciliation of a professional services firm's work-in-progress (WIP) balance against the transactional system of record, built on Databricks Delta Lake.

## The problem

A staging report of unbilled WIP by project stopped tying out to the underlying transaction ledger for a reporting period. Finance flagged three symptoms: the WIP total looked overstated as of period end, billed amounts on several projects looked wrong, and a handful of clients were showing up twice with different regions attached.

This project traces each symptom to its root cause and produces a variance bridge that reconciles the staging total to the source system, to the dollar.

## What was built

- Three source files (transactions, WIP staging balance, project dimension) loaded into managed Delta tables on Databricks Free Edition serverless compute
- A raw to clean to reconciliation layered pipeline, with the clean layer classifying records rather than deleting them, so every exclusion stays traceable
- A variance bridge that walks the staging total through each adjustment driver to a reconciled figure
- `DESCRIBE HISTORY` used at key points to show Delta's operation lineage and confirm the pipeline is auditable, not just correct once

## Architecture

```
raw layer        3 managed Delta tables, loaded unmodified from CSV
clean layer       classification and validation, no rows dropped
reconciliation    variance bridge + control tests, ties to source
```

Full three-part naming (`catalog.schema.table`) throughout. No `MERGE` used for the initial load; a deliberate `MERGE` is used later to restore a project missing from the dimension table, which is where the Delta history output is worth reading.

## Key findings

[Port in from your write-up. Suggested framing per finding: what looked wrong, how you proved the cause rather than just observed the symptom, and what it's worth in dollars if you're comfortable publishing exact figures.]

- Duplicate transaction rows inflating WIP
- A billing formula defect, isolated with a control test rather than assumed
- A dimension join defect causing the duplicate-client symptom (period-overlap filter instead of point-in-time)
- A project present in the transaction ledger but absent from both the dimension and staging tables
- [Any unrequested finding you want to keep in, e.g. the anomalous invoice pattern]

## Variance bridge

[Table or summary of the bridge: staging total through each driver to reconciled total. Screenshot or markdown table works well here.]

## Stack

Databricks Free Edition (serverless) · Delta Lake · SQL · managed tables with `DESCRIBE HISTORY` / time travel

## Notebook

See [`notebook/wip_reconciliation.ipynb`](notebook/wip_reconciliation.ipynb) for the full pipeline, or [`writeup/findings.md`](writeup/findings.md) for the analysis without the code.
