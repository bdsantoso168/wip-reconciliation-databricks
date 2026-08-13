# WIP Reconciliation: Findings and Recommendations

## Context

A staging report of unbilled work-in-progress (WIP) by project stopped tying out to the transaction source system for a reporting period. Three symptoms were raised: the WIP total looked overstated as of period end, billed amounts looked wrong on several projects, and a handful of clients were appearing twice under different regions.

This document sets out what the reconciliation found, what it proves rather than assumes, and what should change versus what needs confirmation before it does.

## The bridge

The staging total reconciles to the source system through four adjustments:

```
  Staging total (starting balance)
- Held WIP excluded
- Late postings excluded
- Duplicate transactions removed
+ Project absent from staging, restored
= Reconciled total
```

[Dollar amounts per line pending confirmation this is clear to publish alongside the exercise it came from.]

## What each driver is

**Held WIP.** A subset of transaction rows sits in a status that carries WIP value but no invoice reference at all. That's a different population from work that's been billed and closed, and treating it as ordinary open WIP would overstate the balance.

**Late postings.** Some transactions post after the reporting period ends but relate to work inside it. Whether those belong in a period-end balance is a cutoff policy question, not something a pipeline should decide unilaterally. It's flagged as a driver, not resolved as one.

**Duplicate transactions.** A set of transaction rows exist as exact duplicates, identical across every source attribute. Confirmed by comparing full row content rather than just counting IDs, so the removal is lossless with no row-selection judgment buried in it.

**Restored project.** One project had real, ongoing transaction activity but no row in the dimension table the staging pipeline joins against. An inner join with no master data record on the other side drops the row silently, no error, no flag. It's added back at the project level only, since deriving a client or region for it from transaction data would be a guess dressed up as a fact.

## The two provable defects

**Billing formula.** The staging billed figure only ties when computed as gross WIP relieved minus the adjustment, never plus. That's confirmed with a second, independent test on the adjustment column alone, which ties exactly from source to staging. Together the two tests isolate the defect to a single sign in one expression, rather than leaving it as "the numbers look off."

**Duplicate clients.** The dimension table carries history, and the staging join was matching any dimension version whose validity window overlaps the reporting period, not the single version valid at period end. Only the projects with a version change inside the period fanned into duplicates. A narrower "current version" filter would remove the duplication but stamp today's client and region onto a historical report. The fix has to key off a point-in-time date, or it recurs the next time a dimension version changes mid-period.

## Recommendations

**Fix without further confirmation:** the billing sign error, the dimension join's date logic, and the project restoration. All three are built as general fixes rather than patches to the specific case found this time. The join fix and the restoration both use the same principle: a defect that can recur on the next project or the next mid-period dimension change needs a fix that survives that recurrence, not one scoped to today's single instance.

**Confirm before changing:** the cutoff treatment for late postings, and the held-WIP status definition. Both are policy calls that belong to Finance, not something to resolve by assumption inside the pipeline.

## The sharper point

The restoration and the join fix share the same lesson: point fixes for point failures don't hold. A hardcoded exception for one project handles this month and fails quietly next month for a different one. The same is true of the dimension join, patched narrowly it fixes today's two duplicates and breaks again on the next mid-period change. Both fixes here are written to be correct the next time they run, not just this time.
