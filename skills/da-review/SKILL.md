---
name: da-review
description: Review of an existing data architecture (data warehouse, data marts, lake, integration pipelines, reporting). Reconstructs the actual state instead of the documented one, walks the staging, core and mart layers with a checklist, backs every finding with a query, sets severity and root cause, and routes each finding to the pipeline step that fixes it. Use it when a solution exists and the question is "what is wrong with it": "review our data warehouse", "why don't the numbers match between reports", "audit our data architecture", "we have tech debt in our pipelines", "we want to migrate the old warehouse, what do we do with it", "assess my colleague's design", even if the user never says review. Keywords: architecture review, audit, as-is, technical debt, single source of truth, inconsistent reports, legacy DWH, migration, anti-pattern.
---

# Review: what is wrong with the current architecture, and why

Entry point of review mode. The result is a finding catalogue with evidence, priority and a link to
the step that fixes each finding. A review does not design a new architecture until it is clear what
is wrong and why. Remediation design comes after prioritisation, and only for the affected layers.

Place in the pipeline: first in review mode. The checklist draws on every step skill; each has its own
"Review" section.

## Step 1: reconstruct the actual state

Documentation describes what was meant to be built. A review judges what was built. Reconstruct:

| What | How | Why |
|---|---|---|
| sources and how they are read | load configuration, logs, metadata queries | an undocumented source is a frequent cause of differences |
| layers and their tables | `information_schema`, naming conventions | checks whether staging, core and mart really exist separately |
| data flows from source to report | lineage from the tool, otherwise by hand from transformation code | without the flow you cannot find where a difference arises |
| who uses what | list of reports and their users, query logs | unused parts are candidates for removal; use outside the warehouse signals a gap |
| decisions and their rationale | decision records, if any | a missing rationale is a finding in itself |

But start with context (`da-context`): without a problem and goals you cannot tell whether the
architecture is wrong or just different from what the reviewer would have built.

## Step 2: the main test, a single version of the truth

The main reason a data warehouse exists is a single version of the truth: the same question gets
everyone the same answer. The typical state without it: finance, sales and marketing each have their
own sales numbers and none of them add up. The risk is also inside the warehouse: once it loses track
of what already exists and values get recalculated, conflicting reports come back and users stop
trusting it.

So take 2–3 key measures from `da-context` and compute them along every path that exists in the
warehouse:

```sql
-- the same measure from two marts and from core, same period
SELECT 'mart_sales' AS path, SUM(revenue) FROM mart_sales.f_sales WHERE sale_date BETWEEN :from AND :to
UNION ALL
SELECT 'mart_finance', SUM(income) FROM mart_finance.f_income WHERE booking_date BETWEEN :from AND :to
UNION ALL
SELECT 'core', SUM(amount) FROM core.invoice_line WHERE delivery_date BETWEEN :from AND :to;
```

A difference is not automatically an error, but every difference must be explained by a definition
(with or without cost of goods, with or without VAT). An unexplained difference is a high-severity
finding. The classic cause in the bottom-up approach: a field called "revenue" in two marts, one
including cost of goods and one without.

## Step 3: checklist by layer

