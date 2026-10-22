# da-skills

Claude Code skills for designing or reviewing a data architecture.

[![ci](https://github.com/koprjaa/da-skills/actions/workflows/ci.yml/badge.svg)](https://github.com/koprjaa/da-skills/actions/workflows/ci.yml)
![license](https://img.shields.io/badge/license-MIT-A31F34?style=flat-square)

Based on the course 4IT419 Data Solution Architectures at Prague University of Economics and Business.

## Skills

To design from scratch, start with `da-context`. To check an existing solution, start with `da-review`. Then follow the order below.

| Skill | What it does |
|---|---|
| `da-pipeline` | Start here if unsure. Explains the steps and the shared formats. |
| `da-review` | Reviews an existing architecture and lists what is wrong. |
| `da-context` | Clarifies the problem, goals and requirements. |
| `da-inventory` | Maps the source systems and what data they hold. |
| `da-approach` | Picks the approach: Inmon, Kimball, Data Vault or lakehouse. |
| `da-consolidation` | Decides which records from different systems are the same customer, product and so on. |
| `da-reference-data` | Unifies code lists across systems. |
| `da-core-model` | Designs the central data model. |
| `da-integration` | Designs how data is loaded and checked. |
| `da-mart` | Designs reporting tables for specific questions. |
| `da-governance` | Sets ownership, documentation and access. |
| `da-handover` | Puts together the documentation and the presentation. |

When `da-approach` picks Data Vault, the core is designed and built by a separate branch:

| Skill | What it does |
|---|---|
| `da-data-vault` | Designs the Data Vault: hubs, links, satellites, business vault, changes in the sources. |
| `da-dv-config` | Installs and configures datavault4dbt in a dbt project. |
| `da-dv-build` | Builds the models in dbt. |
| `da-dv-test` | Adds the Data Vault tests. |
| `da-dv-troubleshoot` | Finds out why a datavault4dbt model misbehaves. |
| `da-dv-rehash` | Recalculates hashes after the hash settings change. |

The five `da-dv-*` skills are adapted from Scalefree's [datavault4dbt Agent Skills](https://github.com/ScalefreeCOM/datavault4dbt-agent-skills) under Apache 2.0. See [NOTICE](NOTICE).

## Install

```bash
git clone https://github.com/koprjaa/da-skills.git
cp -R da-skills/skills/* ~/.claude/skills/
```

## Use

Ask in plain words, for example "design a data warehouse on top of our ERP and e-shop" or "every report shows a different revenue, review our warehouse", or call a skill directly with `/da-review`.

Data quality steps use [dq-skills](https://github.com/koprjaa/dq-skills), so install those too.

## License

[MIT](LICENSE). The `da-dv-*` skills are Apache 2.0, see [NOTICE](NOTICE).
