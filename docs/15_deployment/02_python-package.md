# The Production Python Package

The repository has two tracks that are deliberately kept apart:

| Track | Location | Purpose | Deployed by the incremental job |
| --- | --- | --- | --- |
| **Production package** | `src/formula1_lakehouse/` | The incremental pipeline as a tested, versioned Python wheel | Yes |
| **Experimental notebooks** | `notebooks/` | The course material and exploratory work the package was derived from | No |

The notebooks stay because they teach the concepts step by step. They are not a
place to put logic that production depends on: notebooks cannot be imported, unit
tested or type-checked, and `%run` shares one global namespace between files.

## Package layout

```text
pyproject.toml              # metadata, entry points, tool configuration, dependency groups
uv.lock                     # locked development environment
src/formula1_lakehouse/
    config.py               # Settings: catalog, landing path, table names
    batch.py                # batch id rules
    sources.py              # raw file definitions and enforced schemas
    bronze.py               # landing files -> batch-partitioned Delta tables
    silver.py               # pure transform_* functions + merge definitions
    gold.py                 # pure build_* functions + merge definitions + reference data
    merge.py                # order-safe MERGE INTO used by silver and gold
    control.py              # batch_control: identify, register, complete
    analytics.py            # standings views (SQL lives in sql/*.sql)
    runtime.py              # the only adapter for Databricks runtime objects
    cli.py                  # one sub-command per job task
tests/                      # pytest, hermetic local Spark + Delta
```

Pure transformations (`transform_*`, `build_*`) take DataFrames and return a
DataFrame. Everything that reads or writes tables takes the `SparkSession` and
`Settings` as arguments. Nothing in the package imports `dbutils` or reads
widgets; `runtime.py` is the single place that touches Databricks runtime
objects, and it imports them lazily because `databricks.sdk.runtime` tries to
authenticate as soon as it is imported outside Databricks.

## How the job runs it

Every task in `resources/incremental.job.yml` is a `python_wheel_task` that calls
the package entry point `main` with one sub-command:

| Sub-command | Job task(s) | What it does |
| --- | --- | --- |
| `init-control` | `initialize_control_table` | Creates the control schema and `batch_control` |
| `identify-next-batch` | `identify_next_batch` | Publishes `p_batch_id` and `has_batch` as task values |
| `create-batch` / `complete-batch` | `create_batch`, `complete_batch` | Batch lifecycle |
| `bronze --entity <name>` | `bronze_*` | Ingest one entity of the batch |
| `silver --entity <name>` | `silver_*` | Transform and merge one entity |
| `build-reference` | `gold_nationality_reference` | Refresh the nationality-to-region table |
| `gold --table <name>` | `gold_*` | Build and merge one gold table |
| `build-view --view <name>` | `analytics_*` | Create or replace a standings view |

All sub-commands take `--catalog-name`; batch-aware ones take `--batch-id`, and
`bronze` and `identify-next-batch` also take `--landing-folder-path`. The job
passes them explicitly from `{{job.parameters.*}}` and
`{{tasks.identify_next_batch.values.p_batch_id}}`, so a run can still be started
with different job parameters. The same commands work from a terminal:

```bash
formula1-lakehouse silver --entity results --batch-id 2026-09-08 --catalog-name formula1_incr
```

Job parameters are not forwarded to a wheel task implicitly: `argv` contains
exactly the task's `parameters`, so every value a command needs must be passed
explicitly, as the job does. `identify-next-batch` publishes its task values with
`dbutils.jobs.taskValues` through `databricks.sdk.runtime`; this works from a
`python_wheel_task` on serverless environment version 5, and the condition task
and the `{{tasks.identify_next_batch.values.*}}` references resolve as usual.

The task graph, parallelism, condition task and no-op behaviour are unchanged
from the notebook version.

## Build and deploy

`databricks.yml` declares the wheel as a bundle artifact
(`build: uv build --wheel`), and the job's serverless environment installs
`../dist/*.whl`. `bundle deploy` therefore builds, uploads and installs the
package; install [uv](https://docs.astral.sh/uv/) once on the machine that
deploys.

```bash
databricks bundle validate -t dev --profile <profile>
databricks bundle deploy -t dev --profile <profile>
databricks bundle run formula1_incremental -t dev --profile <profile>
```

The version comes from `__version__` in `src/formula1_lakehouse/__init__.py`.
Bump it when you deploy a change, otherwise an environment may reuse a cached
wheel with the same version.

The wheel declares no runtime dependencies. `pyspark`, `delta-spark` and
`databricks-sdk` are provided by the Databricks environment; listing them would
make the environment try to install or replace them.

## Local development

```bash
uv sync                      # creates .venv with the dev and spark dependency groups
uv run ruff check src tests
uv run ruff format --check src tests
uv run mypy                  # strict
uv run pytest
```

The tests run without a workspace. They start a local Spark session with Delta
Lake and use Spark's built-in session catalog, addressable as `spark_catalog`,
to get the same three-part table names as production. This needs Java 17 (or
11). On Windows, run the suite in WSL, or install `winutils` for Hadoop; local
Spark does not start on Windows without it.

`pyspark` and `delta-spark` are pinned as a pair in `pyproject.toml`. Do not
float them to the newest 3.5.x without running the tests: `pyspark` 3.5.9
breaks Delta overwrites (`does not support truncate in batch mode`).

To run the package against a workspace from your machine, use the `connect`
dependency group instead of `spark` (Databricks Connect brings its own
`pyspark`, so the two groups are mutually exclusive):

```bash
uv sync --group connect --no-group spark
```

## Behaviour that differs from the notebooks

The package reproduces the notebook logic, with these deliberate differences:

- **Null business keys are dropped in every silver table.** The notebooks did
  this for circuits, results and sprints only. A row with a null key part can
  never match in the `MERGE`, so it would be inserted again on every run.
- **`complete-batch` fails for a batch that was never registered.** The notebook
  logged success even when no control row changed.
- **Merges are SQL `MERGE INTO`,** not the `DeltaTable` Python API, so they
  behave identically on Databricks, serverless, Databricks Connect and local
  Delta. The order guard is unchanged: a matched row is updated only when the
  incoming `batch_id` is the same or newer, or the stored one is null.
- **The control table is created `USING DELTA` explicitly.**
- **Landing folders are listed with `pathlib`** on the mounted volume path,
  instead of `dbutils.fs.ls`.
