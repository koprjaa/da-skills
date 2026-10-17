---
name: da-inventory
description: Source inventory for a data architecture. Maps the source systems (data and technology), entities and their role (master, transactional, reference), keys, the grain of every transactional table, time range, overlaps of the same entity across sources, and draws an ERD of the sources. Proves every claim with a query. In review mode it compares the documented sources with the ones actually connected. Use it when "we have three systems and want a warehouse on top", "find out what is in these databases", "what is the grain of this table", "which systems hold customers", "draw an ERD of the sources", "source data inventory", or when following up on data profiling. Keywords: inventory, source systems, grain, granularity, ERD, entities, keys, OLTP, source overlap, source mapping.
---

# Inventory: what the sources really contain

Second step of design mode. An architecture is built on the sources that exist, not on how somebody
describes them. Before connecting anything, analyse the source systems carefully, from their data down
to their technology.

Place in the pipeline: after `da-context`, before `da-approach`. The quality of data inside the sources
(completeness, formats, duplicates) is measured by `dq-profiler`. The inventory asks about structure:
what an entity is, what a key is, what the grain is and where sources overlap.

## Why the sources matter

Source systems are operational (OLTP): they exist to capture data, are usually modelled in third normal
form and optimised for inserting and updating single rows, not for analysis. Departments often have
their own systems, or there are systems with special functions. So they run on different technologies,
and integration runs into different data types, encodings and conventions.

A data warehouse, in contrast, should be subject-oriented (data about one area together regardless of
origin), integrated (from several systems under shared rules), time-variant (snapshots after period
close, with history) and non-volatile (data does not change once loaded). The inventory measures how
far the sources are from that target.

## Procedure

### 1. List of sources

| Source | Owner | Technology | Access | Volume | Time range | Change frequency |
|---|---|---|---|---|---|---|
| web shop | sales | PostgreSQL | read-only replica | 2.1 M rows | 2019– | continuous |
| point of sale | retail | daily CSV export | files | 9.4 M | 2021– | daily |

Find the owner right away: without one you cannot confirm what a column means or agree a change in the
source.

### 2. Entities and their role

For each table set the role: **master** (customer, product, store), **transactional** (order, invoice,
payment), **reference** (code list, classification), or a link table. The role decides what happens
next: master data goes to consolidation, code lists to code mapping, and transactions set the grain of
facts.

### 3. Keys

For each entity, prove with a query what actually identifies it:

```sql
-- is the declared key really unique?
SELECT customer_no, COUNT(*) FROM src_a.customer GROUP BY customer_no HAVING COUNT(*) > 1;

-- is there a natural (business) key next to the technical id?
SELECT COUNT(*), COUNT(DISTINCT email), COUNT(DISTINCT id) FROM src_a.customer;
```

Separate the system's technical key (a sequence number, valid only inside the system) from the business
key (customer number, product code, company ID) that makes sense outside the system. The business key is
the basis for consolidating across sources.

### 4. Grain of every transactional table

Grain is the level of detail one row represents. State it in a sentence ("one row = one receipt line")
and prove with a query that the column combination expressing it is unique.

```sql
SELECT receipt_id, line_no, COUNT(*) FROM src_b.receipt_line
GROUP BY receipt_id, line_no HAVING COUNT(*) > 1;
```

Why grain matters so much: from the lowest level of detail you can aggregate up, never the other way.
Population data per country cannot answer a question about individual regions. A wrong grain can make
it impossible to answer the questions the architecture is being built for. Watch for tables with mixed
grain (header and lines in one table, monthly totals next to daily records).

### 5. Overlaps between sources

An entity × source matrix shows where consolidation will be needed:

| Entity | Source A | Source B | Source C | Shared key? |
|---|---|---|---|---|
| customer | yes (e-mail) | yes (loyalty card) | yes (company ID, businesses only) | no |
| product | yes (SKU) | yes (EAN) | no | partly |
| invoice | no | yes | yes (accounting copy) | document number |

A row without a shared key is input for `da-consolidation`. The same transaction in two sources (an
invoice in both the sales and the accounting system) is a double-counting risk. Write down which source
is primary for that transaction.

### 6. ERD of the sources

Draw an entity-relationship diagram of each source: entities, keys, relationships, cardinalities. It is
a shared language between business and technical people and the basis for mapping specs in
`da-integration`. Draw it from the real data (keys and relationships verified by query), not from the
documentation.

## Rules

- **Sources are not changed.** The inventory only reads. Data fixes belong in transformation or in the
  source system, never in the copy you analyse. A modified copy destroys the evidence.
- **Every claim with a query and a universe.** "The key is unique" only with a query that returned zero
  and the row count it ran on.
- **Write down what you do not know.** A column whose meaning nobody knows is an open question for the
  source owner, not an assumption.
- **Snapshot.** Fix the date the inventory is valid for. Sources change.

## Review

| Check | Anti-pattern | Finding |
|---|---|---|
| do documented sources match the ones actually connected? | a source connected outside the documentation, or documented and long disconnected | S: as-is differs from documentation |
| is the grain known for every transactional table in the warehouse? | grain written nowhere, inferred from a report | S: undefined grain |
| is the primary source defined for a transaction held in several systems? | the invoice loaded from both systems | S: double-counting risk |
| are business keys identified? | consolidation on technical ids that mean nothing outside their system | S → `da-consolidation` |
| are technology and type differences handled? | numbers as text, different character encodings, time zones | S → `da-integration` |

## Output of the inventory

A source table, a list of entities with role and keys, a grain table for transactional tables with
verification queries, an entity × source matrix with the primary source for shared transactions, an ERD
of each source, open questions for owners. In review mode, S-series findings. Continue with
`da-approach`.
