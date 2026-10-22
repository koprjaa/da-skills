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

## Events and late data

Events (match actions, line items, payments) go to a non-historised link. Three rules keep it correct when
data arrives late, is corrected or comes from a stream:

- **Two times.** Store when the event happened (`action_timestamp`, minute of the match) apart from when it
  was loaded (`ldts`). A red card given after the match by a disciplinary decision arrives days later: it
  is a new row with an early event time and a late load time. Reports use the event time, audit uses the
  load time. Nothing is reordered or rewritten.
- **Corrections are new events.** The course states that changes in transactional data show up as new
  transactions or reversal entries, not as updates. A yellow card upgraded to red is two new events (the
  reversal of the yellow and the red), the original row stays. If the source instead updates the event in
  place, a non-historised link cannot hold the correction; ask the source owner, and if updates are real,
  model the event as a standard link with a satellite.
- **A source identifier per event.** Build the link hash key from the hub keys plus an identifier the source
  gives the event (`rowguid`, invoice number), never from the arrival time. A stream that delivers a message
  twice then produces the same hash key and the second copy is not loaded again. (Duplicate delivery in
  streams is general practice, not in the course materials.)

Real-time sources load straight into the raw vault, next to the batch path through staging. Hash keys
make this work: a link computes the player's hash key itself and does not wait for anything. The player
hub is loaded from every stage that carries the key, the stream included; only the player's satellite,
with name and attributes, waits for the batch from the player source.

## Checklist for any change

| Question | If yes |
|---|---|
| does the change add a business key? | new hub |
| does it add a relationship stated in the source? | new link |
| does it exist only through a rule? | exploration link in the business vault |
| does it add or change attributes? | new satellite, or a documented change of an existing one |
| does it bring a second key for known objects? | same-as link |
| is it an event, or a correction of one? | a row in the non-historised link with its source identifier; a correction is a reversal plus a new event |
| does any existing table have to be rebuilt? | stop: look for the additive way |
