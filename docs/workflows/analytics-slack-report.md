# Analytics Slack Report

Scheduled workflow that runs one Athena query and posts the result to a Slack Incoming
Webhook as a Block Kit message.

Workflow file: [workflows/analytics-slack-report.json](../../workflows/analytics-slack-report.json)

Verified against n8n `2.20.7`.

## Portability

The workflow is **self-contained**. It carries its own settings in a `Config` node and reads
nothing from the host: no `$env`, no `$vars`, no dependency on this repository's
`docker-compose.yml` or `.env`. The file imports and runs unchanged on any n8n instance.

The repository stores the file as the source of truth for review and version history — it is
not part of the runtime.

## Node chain

```text
Schedule Trigger
  -> Config                        (deployment settings)
  -> Execute Athena Query          (StartQueryExecution)
  -> Wait 5s
  -> Poll Athena Query Status      (GetQueryExecution)
  -> Athena Query State            (Switch)
       SUCCEEDED  -> Get Athena Results (GetQueryResults)
                     -> Normalize Analytics Data
                     -> Build Slack Payload
                     -> HTTP Request — Send Analytics to Slack
       QUEUED     -> Poll Attempts Remaining? -> back to Wait 5s
       RUNNING    -> Poll Attempts Remaining? -> back to Wait 5s
       FAILED     -> Athena Query Failed   (Stop and Error)
       CANCELLED  -> Athena Query Failed   (Stop and Error)
       fallback   -> Athena Query Failed   (Stop and Error)
```

Only the `SUCCEEDED` branch reaches Slack, and only one node connects to the Slack request,
so a successful execution delivers exactly one message.

## Setup on a new n8n instance

### 1. Import

```text
Workflows -> Import from File -> analytics-slack-report.json
```

Importing the same file twice creates two workflows. To update an existing one, open it and
import from inside its canvas. See [authoring.md](authoring.md).

### 2. Fill in the `Config` node

Open `Config` and set the six values. Everything downstream reads them through
`$('Config').first().json.<field>`.

| Field | Example | Notes |
|---|---|---|
| `athenaRegion` | `eu-central-1` | Region of the Athena endpoint. Keep it equal to the region set on the AWS credential, which is where the STS call goes. |
| `athenaDatabase` | `operator_ro_normcasino` | Query context. The SQL fully qualifies its tables, so this mainly satisfies Athena's requirement for a database. |
| `athenaCatalog` | `AwsDataCatalog` | Falls back to `AwsDataCatalog` when left empty. |
| `athenaWorkgroup` | `primary` | Falls back to `primary` when left empty. |
| `athenaOutputLocation` | `s3://bucket/prefix/` | Leave **empty** if the workgroup enforces a result location — the field is then omitted from the request entirely. |
| `slackWebhookUrl` | `https://hooks.slack.com/services/...` | **Secret.** Ships as a placeholder. |

**On the webhook.** `slackWebhookUrl` is the one secret in this workflow. Fill it in on the
n8n instance, never in the file committed here. If you export the workflow back out, clear
that field before saving it to Git. The committed file must always contain the placeholder
`https://hooks.slack.com/services/REPLACE/WITH/YOUR-WEBHOOK`.

### 3. AWS credential — assume the cross-account role

Athena is reached through n8n's predefined **AWS (Assume Role)** credential
(`awsAssumeRole`), selected on the three Athena `HTTP Request` nodes. It performs the STS
call itself, so no extra node is needed:

```text
IAM user Access Key ID + Secret Access Key
  -> sts:AssumeRole   (POST https://sts.<region>.amazonaws.com, signed with the user's keys)
  -> temporary AccessKeyId + SecretAccessKey + SessionToken
  -> Athena           (SigV4 with the temporary keys, X-Amz-Security-Token header)
```

The long-lived IAM user keys never sign an Athena request; they only sign the STS call.

Create it once:

```text
Settings -> Credentials -> New -> AWS (Assume Role)
```

| Field | Value |
|---|---|
| Region | Same region as `Config.athenaRegion`. This picks the STS endpoint. |
| Use System Credentials | Off. |
| STS Access Key ID | The IAM user's access key ID. |
| STS Access Key Secret | The IAM user's secret access key. |
| STS Session Token | Leave empty — the IAM user has permanent keys. |
| Role ARN | The cross-account Athena role, e.g. `arn:aws:iam::<account>:role/<role>`. |
| External ID | See below. |
| Role Session Name | e.g. `n8n-analytics-slack-report`. Shows up in CloudTrail. |

Then open `Execute Athena Query`, `Poll Athena Query Status` and `Get Athena Results` and
select the credential under *Credential for AWS Assume Role*.

**External ID is mandatory in n8n's UI, but optional in AWS.** The credential throws
`External ID is required when assuming a role` before it makes any network call, so the field
cannot be left blank.

- If the role's trust policy has a `Condition` on `sts:ExternalId`, enter that exact value.
- If it has no such condition — the case here — enter a placeholder such as
  `n8n-analytics-slack-report`. STS does not evaluate an External ID the trust policy never
  asks for, so the value is ignored. It must satisfy the API's own format rules: at least
  2 characters, drawn from `[A-Za-z0-9+=,.@:/-]`.

Check which case applies under IAM -> Roles -> *the role* -> Trust relationships.

Use *Test* on the credential page to validate the whole chain: it assumes the role and then
calls `sts:GetCallerIdentity` with the temporary credentials, so a green result means the
`AssumeRole` leg succeeded.

The IAM user needs only `sts:AssumeRole` on the target role — which the existing
`AssumeAthenaXCrossAccountRole` policy already grants. The Athena, Glue and S3 permissions
belong to the assumed role, not the user. No IAM change is required for this workflow.

