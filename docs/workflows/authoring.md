# Authoring n8n Workflows as Files

Workflows in n8n are plain JSON. They can be written as files, kept in Git, and imported into the local instance. Building them by hand in the web editor is optional.

Verified against n8n `2.20.7`.

## File layout

One workflow per file, kebab-case, under `workflows/`:

```text
workflows/
  webhook-order-intake.json
  news-classifier.json
```

Workflows that need their own setup notes get a companion document under `docs/workflows/`:

```text
docs/workflows/analytics-slack-report.md
```

## JSON shape

Minimum structure n8n needs:

```json
{
  "name": "Webhook Order Intake",
  "nodes": [],
  "connections": {},
  "settings": { "executionOrder": "v1" }
}
```

Each node object uses these keys:

```json
{
  "parameters": {},
  "type": "n8n-nodes-base.set",
  "typeVersion": 3.4,
  "position": [220, 0],
  "id": "33333333-3333-4333-8333-333333333333",
  "name": "Edit Fields"
}
```

- `name` must be unique inside the workflow. Connections reference nodes by `name`, not by `id`.
- `id` is a UUID, local to the node.
- `position` is `[x, y]` on the canvas. Roughly 200 units of horizontal spacing keeps nodes readable.
- `typeVersion` is per node type and matters. Node types currently in use in this project:

  ```text
  n8n-nodes-base.manualTrigger   1
  n8n-nodes-base.webhook         2.1
  n8n-nodes-base.set             3.4
  n8n-nodes-base.httpRequest     4.4
  ```

`connections` is keyed by the source node name. Each entry lists output slots, and each slot lists its targets:

```json
{
  "Webhook": {
    "main": [
      [{ "node": "Edit Fields", "type": "main", "index": 0 }]
    ]
  }
}
```

The outer array is the source node's output index (an IF node has two: `true` at `0`, `false` at `1`). The inner array holds every target fed by that output. `index` is the *target* node's input slot.

## Import into n8n

In the editor:

```text
Workflows -> Import from File
```

Or drag the `.json` file onto an open canvas.

The import creates a new workflow and assigns it a fresh instance ID. Importing the same file twice creates two workflows — it does not update the first one. To update an existing workflow, open it and use `Import from File` on that canvas.

## Export back out

After editing in the UI, export and save over the file in `workflows/`:

```text
Workflow menu (...) -> Download
```

Then strip the instance-local keys before committing. They change on every save and produce noise in diffs:

```bash
jq '{name, nodes, connections, settings}' workflows/my-flow.json > workflows/my-flow.tmp \
  && mv workflows/my-flow.tmp workflows/my-flow.json
```

The full set of keys n8n may emit but that do not belong in the repo:

```text
id  versionId  activeVersionId  versionCounter  versionMetadata
createdAt  updatedAt  active  isArchived  triggerCount
staticData  pinData  meta  tags  shared
```

`pinData` deserves a specific mention: it holds pinned test data from the editor, which is often real payloads from a live system. Never commit it.

## Credentials

Never put credential values into workflow JSON.

n8n stores credentials in its own database, encrypted with `N8N_ENCRYPTION_KEY`. Exported workflow JSON only references them:

```json
"credentials": {
  "httpHeaderAuth": { "id": "aBcD1234", "name": "My API Key" }
}
```

On a fresh import those references point at credentials that may not exist yet. Create the credential once in the UI under `Settings -> Credentials`, then reopen the node and select it.

Never regenerate `N8N_ENCRYPTION_KEY` while local data exists — every stored credential becomes unreadable. See the warning in [README.md](../../README.md).

## Webhook URLs

The local base URL comes from `WEBHOOK_URL` in [docker-compose.yml](../../docker-compose.yml):

```text
http://n8n-homerun.local:15678/
```

A Webhook node exposes two URLs:

```text
http://n8n-homerun.local:15678/webhook-test/<path>   # test, only while "Listen for test event" is active, one request
http://n8n-homerun.local:15678/webhook/<path>        # production, only while the workflow is active
```

The production URL returns 404 until the workflow is activated.

## Activation

Imported workflows arrive inactive. Toggle `Active` in the editor. Only trigger-based workflows (Webhook, Schedule Trigger) can be activated; a workflow whose only trigger is Manual cannot.

## Alternative: CLI import

Useful for bulk imports. `./data/n8n` is already mounted into the container at `/home/node/.n8n`, so a file placed there is visible to the CLI:

```bash
cp workflows/my-flow.json data/n8n/
docker compose exec n8n n8n import:workflow --input=/home/node/.n8n/my-flow.json
```

This path requires a top-level `id` in the JSON — unlike the UI import, the CLI does not generate one, and fails with a not-null constraint error on `workflow_entity.id` if it is missing. It also writes straight to the database, and n8n has no `delete:workflow` CLI command, so a mistaken import has to be cleaned up in the UI.

List what is currently in the instance:

```bash
docker compose exec n8n n8n list:workflow
```
