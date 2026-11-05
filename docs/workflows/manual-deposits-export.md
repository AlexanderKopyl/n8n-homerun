# Manual Deposits Export

Daily workflow that reads yesterday's manual deposits (`payment_method_name = 'ManualDeposit'`)
from Athena and posts them to a Slack channel as an `.xlsx` attachment, with a one-line
summary as the message text.

Workflow file: [workflows/manual-deposits-export.json](../../workflows/manual-deposits-export.json)

Verified against n8n `2.20.7` node versions; not yet run on the instance (see *Testing*).

## Why a bot token and not the Incoming Webhook

A Slack Incoming Webhook (`https://hooks.slack.com/services/...`) can only post message text
and blocks. It has no file upload at all, and 100–150 rows a day do not fit a message. The
file therefore goes through the Slack Web API with a **bot token**, using the three-step
external upload (`files.getUploadURLExternal` → upload bytes → `files.completeUploadExternal`).
The webhook URL is not used by this workflow.

## Node chain

```text
Schedule Trigger                   01:00 UTC daily
  -> Config                        (deployment settings)
  -> Execute Athena Query          (StartQueryExecution)
  -> Wait 5s
  -> Poll Athena Query Status      (GetQueryExecution)
  -> Athena Query State            (Switch)
       SUCCEEDED  -> Get Athena Results Page (GetQueryResults, 1000 rows)
                     -> Parse Athena Page
                     -> More Pages?   true  -> Get Athena Results Page (next page)
                                      false -> Collect Export Rows
                                               -> Rows Found?
                                                    true  -> Build Export Rows
                                                             -> Convert to XLSX
                                                             -> Prepare Slack Upload
                                                             -> Get Slack Upload URL     (files.getUploadURLExternal)
                                                             -> Check Upload URL
                                                             -> Upload File to Slack     (POST bytes to upload_url)
                                                             -> Complete Slack Upload    (files.completeUploadExternal)
                                                             -> Verify Slack Response
                                                    false -> Post Empty Notice          (chat.postMessage)
                                                             -> Verify Slack Response
       QUEUED     -> Poll Attempts Remaining? -> back to Wait 5s
       RUNNING    -> Poll Attempts Remaining? -> back to Wait 5s
       FAILED     -> Athena Query Failed   (Stop and Error)
       CANCELLED  -> Athena Query Failed   (Stop and Error)
       fallback   -> Athena Query Failed   (Stop and Error)
```

The Athena half is the same pattern as the other exports in this repository; the Slack half
is new.

## What lands in Slack

One message per day in the configured channel, posted by the bot:

```text
Ручные депозиты ManualDeposit за 2026-11-04 (UTC): 123 шт.
По причине: Compensation: 117, Courier deposit: 6
По статусу: success: 120, declined: 3
📎 manual_deposits_2026-11-04.xlsx
```

The spreadsheet has one sheet, `ManualDeposit`, a header row, and one row per manual deposit
in the query's order (newest first). Columns repeat the UBO transactions export:

```text
ID  Manual  Player ID  Brand  Risk casino segment  Created  Updated  Creating Reason
Transaction type  Transaction status  Paid amount  Currency  Modified by  Decline reason
Notes  Payment method
```

`Paid amount` is written as a number; every other column as text, so IDs keep their digits.
NULL becomes an empty cell. Times are UTC, as in the data.

On a day with zero rows the bot posts the first line of the summary plus
`Файл не прикреплён: за сутки нет ни одной строки.` and no file, so a quiet day looks
different from a broken run.

## Schedule

`Schedule Trigger` runs `0 1 * * *` and the workflow timezone is **UTC**, so that is 01:00 UTC
(03:00 or 04:00 in Kyiv depending on DST). The source table `f_payment` is loaded once a day
at 00:00 UTC with the previous day's data, so the run must never start before 00:30 UTC. The
extra half hour is a buffer for a slow load. Keep the timezone at UTC if you change the cron;
the SQL's "yesterday" is `date_add('day', -1, current_date)` evaluated by Athena in UTC, and
the file name and message date are computed the same way.

## Setup on a new n8n instance

### 1. Import

```text
Workflows -> Import from File -> workflows/manual-deposits-export.json
```

### 2. Config node

| Field | Example | Notes |
|---|---|---|
| `athenaRegion` | `eu-central-1` | Region of the Athena endpoint. Keep it equal to the region on the AWS credential. |
| `athenaDatabase` | `operator_ro_normcasino` | Query context. The SQL fully qualifies its tables as `database.table`. |
| `athenaCatalog` | `AwsDataCatalog` | Falls back to `AwsDataCatalog` when left empty. |
| `athenaWorkgroup` | `primary` | Falls back to `primary` when left empty. |
| `athenaOutputLocation` | `s3://bucket/prefix/` | Leave **empty** if the workgroup enforces a result location. |
| `slackChannelId` | `C0123456789` | The channel **ID**, not its name. Ships as a placeholder; the workflow refuses to upload while it starts with `REPLACE`. |

To find the channel ID: open the channel in Slack → channel name → *About* → the ID is at
the bottom. The bot must be a member of the channel (`/invite @<app name>` in the channel).

### 3. AWS credential

