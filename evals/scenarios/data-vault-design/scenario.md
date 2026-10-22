# Scenario: designing a Data Vault for a bike rental

## Task given to the agent

> We have decided to build the core of our warehouse as a Data Vault. The extracts and notes are in
> `context/`. Design the raw vault and the business vault: which hubs, links and satellites, what goes
> where, and how the column change and the partner app are handled. List the entities as a table.

Context (invented data): `crm_members.csv`, `crm_members_v2.csv`, `rentals.csv`,
`partner_app_users.csv`, `notes.md`.

## Grading criteria

| # | Criterion | Pass condition |
|---|---|---|
| 1 | Hubs on business keys | hubs for member, bike and station on their business keys; no hub for the rental itself unless it has its own business key |
| 2 | Key normalisation | `m-103` and `M-103` are named as the same key and a normalisation rule before hashing is stated |
| 3 | Events kept apart | rentals are modelled as events (non-historised link or link with an event identifier); the two rentals of `M-101` on `B-7` at `S-1` stay two rows |
| 4 | Privacy split | member names, e-mail, birth date and phone are in a personal-data satellite apart from non-personal attributes |
| 5 | Satellite per source | the partner app gets its own satellite(s); CRM satellites are not reused for it |
| 6 | Different keys, no raw merge | partner `user_uuid` is not merged into CRM members in the raw vault; matching (for example on e-mail) is a same-as link in the business vault, with its evidence |
| 7 | Column change without rewrite | the `phone` → `phone_country` + `phone_number` change keeps the old history; the choice (new satellite, or the same satellite with NULLs) is stated with its consequence |
| 8 | Soft rule in the business vault | "home station" is computed in the business vault (computed satellite or exploration link), not a raw vault link |
| 9 | Raw stays raw | no cleaning or derivation in raw vault satellites; changed e-mail of `M-102` is a new satellite version, not an overwrite |
| 10 | Tests | hub, link and satellite tests are listed (unique key, existing references, no NULL references) |

## Results

| Run | Model | Passed | Missed |
|---|---|---|---|
| baseline, 2026-10-22 | Sonnet | 8 of 10 | 4 (personal data left in the source satellites "to keep the model small"), 10 (no tests) |
| with `da-data-vault`, 2026-10-22 | Sonnet | 10 of 10 | none; also wrote decision records D-01 to D-05 and an orphan test for `m-103` |

The baseline is strong on Data Vault basics; criteria 4 and 10 separate the runs.
