---
name: da-integration
description: Designs how the warehouse layers are loaded, the path source → landing/staging → core → mart. Covers ETL versus ELT, full and incremental loads, deletions in the source, batch versus streaming and CDC, source-to-target mapping specs with lineage, pipeline principles (reproducibility, idempotence, every pipeline writes only to its own tables, no circular dependencies, code in Git, a test environment), tests on every run and reconciliation with the source. In review mode it finds edits in staging, pipelines with side effects, missing reconciliation and reports running on operational systems. Use it for "how do we load the warehouse", "ETL or ELT", "incremental load", "we need real time", "mapping spec", "the pipeline fails and nobody knows why", "how to test the loads", "do the counts match the source?", even if the user never says integration. Keywords: integration, ETL, ELT, staging, landing zone, full load, incremental load, CDC, streaming, batch, pipeline, idempotence, mapping, lineage, reconciliation, tests.
---

# Integration: how data gets into the layers

Seventh step of design mode. The layer models say what the data should look like; integration says how
it gets there, reliably and repeatably. Most operational problems of a warehouse (numbers that do not
match after the morning run, a pipeline nobody dares to change) are born here.

Place in the pipeline: after `da-core-model`, before `da-mart`. Quality controls inside the pipeline
(DQ firewall, monitoring) are designed in `dq-guard`; this skill designs the structure of the flow.

## ETL or ELT

| | ETL | ELT |
|---|---|---|
| order | extract → transform on a separate system → load | extract → load into the warehouse → transform with the warehouse's compute |
| advantages | lower storage needs in the warehouse; sensitive data (PII) can be dropped before it ever enters | a copy of the raw data stays in the warehouse; a transformation bug needs no new pull from the source; suits near real time |
| when | strict requirements on what may enter the warehouse | modern platforms with ample storage, a need to recompute |

## Landing and staging

Whatever the load style, store new data first in a dedicated zone (landing zone, staging) where it can
be explored and checked before users see it. Loading raw data straight into the serving layer means
handing users "garbage in, garbage out".

Staging is a technical copy of the source: every row, the original text values, no change of meaning.
Only technical metadata is added: **load timestamp**, **record source**, and hash keys and hash diff for
Data Vault. The purpose is to get data out of the source quickly and take load off the operational
systems. Staging can be persistent (full history) or hold only the current batch.

## Loading

| Type | How | When |
|---|---|---|
| **full load** | empty the target table and load everything from the source | first (initial) load; the source has no timestamp or key for a merge; recovery after an error |
| **incremental** | move only the changes from the last period, target data stays | the source has a reliable timestamp or change flag |
| **streaming / CDC** | capture change events in the source and push them to a queue | events arrive unpredictably, a process is waiting to react |

The traps of incremental loads: there must be a way to detect a change (a timestamp, an updated flag) and
the source must support it. The delta must be merged with the previous state, which means more jobs. And
an increment does not see **deletions**: a record deleted in the source never shows up, so you need a
periodic full pass (for example once a month), or CDC that reports deletes.

Streaming needs a different mindset: a push model, an event queue, compute running all the time. Before
choosing it, check whether anyone really has to react in real time. It is rare. Maintaining business
logic in streaming transformations is much harder than in batch, so compute the warehouse's core metrics
in batch and build a dedicated solution for the specific real-time case.

## Pipeline principles

| Principle | What it means | Why |
|---|---|---|
| pipeline as code | transformations in a repository (Git), changes through a test environment, then production | changes are traceable and reversible |
| reproducibility | the same input always gives the same result | an error can be reproduced and traced; what passed testing works in production |
| idempotence, no side effects | a run changes only its own data, a repeated run breaks nothing | the state of the data does not depend on the order in which jobs finished |
| every pipeline writes only to its own tables | if the customer table needs a figure from another area, a later step combines them | otherwise a table's state depends on what ran after it, and bugs become very hard to find |
| no circular dependencies | a step never reads a table produced by a later step | on a clean test environment the pipeline fails, in production it reads yesterday's data |
| tests on every run | constraints and checks run on every run, not only at deployment | both the code and the source data can change |

The correctness test: the pipeline must run from scratch on a clean test environment where none of the
output tables exist yet. If it fails, it has a hidden dependency on a previous run.

## Mapping spec

For every target table, a source → target map. It is the specification for the build and the lineage for
traceability:

| Target | Source | Transformation | Rule / decision |
|---|---|---|---|
| core.customer.core_customer_id | customer_xref | identity lookup | H-01…H-n in the consolidation catalogue |
| core.customer.country_code | src_b.customer.country | code map `country` | `da-reference-data` |
| core.payment.amount | src_c.payment.amount | text → DECIMAL(12,2), negative = reversal | D-07 |
| core.payment.paid_ts | src_a — | cannot be determined, stays NULL | limitation L-03 |

The last row matters as much as the others: what the input cannot determine is recorded as a limitation,
not filled in with a guess.

## Reconciliation

After every load, prove nothing was lost or duplicated:

```sql
-- counts and totals by layer for the same period and the same meaning of amount
SELECT 'stage' AS layer, COUNT(*), SUM(amount_net) FROM stage.invoices_c WHERE period = :p
UNION ALL
SELECT 'core',  COUNT(*), SUM(amount_net) FROM core.invoice WHERE period = :p
UNION ALL
SELECT 'mart',  COUNT(*), SUM(revenue_net) FROM mart.f_revenue WHERE period = :p;
```

Mind the meaning: compare amounts of the same kind (with VAT against with VAT, after credit notes against
after credit notes). Every source representation of a transaction must be traceable in core through the
XREF; a difference in count between staging and core is fine only when consolidation explains it (two
representations of one invoice) and that is written down.

## Review

| Check | Anti-pattern | Finding |
|---|---|---|
| does reporting read from operational systems? | a dashboard straight on the production database | I: load on operations, no warehouse in between |
| is staging a faithful copy? | manual fixes in staging, business rules at extraction | I: lost audit trail |
| does each pipeline write only to its own tables? | pipeline B fills a column in pipeline A's table | I: side effects |
| can the pipeline be built from scratch? | dependence on tables from the last run | I: hidden dependency |
| do increments handle deletes? | orders deleted in the source live on in the warehouse | I: difference from the source |
| is there reconciliation? | counts and totals are never checked, users find the differences | I: missing control |
| is real time justified? | streaming for a report read once a week | I: needless complexity |
| is the pipeline in a repository and tested? | changes made straight in production | I: irreversible changes |

## Output of integration

A flow diagram source → landing/staging → core → mart, the load type for each source including how
deletes are handled, mapping specs for all target tables linked to decisions and limitations, the list of
tests in the pipeline, and reconciliation queries. Continue with `da-mart`.
