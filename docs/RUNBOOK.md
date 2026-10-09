# Runbook

Every step here is a plain `databricks bundle` command. Always pass `-t <target>` (`dev`, `staging` or `prod`). On a laptop, add `-p <profile>`. In CI, authenticate a service principal with `DATABRICKS_CLIENT_ID` and `DATABRICKS_CLIENT_SECRET`; the host always comes from the target file. `<key>` is a dashboard key such as `nyctaxi_trips`.

Validate dev with plain `validate`: dev can warn about permissions on your own user folder, and `--strict` turns that warning into an error. Validate staging and prod with `validate --strict`.

## Playbooks

### Add a dashboard

1. Build it in the dev workspace UI. Use unqualified table names (`FROM trips`); see [BYO-DASHBOARDS](BYO-DASHBOARDS.md).
2. Pull it into the repo (this writes local files only):
   `databricks bundle generate dashboard --existing-id <dashboard-id> --key <key> -s dashboards/<key> -d dashboards/<key> -t dev`
3. Copy `dashboards/nyctaxi_trips/dashboard.yml` to `dashboards/<key>/dashboard.yml` and rename the variable to `dash_<key>` and the file to `./<key>.lvdash.json`, and set its `display_name` and `description`. Delete the generated `<key>.dashboard.yml`.
4. Run the [BYO checklist](BYO-DASHBOARDS.md#checklist), then `databricks bundle validate -t dev`.
5. Open a pull request. Nothing is deployed until a target assigns the dashboard.

### Assign or unassign a dashboard

- **Assign:** add `<key>: ${var.dash_<key>}` under `resources.dashboards` in the target file. `databricks bundle plan -t <target>` should show only `create`.
- **Unassign:** this deletes the dashboard from that workspace unless you unbind it first. Follow [Retire a dashboard](#retire-a-dashboard).

### Add a workspace

1. Copy `targets/prod.yml` (a shared workspace) or `targets/dev.yml` (personal copies) to `targets/<name>.yml`.
2. Rename the target key to `<name>`. Set `host`, `catalog` and `schema`, and keep only the dashboards this workspace should get. For shared workspaces keep `root_path` and `protect: true`. Protect blocks deleting a dashboard while it is still defined in the bundle; removing it from the target is how you retire it (see [Retire a dashboard](#retire-a-dashboard)).
3. If the workspace has no warehouse named `Serverless Starter Warehouse`, set `warehouse_id` under the target's `variables`.
4. Give viewers `USE CATALOG`, `USE SCHEMA` and `SELECT` on the data.
5. Run `databricks auth login --host <workspace-url> -p <name>`, then `databricks bundle validate --strict -t <name> -p <name>` and `databricks bundle plan -t <name> -p <name>` (expect only `create`).
6. Add the target to your CI pipeline and open a pull request.

### Change an existing dashboard

1. Deploy your dev copy: `databricks bundle deploy -t dev`.
2. Edit your `[dev <name>]` copy in the UI.
3. Pull the edit back into the JSON: `databricks bundle generate dashboard --resource <key> --force -t dev`. Here `--force` only overwrites local files. The first pull may reformat the `.lvdash.json` (for example, joined query lines); that one-time diff is expected. Always pull from dev, never from staging or prod.
4. Review the diff, commit and open a pull request.
5. CI runs `plan` for every target. Check that only the expected dashboards change and nothing is deleted or recreated.
6. After merge, CI deploys staging, then prod.

### Promote

The order is dev, then staging, then prod. People deploy dev from their laptops. CI deploys staging and prod from `main`, each behind its own approval. Each shared stage runs:

```bash
databricks bundle validate --strict -t <target>
databricks bundle plan -t <target>
databricks bundle deploy -t <target>
```

If a stage fails, the later stages do not run.

### Roll back

Run `git revert <commit>`, open a pull request and read the plans (a content rollback shows `update`). Merge and promote as usual. If the revert removes an assignment, it is a delete: follow [Retire a dashboard](#retire-a-dashboard) instead.

### Retire a dashboard

Removing the assignment line without unbinding first deletes the dashboard. To keep the dashboard in the workspace but stop managing it:

1. Open a pull request that removes `<key>: ${var.dash_<key>}` from the target file. Do not merge it yet.
2. Pause deploys of that target in CI.
3. From a checkout of `main`, a person runs `databricks bundle deployment unbind <key> -t <target>`.
4. Merge the pull request. `databricks bundle plan -t <target>` should show no change for `<key>`. Resume deploys.
5. When no target assigns the dashboard any more, delete `dashboards/<key>/` in a later pull request.

To delete the dashboard instead, merge the removal, then a person runs a plain `databricks bundle deploy -t <target>` in a terminal and answers the delete prompt. The dashboard goes to Trash.

### Adopt an existing dashboard

Use this when a dashboard already exists in a workspace and must keep its ID and URL.

1. Pull it in with `databricks bundle generate dashboard --existing-id <dashboard-id> --key <key> -s dashboards/<key> -d dashboards/<key> -t <target>`, then convert it as in [Add a dashboard](#add-a-dashboard).
2. If the dashboard is not in the target's default folder (`<root_path>/resources`), add `parent_path: /Workspace/<its current folder>` to the definition. Dashboards cannot be moved, so a different folder means a new dashboard.
3. Assign it in the target file. `databricks bundle plan -t <target>` shows `create`. Do not deploy yet, or you get a second copy.
4. A person runs `databricks bundle deployment bind <key> <dashboard-id> -t <target>` and confirms the prompt.
5. `databricks bundle plan -t <target>` must now show `update`, not `create` or `recreate`. If it does not, run `databricks bundle deployment unbind <key> -t <target>`, fix the folder and repeat from step 3.
6. Open a pull request and deploy through CI. Repeat for each workspace that already has a copy.

## CI wiring

This repo ships no pipeline file. Every stage starts from a fresh checkout, installs Databricks CLI v1.20.x and signs in with that environment's service principal (for example OAuth M2M: `DATABRICKS_CLIENT_ID` and `DATABRICKS_CLIENT_SECRET`). Each stage needs credentials for its own target workspace, including the `dev` validate.

**On every pull request** (no approval needed):

```bash
# Each check must print nothing
grep -nE '^[[:space:]]+dash_[a-z0-9_]+:' targets/*.yml     # targets must not redefine dashboards
grep -n '^variables:' targets/*.yml                        # targets must not add top-level variables
grep -hE '^  [A-Za-z0-9_]+:' databricks.yml dashboards/*/dashboard.yml | sed 's/:.*//' | sort | uniq -d   # duplicate variable names
jq -r '.datasets[].queryLines[]?' dashboards/*/*.lvdash.json | grep -E '[A-Za-z0-9_`-]+\.[A-Za-z0-9_`-]+\.[A-Za-z0-9_`-]+'   # 3-part table names

# Then, for every target
databricks bundle validate -t dev
databricks bundle validate --strict -t staging
databricks bundle validate --strict -t prod
databricks bundle plan -t <target>
```

`grep` succeeds when it finds something, so make any output fail the build. Post the plans on the pull request.

**After merge to `main`:** one stage per shared target (staging, then prod). Each stage has its own CI environment, required approvers and credentials, and runs the three commands in [Promote](#promote). Allow only one deploy per target at a time.

Also fail the build if your pipeline file contains `deploy --auto-approve`, `deploy --force`, `deploy --plan` or `--force-lock`.

## Safety rules

Only ever run a plain `databricks bundle deploy -t <target>`. Never add:

- `--auto-approve`: it approves deletes and recreates without asking.
- `--force`: it skips the Git branch check and the "modified remotely" check, so it can overwrite UI edits on every dashboard in the target.
- `--plan <file>`: it skips the check for UI edits made after the plan was saved.
- `--force-lock`: it lets two deploys of the same target run at once.

Without these flags, a deploy in CI refuses to delete or recreate a dashboard, and stops if a dashboard was edited in the UI. That refusal is the safety net, not a bug.

## Things to know

- **Use unqualified table names** (`FROM trips`). A full `catalog.schema.table` name ignores the target's `catalog` and `schema`.
- **UI edits stop the deploy.** If someone edits a deployed staging or prod copy, `plan` warns and `deploy` stops with `has been modified remotely`. Make the change in dev and promote it instead.
- **A timed-out create may still have worked.** If a deploy times out while creating a dashboard, do not just retry. Look for it with `databricks lakeview list`. If it exists, a person binds it (`databricks bundle deployment bind <key> <dashboard-id> -t <target>`), then runs `plan` and deploys.
- **Unbind before removing from config.** Otherwise the dashboard is deleted.
- **Never commit `.databricks/`.** It is a local cache, keyed by target name.
- **Viewers need data access.** Dashboards run with the viewer's own permissions.
