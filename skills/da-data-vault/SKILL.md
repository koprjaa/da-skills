---
name: da-data-vault
description: Designs the core of a data warehouse as a Data Vault 2.0. Identifies business objects and business keys, normalises keys before hashing, models the raw vault (hubs, links, non-historised links, satellites, multi-active satellites), splits satellites by privacy, source, security and business criteria, places business rules in the business vault (PIT, bridge, exploration and same-as links, computed satellites), and handles change in the sources (new and dropped columns, a new source with different keys). Ends with an entity table that maps one to one onto datavault4dbt models. Branch of the pipeline after da-core-model when da-approach chose Data Vault. Use it for "model this as a Data Vault", "which hubs, links and satellites", "raw vault vs business vault", "how to add a new source to the vault", "a column changed in the source", even if the user only says hub or satellite. Keywords: Data Vault, Data Vault 2.0, hub, link, satellite, raw vault, business vault, hash key, hash diff, business key, non-historised link, PIT, bridge, same-as link, satellite split.
---

# Data Vault: designing the core

Branch of the pipeline. `da-approach` chose Data Vault for the core, `da-core-model` holds what both
variants share (choosing the business key, constraints, the data dictionary). This skill designs the
vault itself. The implementation in dbt continues with `da-dv-config` (install the package) and
`da-dv-build` (the models), testing with `da-dv-test`.

Data Vault 2.0 is a system of business intelligence with three pillars: **methodology** (consistent,
repeatable, pattern based, with implementation as a sub-pillar), **architecture** (multi-tier,
scalable, supports NoSQL) and **model** (flexible, scalable, hub and spoke). This skill covers the model
and the parts of the architecture it depends on.

## Layers and where rules apply

| Layer | Content | Rules |
|---|---|---|
| staging | raw extract mirroring the source tables, persistent (whole history) or transient (current batch) | **hard rules** only: types, hash keys, hash diffs, load timestamp, record source |
| raw vault | integrated, unmodified data from the sources as hubs, links and satellites, at the source grain | none beyond hard rules: one version of the facts, auditable |
| business vault | business-rule changed data on top of the raw vault | **soft rules**: cleaning, derivations, reusable logic, query helpers |
| information marts | subject-oriented presentation, any dimensional model | soft rules, built for users |

A hard rule does not change the content of a field or the grain. A soft rule changes the content, turns
data into information and may change over time. The question for every transformation: does it change
content? If yes, it does not belong in the raw vault. Business users get information marts; only a power
user works on the raw vault directly, in a separate user space.

## Step 1: business objects and business keys

From `da-inventory` list the business objects and how they relate (a simple conceptual model), then give
each one a business key. Ask of every candidate, as in `da-core-model`: unique across the enterprise,
unique over time, contains personal data, known to business users, natural or technical. The source
system's technical id is a candidate only when the business uses it and it does not collide across
systems; name and e-mail usually fail (personal data, not stable).

**Normalise the key before hashing**, by one written rule (case, whitespace, the separator of a
composite key). `B-485` and `b-485` give different hashes and so two hub rows for one object.

Why hash keys and not sequences: one deterministic reference for a key across sources and formats,
fixed length for indexing, shorter strings for composite keys, and above all **parallel loading** of
hubs, links and satellites without waiting for a sequence (which is what Data Vault 1.0 used). Hash keys
are recommended, not mandatory; the business key itself is the other option.

## Step 2: the raw vault

| Data | Entity | Structure |
|---|---|---|
| business object | **hub**: a distinct list of business keys | hash key (PK), load timestamp, record source, business key (may be composite) |
| relationship between keys | **link**: a distinct list of relationships | hash key (PK), load timestamp, record source, typically two or more hub hash keys (FK) |
| immutable event or transaction | **non-historised link** (transactional link) | as a link, plus the event's identifier and its descriptive attributes on the link itself |
| descriptive data over time | **satellite** on a hub or a link | parent hash key + load timestamp (PK), record source, optional hash diff, attributes |
| several values of one attribute at once | **multi-active satellite** | as a satellite, plus a sequence number within the parent (e.g. phone 1, 2, 3) |

