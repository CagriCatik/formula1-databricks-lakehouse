# Operating the Incremental Job

## Deploy

```bash
databricks bundle validate -t dev --profile <profile>
databricks bundle deploy -t dev --profile <profile>
```

## Run

```bash
databricks bundle run formula1_incremental -t dev --profile <profile>
```

The job-level `catalog_name` and `landing_folder_path` parameters are passed to
every Python wheel task as command-line arguments. The discovery task publishes
`p_batch_id` and `has_batch` as task values. The bundle resource passes
`p_batch_id` into every batch-aware task and uses `has_batch` to gate the
processing branch.

## Observe

For each run, check:

1. `identify_next_batch` selected the expected folder.
2. Bronze row counts and source-file metadata match the batch contents.
3. Silver quality checks did not reject unexpected records.
4. Gold merge metrics are consistent with inserts and updates expected for the batch.
5. `complete_batch` changed the control row to `completed`.

Useful SQL:

```sql
SELECT *
FROM formula1_incr.control.batch_control
ORDER BY created_timestamp DESC;

DESCRIBE HISTORY formula1_incr.gold.fact_session_results;
```

## Failure behavior

- No available batch is a successful no-op.
- A task failure prevents downstream tasks and leaves the batch
  `in_progress`.
- Job concurrency is limited to one run.
- Job queuing is enabled, so a trigger received during an active run waits.
- Repair the failed run after resolving its cause; do not launch an unrelated
  run to bypass an `in_progress` batch.

