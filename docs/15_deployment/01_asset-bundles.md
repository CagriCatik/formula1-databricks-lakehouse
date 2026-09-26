# Databricks Bundle Deployment

The root `databricks.yml` and files in `resources/` define a Databricks
Declarative Automation Bundle. It deploys two Lakeflow Jobs and one AI/BI
dashboard. The incremental job runs the Python wheel built from `src/`
(see [The Production Python Package](02_python-package.md)); the full-refresh job
runs the notebooks under `notebooks/`, which the bundle synchronizes to the
workspace.

Building the wheel requires [uv](https://docs.astral.sh/uv/) on the machine that
runs `bundle deploy`.

## Authentication

The repository contains no workspace URL or credential. Configure a CLI profile:

```bash
databricks auth login --host https://<workspace-url> --profile formula1-dev
databricks auth profiles
```

For CI/CD, prefer workload identity federation or another short-lived machine
identity. Avoid personal access tokens when a federated option is available.

## Validate, deploy, run

```bash
databricks bundle validate -t dev --profile formula1-dev
databricks bundle deploy -t dev --profile formula1-dev
databricks bundle run formula1_full_refresh -t dev --profile formula1-dev
```

The production target changes the default catalog names. Provision those
catalogs and landing volumes before deployment, or override the variables with
names already approved in your workspace.

```bash
databricks bundle deploy -t prod --profile formula1-prod \
  --var="full_refresh_catalog=analytics,incremental_catalog=analytics_incr"
```

## Dashboard as code

`resources/formula1.dashboard.yml` deploys
`dashboards/formula1_lakehouse.lvdash.json`, an AI/BI dashboard with three pages:
season standings, all-time dominance, and pipeline health. Its dataset queries
use two-part names such as `gold.v_driver_standing`; the bundle supplies the
catalog through `dataset_catalog`, so the dashboard follows `incremental_catalog`
in each target.

The `warehouse_id` variable resolves the warehouse named
`Serverless Starter Warehouse` at deploy time. Override it when your workspace
uses a different warehouse:

```bash
databricks bundle deploy -t dev --profile formula1-dev \
  --var="warehouse_id=<warehouse-id>"
```

By default viewers run the queries with their own credentials, so they need
`SELECT` on the gold tables and views and on `control.batch_control`. To edit the
dashboard in the workspace and keep Git in sync, run
`databricks bundle generate dashboard --resource formula1_lakehouse --force`
after deploying.

The Season filter opens on 2025. That default is stored in the dashboard file, so
change it there (or in the workspace, then regenerate) when newer seasons load.

## Configuration boundaries

Bundle variables cover environment-specific object names and paths. Secrets do
not belong in bundle variables because resolved bundle configuration and job
parameters can be visible to operators. Store secrets in secret scopes or use
Unity Catalog storage credentials and external locations.

The jobs use serverless compute. The incremental job declares a serverless
environment that installs the wheel; the full-refresh job omits cluster
configuration. If the target
workspace does not support serverless jobs, add an approved `job_clusters`
definition or reference an existing cluster according to your platform policy.

