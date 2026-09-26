# 08 — Bronze-to-Silver Transformation

The screenshots extend the raw ERD into a Lakehouse pipeline. The raw model should not be confused with the Silver model: source duplication is acceptable in raw/Bronze but should be reconciled in Silver.

## Transformation strategy

```mermaid
flowchart LR
    B[Bronze near-source table] --> T1[Type enforcement]
    T1 --> T2[Null and format normalization]
    T2 --> T3[Deduplication]
    T3 --> T4[Referential-integrity checks]
    T4 --> T5[Conformance]
    T5 --> S[Silver domain table]
```

## Example: repeated race attributes

The raw `results` and `sprints` datasets repeat `date`, `raceName` and `url`, even though comparable information already exists in `races`. Silver can use `(season, round)` as the authoritative event key and compare the repeated fields for consistency.

```mermaid
flowchart TB
    R[races\nseason + round\nraceName/date/url] --> C{Compare repeated\nsource values}
    RE[results\nraceName/date/url] --> C
    SP[sprints\nraceName/date/url] --> C
    C -->|consistent| S[Conformed Silver event/session data]
    C -->|mismatch| Q[Quarantine / data-quality issue]
```

## Suggested Silver responsibilities by entity

| Entity | Silver responsibility |
| --- | --- |
| `circuits` | type coordinates, standardize text/country fields, enforce unique `circuitId` |
| `races` | enforce date/season/round types, validate circuit, standardize event identity |
| `constructors` | normalize names/nationalities, enforce unique `constructorId` |
| `drivers` | normalize names, dates and nationalities, enforce unique `driverId` |
| `results` | type metrics, validate all foreign keys, deduplicate composite key |
| `sprints` | same validation pattern as results, maintained as separate session domain |

## Optional conformed session view

Source tables should remain separate in raw/Bronze. Silver or Gold may expose a unioned analytical view if useful:

```mermaid
flowchart LR
    R[Silver results] --> U[Unified session result view]
    S[Silver sprints] --> U
    U --> ST[session_type\nRACE or SPRINT]
```

This is an analytical convenience, not a reason to destroy source lineage.

Next: [09 — Gold Dimensional Model](09_gold_dimensional_model.md)
