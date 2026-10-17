---
name: da-pipeline
description: Entry point and shared standard for data architecture work, whether designing from scratch or reviewing an existing solution (data warehouse, lakehouse, integration layer, reporting). Holds the step map, the two modes (design and review), the staging, core and mart layers, the architecture decision format and the finding format. Use it whenever a data architecture has to be built or checked, "design a data warehouse", "how do we build a DWH on top of these systems", "review our data architecture", "every report shows different numbers", "consolidate data from several systems", "Inmon or Kimball", "we want a data platform", "what is wrong with our warehouse", even if the user never says the word architecture. Keywords: data architecture, data warehouse, DWH, design, review, Inmon, Kimball, Data Vault, staging, core, data mart, lakehouse.
---

# DA pipeline: map and shared standard

Twelve skills for data architecture. They run in two modes: **design** builds an architecture from
scratch, **review** goes through an existing solution and says what is wrong with it and which step
fixes it. The steps are the same in both modes, only the question changes. In design it is "how do
we do this", in review it is "was this done well, and how do I tell".

```
da-context → da-inventory → da-approach                     (analysis: need, sources, style)
     ↓
da-consolidation → da-reference-data → da-core-model → da-integration → da-mart → da-governance
                                                            (layer design and loading)
da-handover                                                 (package, decisions, presentation)

da-review  ── entry for review mode: walks the current solution layer by layer and sends
              each finding to the step that fixes it
```

| Skill | What it does | When |
|---|---|---|
| `da-review` | review of an existing architecture, finding catalogue, priority, link to the fixing step | first in review mode |
| `da-context` | problem, goals and metrics, stakeholders, requirements, scope, link to enterprise architecture | first in design mode |
| `da-inventory` | source systems, entities, grain, keys, overlaps, evidence by query | after context |
| `da-approach` | choice of style (Inmon, Kimball, Data Vault, lakehouse, mesh, fabric) and its justification | after inventory |
| `da-consolidation` | identity across sources: business key, merge hypotheses, XREF | after the approach |
| `da-reference-data` | reference data, code mapping between sources | alongside consolidation |
| `da-core-model` | integrated core: entities, keys, constraints, history (SCD2) | after consolidation |
| `da-integration` | staging → core → mart, mapping specs, loading, tests and reconciliation | after the core model |
| `da-mart` | dimensional model for specific questions: facts, dimensions, grain, KPIs | after integration |
| `da-governance` | metadata, lineage, ownership, quality as a control, security | throughout, closes the design |
| `da-handover` | architecture package, decision records, presentation and questions | throughout, at the end |

The order has a reason. A core model designed before consolidation bakes in the wrong identities: two
records of the same customer stay two entities and every mart on top counts them twice. A mart designed
before the core model brings back the very problem the warehouse exists to solve, several versions of
the truth. You can skip steps the domain does not have (one source → consolidation is trivial), but do
not reorder them.

## Two modes

**Design.** Start with `da-context`. Without a problem and goals you cannot tell what the right
architecture is: an architecture is always designed for a specific purpose. Every step ends with a
decision in the format below.

**Review.** Start with `da-review`. First reconstruct what actually exists (not what the documentation
claims), then walk the steps as a checklist. Every step skill has a "Review" section listing what to
verify and which anti-patterns to look for. The output is a finding catalogue, not a new design. A
remediation design comes only after prioritisation.

The modes combine. A review often ends with one layer being redesigned, and for that layer you switch
to design mode.

## Data warehouse layers

The classic three-layer split:

| Layer | Purpose | Rules |
|---|---|---|
| **L0 staging** | temporary place where source data is copied one to one | no business logic; some warehouses do without it |
| **L1 core** | integrated, cleaned data of the whole organisation, a single version of the truth | 3NF (Inmon) or Data Vault, depending on the approach |
| **L2 mart** | data prepared for one department or question | dimensional, may be denormalised |

Why staging exists: sources cannot always be extracted at the same moment (time zones, load on
operational systems, network capacity), and data has to be ready before it is integrated. Staging can
keep the full history (persistent) or only the current batch (transient). In Data Vault only "hard"
rules that do not change meaning are applied in staging: hash key calculation, load timestamp and
record source. Business rules belong to the higher layers.

The Data Vault variant splits the core into a **raw vault** (integrated raw data) and a **business
vault** (business logic), with **information marts** for presentation. Details in `da-approach` and
`da-core-model`.

