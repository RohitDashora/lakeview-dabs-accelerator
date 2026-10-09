# Bring your own dashboard

This repo deploys the same `.lvdash.json` to every workspace and changes only the catalog, schema, warehouse, viewers and display-name suffix per target. That only works if the JSON follows a few rules. A file that breaks them can deploy without errors and still read the wrong data.

You need `jq`, `grep` and the Databricks CLI. Set the file you are checking once:

```bash
F=dashboards/<key>/<key>.lvdash.json
```

To bring in a dashboard that already exists in a workspace and must keep its ID and URL, see [RUNBOOK: Adopt an existing dashboard](RUNBOOK.md#adopt-an-existing-dashboard).

## Example: before and after

A typical export uses full table names, which ignore the per-target catalog and schema:

```json
"queryLines": [ "SELECT pickup_zip, fare_amount ", "FROM samples.nyctaxi.trips" ]
```

The fixed version uses the bare table name, and the bundle supplies the catalog and schema:

```json
"queryLines": [ "SELECT pickup_zip, fare_amount ", "FROM trips" ]
```

Keep each dataset's own `"catalog": "samples"` and `"schema": "nyctaxi"` keys (the dev values). They let the dashboard open in the dev UI, and the bundle replaces them on deploy.

## Checklist

Each check prints nothing when the file is fine.

1. **No 3-part table names.** Use `FROM trips`, not `FROM catalog.schema.trips`. For metric views, `asset_name` must not be 3-part either.

   ```bash
   jq -r '.datasets[].queryLines[]?' "$F" | grep -E '[A-Za-z0-9_`-]+\.[A-Za-z0-9_`-]+\.[A-Za-z0-9_`-]+'
   grep -nE '"asset_name": *"[^"]*\.[^"]*\.[^"]*"' "$F"
   ```

   A struct field such as `a.b.c` in a column expression also matches; check such hits by hand.

2. **No `${...}` in the JSON.** Variables are not substituted inside the file. Use dashboard parameters (`:name`) for anything that varies at run time.

   ```bash
   grep -nF '${' "$F"
   ```

3. **Dataset `catalog` and `schema` keys are the dev values only.** One catalog and schema per dashboard.

   ```bash
   jq -r '.datasets[] | select((.catalog // "samples") != "samples" or (.schema // "nyctaxi") != "nyctaxi") | .name' "$F"
   ```

   If your dev data is not `samples.nyctaxi`, use your dev `catalog` and `schema` values in this check.

4. **No server-owned fields** such as `etag`, `dashboard_id`, `create_time`, `update_time` or `lifecycle_state`.

   ```bash
   jq -r 'keys - ["datasets","pages","uiSettings"] | .[]' "$F"
   ```

   To remove them: `jq 'del(.etag, .dashboard_id, .create_time, .update_time, .lifecycle_state)' "$F" > "$F.tmp" && mv "$F.tmp" "$F"`.

5. **Names match.** The folder, the JSON file and the variable all use the same snake_case `<key>`: `dashboards/<key>/<key>.lvdash.json` and `dash_<key>` in `dashboards/<key>/dashboard.yml`. The key cannot change after the first deploy.

   ```bash
   for d in dashboards/*/; do k=$(basename "$d"); [ -f "$d$k.lvdash.json" ] && grep -q "^  dash_$k:" "${d}dashboard.yml" || echo "mismatch: $k"; done
   ```

6. **The definition uses variables.** `file_path` (not `serialized_dashboard`), `warehouse_id: ${var.warehouse_id}` (not a literal ID), plus `dataset_catalog`, `dataset_schema`, `embed_credentials: false`, `permissions` and `lifecycle`. Copy `dashboards/nyctaxi_trips/dashboard.yml` to start.

   ```bash
   grep -nE '^[^#]*serialized_dashboard|warehouse_id: *[0-9a-f]{16}' dashboards/*/dashboard.yml
   ```

Then assign the dashboard to `dev` in your local copy, run `databricks bundle deploy -t dev`, and open the URL from `databricks bundle summary -t dev` to check that every widget shows data.
