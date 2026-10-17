# da-skills

Twelve Claude Code skills that design a data architecture from scratch or review an existing one, from requirements and source inventory through the choice of Inmon, Kimball or Data Vault to the core model, loading, data marts and governance.

![claude-code](https://img.shields.io/badge/Claude%20Code-skills-D97757?style=flat-square)
![license](https://img.shields.io/badge/license-MIT-A31F34?style=flat-square)
![status](https://img.shields.io/badge/status-active-22863A?style=flat-square)
[![ci](https://github.com/koprjaa/da-skills/actions/workflows/ci.yml/badge.svg)](https://github.com/koprjaa/da-skills/actions/workflows/ci.yml)

The method comes from the course 4IT419 Data Solution Architectures at Prague University of Economics and Business: its lectures, the readings on Inmon, Kimball and Data Vault 2.0, and a warehouse case study built on several overlapping source systems. Nothing in the skills is tied to that case study, and the steps work the same for an insurer or a public registry.

The companion set is [dq-skills](https://github.com/koprjaa/dq-skills), which measures and fixes the quality of the data itself. Consolidation, governance and source profiling hand off to it.

## Two modes

In design mode you start from a problem and goals and end with an architecture you can defend. Every step closes with a decision record.

In review mode you first reconstruct what is actually deployed, since documentation drifts from it. Then you walk the same steps as a checklist and end with a prioritised finding catalogue, each finding pointing to the step that fixes it.

The modes combine. A review often ends with one layer being redesigned, and that layer goes through design mode.

## Pipeline

```
da-context > da-inventory > da-approach                       analysis
     |
da-consolidation > da-reference-data > da-core-model > da-integration > da-mart > da-governance
                                                              layer design and loading
da-handover                                                   package, decisions, presentation

da-review    entry for review mode, sends each finding to the step that fixes it
```

| Skill | Function |
|---|---|
| `da-pipeline` | Entry point. Step map, the two modes, staging/core/mart layers, decision and finding formats. |
| `da-review` | Reconstructs the current state, finding catalogue, priority, link to the fixing step. |
| `da-context` | Problem and cause, goals with metrics, stakeholders, requirements, scope, link to enterprise architecture. |
| `da-inventory` | Source systems, entities and their role, grain, keys, overlaps between sources, evidence by query. |
| `da-approach` | Inmon, Kimball, Data Vault, lakehouse, mesh, fabric, with decision criteria and a justified choice. |
| `da-consolidation` | Identity across sources: business keys, merge and do-not-merge hypotheses, cross-reference tables. |
| `da-reference-data` | Core code lists with the meaning of each code, mapping from source codes, conflicts. |
| `da-core-model` | Integrated core in 3NF or Data Vault, business and surrogate keys, constraints, history (SCD2). |
| `da-integration` | Source to staging to core to mart, ETL or ELT, mapping specs, incremental loads, tests, reconciliation. |
| `da-mart` | Dimensional model: business process, grain, facts, conformed dimensions, KPIs. |
| `da-governance` | Ownership and stewardship, metadata and lineage, quality as a control, security, operating model. |
| `da-handover` | Architecture package, decision log, an 8 to 10 minute presentation, the questions to expect. |

Run the steps in this order. A core model designed before consolidation bakes in the wrong identities, so two records of one customer stay two entities and every mart counts them twice. A mart designed before the core brings back several versions of the truth, the problem the warehouse exists to remove. Skip steps the domain does not have; do not reorder them.

## Decision format

Each design step ends with a record in the same shape, so the handover package is mostly a matter of collecting them:

```
Problem / Options / Criteria / Choice / Assumptions / Risks / Consequences
```

Review findings carry an ID by area (`X` context, `S` sources, `C` core, `I` integration, `M` mart, `G` governance), the evidence query, the impact, and the step that fixes it.

## Install

```bash
git clone https://github.com/koprjaa/da-skills.git
cp -R da-skills/skills/* ~/.claude/skills/
```

For one project only, copy into `.claude/skills/` inside that project. Install [dq-skills](https://github.com/koprjaa/dq-skills) next to it if you want the hand-offs to resolve.

## Use

The skills activate from their description. Write what you want:

```
design a data warehouse on top of our ERP, e-shop and CRM
every report shows a different revenue, review our warehouse
Inmon or Kimball for an insurer that has to pass an audit
how do we merge customers from three systems without a shared ID
```

You can also call one directly with `/da-pipeline` or `/da-review`. Start with `da-pipeline` when you do not know which step comes first.

## Principles

- No claim about the sources without a query and a number.
- An architecture is designed for a stated problem. Without goals there is no right answer.
- Do not merge records you cannot prove are the same entity. A wrong merge is harder to undo than a missed one.
- Every choice names the options it rejected and why.
- A warehouse is finished when it has an owner and reconciliation with the sources passes.

## How it was tested

The skills were evaluated with Anthropic's skill-creator: three tasks (a design from three overlapping sources, a review of a broken warehouse, an approach choice for an audited insurer), each run with and without the skills. With the skills, answers were more explicit about evidence, rejected options and merge risks, at roughly twice the time and about 50 % more tokens. A strong model already knows the textbook answers, so the skills mostly add rigour.

## Limits

The skills contain documentation and SQL sketches. They ship no runnable code and connect to no database. The SQL is generic and needs porting to your dialect; `dq-pipeline` in dq-skills has a translation table.

## Development

```bash
cargo fmt --check
cargo clippy --all-targets -- -D warnings
cargo test
```

The tests check the shape of each skill file. The frontmatter has to parse, the name has to match the directory, and the description has to be long enough to match on and end with a keyword list. Every pipeline step has to appear in `da-pipeline`. Claude Code skips a skill with broken frontmatter without an error, so without these tests a missing step shows up only when the pipeline runs past it.

## License

[MIT](LICENSE)
