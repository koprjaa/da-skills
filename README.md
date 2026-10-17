# da-skills

Claude Code skills for designing or reviewing a data architecture.

![claude-code](https://img.shields.io/badge/Claude%20Code-skills-D97757?style=flat-square)
![license](https://img.shields.io/badge/license-MIT-A31F34?style=flat-square)
[![ci](https://github.com/koprjaa/da-skills/actions/workflows/ci.yml/badge.svg)](https://github.com/koprjaa/da-skills/actions/workflows/ci.yml)

Twelve skills: context, source inventory, choice of approach (Inmon, Kimball, Data Vault, lakehouse), consolidation, reference data, core model, loading, data marts, governance and handover. The method comes from the course 4IT419 Data Solution Architectures at Prague University of Economics and Business.

Data quality work (profiling, deduplication, constraints) goes to [dq-skills](https://github.com/koprjaa/dq-skills).

## Two modes

Design starts at `da-context` and produces a decision record for each step.

Review starts at `da-review`. It reconstructs what is actually deployed, walks the same steps as a checklist, and returns prioritised findings, each linked to the step that fixes it. A layer that needs rebuilding then goes through design.

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

Keep the order. A core model built before consolidation keeps one customer as two entities, and every mart then counts them twice. Steps the domain does not need can be skipped.

## Decision format

Each design step ends with a decision record:

```
Problem / Options / Criteria / Choice / Assumptions / Risks / Consequences
```

A review finding has an ID by area (`X` context, `S` sources, `C` core, `I` integration, `M` mart, `G` governance), an evidence query, the impact and the fixing step.

## Install

```bash
git clone https://github.com/koprjaa/da-skills.git
cp -R da-skills/skills/* ~/.claude/skills/
```

For one project only, copy into `.claude/skills/` inside that project. Install dq-skills too for the delegated steps.

## Use

Plain requests work:

```
design a data warehouse on top of our ERP, e-shop and CRM
every report shows a different revenue, review our warehouse
Inmon or Kimball for an insurer that has to pass an audit
how do we merge customers from three systems without a shared ID
```

Or call one directly, for example `/da-review`. If you are not sure where to start, use `/da-pipeline`.

## Principles

- Every claim about a source has a query and a number.
- Goals come first; without them a design cannot be judged.
- Merge records only with evidence they are the same entity. A wrong merge is harder to undo than a missed one.
- Every decision lists the rejected options and why.
- Loads are reconciled against the sources, and every layer has an owner.

## How it was tested

Three tasks ran with and without the skills in Anthropic's skill-creator: a design over three overlapping sources, a review of a broken warehouse, and an approach choice for an audited insurer. With the skills, answers cited more evidence, listed rejected options and flagged merge risks, and took about twice as long with about 50 % more tokens.

## Limits

The skills are documentation and SQL sketches, with no runnable code. To port the SQL to another dialect, use the translation table in `dq-pipeline` from dq-skills.

## Development

```bash
cargo fmt --check
cargo clippy --all-targets -- -D warnings
cargo test
```

The tests check that every skill file has valid frontmatter, a name matching its directory and a description with keywords, and that every step is listed in `da-pipeline`. Claude Code silently skips a skill with broken frontmatter.

## License

[MIT](LICENSE)
