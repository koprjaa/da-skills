---
name: da-core-model
description: Designs the integrated layer (core) of a data warehouse, logically and physically. Either 3NF (Inmon) or Data Vault 2.0 (hub, link, satellite, hash keys, raw and business vault, PIT and bridge tables). Covers choosing the business key, surrogate and hash keys, PK, FK, UNIQUE, NOT NULL and CHECK as a safeguard for assumptions, history (SCD types 1, 2, 3 and history tables), separating identity from attribute versions, the data dictionary and attribute lineage. In review mode it finds a core that is just a copy of the sources, missing constraints and overwritten history. Use it for "design the core model", "how to model the warehouse core", "hub link satellite", "SCD2", "which keys for the DWH", "we need change history", "ERD of the integrated layer", even if the user never says core. Keywords: core model, integrated layer, 3NF, Data Vault, hub, link, satellite, hash key, business key, surrogate key, SCD2, constraints, data dictionary, lineage.
---

# Core model: the integrated heart of the warehouse

Sixth step of design mode. The core is the consolidation layer: it holds integrated, historical data at
the same grain regardless of which source it came from. It is the only source for the marts, so anything
that goes wrong here spreads into every report.

Place in the pipeline: after `da-consolidation` and `da-reference-data` (identities and codes are already
decided), before `da-integration`. The modelling variant was chosen in `da-approach`.

## Variant A: 3NF

Third normal form minimises redundancy: every non-key column depends only on the table's primary key.
Hierarchies inside a table (an address in the customer table, a group in the product table) are split
into their own tables linked by keys. It suits the integration layer especially when sources come at
different natural grain, because a normalised structure can absorb data from any source. It is usually
not used for the presentation layer because of the number of joins and the difficulty for business users.

Procedure: conceptual model (entities and relationships from `da-inventory` and `da-consolidation`) →
logical model (attributes assigned through normalisation, independent of technology) → physical model
(data types, keys, constraints, indexes for a specific database).

## Variant B: Data Vault 2.0

The core is split into hubs (business keys), links (relationships between keys) and satellites
(descriptive data over time), with a raw vault holding unmodified data and a business vault holding the
business rules. The design of the vault (keys and hashing, entity types, satellite splits, business
vault entities, changes in the sources) continues in `da-data-vault`; the implementation in dbt in
`da-dv-build`. The sections below (business key, constraints, history, dictionary) apply to both
variants.

## Choosing the business key

Questions for every candidate key:

1. Is it unique across the whole organisation?
2. Is it unique over time (never reassigned)?
3. Does it contain personal data?
4. Do business users know and use it?
5. Is it natural, or a technical (surrogate) key of one system?

The technical id of one system is a weak business key: outside the system it means nothing. The
warehouse surrogate key (sequence or hash) is an internal row identifier; the business key stays stored
next to it so the origin can be traced. The binding of source keys to the core identity lives in the
XREF from `da-consolidation`.

## Constraints as a safeguard for assumptions

Back every assumption about the data with a database constraint, otherwise the design is "happy path"
and fails silently:

| Assumption | Constraint |
|---|---|
| the entity has an identity | `PRIMARY KEY` |
| the business key is unique | `UNIQUE` |
| the referenced record exists | `FOREIGN KEY` |
| the attribute is always filled | `NOT NULL` |
| the value has an allowed range | `CHECK` (amount ≥ 0, valid_from < valid_to) |

Add constraints to intermediate transformation tables too. An error then shows up at the step that caused
it, not in the report. The practical effect: a category table that should have a few thousand rows and,
because of a bug in a previous transformation, has hundreds of millions is caught immediately with a
unique constraint; without one it just slows down and inflates the cost of every query. Where a value
genuinely can be missing, use an explicit unknown member (an "unknown" row in the code list) instead of
NULL, so it does not drop out of joins.

For Data Vault the standard tests are listed in `da-data-vault` and implemented in `da-dv-test`.

## History

Decide how a change of attribute is handled. The choice determines what historical reports show.

| Type | How | Effect on history |
|---|---|---|
| **SCD 1** | overwrite the value | history is lost, historical reports change |
| **SCD 2** | new row version with a new key and valid from/to | historical facts point to the old version, history preserved |
| **SCD 3** | a column with the previous value | only limited history, changes reports |
| **history table** | a row for every state of the entity over time | full history, simple logic |
| **attribute on the event** | the value valid at the time of the transaction stored with the transaction (delivery address on the order) | history without versioning the dimension |

The last two are now a common alternative to classic SCDs, because storage is cheap and the logic is
simpler. In Data Vault, satellites handle history by themselves.

**Separate identity from versions.** The core identity of a customer (from the XREF) does not change;
the versions of its attributes do. If identity and SCD2 are merged into one key, the next batch creates a
"new customer" for every change of address.

**Name the timestamps.** One entity often has several times: order created, shipped, the customer's first
order. They confuse users unless explicitly named; filtering revenue by ship date gives a different
number than by order date. Name every time column after its event.

## Data dictionary and lineage

For every core table: meaning, grain, keys (PK, business, FK), cardinality and optionality of
relationships, historisation. For every attribute: meaning, data type, mandatory or not, source
(system.table.column) and transformation. Attribute lineage is the input for `da-integration` and for
traceability in `da-governance`.

## Review

```sql
-- core as a copy of a source: core tables that mirror one source table 1:1
-- (compare column names via information_schema; a high match = suspicion)

-- broken history: two versions of the same identity valid at the same moment
SELECT a.customer_id, COUNT(*) AS overlaps
FROM core.customer_hist a JOIN core.customer_hist b
  ON a.customer_id = b.customer_id AND a.version_id < b.version_id
 AND a.valid_from < b.valid_to AND b.valid_from < a.valid_to
GROUP BY a.customer_id;
```

| Check | Anti-pattern | Finding |
|---|---|---|
| is the core integrated, or a copy of the sources? | every source has its own customer tables even in core | C: no integration |
| are assumptions enforced by constraints? | no PK, FK or UNIQUE, checks "sometimes by hand" | C: happy path |
| is history managed? | SCD1 where reports need the state as of a date | C: lost history |
| are identity and versions separate? | a new surrogate key for every attribute change, no stable identity | C: identity breaks apart |
| is the hash key deterministic? | hash from an unnormalised key, different spellings give different hubs | C: duplicate hubs |
| is there a dictionary and lineage? | only the author knows what the columns mean | C/G: missing documentation |

## Output of the core model

Conceptual, logical and physical model (ERD), DDL with constraints, the rule for normalising and hashing
business keys, a historisation decision for every entity (in the `da-pipeline` format), and a data
dictionary with attribute lineage. Continue with `da-integration`.
