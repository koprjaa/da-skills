# Change in the sources

Data Vault is chosen because sources keep changing. The rule for every change: add entities, do not
rebuild or overwrite existing ones. The raw vault is insert-only and keeps its history.

Sources: the workshop exercises (new requirement, column changes, new regional sources) state the
problems. Where the course materials give no answer, the text below says so.

## A new requirement needs new source objects

Bring the new source tables into staging (one stage per source table), add hubs only for new business
keys, links for the new relationships and satellites for the new attributes. Existing entities stay as
they are. Example from the workshop: sales from a web shop added to a football model meant new hubs
(order, product), a non-historised line item link, a link between order, customer and match, and their
satellites.

**A relationship defined by a business rule is not a raw vault link.** When the connection exists only
through a rule (a product belongs to a player because the player's name appears in the product name),
there is no key in the source that states it. Model it as an exploration link in the business vault, so
the rule can change without touching raw data.

## New and dropped columns in a source

The course materials pose the question but do not give a prescribed answer. The options follow from the
principles above (insert-only, one version of the facts, satellite splits):

- **A dropped column** stays in the satellite; new rows carry NULL. History is not rewritten.
- **New columns**, two options:
  - add them to the existing satellite: the hash diff input changes, so every key gets one new version
    on the next load even though nothing changed in reality;
  - open a new satellite for the new structure and stop loading the old one: the delta check stays
    clean and the old history is untouched.
- **A changed representation** (a date of birth split into year, month and day) is reconciled in the
  business vault: a computed satellite builds one attribute from both versions, so marts do not notice.

Whichever you choose, write it down as a decision in the `da-pipeline` format. Neither option is
prescribed by the course materials; the second keeps the delta check clean at the cost of one more
satellite to join.

## A new source with different keys

A new source of the same business objects (players, teams, matches from regional marketing) gets its own
staging tables and its **own satellites per source** on the existing hubs (`regional_player_s` next to
`player_s`). Existing satellites are not changed.

The keys decide the rest:

- **Same key, same meaning** (the regional system uses the global player id): load into the same hub.
- **Different keys for the same real objects** (`playerUUID` against `player_id`): both keys land in the
  hub as separate rows. A **same-as link** in the business vault records which of them are one player,
  with the evidence from `da-consolidation`.
- **Keys that can collide in value but mean different things** (team number 12 in both systems, two
  different teams): keep them apart, either with a separate hub or with a business key collision code
  added to the key. The collision code is standard Data Vault practice but is not in the course
  materials.

Events from the new source (regional match actions) go to the existing non-historised link with their
own record source, or to a separate one if the grain differs.

## Checklist for any change

| Question | If yes |
|---|---|
| does the change add a business key? | new hub |
| does it add a relationship stated in the source? | new link |
| does it exist only through a rule? | exploration link in the business vault |
| does it add or change attributes? | new satellite, or a documented change of an existing one |
| does it bring a second key for known objects? | same-as link |
| does any existing table have to be rebuilt? | stop: look for the additive way |
