# 02 — Lakehouse Layers

The screenshots depict a classic layered Lakehouse progression. The following diagram expresses that structure explicitly.

```mermaid
flowchart TB
    A[Landing / source files\nExact delivered datasets] --> B[Bronze\nDelta + ingestion metadata]
    B --> C[Silver\nCleaned, typed, standardized, deduplicated]
    C --> D[Gold\nBusiness-ready dimensions and facts]

    B -. preserves source traceability .-> A
    C -. validates against .-> B
    D -. aggregates and models .-> C
```

## Landing and raw

Landing is the immutable or minimally touched delivery area. The raw model mirrors the Formula 1 source structure and therefore deliberately retains repeated race attributes and separate `results`/`sprints` datasets.

## Bronze

Bronze is the first managed Lakehouse representation. A practical implementation keeps the source columns intact while adding technical ingestion information such as load timestamps, source-file lineage or batch identifiers. The exact technical column names are implementation-specific and should match the physical catalog.

## Silver

Silver is where source irregularities are resolved. Typical responsibilities are:

- parse and enforce data types;
- standardize null handling;
- validate primary and foreign keys;
- remove duplicate deliveries;
- standardize names and dates;
- reconcile repeated `raceName`, `date`, and `url` values against the authoritative race entity;
- expose reusable domain tables.

## Gold

Gold transforms the conformed domain into analytics-ready structures. The screenshots indicate a dimensional/star-model direction, where descriptive entities become dimensions and measurable race/session outcomes become facts.

```mermaid
flowchart LR
    B1[Bronze circuits] --> S1[Silver circuits]
    B2[Bronze races] --> S2[Silver races]
    B3[Bronze constructors] --> S3[Silver constructors]
    B4[Bronze drivers] --> S4[Silver drivers]
    B5[Bronze results] --> S5[Silver results]
    B6[Bronze sprints] --> S6[Silver sprints]

    S1 --> G1[Gold dimensions]
    S2 --> G1
    S3 --> G1
    S4 --> G1
    S5 --> G2[Gold facts]
    S6 --> G2
```

Next: [03 — Raw Model Overview](03_raw_model_overview.md)
