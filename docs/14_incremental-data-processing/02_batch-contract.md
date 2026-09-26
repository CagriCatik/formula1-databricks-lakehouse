# Batch Contract

Each immediate child directory of the configured landing volume is treated as a
batch identifier. The job sorts folder names lexically and processes the first
one that is not present in the control table.

Use names whose lexical order matches business order:

- ISO dates: `2026-09-01`, `2026-09-08`
- Zero-padded sequence numbers: `batch_0001`, `batch_0002`

Avoid unpadded names such as `batch_1`, `batch_2`, `batch_10`, because lexical
ordering would place `batch_10` before `batch_2`.

Batch identifiers must match `[A-Za-z0-9][A-Za-z0-9._-]*`. This keeps volume
paths and Delta `replaceWhere` predicates unambiguous; whitespace, slashes, and
quotes are rejected by the Bronze writer.

## Expected layout

```text
/Volumes/<catalog>/landing/files/<batch_id>/
├── circuits.csv
├── races.csv
├── constructors.json
├── drivers.json
├── results/
│   └── *.json
└── sprints/
    └── *.json
```

Publish a batch atomically: upload to a temporary location, verify all expected
objects, and only then move or copy it beneath the watched landing path. This
prevents the job from observing a partially uploaded batch.

## Replay and recovery

Failed runs leave the batch as `in_progress` so it is not silently selected as
new work. After correcting the underlying problem, use the Lakeflow Jobs repair
run feature to rerun failed and downstream tasks. The Bronze, Silver, and Gold
writers are designed around deterministic keys and Delta merges.

To intentionally replay a completed batch, first assess downstream effects,
then remove or reset only that batch's control row and start a new run. Treat
this as an operator action; do not automate deletion of control history.
