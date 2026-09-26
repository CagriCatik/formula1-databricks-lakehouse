# 14 — Glossary

| Term | Meaning in this project |
| --- | --- |
| Landing | File-oriented delivery area before managed transformation |
| Raw | Source-aligned logical model retaining original payload semantics |
| Bronze | First managed/Delta representation with lineage and ingestion context |
| Silver | Cleaned, typed, deduplicated and conformed domain data |
| Gold | Business-ready dimensional/fact model for analytics |
| Grain | What exactly one row represents |
| Primary key | Column or column set uniquely identifying a row |
| Foreign key | Column or column set referencing another entity |
| Composite key | Key built from more than one column |
| Natural/business key | Identifier derived from the source/domain, such as `driverId` |
| Surrogate key | Warehouse-generated dimension key, usually used in Gold |
| Referential integrity | Requirement that referenced parent records actually exist |
| Conformance | Standardizing entities/attributes so they are reused consistently |
| Fact | Table of measurable business events/outcomes |
| Dimension | Descriptive table used to analyze facts |
| Session type | Analytical discriminator such as `RACE` or `SPRINT` |

Back to [README](README.md).
