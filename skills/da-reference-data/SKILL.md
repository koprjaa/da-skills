---
name: da-reference-data
description: Unifies code lists and reference data across sources in the integrated layer. Designs core code lists with the meaning of each code, maps from source code to core code (code maps), rules for uniqueness and coverage, handling of unknown and unmapped values, and controlled change of reference data. In review mode it finds mappings hidden in reports, ambiguous maps and values that lead nowhere. Use it when "every system codes order status differently", "unify payment methods and categories", "map countries and currencies", "code map", "reference data", "why is there a category Other in the report", even if the user never says code list. Keywords: code list, reference data, code map, code mapping, lookup, LOV, classification, countries, currencies, categories, mapping coverage.
---

# Reference data: one code for one meaning

Runs alongside consolidation. Every source codes statuses, types and categories its own way: "PAID",
"zaplaceno", "2". The integrated layer needs one core code with a clear meaning and a map that assigns
every source value exactly once.

Place in the pipeline: after `da-approach`, alongside `da-consolidation`, before `da-core-model`. The
surface form of values (case, whitespace, typos) is normalised by `dq-standardizer`; this skill decides
about meaning and structure.

## Reference data versus master data

Reference data is non-transactional, classifies other data and changes rarely: currency codes, country
lists, product categories, loyalty tiers, time zones. Master data, in contrast, describes specific
subjects (customer, product) and is consolidated in `da-consolidation`. Keep them apart, because they are
governed differently: a code list is mapped, a master entity is matched.

| Kind | Examples | Who is the authority |
|---|---|---|
| external standard | countries, currencies, units, industry classification | the standard (ISO and similar), adopted, not invented |
| internal domain | order status, payment type, product category, promotion type | the process owner in the organisation |
| derived group | price band, age cohort | a definition in the data dictionary, not in the source |

Reference data management covers defining values, categorising them and maintaining them. Reference data
is shared across the organisation and has a steward; a change of value should go through a defined
process and be approved and announced before it is deployed.

## Procedure

### 1. Value inventory

For every code attribute in every source, list all values actually used, with their frequency:

```sql
SELECT 'B' AS source, payment_method AS value, COUNT(*) AS n FROM src_b.payment GROUP BY payment_method
UNION ALL
SELECT 'C', payment_type, COUNT(*) FROM src_c.payment GROUP BY payment_type
ORDER BY source, n DESC;
```

Include values from the source's own code lists that no transaction uses yet. A code list is valid even
without transactions, and if you leave it out, the first new transaction with that value ends up
unmapped.

### 2. Core code list

Design the core codes with their meaning. A code without a definition is just another spelling of the
same confusion.

| core_code | Meaning | Valid from | Valid to |
|---|---|---|---|
| CARD | card payment (online and terminal) | 2019-01-01 | |
| CASH | cash at the till | 2019-01-01 | |
| TRANSFER | bank transfer against an invoice | 2019-01-01 | |
| UNKNOWN | the source does not state the value | | |

Set the granularity of the core code list from the questions in `da-context`. If nobody needs to tell
online card payments from terminal ones, do not split them. If someone does, do not merge them, because
splitting them later will not be possible.

### 3. Code map

```sql
CREATE TABLE core.code_map (
  domain        VARCHAR(30)  NOT NULL,   -- 'payment_method', 'country', ...
  source_system VARCHAR(10)  NOT NULL,
  source_value  VARCHAR(100) NOT NULL,   -- original value, unchanged
  core_code     VARCHAR(30)  NOT NULL,
  note          VARCHAR(200),
  PRIMARY KEY (domain, source_system, source_value),   -- every source value exactly once
  FOREIGN KEY (domain, core_code) REFERENCES core.code_list (domain, code)
);
```

Rules of the map:

- **Every source value in use is mapped exactly once.** The primary key enforces it.
- **The map does not create source values that do not exist.** Do not add rows "just in case" for values
  absent from the source; it blurs coverage.
- **No ambiguity.** When one source value means two things depending on context ("OTHER"), record it as
  a limitation and map it to a code that admits the uncertainty, not to one of the two meanings at random.
- **The original value stays.** Core carries the core code; the original text stays in staging and in
  lineage.
- **Unknown is not an error.** A missing value in the source goes to `UNKNOWN`, not to the most frequent
  code.

### 4. Coverage

```sql
-- unmapped values that occur in the data
SELECT s.source_system, s.value, s.n
FROM stage_values s
LEFT JOIN core.code_map m ON m.domain = s.domain AND m.source_system = s.source_system
                         AND m.source_value = s.value
WHERE m.core_code IS NULL;
```

Coverage report: number of source values, how many are mapped, how many go to `UNKNOWN`, how many remain
unmapped and why. An unexplained unmapped value is a defect of the map, not of the data.

Put the check into the pipeline: a value that does not exist today and arrives tomorrow must stop the
load or land in quarantine, not quietly fall into `UNKNOWN`.

## Code list history

Code lists change: a category is renamed, split, retired. Decide whether reports should see the state as
of the transaction date or the current state. For the first, keep valid from/to (the same principle as
SCD2 in `da-core-model`). A category renamed today must not quietly rewrite past reports unless the
context wants that.

## Review

| Check | Anti-pattern | Finding |
|---|---|---|
| is the mapping stored as data? | mapping in CASE expressions inside reports, different in every report | C/G: map outside the warehouse |
| is the map unambiguous and complete? | the same source value in two map rows, unmapped values in the data | C: ambiguity, coverage gap |
| do core codes have a definition? | codes without meaning, "Other" as a dumping ground | C/G: missing definition |
| are source values overwritten? | original text lost, the mapping cannot be verified | I: lost lineage |
| is code list change controlled? | a category renamed in production, historical reports changed | G: uncontrolled change |
| does a new value fall into `UNKNOWN` without a warning? | the share of unknowns grows unnoticed | I: missing control |

## Output of reference data

A list of domains, core code lists with meaning and validity, a code map for each domain, a coverage
report explaining unmapped and unknown values, and the rule for new values in the pipeline. Continue with
`da-core-model`.
