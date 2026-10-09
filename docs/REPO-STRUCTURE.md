# Repo structure and ownership

## Layout

```
.
├── databricks.yml                  # bundle name, CLI version, includes, shared variables
├── dashboards/
│   ├── nyctaxi_trips/
│   │   ├── dashboard.yml           # defines dash_nyctaxi_trips
│   │   └── nyctaxi_trips.lvdash.json
│   └── nyctaxi_zones/
│       ├── dashboard.yml           # defines dash_nyctaxi_zones and nyctaxi_zones_viewers
│       └── nyctaxi_zones.lvdash.json
├── targets/
│   ├── dev.yml                     # personal dev copies
│   ├── staging.yml                 # shared, trips only
│   └── prod.yml                    # shared, trips and zones
├── docs/                           # RUNBOOK, BYO-DASHBOARDS, this file
├── CODEOWNERS
└── README.md
```

What a dashboard **is** lives in `dashboards/`. **Where** it is deployed, and with which values, lives in `targets/`. The two never share a file, so each folder can have its own owner.

## Who owns what

| Role | Owns | Typical change |
|---|---|---|
| Dashboard builders | `dashboards/`, `docs/BYO-DASHBOARDS.md` | Add or change a dashboard |
| Platform ops | `targets/`, `docs/RUNBOOK.md` | Add a workspace, assign or unassign a dashboard, set catalog, schema or viewers |
| CI/CD | `databricks.yml`, `CODEOWNERS`, `README.md`, the pipeline | CLI version, shared variables, CI checks |
| Reviewers | Pull requests | Approve changes; read the plan for every target |
| Release | Promotion approvals | Approve each staging and prod deploy |

Rules that keep the split clean:

- Targets set values (`catalog`, `schema`, `name_suffix`, `protect`, `warehouse_id`, and per-dashboard settings that builders declare, such as `nyctaxi_zones_viewers`). Targets never set a `dash_*` variable, because that silently replaces the whole dashboard definition for that workspace.
- A new per-dashboard setting is declared by builders in `dashboard.yml` first, then set by ops in a target.
- Builders who want to see a new dashboard in dev assign it in their local copy only; ops commit the real assignment.

## How it is enforced

- **CODEOWNERS.** Each path has one owning team, so a change to `dashboards/` needs a builder's approval and a change to `targets/` needs platform ops. Replace the `@your-org/...` placeholders with your teams.
- **Branch protection on `main`.** Require pull requests, code owner review, at least one approval and a passing CI check; block force-pushes.
- **CI environment approvals.** Staging and prod each have their own CI environment with the release team as required approvers and their own credentials, so staging credentials cannot reach prod.
- **The `dash_` check.** CI fails if any `dash_*` key appears under `targets/` (see [RUNBOOK: CI wiring](RUNBOOK.md#ci-wiring)). CODEOWNERS cannot check file contents, so this check covers that gap.
