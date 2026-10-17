---
name: da-mart
description: Designs the presentation layer of a data warehouse, the data marts for specific departments and questions. Follows Kimball's four steps (process, grain, dimensions, facts), star and snowflake schemas, conformed dimensions, KPI definitions with their limitations, dependent marts that read only from core, dimension history, OLAP cubes and the semantic layer. In review mode it finds mixed grain, double counting, different definitions of the same measure in different marts and logic hidden in the reporting tool. Use it for "design a mart for sales", "star schema", "facts and dimensions", "how to calculate a KPI", "why does every report show a different revenue", "dashboard on the warehouse", "cube", even if the user never says mart. Keywords: data mart, presentation layer, star schema, snowflake, fact table, dimension, grain, conformed dimension, KPI, OLAP, cube, Kimball.
---

# Mart: data for specific questions

Eighth step of design mode. A mart is a subset of the warehouse for one department or area (sales,
finance, marketing), shaped so that questions are easy to ask. Users mostly see this layer, so this is
where it is decided whether the warehouse gets used and trusted.

Place in the pipeline: after `da-integration`, before `da-governance`. A mart reads only from core. A mart
loaded straight from sources (an independent mart) bypasses consolidation and brings back several
versions of the truth.

## Dependent or independent

| | Dependent mart | Independent mart |
|---|---|---|
| source | the warehouse core | operational systems directly |
| identities and codes | shared, from consolidation | each mart solves them alone |
| consequence | the same number in every mart | unintegrated data, unaligned metadata, hard cross-mart analysis |

The default in this pipeline is a dependent mart. An independent mart is acceptable only as a temporary
solution with a recorded decision and a plan to integrate it.

## The four steps

**1. Choose the business process.** Invoicing, orders, production quality, marketing campaigns. Not a
department and not a report, but a process people ask questions about. The questions are in
`da-context`.

**2. Declare the grain.** The grain of the fact is the level one row represents, and it should be as low
as possible, a level that cannot be split further: a receipt line, not a receipt; one payment, not a daily
total. From the lowest grain you can aggregate up, never the other way. A wrong grain can make it
impossible to answer the questions the mart exists for. State the grain in a sentence and enforce it with
a unique key.

**3. Identify the dimensions.** The context that describes the facts: time (year, quarter, month, day),
place (address, region, country), customer, product, store, channel. Ask: how do users describe the data
this process produces? Feedback from analysts who work with the data helps.

**4. Identify the facts.** Numeric measures for each row: quantity, amount, tax, duration. Ask what you are
answering. Every measure must be true at the chosen grain: a receipt-level discount placed on every line
multiplies when summed.

The result is a star: a fact table in the middle, dimensions around it, linked by foreign keys.

## Facts and dimensions

| | Fact table | Dimension |
|---|---|---|
| content | measures of one event + dimension keys | descriptive attributes (name, category, country, segment) |
| size | large, grows with transactions | smaller, more stable |
| shape | narrow, many rows | denormalised for simple queries |

A "strategic customer" flag is a dimension attribute, not a fact. A number that cannot be summed or
averaged does not belong in a fact.

**Star versus snowflake.** In a snowflake at least one dimension is attached through another dimension
(country through customer). It saves redundancy and eases dimension maintenance, but adds joins and is
harder to read. The default for the presentation layer is the star; use a snowflake only where a
dimension hierarchy truly needs separate maintenance.

## Conformed dimensions

Calendar, customer, product and store should be one dimension shared by all marts (conformed), not a copy
in each mart. Only then can facts from different marts be combined and compared. Load the shared
dimension from the core identity (XREF from `da-consolidation`), not from a source.

Handle dimension history according to the decision in `da-core-model`: SCD2 with valid from/to, or the
attribute valid at the time of the transaction stored directly on the fact (delivery address on the
order). Whoever filters by date must know which date: order date and ship date give different revenue.

## KPIs

For every measure, a definition, not just a formula:

| KPI | Definition | Formula | Calculation grain | Limitations |
|---|---|---|---|---|
| revenue | sales net of VAT after reversals, as of the sale date | `SUM(f_sales.amount_net)` | line | returns only from 2021 |
| revenue by store | revenue split by store | `GROUP BY d_store` | line | online orders have no store → category "not stated" |

When a measure cannot be calculated unambiguously from the input, do not estimate it. Record the
limitation and either calculate it with an explicit "not stated" category or leave it out. A silent
estimate is worse than an admitted gap.

KPI definitions belong in the data dictionary (`da-governance`) and the calculation in the mart or the
semantic layer, not in individual reports. Users will use different reporting tools; when the main logic
is calculated in one of them, the others get a different number. Leave only minor touches in the
reporting tool: labels, formatting, documentation.

## Cubes and the semantic layer

An OLAP cube stores data multidimensionally and lets measures be aggregated and broken down by dimension
(region × year × product); more than three dimensions make a hypercube. Variants: MOLAP (multidimensional
storage), ROLAP (multidimensionality on a relational database), HOLAP (combination), DOLAP (a local subset
on a device). A modern semantic model in a reporting tool plays a similar role. Both are an optional layer
on top of the mart, not a replacement for it.

## Review

```sql
-- mixed grain / duplicates: the combination that should be unique
SELECT receipt_id, line_no, COUNT(*) FROM mart.f_sales
GROUP BY receipt_id, line_no HAVING COUNT(*) > 1;

-- facts without a dimension (anti-join)
SELECT COUNT(*) FROM mart.f_sales f LEFT JOIN mart.d_customer c ON c.customer_key = f.customer_key
WHERE c.customer_key IS NULL;

-- the same measure in two marts
SELECT SUM(revenue) FROM mart_sales.f_sales WHERE year = 2025;
SELECT SUM(income)  FROM mart_finance.f_income WHERE year = 2025;
```

| Check | Anti-pattern | Finding |
|---|---|---|
| does every fact have one recorded and enforced grain? | header and lines in one table, a discount applied on every line | M: mixed grain, double counting |
| does the mart read only from core? | a mart loaded straight from a source | M: independent mart, bypasses consolidation |
| are dimensions shared? | a different calendar and a different customer in each mart | M: marts cannot be compared |
| does the same measure have the same definition everywhere? | "revenue" with cost of goods in sales, without it in finance | M/G: conflicting KPIs |
| where is the logic calculated? | calculations in the reporting tool, different in every report | M: logic outside the warehouse |
| are KPI limitations admitted? | a measure estimated without a flag | M: hidden estimate |

## Output of the mart

For each mart: the business process, the grain as a sentence with a unique key, the star (fact,
dimensions, keys), the list of conformed dimensions, a KPI catalogue with definition, formula and
limitations, the dimension history decision, and optionally a cube or semantic model. Continue with
`da-governance`.
