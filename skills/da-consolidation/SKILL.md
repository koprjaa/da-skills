---
name: da-consolidation
description: Consolidates identities across sources in the integrated layer of a data warehouse. States merge hypotheses (merge / do not merge) backed by evidence from the input, defines the business key, builds XREF tables from source record to core identity, separates master entities from transactions and guards against the critical errors (merging across a conflict, two records of one source in one identity, double counting). In review mode it finds duplicate identities and double counting in the warehouse. Use it when "the same customers are in several systems", "how do we join products from the web shop and the ERP", "we need a single customer", "XREF", "master data", "the invoice is in both systems", "the numbers are doubled", even if the user never says consolidation. Keywords: consolidation, master data, MDM, identity, business key, XREF, cross-reference, golden record, merge, double counting.
---

# Consolidation: one identity across sources

Fourth step of design mode and the most common source of critical errors. The integrated layer (core)
exists to join data about the same thing from different systems, which means deciding when two records
are the same thing. That decision is made with a hypothesis and evidence, not by similarity.

Place in the pipeline: after `da-approach`, alongside `da-reference-data`, before `da-core-model`. The
technical side of matching (match code, blocking, thresholds, survivor selection, golden record) is in
`dq-deduplicator`. This skill makes the architectural decision: what an identity is, where it lives and
how it is traced.

## What gets consolidated

Consolidation makes sense where you join things that are clearly alike: a person held in several
systems, each with a different set of attributes, ends up in one table of persons in the target store.
Typically master data: customer, product, store, supplier. Master data is shared across the
organisation, belongs to the organisation rather than to one application, and its changes should be
controlled.

Transactions are consolidated differently from master data. An order in the web shop and an order at
the till are two different events, even with the same customer and amount. Merging them would be wrong.
But an invoice in the sales system and the same invoice in the accounting system are **two
representations of one event**, and if both are loaded as separate records they are counted twice.

## Business key

Source ids are valid only inside their own system. Customer 1045 in the web shop and 1045 in the ERP
have nothing to do with each other. Identity across sources is carried by the **business key**: an
attribute or combination that makes sense outside the system and identifies the thing in reality.

| Entity | Strong business key | Weaker combination | Not enough |
|---|---|---|---|
| person | identifier with a checksum (national ID) | first name + last name + date of birth + address | name alone, e-mail alone for shared mailboxes |
| company | company registration number | name + registered address | name alone |
| product | EAN / GTIN, manufacturer part number | name + brand + variant | one system's internal SKU |
| store | branch code in a shared registry | address | name ("Downtown") |

When the business key cannot be built from the input (an attribute is missing in one source), do not
fill it in by guessing. Do not invent a missing date of birth, do not infer a name from an e-mail
address. The record stays a separate identity and the limitation is written down.

## Merge hypotheses

Record every merge or no-merge decision as a hypothesis:

| ID | Entity | Candidate | Hypothesis | Evidence | Result |
|---|---|---|---|---|---|
| H-01 | customer | A:1045 ↔ B:88213 | same person | matching name, date of birth and e-mail, no conflict in stable attributes | MERGE |
| H-02 | customer | A:2210 ↔ C:5001 | same person | matching name, but a different date of birth | DO_NOT_MERGE |
| H-03 | invoice | B:F-7781 ↔ C:AR-7781 | two representations of one invoice | explicit reference from C to the B document number, same amount | XREF, not a new record |

Rules:

- **Similarity is not evidence.** A shared part of an id or a similar name is a candidate, not an
  identity. Evidence is a business key match without a conflict in stable attributes, or an explicit
  reference between systems.
- **A conflict in a stable attribute blocks the merge.** A stable attribute does not change in reality
  (date of birth, national ID). A different value means either two people or a data error; in both cases
  you do not merge.
- **Not merging is also a decision.** A DO_NOT_MERGE with its reason belongs in the catalogue, so nobody
  tries it again later without new evidence.
- **Negative test.** Add a query to the catalogue that proves nothing is merged by mistake: the number of
  identities with a conflict in a stable attribute must be zero.

How many false merges can the purpose tolerate? Usually none. Two different people in one identity is
worse than one person in two identities: the first corrupts every calculation, the second only
understates the number of unique customers and can be fixed later.

## XREF

The result of consolidation is stored in a cross-reference table, not by overwriting source data:

```sql
CREATE TABLE core.customer_xref (
  core_customer_id  BIGINT       NOT NULL REFERENCES core.customer,
  source_system     VARCHAR(10)  NOT NULL,
  source_key        VARCHAR(50)  NOT NULL,
  hypothesis_id     VARCHAR(10)  NOT NULL,   -- link to the catalogue, for traceability
  valid_from        TIMESTAMP    NOT NULL,
  PRIMARY KEY (source_system, source_key),          -- every source record exactly once
  UNIQUE (core_customer_id, source_system)          -- at most one representation per source per identity
);
```

Why these two constraints: the first guarantees completeness and uniqueness (every source record has
exactly one core identity), the second prevents two records of the same source from merging into one
identity. A record with no match gets its own identity; it is never dropped.

The same principle applies to transactions: `invoice_xref` binds the representations of one invoice
from different systems to one core invoice. Measures are then computed from the core invoice, never from
the sum of its representations.

## Critical errors

| Error | Why it is critical | How to prevent it |
|---|---|---|
| merging across a conflict in stable attributes | joins two real people, corrupts every calculation | a conflict blocks MERGE, negative test |
| two records of the same source in one identity | a hidden deduplication of the source without a decision | `UNIQUE (core_id, source_system)` |
| changing source data to make a match | destroys the audit trail, cannot be repeated | fix only in transformation, source unchanged |
| double counting a transaction | inflated revenue and payments | XREF of representations, measures from the core event |
| filling a missing attribute with a guess | invented data passed off as fact | the attribute stays NULL, the limitation is recorded |

## Review

```sql
-- duplicate identities: same business key, several core identities
SELECT bk, COUNT(DISTINCT core_customer_id) FROM core.customer_bk
GROUP BY bk HAVING COUNT(DISTINCT core_customer_id) > 1;

-- double counting: a transaction with representations in two sources, both in the fact table
SELECT x.core_invoice_id, COUNT(*) FROM core.invoice_xref x
JOIN mart.f_revenue f ON f.source_system = x.source_system AND f.source_key = x.source_key
GROUP BY x.core_invoice_id HAVING COUNT(*) > 1;
```

| Check | Anti-pattern | Finding |
|---|---|---|
| does every master entity have an identity across sources? | customers of each source side by side with no link | C: no consolidation |
| can each merge be traced to its evidence? | merged "by name" in a script with no catalogue | C: undocumented identity |
| is there an XREF with constraints? | mapping lives in transformation code and is stored nowhere | C: untraceable lineage |
| are transaction representations linked? | invoices from both the sales and the accounting system summed up | C/M: double counting, critical |
| are missing attributes filled with guesses? | country derived from the phone number without a flag | C: invented data |

## Output of consolidation

A hypothesis catalogue (MERGE and DO_NOT_MERGE) with evidence and profiling queries, the business key
definition of each entity, XREF tables for master entities and transaction representations with their
constraints, the negative test, and a list of limitations (what the input cannot determine). Continue
with `da-reference-data` and `da-core-model`.
