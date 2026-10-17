---
name: da-governance
description: Governance of a data architecture, the part that keeps it usable after handover. Roles (owner, steward, users, supporting roles), processes (classification, lineage, quality definition, change control), three kinds of metadata (business, technical, operational) and the catalogue, a dictionary of terms and KPIs, tracing a number from report back to source, classification of sensitive data, access control, masking and encryption, and the governance model (centralised, federated). Delegates quality controls to dq-guard. In review mode it finds terms without an owner, untraceable numbers and uncontrolled sensitive data. Use it for "who owns this data", "we need a data catalogue", "where does this number come from", "lineage", "business glossary", "who may see personal data", "data governance", "so the warehouse does not fall apart", even if the user never says governance. Keywords: data governance, data owner, data steward, metadata, data catalogue, lineage, business glossary, data dictionary, classification, PII, access control, masking, DAMA-DMBOK.
---

# Governance: making the architecture last

Ninth step, but one that runs throughout. An architecture without governance falls apart: terms drift
apart, nobody knows where a number comes from, new sources get bolted on without a decision. Governance is
the exercise of authority and control (planning, monitoring, enforcement) over the management of data
assets; it is a subset of data management, which also covers the technical activities.

Place in the pipeline: after `da-mart`, closing the design, but set up the roles and the dictionary from
`da-context` onwards. Quality controls, the firewall and monitoring are designed in `dq-guard`; this skill
designs who is responsible for what and how things are traced.

## Roles

| Category | Role | Responsibility |
|---|---|---|
| **governors (approvers)** | data owner, data steward | accountable for the data, implement the rules, classify data, decide on access |
| **users** | analysts, data scientists, managers | use data to make decisions, report problems |
| **supporting roles** | executives, legal | fund and lead the strategy, interpret regulation |

Every key entity (customer, product, invoice) and every key measure has a named owner. Master and
reference data belong to the organisation, not to an application or department, and therefore need a
steward who approves their changes.

Governance model: **centralised** (one place for governance, quality and integration) gives uniformity and
control but risks bureaucracy and slowness towards business units' needs. **Federated** (typical for data
mesh) lets domains own their data but keeps shared rules, otherwise every domain does things its own way.
Record the choice as a decision.

## Processes

1. **Know the data**: what we have and what it represents.
2. **Classification**: which data is sensitive (personal, financial, trade secret).
3. **Lineage**: where the data comes from and what it went through.
4. **Quality**: what good and bad data is, how often it is checked, what happens to bad data.
5. **Change control**: a change of definition, code list or structure is approved and announced before
   it is deployed.

Governors usually own these processes. Automate them where you can (a catalogue collects metadata by
itself, an identity service controls access); manual governance does not keep up as the platform grows.

## Metadata

| Kind | Content | Who uses it |
|---|---|---|
| **business** | names and definitions of terms, subject areas, entities and attributes, data models, refresh schedules, standards | users, stewards |
| **technical** | tables and columns, physical models, ETL jobs, access rights | engineers, database administrators |
| **operational** | run logs, errors, results, SLAs, technical roles and contacts | operations |
| **usage** | who accessed the data, when and how | security, audit |

Metadata is stored in a **catalogue**: an indexed directory where people find and understand data. A
catalogue makes sense for an organisation of any size; fast and consistent access to data is one of the
biggest pains of knowledge workers.

## Data dictionary and tracing a number

The dictionary links a business term to its technical implementation:

| Term | Definition | Owner | Where in the warehouse | Calculation | Lineage |
|---|---|---|---|---|---|
| revenue | sales net of VAT after reversals, as of the sale date | head of sales | `mart.f_sales.amount_net` | `SUM` over lines | src_b.order_line → core.sales_line → mart.f_sales |

The traceability test: take a number from a report and walk it back to the source. It should be possible
without the report's author, using only the dictionary, lineage and mapping specs (`da-integration`). A
dimensional model helps: from a well-built model documentation and lineage can be generated
automatically, from random queries and filters in a reporting tool they cannot.

## Security and privacy

Security protects data from unauthorised access, corruption and theft; privacy protects personal data from
misuse. Measures fall into three groups:

| Group | Measures |
|---|---|
| data protection | encryption in transit and at rest, masking of sensitive values, anonymisation |
| access control | roles and permissions (identity service), access to sensitive data only under conditions |
| network | firewall, separated networks |

Order of rollout: access control, audit logging and encryption are usually easier and can come first;
network security and security testing take more work. Classification comes before access: what is not
marked sensitive does not get protected. Regulatory compliance (data protection) is interpreted by the
legal team; the architecture must make it possible to mask, restrict access and trace processing.

Watch for business keys that carry personal data (national ID, e-mail): as a key they spread into every
layer. Consider it already when choosing the key in `da-core-model`.

## Quality as part of governance

Data quality is contextual: data is of high quality when it meets the expectations and needs of its users.
Follow the principles: focus on the data critical to the organisation (regulatory and financial reporting,
operations), manage quality across the whole life cycle, prevent errors and remove root causes. Design the
specific controls, scores and monitoring with `dq-validator` and `dq-guard`.

## Review

| Check | Anti-pattern | Finding |
|---|---|---|
| do key terms have a definition and an owner? | "customer" means something different in each department | G: no dictionary |
| can a report number be traced to its source? | the measure is calculated only in the reporting tool and its author has left | G: untraceable number |
| is there a catalogue, or at least a dictionary? | documentation lives in people's heads | G: no metadata |
| is sensitive data classified and controlled? | personal data visible to every warehouse user | G: no classification, breach risk |
| is change of definitions and code lists controlled? | a change in production without notice, historical reports changed | G: uncontrolled change |
| is the governance model a conscious choice? | domains do everything their own way with no shared rules | G: no federated rules |

## Output of governance

A role matrix (owner and steward for key entities and measures), a description of the classification,
lineage, quality and change processes, the catalogue structure and the data dictionary with lineage, the
classification of sensitive data and the access control design, and the governance model decision.
Continue with `da-handover`.