| Layer | Check | Anti-pattern | Fix |
|---|---|---|---|
| sources | are all sources mapped and analysed (data and technology)? | a source connected "temporarily" without analysis, different data types across platforms | `da-inventory` |
| sources | does reporting read from operational systems? | reports straight on OLTP overload operations | `da-integration` |
| staging | is staging a faithful copy of the source? | edits in staging, business logic already at extraction | `da-integration` |
| core | is there an integrated layer, or just copies of source tables? | normalised source tables imported one to one and reports pointed straight at them | `da-core-model` |
| core | does every entity have one identity across sources? | two records of the same person from two systems with no link | `da-consolidation` |
| core | are code lists unified? | the same status coded differently in every source, mapping done in reports | `da-reference-data` |
| core | is history managed (valid from/to)? | values overwritten, historical reports change retroactively | `da-core-model` |
| integration | does each pipeline write only to its own part? | a pipeline changes another pipeline's tables, circular dependencies | `da-integration` |
| integration | can the load be rebuilt from scratch on a clean environment? | dependence on tables from the previous run | `da-integration` |
| integration | are there tests and reconciliation with the source? | row counts and totals are never checked | `da-integration` |
| mart | does each fact have a clear, enforced grain? | mixed levels of detail in one table, double counting | `da-mart` |
| mart | are shared dimensions actually shared (conformed)? | every mart has its own calendar and its own customer | `da-mart` |
| mart | is the model internally consistent and explainable? | ad hoc denormalisation with no method (neither Kimball nor Inmon) | `da-approach`, `da-mart` |
| governance | does every key term have a definition and an owner? | "revenue" means something different in every department | `da-governance` |
| governance | can you trace where a report number comes from? | the measure is calculated inside the reporting tool | `da-governance` |
| governance | is personal and sensitive data controlled? | the solution ignores data protection requirements | `da-governance` |
| operations | are costs, scalability and performance known? | expensive maintenance, growing volume slows down loads | `da-approach` |

Record positive results too: "the sales fact grain is one receipt line, enforced by a unique key" is as
valuable as a defect, because it tells the remediation what not to break.

## Evidence queries

Every finding needs a query with a count and a universe. The most common ones:

```sql
-- grain: is the combination that should be unique actually unique?
SELECT receipt_id, line_no, COUNT(*) FROM mart.f_sales
GROUP BY receipt_id, line_no HAVING COUNT(*) > 1;

-- identity: how many business keys have more than one record in core
SELECT COUNT(*) FROM (SELECT bk_customer FROM core.customer GROUP BY bk_customer HAVING COUNT(*) > 1) d;

-- referential integrity: facts without a dimension (anti-join, not NOT IN)
SELECT COUNT(*) FROM mart.f_sales f LEFT JOIN mart.d_product p ON p.product_key = f.product_key
WHERE p.product_key IS NULL;

-- reconciliation: count and total in the source against core, same period
SELECT COUNT(*), SUM(amount) FROM stage.invoices WHERE invoice_date BETWEEN :from AND :to;
SELECT COUNT(*), SUM(amount) FROM core.invoice   WHERE invoice_date BETWEEN :from AND :to;
```

Do not measure the quality of the data itself here (completeness, formats, duplicates inside a source);
delegate that to `dq-profiler` and `dq-validator`. An architecture review asks whether the structure and
flows allow correct numbers, not whether every value is right.

## Step 4: cause, priority, fix

For each finding, separate the symptom from the cause. "The revenue report does not match" is a
symptom; the cause is, for example, missing customer consolidation or an undefined grain. Fix the cause,
or the symptom comes back somewhere else.

Set priority from severity (scale in `da-pipeline`) and from how many other findings rest on the same
cause. One missing consolidation can explain five mismatched reports. Such a cause goes first, even if
it is laborious.

A review also judges sustainability. Early on you iterate fast; as the complexity of data, processes and
pipelines grows, change slows down and technical debt piles up. Signals: fixing one pipeline breaks
another, nobody knows how revenue is calculated, a change takes weeks. Typical reasons to migrate an old
solution are high maintenance cost, limited scalability and performance. But a migration is a decision
with its own risks (deadline, missing team skills, complex logic to port). Record it as an option in the
`da-pipeline` decision format, not as a given.

## Output of the review

```
1. Review context: purpose, scope, what was and was not reviewed, snapshot and date
2. Actual state: sources, layers, flows (diagram), users
3. Single-version-of-truth test: measures, paths, differences and their explanation
4. Finding catalogue: table in the da-pipeline format (ID, layer, finding with evidence,
   severity, impact, fix = skill), including positive findings
5. Causes: "symptom → cause" tree, findings grouped by cause
6. Recommendations and order of remediation, larger changes as options in the decision format
7. Not verified: what could not be checked and why
```

Continue with the fixing step for the highest-priority findings. For a layer being reworked, switch to
that skill's design mode.