Same `AWS (Assume Role)` credential as the other exports. Select it on `Execute Athena Query`,
`Poll Athena Query Status` and `Get Athena Results Page`. Creating it is described in
[analytics-slack-report.md](analytics-slack-report.md#3-aws-credential--assume-the-cross-account-role).

### 4. Slack credential (bot token)

Create a Slack app once (https://api.slack.com/apps → *Create New App* → *From scratch*), in
the target workspace:

1. *OAuth & Permissions* → *Bot Token Scopes*: add `files:write` and `chat:write`.
2. *Install to Workspace*. Copy the **Bot User OAuth Token** (`xoxb-...`).
3. In the channel: `/invite @<app name>`.

In n8n:

```text
Settings -> Credentials -> New -> Slack API
  Access Token: xoxb-...
```

Select it under *Credential for Slack API* on `Get Slack Upload URL`, `Complete Slack Upload`
and `Post Empty Notice`. `Upload File to Slack` needs no credential: the one-time
`upload_url` is itself the authorization.

The token is stored encrypted in n8n and never in this repository.

### 5. Activate

Imported workflows arrive inactive. Toggle `Active` after the test below.

## Athena

The SQL lives in `Execute Athena Query`, in the JSON body of `StartQueryExecution`. It is the
single source of truth for the day window, the filters and the column set.

- Window: yesterday, whole UTC day (`action_date >= yesterday and < today`).
- Filter: `payment_method_name = 'ManualDeposit'` (both `adjustment deposit` /
  *Compensation* and `deposit` / *Courier deposit*), `brand <> 'SBNORM'`. Streamers and test
  players are **not** excluded on purpose.
- One row per `apm_order_id` (latest `update_ts` wins) as a guard against duplicates.
- `Creating Reason` comes from `ref_apm_transaction_reasons` joined on `reason_id` and
  `transaction_type`.
- Tables are referenced as `operator_ro_normcasino.<table>`; the `"Athena-bebohuex"` prefix
  from the DBeaver version of the query is a client connection name, not an Athena catalog.
- The query contains no `?` and no trailing `;`.

Adding or renaming a column means editing three places together: the SQL, `COLUMNS` in
`Parse Athena Page` (which fails loudly if a column is missing) and `COLUMNS` in
`Build Export Rows` (which sets the spreadsheet column order).

## Failure behaviour

| Situation | Result |
|---|---|
| Athena `FAILED` or `CANCELLED`, or an unknown state | `Athena Query Failed` stops the run with the query id and reason. Nothing is sent to Slack. |
| Query still running after 120 polls (~10 min) | `Athena Poll Timeout` stops the run. Nothing is sent. |
| A column is missing from the result | `Parse Athena Page` throws naming the column. |
| More than 100 pages (100 000 rows) | `Parse Athena Page` throws; the date filter has almost certainly been lost. |
| Zero rows | `Post Empty Notice` posts the summary line without a file. |
| `slackChannelId` left as the placeholder | `Prepare Slack Upload` throws before any Slack call. |
| Slack answers `ok: false` (e.g. `not_in_channel`, `missing_scope`, `invalid_auth`) | `Check Upload URL` / `Verify Slack Response` throw with the Slack error code and a hint. Slack returns HTTP 200 on these, so without the check the run would look green. |
| Slack 429 / 5xx or a timeout | The HTTP node retries 3 times, 5 s apart, then fails visibly. |
| `files.completeUploadExternal` fails after the bytes were uploaded | The file is discarded by Slack; nothing is posted. Re-running the workflow uploads a fresh file. |

Successful and failed executions are both saved. There is no separate error workflow.

## Testing

Select a node and press `D` to disable or re-enable it.

**1. Athena only.** Disable `Build Export Rows` and `Post Empty Notice`, click
`Execute workflow`, and open `Collect Export Rows`. Check `rowCount` is in the expected range
(roughly 100–150 on a weekday), `reportDate` is yesterday, and `summaryText` reads well.

**2. Spreadsheet, without Slack.** Re-enable `Build Export Rows`, disable
`Get Slack Upload URL`, run, and download the binary from `Convert to XLSX` to open it in
Excel. `Prepare Slack Upload` shows the byte length that will be sent to Slack.

**3. Delivery to a test channel.** Put a test channel's ID in `Config.slackChannelId`, invite
the bot there, re-enable everything, and run once. The channel must receive one message with
the file attached. Only then switch to the production channel ID.

**4. Empty day.** Temporarily change the SQL filter to a value that matches nothing (for
example `payment_method_name = 'ManualDepositXX'`), run, confirm the notice without a file
arrives, and revert the SQL.

## Known limitations

- Slack's external upload is three calls; the file id from step 1 is valid for a short time
  only, so the chain must not be paused between `Get Slack Upload URL` and
  `Complete Slack Upload`.
- `Convert to XLSX` keeps the whole spreadsheet in memory. At a few hundred rows a day this is
  tens of kilobytes; it is not a concern unless the export grows by orders of magnitude.
- All failures surface only in n8n `Executions`. If the run fails, nothing reaches Slack, so
  a missing morning message is the signal to check n8n.
- `Wait 5s` stays under n8n's 65-second threshold, so the execution is held in memory rather
  than suspended and resumed.
