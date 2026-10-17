---
name: da-approach
description: Chooses the architectural style and modelling approach for a data solution and justifies it. Compares Inmon (top-down, 3NF core), Kimball (bottom-up, star schema), the hybrid, Data Vault 2.0, snowflake, One Big Table, activity schema and entity-centric modelling, and at platform level the data warehouse, data lake, lakehouse with medallion layers, data fabric, data mesh, and Lambda and Kappa processing. Decides from the criteria found in context and inventory and records the decision with its options. In review mode it tells whether an existing solution follows any approach at all. Use it for "Inmon or Kimball", "do we need Data Vault?", "lakehouse or warehouse", "should we do data mesh", "which model for the DWH", "star or 3NF", "medallion bronze silver gold", even if the user only asks about one of them. Keywords: Inmon, Kimball, Data Vault, star schema, snowflake, OBT, lakehouse, medallion, data lake, data mesh, data fabric, Lambda, Kappa, approach selection.
---

# Approach: which architectural style, and why

Third step of design mode. Decisions are made on two levels: the **platform** (warehouse, lake,
lakehouse, fabric, mesh) and the **modelling** of each layer (3NF, Data Vault, star and others). There is
no right answer in general, only for specific questions, volumes, users and team. That is why the choice
comes after `da-context` and `da-inventory` and is recorded as a decision with options.

Place in the pipeline: after inventory, before consolidation. The choice shapes `da-core-model` and
`da-mart`.

## Decision criteria

Score the options on these criteria. The weights come from the context.

| Criterion | Question |
|---|---|
| analytical requirements | what questions will be asked and what query pattern follows from them? |
| volume and scalability | how much data is there now and how much will there be? |
| ease of use | who will write the queries, an engineer or a business user? |
| flexibility | how often do sources, rules and questions change? |
| performance | how much does query speed matter against data redundancy? |
| auditability and history | must every value be traceable to its origin, with history of relationships and sources? |
| team and time | what skills does the team have and when must the first value arrive? |

## Inmon, Kimball and the hybrid

| | Inmon (top-down) | Kimball (bottom-up) |
|---|---|---|
| starts with | an enterprise data model of the main subject areas (customer, product, supplier) → logical → normalised physical | a business process and the questions it must answer |
| data flow | sources → warehouse in 3NF at the lowest grain → departmental marts | sources → marts in star schemas; the warehouse is the logical union of marts |
| precondition | the organisation agrees on definitions, cleaning and rules up front | definitions are agreed department by department |
| advantages | single version of the truth, normalisation saves space, new marts are easy | fast start, lower initial cost, a model users understand |
| disadvantages | more joins and slower queries, long upfront work, higher initial cost | longer ETL, risk of duplicate and conflicting data across marts, ongoing development |

The risk of Kimball without shared definitions: a "revenue" field in two marts, once with cost of goods
and once without. Kimball's answer is **conformed dimensions**, for example one calendar shared by all
marts.

In practice the **hybrid** dominates: a normalised warehouse (Inmon) as the single source of truth, star
schema marts (Kimball) on top, optionally cubes or a semantic model. It is not a template, some layers
can be skipped, but reports straight on the source systems are usually not the way, because they load the
operational systems.

## Modelling approaches

| Approach | Where it fits | Pros | Cons |
|---|---|---|---|
| **3NF** | integration layer (core, silver), especially with several sources at different grain | little redundancy, integrity through keys, flexibility | many joins, hard for business users; usually not used for the presentation layer |
| **star (Kimball)** | presentation layer, marts | fast queries, intuitive, easy aggregations | redundancy, dimension history is tricky, less flexible |
| **snowflake** | marts with deep dimension hierarchies | less redundancy than a star, dimensions easier to maintain | more joins, harder to read |
| **Data Vault 2.0** | core with many changing sources, audit requirement | handles changing requirements without rebuilds, history and traceability, parallel and incremental loading | complex design, many tables and joins, hard-to-read queries |
| **One Big Table** | single-purpose analysis, data science | fastest queries, everything in one place | high redundancy, changes are expensive |
| **activity schema** | events of one entity (customer activity) | few tables, new activities without schema change | new approach, not for every domain |
| **entity-centric** | entities with time-bound metrics in one row | flexible, few tables | complex queries over semi-structured data, weaker integrity |

