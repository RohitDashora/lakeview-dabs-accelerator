# Lakeview dashboards with Databricks Asset Bundles

> **Reference implementation for demo and learning purposes only.** This is not an official Databricks product and is not supported by Databricks. It is provided as-is, without warranty. Review, test and adapt it before using it with your own workspaces or data.

A small starter repo for keeping AI/BI (Lakeview) dashboards in Git and deploying them to several Databricks workspaces with plain `databricks bundle` commands. There are no scripts or custom tools. Copy it, replace the placeholders and adapt it to your own dashboards and workspaces.

It deploys dashboards only. It does not create data, catalogs or permissions on data.

## How it works

Each dashboard is **defined once** as a variable named `dash_<key>` in `dashboards/<key>/dashboard.yml`:

```yaml
# dashboards/nyctaxi_trips/dashboard.yml (shortened)
variables:
  dash_nyctaxi_trips:
    type: complex
    default:
      display_name: "NYC Taxi Trip Overview${var.name_suffix}"
      file_path: ./nyctaxi_trips.lvdash.json
      warehouse_id: ${var.warehouse_id}
      dataset_catalog: ${var.catalog}
      dataset_schema: ${var.schema}
```

Each workspace has one target file that sets its own values and **assigns** dashboards with one line each:

```yaml
# targets/prod.yml (shortened)
targets:
  prod:
    variables: { catalog: samples, schema: nyctaxi }
    resources:
      dashboards:
        nyctaxi_trips: ${var.dash_nyctaxi_trips}
```

The `${var.*}` values are resolved per target, so the same definition reads a different catalog, gets a different viewer list and so on in each workspace. A dashboard is only deployed where a target assigns it.

| Target | Mode | Dashboards |
|---|---|---|
| `dev` (default) | development: each person gets a personal `[dev <name>]` copy | trips, zones |
| `staging` | production, fixed folder, protected (see RUNBOOK) | trips |
| `prod` | production, fixed folder, protected (see RUNBOOK) | trips, zones (zones view-only) |

All three read the built-in `samples.nyctaxi` data. Point `catalog` and `schema` at your own data.

## Layout

| Path | What it holds |
|---|---|
| `databricks.yml` | Bundle name, CLI version, includes, shared variables |
| `dashboards/<key>/` | `dashboard.yml` (the definition) and `<key>.lvdash.json` (the dashboard itself) |
| `targets/<name>.yml` | One file per workspace: host, mode, values, assigned dashboards |
| `CODEOWNERS` | Which team approves changes to which folder |
| `docs/` | Runbook, guide for your own dashboards, repo structure and ownership |

## Quickstart

You need:

1. Databricks CLI **v1.20.x** (`databricks.yml` requires `>= 1.20.0, < 1.21.0`). See the [install guide](https://docs.databricks.com/aws/en/dev-tools/cli/install).
2. In each workspace, a SQL warehouse named `Serverless Starter Warehouse`, or change the name in `databricks.yml`.
3. The `samples` catalog available in each workspace (or your own data, see below).

Steps:

1. Replace the placeholder `host` in `targets/dev.yml`, `targets/staging.yml` and `targets/prod.yml` with your workspace URLs. Validation fails until you do.
2. Create one auth profile per workspace:

   ```bash
   databricks --version                                   # Databricks CLI v1.20.x
   databricks auth login --host <dev-workspace-url> -p dev
   databricks auth login --host <staging-workspace-url> -p staging
   databricks auth login --host <prod-workspace-url> -p prod
   databricks auth profiles                               # all three should be valid
   ```

3. Validate, preview and deploy to dev:

   ```bash
   databricks bundle validate -t dev -p dev
   databricks bundle plan     -t dev -p dev               # expect create only
   databricks bundle deploy   -t dev -p dev
   databricks bundle summary  -t dev -p dev               # shows the dashboard URLs
   ```

4. Check staging and prod without deploying: `databricks bundle validate --strict -t staging -p staging`, then `databricks bundle plan -t staging -p staging` (same for `prod`).

Deploy staging and prod from CI after review; see the [runbook](docs/RUNBOOK.md).

## Make it yours

1. **Workspaces.** Set each `host`. Add a workspace by copying `targets/prod.yml` (shared) or `targets/dev.yml` (personal) and renaming the target.
2. **Data.** Set `catalog` and `schema` in each target. Viewers need `USE CATALOG`, `USE SCHEMA` and `SELECT` on that data, because dashboards run with the viewer's own permissions.
3. **Names.** Choose your `bundle.name` and target names before the first shared deploy. They are part of the deployed paths, so renaming later creates new dashboards.
4. **Dashboards.** Add yours under `dashboards/<key>/` following [BYO-DASHBOARDS](docs/BYO-DASHBOARDS.md), and remove the NYC taxi examples when you no longer need them.
5. **Teams.** Replace the `@your-org/...` handles in `CODEOWNERS`.
6. **CI.** Add a pipeline as described in [RUNBOOK: CI wiring](docs/RUNBOOK.md#ci-wiring).

## Documentation

- [docs/RUNBOOK.md](docs/RUNBOOK.md): step-by-step playbooks, safety rules and CI wiring.
- [docs/BYO-DASHBOARDS.md](docs/BYO-DASHBOARDS.md): making your own `.lvdash.json` work in this repo.
- [docs/REPO-STRUCTURE.md](docs/REPO-STRUCTURE.md): who owns what and how it is enforced.
