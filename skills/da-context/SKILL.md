---
name: da-context
description: Maps the context of a data architecture before a single table is drawn. States the problem and its cause, goals with metrics, scope and out of scope, stakeholders and their interests, functional, non-functional and implicit requirements, and the link to the business, application and technology architecture. In review mode it checks whether the existing solution answers the real business questions. Use it when a data project starts, "what do we actually need from the data warehouse", "write down the reporting requirements", "who are the stakeholders", "why does nobody use our warehouse", "what are the non-functional requirements", or whenever a brief sounds technical and has no why in it. Keywords: context, requirements, stakeholders, goals, metrics, scope, non-functional requirements, enterprise architecture, TOGAF, Zachman, ArchiMate.
---

# Context: problem, goals, people, requirements

First step of design mode and the first check in review mode. An architecture is always designed for a
specific purpose, so without the purpose it can neither be designed nor judged. The output is a short
document that every later decision in the pipeline refers back to.

Place in the pipeline: before `da-inventory`. In review mode `da-review` runs it as the first check.

## First, set up the engagement

Before mapping anything, settle the frame of the work. Without it the design drifts.

| Question | Why it matters |
|---|---|
| Why is this work being done, what need triggered it? | defines success |
| Who produces the design and who provides input? | without access to people and data nothing can be verified |
| What must the architecture package contain? | avoids a dispute at the end |
| What is the time frame? | sets the depth of analysis |
| What are the acceptance criteria? | tells you when it is done |

## The problem and its cause

The problem comes from a need, but it usually has to be worked out. Look for causes, not symptoms.

- Symptom: "Sales and finance report different revenue for the quarter."
- Causes to verify: a different definition of revenue (with VAT, without, after credit notes), a
  different source, a different booking moment, the same invoice counted twice from two systems.

Ask about business questions, not about the report layout. Do not ask "what chart do you want", ask
"what problem do you want to improve and what information does the team need for it". The questions
tell you which data, at what level of detail and how fresh. The chart comes last.

## Goals and metrics

Every goal gets a metric that shows it is met. A goal without a metric cannot be verified, not even in a
review.

| Goal | Metric |
|---|---|
| one version of revenue across departments | difference between sales and finance revenue reports = 0 |
| report available in the morning | yesterday's data loaded by 7:00 on ≥ 95 % of working days |
| a consolidated view of the customer | share of customers with a single identity across sources |

Include a **scope statement**: what the design covers and, explicitly, what it does not. Without "out of
scope" the scope quietly grows.

## Stakeholders

A stakeholder is anyone with an interest in the architecture. They have different interests and
different power, so map them and choose a communication approach for each group. One presentation for
everyone does not work; the message has to fit each group's interests and concerns.

| Group | Typical roles | What they need |
|---|---|---|
| approvers and stewards | data owner, data steward | accountability, classification, access |
| users | analysts, data scientists, managers with reports | correct, available data and clear terms |
| supporting roles | executives, legal, security | regulatory compliance, funding, risk |
| data providers | owners of source systems | what you want from them and what load it brings |
| builders | data engineers, database administrators | a buildable specification |

Write responsibilities as a matrix of roles and activities: defining terms, approving the model, owning
a source, running the loads, resolving a data incident. The RACI form (responsible, accountable,
consulted, informed) is a practical convention, not a requirement of the method. What matters is that
each activity has exactly one accountable person and that the data owner and steward are named.

## Requirements

Three kinds, write down all of them:

- **Functional**: which questions the data must answer, broken down how and at what level of detail.
- **Non-functional**: data freshness (daily batch, near real time, real time), availability,
  scalability with volume, flexibility for new data types and sources, security and access control,
  history (how far back and how precise as of a date).
- **Implicit**: expectations nobody states because they are "obvious". Typically: the numbers match
  accounting, a historical report does not change after a data fix without explanation, not everyone
  sees sensitive data.

Freshness ties into integration. Batch processing keeps the target stable and limits the load on
sources; real time needs a different kind of integration (operational data store, streaming). A
"real-time" requirement without a business reason is expensive, so ask which decision actually depends
on that freshness.

For distributed storage the CAP theorem applies: of consistency, availability and partition tolerance
you can have at most two. If the requirements ask for all three, that is a conflict the business has to
resolve.

## Link to the enterprise architecture

Data architecture is one of four basic architectures: business (goals, organisation, processes,
functions, information), application (applications, their functions, integrations), data (information,
data, data management tools such as warehouses and ETL) and technology (compute, storage, networks). Next
to them stand the complementary ones: security, motivation (goals, requirements, drivers, constraints),
integration and physical.

Data architecture is bound on three sides: the business layer needs information for processes and
decisions, applications process data (and integration between applications is mostly about data), and
technology stores and moves it. Its inputs come from the enterprise and business architecture, IT
standards and the data strategy.

When the organisation uses a framework, place the work in it:

| Framework | What to take from it |
|---|---|
| TOGAF | domains business, application, data, technology; data architecture is produced mainly in phases B to D of the ADM cycle, with phase C the key one |
| ArchiMate | a language for diagrams of layers and their relationships |
| Zachman | a matrix of questions (what, how, where, who, when, why) and perspectives; the data architect deals with "what" between the logical and physical view |
| DAMA-DMBOK | data architecture as one discipline of data management, linked to governance |

The framework is not the goal. Architecture development is iterative, and each iteration should have a
specific business purpose and demonstrable value. Filling in every artefact of a framework without
purpose is an anti-pattern.

## Review

Check what of the above exists and whether it holds:

| Check | Anti-pattern | Finding |
|---|---|---|
| is the problem and its cause written down? | a warehouse built "because one should have one", with no question | X: architecture without purpose |
| do goals have metrics? | "better reporting" with no measure | X: unverifiable goal |
| are scope and out of scope defined? | every new source gets added without a decision | X: uncontrolled scope |
| are stakeholders mapped and owners named? | data "belongs to IT", nobody approves definitions | X + G: no ownership |
| does the solution answer the original questions? | marts for reports nobody reads, while the ones computed by hand in spreadsheets are missing | X: mismatch with the need |
| are non-functional requirements stated? | real time with no reason, or batch where a process waits | X: unjustified freshness |

Useful evidence in a review: a list of reports and calculations people produce outside the warehouse
(manual exports, spreadsheets). Each one signals a question the architecture does not answer.

## Output of context

A document (1–3 pages) with sections: problem and cause, goals and metrics, scope and out of scope,
stakeholders with communication and a responsibility matrix, requirements (functional, non-functional,
implicit), link to the enterprise architecture, assumptions and open questions. In review mode, also
X-series findings in the `da-pipeline` format. Continue with `da-inventory`.