## What a data architecture answers

A data architecture is more than a table schema. It answers:

- who is responsible for the data and what they do with it (roles, organisational units),
- who uses the data and in which processes,
- what the structure of the data is and how it relates,
- what means are used to process and store it,
- what the reference data and quality rules are,
- what the data life cycle is and where the data flows,
- how security and access are controlled.

Tell three types of data apart, because each is handled differently: **transactional** (events),
**master** (customer, product, location) and **reference** (code lists and classifications). And three
forms: structured, semi-structured, unstructured.

Models come in three levels. **Conceptual**: entities and relationships for the business. **Logical**:
detailed, attributes assigned through normalisation, independent of technology. **Physical**: for a
specific database. A review often finds that the logical model is missing and the physical one was
derived straight from the source tables.

## Architecture decision format

Record every significant decision (style, key, grain, history, tooling) like this:

```
### D-03 Customer history
Problem:      address and segment change; reports need the state as of the transaction date
Options:      A) overwrite (SCD1)  B) new row version (SCD2)  C) monthly snapshot
Criteria:     correctness of historical reports, volume, query complexity
Choice:       B, because revenue by segment must be counted as of the purchase day
Assumptions:  the source sends a change date; a customer changes a few times a year at most
Risks:        the source has no change date → mitigation: derive it from load time, flag as estimate
Consequences: every dimension carries valid from/to and a current-version flag
```

Why this shape: options must be comparable, the chosen one needs a justification, and the design only
holds under the stated assumptions. Without assumptions nobody can tell when the decision stopped
holding. The records are collected into a log in `da-handover`.

## Finding format (review mode)

| ID | Layer / area | Finding with evidence | Severity | Impact | Fix |
|---|---|---|---|---|---|
| C2 | core / customer | 2,140 of 18,300 customers (11.7 %) exist in two sources with no link, query Q-07 | High | revenue per customer understated, duplicate targeting | `da-consolidation` |

ID coding: the letter is the layer or area (X context, S sources, C core, I integration, M mart,
G governance), the number is the order. Every finding carries evidence: a query, a count and its
universe, not an impression.

| Severity | Criterion |
|---|---|
| **Critical** | report numbers are wrong and decisions rely on them, data loss, regulatory breach |
| **High** | several versions of the truth, manual workarounds in a key process, the error grows with every load |
| **Medium** | worse maintainability and performance, undocumented decisions |
| **Low** | naming, consistency, cosmetics |

The scale is an operating convention of this set, not an industry standard.

## Portability

The skills describe roles, not specific tables. Before you start, assign the roles:

| Role | What it is | Examples |
|---|---|---|
| **source system** | operational system that creates the data | web shop, point of sale, ERP, hospital system, registry |
| **master entity** | the subject everything else is about | customer, patient, product, store, applicant |
| **transaction** | an event in time | order, invoice, payment, visit, filing |
| **code list** | internal domain of values | order status, payment type, category |
| **reference registry** | external authoritative source | ISO country codes, industry classification, address registry |

This set does not measure data quality; it delegates it to `dq-skills`: source profiling to
`dq-profiler`, matching and golden record to `dq-deduplicator`, controls and monitoring to `dq-guard`.

## Hard rules

- **Architecture serves a purpose.** Do not judge or design without a stated problem and goals. Start
  with `da-context`, in review too: a solution that is technically clean but does not answer the
  business questions is a bad architecture.
- **No claim about the data without a query.** Prove grain, keys, overlaps and duplicates with a query,
  a count and its universe. If it cannot be checked, write "not verified" and why.
- **Source data is never changed.** A fix belongs in the transformation or in the source system, not in
  the staged copy. A modified copy destroys the audit trail.
- **Look for the cause, not the symptom.** "Reports do not match" is a symptom; the cause is usually
  missing consolidation, an unclear grain or double counting.
- **Every option has a cost.** State assumptions, constraints and risks with mitigation. A decision
  without them can neither be defended nor reviewed later.
- **A positive finding is still a finding.** "The fact grain is unambiguous and enforced by a key"
  belongs in a review just as much as a defect.

## Output of the pipeline

Design mode: an architecture package (`da-handover`) with context, inventory, chosen approach, models of
all layers, mapping specs, tests and a decision log.
Review mode: a `da-review` report with the finding catalogue, priorities and the fixing step, and where
needed a reworked design of the affected layers.