### When Data Vault

Data Vault 2.0 (Dan Linstedt) separates business keys (hub), relationships (link) and descriptive
attributes (satellite). It combines elements of 3NF and the star schema. It pays off when you need to
track history at scale, including changes of relationships and changes of the sources themselves over
time, when sources keep being added and rules keep changing, and when auditability is a requirement
(regulation). It is more work and is not used across the board in practice, so choose it for a specific
requirement, not because it is fashionable. On top of a Data Vault there is always a presentation layer
(information marts) for users; querying the vault directly is hard.

## Platform

| Option | Essence | When |
|---|---|---|
| **data warehouse** | structured data, known purpose, schema on write | known questions, emphasis on correctness and security |
| **data lake** | raw data of all types, schema on read, ELT | unstructured data, purpose not yet known, exploration and data science |
| **lake + warehouse** | lake as staging and preparation, warehouse as the serving and security layer | the most common combination; the warehouse handles access control, concurrency and performance tuning better |
| **lakehouse** | raw, clean and presentation layers (bronze, silver, gold) on a lake format | one platform for both worlds; where the format falls short, part of the data is copied into a relational warehouse |
| **data fabric** | a unified view through metadata, catalogue, lineage and several delivery styles (ETL, streaming, virtualisation) | many heterogeneous sources, the aim of never refusing a new source; not necessarily centralised |
| **data mesh** | domains own their data as a product, self-service, federated governance | only large organisations with serious pain from central IT and a long horizon (years) |

A data lake alone does not replace a warehouse. A lake and a warehouse run separately usually end with
users unable to combine the data, so connect them. Data mesh is mainly an organisational and cultural
principle, not a technology: each domain still uses a warehouse, lakehouse or fabric inside. Its
challenges are cultural change, governance, coordination and technical complexity.

### Processing: Lambda and Kappa

**Lambda** processes batch and stream separately (batch, speed and serving layers) and merges them into
one view. It is flexible but harder to maintain. **Kappa** treats all data as a single stream; history
and recalculation are handled by replaying the whole event source. Choose by the freshness required in
`da-context`: when a daily batch is enough, you need neither.

## How to decide

1. Take the questions, volumes, users and non-functional requirements from `da-context` and the sources
   from `da-inventory`.
2. Decide the platform. The default for structured data with known questions is a warehouse, or lake +
   warehouse once unstructured data arrives.
3. Decide the model of each layer separately: core (3NF or Data Vault), presentation (star).
4. Record the decision in the `da-pipeline` format with at least two options and the criteria.

Example: three operational systems sharing customers and products, a need for one version of revenue,
stable sources, a small team. Inmon + star marts wins on the single-version-of-truth criterion; Data
Vault would add auditability the context does not ask for, at the cost of complexity. Kimball without a
central core would deliver the first report faster but would have to solve customer consolidation again
in every mart.

## Review

| Check | Anti-pattern | Finding |
|---|---|---|
| can you tell which approach the solution follows? | ad hoc denormalisation, internally inconsistent and unexplainable | X/C: no method |
| do an integrated layer and a presentation layer exist at all? | normalised source tables imported one to one, reports pointed straight at them | C → `da-core-model` |
| is the choice backed by criteria? | Data Vault or data mesh chosen because it is fashionable, with no audit requirement or organisation size to justify it | X: unjustified complexity |
| are lake and warehouse connected? | two separate worlds, users cannot combine the data | I → `da-integration` |
| does processing match the required freshness? | streaming where batch would do, or batch where a process waits | X: mismatch with the requirement |

## Output of the approach

A decision record on the platform and on the model of each layer (core, presentation) with options,
criteria, choice, assumptions and risks. A picture of the target architecture: sources → staging → core
→ mart → users. In review mode, findings. Continue with `da-consolidation`.
