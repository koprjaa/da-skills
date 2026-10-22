# Business vault entities

The business vault holds business-rule changed data. Its purpose is to make the end-user structures easy
to build. Besides cleaning it carries reusable logic, performance helpers (PIT and bridge) and the base
for self-service BI. Everything here can be dropped and rebuilt from the raw vault.

## Point-in-time table (PIT)

A query helper attached to a hub or a link with several satellites.

- It simplifies combining several deltas: for every snapshot date it records which version (load
  timestamp) of each satellite was valid.
- It reduces the joins needed to read the current or historical state.

| Column | Content |
|---|---|
| PIT key | PK |
| parent hash key | FK to the hub or link |
| snapshot date | the moment the row describes |
| per satellite: hash key + load timestamp | the satellite version valid at the snapshot, NULL when the satellite had no row yet |

Example: a customer hub with three satellites from three insurance systems. On one snapshot date the
third satellite has no row yet (NULL), on the next it has.

## Bridge table

A query helper that connects hubs and links but holds no satellite attributes.

- It shifts data to the grain the business wants (for example one row per invoice with its customer and
  store).
- It can carry calculated fields such as aggregates, which helps performance downstream.
- It keeps the link hash keys, so link satellites can be joined for descriptive attributes.

Example columns: `hk_invoice`, `hk_customer`, `hk_store`, snapshot date, invoice number, total.

## Exploration link

A computed link created for a business reason. It connects hubs that were not connected in the raw vault.
Example: the raw vault links airplane to part and part (engine) to manufacturer; an exploration link
connects airplane directly with manufacturer.

**Same-as link** is a special exploration link: it identifies clusters of entries in the same hub that
describe one real object (two customer keys from two systems for the same person). It is how the vault
records a consolidation decision from `da-consolidation` without touching the hub.

## Computed aggregate link

Removes one hub from a link and aggregates by the remaining relationships. Example: a flight link
(airport of origin, airport of destination, carrier, flight number) becomes a service link without the
flight hub, with the number of flights aggregated into a `FlightCount` attribute. Once a link is
calculated from raw data (a `GROUP BY`), it belongs to the business vault.

## Computed satellite

Stores business-rule changed data: the result of an aggregation, summarisation, correction or evaluation,
or of a data quality, cleansing or address correction routine. Example: a satellite on the service link
that holds the aggregated flight count. A cleaned copy of a raw satellite (`customer_p_s` →
`customer_p_qa_bs`) is a computed satellite too.

## Where to compute what

| Need | Business vault entity |
|---|---|
| a cleaned or corrected attribute | computed satellite |
| state as of a date across satellites | PIT |
| a different grain for a mart | bridge |
| a relationship that exists only through a rule | exploration link |
| two keys of one real object | same-as link |
| a count or sum over a link | computed aggregate link, or a computed satellite on it |