Choose the non-historised link when the source never updates the record but issues new transactions or
reversals: invoices, sensor data, line items, clicks and sessions. Whatever you choose for events, give
each one its identifier (invoice number, session timestamp); otherwise two events of the same pair of
keys collapse into one link row. A hub is for a key the business cares
about; a key that only relates other keys belongs in a link.

Hubs are stable and rarely change; links allow many-to-many relationships and a change of a relationship
without touching the hubs; satellites hold history without overwriting. A change of descriptive data
therefore never touches keys or relationships.

The hash diff is a hash of the satellite's attributes. It makes the delta check cheap: a new satellite
row only when the hash diff changes.

## Step 3: split the satellites

Split by **privacy class** (personal data apart), **source** (one satellite per source system),
**security class**, **business criteria** (for example rate of change). Two rules:

- A satellite never describes another satellite or more than one parent.
- Every split trades finer organisation for more joins later. Split for a reason you can name.

A common first cut: per hub one satellite with personal data (`_p_s`) and one without (`_n_s`).

## Step 4: the business vault

Everything that changes content goes here, so the raw vault keeps one version of the facts and the
business vault can be rebuilt whenever a rule changes. Cleaning bad data belongs here, and what counts
as bad is decided with business users. The entities (PIT, bridge, exploration and same-as links,
computed aggregate links, computed satellites) are in [references/business-vault.md](references/business-vault.md).

## Step 5: change in the sources

The enterprise warehouse is a product, not a project: it keeps changing. Data Vault absorbs new sources,
new tables, schema changes and new requirements by adding entities, not by rebuilding existing ones. New
and dropped columns, a new source with different keys, and relationships that exist only through a
business rule are in [references/changes.md](references/changes.md).

## Step 6: tests

| Entity | Test |
|---|---|
| hub | the business key is unique (PK test); no NULL or empty business key (or combination) |
| link | the combination of hub references is unique; every hub hash key exists in its hub; no NULL or empty reference |
| satellite | hash key + load timestamp is unique; every parent hash key exists in its hub or link; no NULL or empty reference |

Implementation of these tests in dbt: `da-dv-test`.

## Methodology

Data Vault 2.0 draws on CMMI (aiming at level 5, optimised and automatable), Six Sigma, TQM, PMP and
agile delivery (Scrum). Roles in a team: architecture lead, source and business analysts, data or
analytics engineers, DevOps (CI/CD), testers. Because every entity follows a pattern, the models are
generated from metadata rather than written by hand; that is what `da-dv-build` does with datavault4dbt.

## Review

| Check | Anti-pattern | Finding |
|---|---|---|
| is Data Vault justified? | chosen without an audit requirement, changing sources or scale (`da-approach`) | X: unjustified complexity |
| does the raw vault hold raw data? | cleaning, derivations or business logic in raw vault satellites | C: no single version of the facts |
| is the business key normalised before hashing? | different spellings produce several hub rows for one object | C: duplicate hubs |
| is the hub key a business key? | a technical id that collides across systems | C: two objects in one hub row |
| are events kept apart? | repeated events of the same pair of keys without an event identifier, so they collapse into one row | I: lost or merged events |
| does each satellite have one parent and a reason for its split? | a satellite on two parents, or splits nobody can explain | C: wrong model |
| is personal data separated? | names and contact details in the same satellite as business attributes | G: privacy cannot be controlled |
| do users query marts? | reports written against hubs and satellites | I: no presentation layer |

## Output

An entity table, one row per entity, which `da-dv-build` turns into models:

| Column | Content |
|---|---|
| entity | name with the type suffix: `_h` hub, `_l` link, `_nl` non-historised link, `_s` satellite (`_p_s` / `_n_s` for the privacy split), `_bs` business vault satellite |
| type | hub, link, nh_link, satellite, ma_satellite, pit, bridge, computed |
| business key / parents | the key columns of a hub; the hubs of a link; the parent of a satellite |
| source | staging table and record source |
| payload | attributes of a satellite or a non-historised link |
| hash diff | yes or no |
| rules | soft rules applied (business vault only) |

Plus the written key normalisation rule and the decision record in the `da-pipeline` format. Continue
with `da-dv-config` and `da-dv-build`, then `da-integration` for loading and reconciliation.
