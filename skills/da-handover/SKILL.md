---
name: da-handover
description: Documents and defends a data architecture. Assembles the architecture package (context, options and rationale, layer design, security, operating and migration model, work packages, resources, schedule, assumptions, constraints and risks with mitigation), keeps the decision log, checks that another engineer can find the evidence for every decision, and prepares a short presentation and answers to the typical questions. In review mode it checks whether the existing documentation is enough to take the solution over. Use it for "write the architecture documentation", "prepare the defence of the design", "present the data warehouse to management", "architecture package", "how do we defend this in 10 minutes", "what will they ask us", even if the user never says handover. Keywords: documentation, architecture package, deliverables, ADR, decision log, defence, presentation, schedule, risks, mitigation, acceptance criteria, handover.
---

# Handover: package, decisions, presentation

The last step, but one that grows throughout. An architecture that cannot be explained and taken over is
useless. The package must stand before two audiences: the one who decides (why this option, what it
costs, what the risks are) and the one who will build and run it (what exactly, where is the evidence).

Place in the pipeline: collects the outputs of every step from `da-context` to `da-governance`. Start the
decision log in the very first step, not at the end.

## Architecture package

Structure following the phases of an architecture engagement:

| Part | Content | From |
|---|---|---|
| context | problem and cause, goals and metrics, scope and out of scope, stakeholders, requirements | `da-context` |
| starting point | sources, entities, grain, overlaps; in review mode the finding catalogue | `da-inventory`, `da-review` |
| options | prioritised list of options, the best one and its rationale | `da-approach` |
| design | business layer (how needs are met), application and data layer (consolidation, reference data, core, integration, marts), infrastructure layer (where everything runs) | step skills |
| cross-cutting models | security model, operating model, migration model | `da-governance`, `da-integration` |
| delivery | list of deliverables and work packages with the dependencies between them | — |
| finalisation | resource plan (people, money, IT), cost estimate, schedule, assumptions and constraints, risks with mitigation | — |
| decision log | all decisions in the `da-pipeline` format | all steps |

A **deliverable** is something that exists on its own and has value (the core model, the sales mart). A
**work package** is the work that produces it. Capturing the dependencies between packages is the key
part: the mart cannot be delivered before the core.

For resources, expect people to be the biggest constraint, and make the schedule respect their
availability. Assumptions and constraints state the conditions under which the design holds; a risk is
uncertainty, and every risk gets a mitigation.

## Decision log

Every decision has a number, problem, options, criteria, choice with rationale, assumptions, risks and
consequences (format in `da-pipeline`). "Do not do it" decisions belong in the log too: not merging, not
calculating a KPI, not building streaming. Those are the ones asked about most in a defence.

Do not delete superseded decisions. Mark them as replaced and link to the new one. The history of
decisions explains why the solution looks the way it does.

## Handover test

The acceptance test of the documentation: another data engineer who did not work on the design uses the
package to run the solution and finds the evidence for every decision (query, data, source). Walk through
it yourself before someone else does:

- pick three decisions from the log at random and find their evidence without the author,
- pick a number from a mart and walk it back to the source (dictionary and lineage from `da-governance`),
- run the pipeline from scratch on a clean environment (`da-integration`),
- check that every limitation (what the input cannot determine) is recorded and visible in the report.

## Presentation

Before building slides, answer the preparation questions:

1. **What is the goal?** To inform, or to persuade towards a decision? It shapes the whole structure.
2. **For whom?** Management wants the option, the cost and the risks; engineers the model and the flows;
   users what changes for them. Fit the message to the audience's interests; one presentation for everyone
   does not work, and jargon loses those who do not know it.
3. **Why now?** Which problem the design solves (from `da-context`).
4. **In what format?** Many presentations now happen online, where the audience mostly sees the screen and
   you, so the slide must carry its structure by itself.

Rules for a short format (8–10 minutes):

- From a 20–30 page package pick only the core: the problem, the chosen option and why, the target
  architecture in one picture, the two or three most important decisions, risks, and what needs deciding.
  Details stay in the package for questions.
- A slide has a clear hierarchy. The eye reads left to right and top to bottom and catches mainly what
  stands out visually, so highlight the one main point of each slide.
- Do not read text from the slides. Prepare keywords, not sentences.
- Everyone presenting should be able to explain at least one "merge" decision and one "do not merge" or
  "do not do" decision. It shows the decisions are grounded, not borrowed.

## Typical questions

Prepare an answer of under a minute, with a pointer to the evidence:

| Question | Where the answer is |
|---|---|
| Why this approach and not another (Inmon, Kimball, Data Vault)? | decision from `da-approach`, criteria; for Data Vault also the entity table from `da-data-vault` |
| How did you tell similarity from evidence of identity? Which candidate was rejected? | hypothesis catalogue in `da-consolidation` |
| What does your negative test prove? | `da-consolidation` |
| How do you prevent double counting a transaction held in two systems? | XREF of representations, reconciliation |
| Which measures cannot be calculated unambiguously from the input, and how did you document it? | KPI catalogue with limitations in `da-mart` |
| How do you keep identity separate from attribute history in the next batch? | `da-core-model` |
| What happens when a new source or a new code list value arrives? | `da-integration`, `da-reference-data` |
| Where exactly does this number come from? | dictionary and lineage in `da-governance` |

The same holds for theoretical questions asked without preparation (definitions of terms): short, precise,
with an example.

## Review

| Check | Anti-pattern | Finding |
|---|---|---|
| is there a package with justified options? | the documentation describes tables, not why they exist | X: no rationale |
| are assumptions, constraints and risks written down? | the design presents itself as valid unconditionally | X: unknown limits of validity |
| does the documentation pass the handover test? | only the author can run and explain the solution | G: key-person risk |
| is the decision log complete, including "do not do"? | only what was built is recorded | X: missing negative decisions |

## Output of handover

An architecture package in the structure above, the decision log, the result of the handover test, an
8–10 minute presentation and prepared answers to the typical questions.