Credentials are encrypted per instance and are never written into workflow JSON, so this step
repeats on every instance the workflow is imported to.

### 4. Schedule and activate

`Schedule Trigger` uses a cron expression. The committed value is a **development
placeholder**:

```text
45 9 * * *      # 09:45 daily
```

Set the real production time in the node. The workflow timezone is `Europe/Kyiv`, set in the
workflow settings, so cron fields are Kyiv local time regardless of the host's timezone.

Imported workflows arrive inactive. Nothing runs on a schedule until you toggle `Active`.

## Athena

The SQL lives in the `Execute Athena Query` node, in the JSON body of the
`StartQueryExecution` request. It is the single source of truth for period generation,
aggregation, metric calculation and ordering — nothing is recalculated in JavaScript.

Output contract, one row per period:

```text
level  period  ftd  total_dep  in_out  hold  reg2dep  ftd2std  ar_psp  ar_psp_deposits
```

`level` is one of:

```text
year   current year
month  previous month + current month
week   weeks of the current month
day    completed days of the current/latest week (excludes today by design)
3h     today, in 3-hour buckets
```

`day` deliberately excludes `CURRENT_DATE` while `3h` covers it, so the report shows
completed days followed by today's 3-hour breakdown. This is intended, not a gap.

## Report layout

```text
📊 Analytics Report
26 Aug 2026 • Europe/Kyiv • Updated 09:45

YEAR / MONTHS / WEEKS — <CURRENT MONTH> / DAYS — CURRENT WEEK / 3H — LATEST DAY
```

Each period is one Block Kit `section` with eight `fields` in fixed order:

```text
FTD      | Total Dep
In-Out   | Hold
Reg2Dep  | Ftd2Std
AR PSP   | AR PSP Deposits
```

`ftd`, `total_dep` and `in_out` are formatted with thousands separators; the other five as
percentages with exactly one decimal. NULL, missing or unparseable values render as `—`
rather than failing. A level that returns no rows is skipped instead of printing an empty
heading. The report date, time and current month name are generated at run time in
`Europe/Kyiv`.

## Failure behaviour

| Situation | Result |
|---|---|
| Athena `FAILED` or `CANCELLED` | `Athena Query Failed` stops the run and reports the query execution ID and `StateChangeReason`. No Slack message. |
| Unrecognised Athena state | Same as `FAILED`. |
| Query still running after 120 polls (~10 min) | `Athena Poll Timeout` stops the run and reports the last observed state. No Slack message. |
| Zero analytics rows | `Normalize Analytics Data` throws. No Slack message. |
| Athena result exceeds one page of 1000 rows | `Normalize Analytics Data` throws rather than silently reporting partial data. |
| Athena returns an unsupported `level` | `Normalize Analytics Data` throws, naming the value. |
| `sts:AssumeRole` is denied or the External ID is wrong | The Athena node fails with `Failed to assume role: STS AssumeRole failed: 403 ...`. No Slack message. |
| `slackWebhookUrl` left as the placeholder | Slack returns 404 and the node fails visibly after its retries. |
| Slack returns 429 / 5xx, or the request times out | Node retries 3 times, 5 s apart, then fails the execution visibly. |

All four HTTP nodes retry three times, five seconds apart. Failures surface in
`Executions`; the workflow saves both successful and failed runs. There is no separate error
workflow in this repository, so nothing extra is wired up.

## Testing

Run each stage from the editor. Nothing below sends to Slack until the last step.
Select a node and press `D` to disable or re-enable it.

**1. Athena only.** Disable `Normalize Analytics Data`, click `Execute workflow`, and inspect
`Get Athena Results`. Check the output contains rows for `year`, `month`, `week`, `day` and
`3h`.

Or check connectivity outside n8n:

```bash
aws athena start-query-execution --query-string "SELECT 1" \
  --work-group primary --region eu-central-1
```

**2. Normalization.** Re-enable `Normalize Analytics Data` and run again. Its output must be
a single item shaped like:

```json
{ "year": [], "month": [], "week": [], "day": [], "3h": [] }
```

with rows in Athena's order — not sorted alphabetically.

**3. Slack payload, without delivery.** Disable
`HTTP Request — Send Analytics to Slack`, run the workflow, and open the output of
`Build Slack Payload`. It must be exactly one item:

```json
{ "text": "Analytics Report — ...", "blocks": [] }
```

Paste `blocks` into [Slack's Block Kit Builder](https://app.slack.com/block-kit-builder) to
preview the rendering. Block Kit Builder never contacts your webhook.

**4. Delivery.** Put a **test channel's** webhook in `Config.slackWebhookUrl`, re-enable the
Slack node, and run once. Only then switch it to the production webhook.

Never put a real webhook URL into a test fixture, a Code node, or this file.

## Known limitations

- Reads a single page of up to 1000 Athena rows. The query returns roughly 25, so pagination
  is not implemented; the workflow fails loudly rather than truncating if that ever changes.
- Polling is bounded at 120 attempts of 5 seconds. Longer queries need the limit raised in
  `Poll Attempts Remaining?`.
- Slack caps a message at 50 blocks. The layout produces about 37 at most; beyond that
  `Build Slack Payload` throws instead of sending a truncated report.
- `Wait 5s` stays under n8n's 65-second threshold, so the execution is held in memory rather
  than being suspended and resumed.
- The Slack webhook lives in the `Config` node on the instance, not in an n8n credential —
  n8n has no credential type for a secret URL, and the spec requires an Incoming Webhook via
  the HTTP Request node. Anyone who can open the workflow can read it.
