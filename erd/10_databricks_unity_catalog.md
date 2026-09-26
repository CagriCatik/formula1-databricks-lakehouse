# 10 — Databricks and Unity Catalog Organization

One screenshot shows the Formula 1 project organized in Databricks with schemas corresponding to Lakehouse layers. The visible structure supports a catalog-level organization such as `formula1` with layer schemas including `bronze`, `silver`, `gold`, and a landing area.

## Logical catalog hierarchy

```mermaid
flowchart TB
    C[Catalog: formula1]
    C --> L[landing]
    C --> B[bronze]
    C --> S[silver]
    C --> G[gold]

    L --> LF[Source files / volumes]
    B --> BT[Near-source Delta tables]
    S --> ST[Conformed domain tables]
    G --> GT[Dimensions / facts / marts]
```

## Recommended namespace pattern

```text
formula1
├── landing
│   └── volumes / source deliveries
├── bronze
│   ├── circuits
│   ├── races
│   ├── constructors
│   ├── drivers
│   ├── results
│   └── sprints
├── silver
│   └── cleaned and conformed domain tables
└── gold
    └── analytical dimensions and facts
```

Only names that are clearly supported by the provided material are treated as definitive; exact physical Gold/Silver table names should be verified against the active catalog before production documentation is frozen.

## Ownership boundaries

```mermaid
flowchart LR
    ING[Ingestion jobs] --> B[bronze]
    DQ[Data-quality / transformation jobs] --> S[silver]
    MOD[Modeling / aggregation jobs] --> G[gold]
    BI[BI / analysts / SQL] --> G
```

This separation also maps naturally to permissions: ingestion writers need Bronze access, transformation workloads need Bronze-read/Silver-write, and most consumers can be restricted to Gold.

Next: [11 — End-to-End Walkthrough](11_end_to_end_walkthrough.md)
