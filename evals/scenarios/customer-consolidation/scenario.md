# Scenario: consolidating customers and invoices from three systems

## Task given to the agent

> We are building a warehouse over a web shop, an ERP and a finance system. The files in
> `context/` are extracts. Decide which customers are the same, how invoices from ERP and finance
> relate, and how the result is stored in the warehouse. What is total revenue for May 2026?

Context (invented data): `shop_customers.csv`, `erp_customers.csv`, `erp_invoices.csv`,
`finance_invoices.csv`.

## Grading criteria

| # | Criterion | Pass condition |
|---|---|---|
| 1 | Source ids are not identity | shop `1045` and ERP `1045` are **not** merged (different names and birth dates) |
| 2 | Business key match | shop `1045` and ERP `88213` are merged on name + birth date, with that evidence stated |
| 3 | Conflict blocks the merge | shop `1046` and ERP `88214` are **not** merged: the birth date differs by a year; recorded as DO_NOT_MERGE with the reason |
| 4 | No silent source deduplication | shop `1047` and `1048` are not folded into one identity without an explicit decision; the two-records-of-one-source case is named |
| 5 | No guessing | shop `1049` (e-mail only) stays its own identity; no name is inferred from the e-mail |
| 6 | Hypotheses as a catalogue | merge and no-merge decisions are listed with id, evidence and result |
| 7 | XREF with both constraints | a cross-reference table where every source record maps exactly once and one identity has at most one record per source (or an equivalent rule) |
| 8 | Invoice representations | finance `AR-7781`/`AR-7782` are linked to ERP `F-7781`/`F-7782` as representations, not new invoices |
| 9 | No double counting | May revenue is 16,500 CZK, not 32,200 CZK |
| 10 | Negative test | a check is proposed that proves no identity has a conflict in a stable attribute |

## Results

| Run | Model | Passed | Missed |
|---|---|---|---|
| baseline, 2026-10-22 | Sonnet | 5 of 10, 2 partly (6, 7) | 3 (linked the records with conflicting birth dates), 4 (merged two shop records), 10 (no negative test) |
| with skills (`da-pipeline`, `da-consolidation`) | Sonnet | 10 of 10 | none |

Criteria 3, 4 and 10 separate the runs; the baseline already avoids the id collision and double counting.
