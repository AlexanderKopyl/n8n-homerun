# Depositing Players Export

Scheduled workflow that runs one Athena query and overwrites a Google Sheet tab with every
player who registered in the last 10 days, has **never made a successful deposit**, has a phone
number and is not a streamer — one row per player, with the number of days since registration.

Workflow file: [workflows/depositing-players-export.json](../../workflows/depositing-players-export.json)

Verified against n8n `2.20.7`.

Built on the same Athena transport as [vip-deposit-alert](vip-deposit-alert.md): the
`awsAssumeRole` credential, a `Config` node, the five-state Switch and the bounded poll loop.
What differs is the output: the result is a full player list, far beyond Slack's 100-row
table, so it goes to a Google Sheet instead of a message.

## Portability

The workflow is **self-contained**. It carries its own settings in a `Config` node and reads
nothing from the host: no `$env`, no `$vars`, no dependency on this repository's
`docker-compose.yml` or `.env`.

## Node chain

```text
Schedule Trigger
  -> Config                        (deployment settings)
  -> Execute Athena Query          (StartQueryExecution)
  -> Wait 5s
  -> Poll Athena Query Status      (GetQueryExecution)
  -> Athena Query State            (Switch)
       SUCCEEDED  -> Get Athena Results Page   (GetQueryResults, 1000 rows)
                     -> Parse Athena Page
                     -> More Pages?            (IF, NextToken present)
                          true  -> back to Get Athena Results Page
                          false -> Collect Export Rows
                                   -> Clear Sheet          (values:clear)
                                   -> Write Rows to Sheet  (values.update)
       QUEUED     -> Poll Attempts Remaining? -> back to Wait 5s
       RUNNING    -> Poll Attempts Remaining? -> back to Wait 5s
       FAILED     -> Athena Query Failed   (Stop and Error)
       CANCELLED  -> Athena Query Failed   (Stop and Error)
       fallback   -> Athena Query Failed   (Stop and Error)
```

Pagination is an explicit loop rather than the HTTP Request node's built-in pagination:
Athena answers with `Content-Type: application/x-amz-json-1.1`, which n8n's paginator does
not parse as JSON, so `$response.body.NextToken` would never resolve.

The sheet is touched only after every page has been read. An Athena failure, a poll timeout
or a malformed page leaves yesterday's export in place.

## Setup on a new n8n instance

### 1. Import

```text
Workflows -> Import from File -> depositing-players-export.json
```

### 2. Prepare the spreadsheet

Create the spreadsheet and a tab for the export (default name `Depositing Players`). The tab
must exist — the Sheets API does not create it — and columns `A:Z` of it belong to this
workflow: they are cleared and the data is rewritten into `A:E` on every run. Keep notes and formulas on another tab.

### 3. Fill in the `Config` node

| Field | Example | Notes |
|---|---|---|
| `athenaRegion` | `eu-central-1` | Keep it equal to the region set on the AWS credential. |
| `athenaDatabase` | `operator_ro_normcasino` | Query context. The SQL fully qualifies its tables. |
| `athenaCatalog` | `AwsDataCatalog` | Falls back to `AwsDataCatalog` when left empty. |
| `athenaWorkgroup` | `primary` | Falls back to `primary` when left empty. |
| `athenaOutputLocation` | `s3://bucket/prefix/` | Leave **empty** if the workgroup enforces a result location. |
| `googleSheetId` | `1AbC...xYz` | The long id between `/d/` and `/edit` in the spreadsheet URL. |
| `googleSheetTab` | `Depositing Players` | Tab to overwrite. Quotes in the name are handled. |

### 4. Credentials

- **AWS (Assume Role)** on `Execute Athena Query`, `Poll Athena Query Status` and
  `Get Athena Results Page`. Identical to the analytics report — see
  [analytics-slack-report.md, section 3](analytics-slack-report.md#3-aws-credential--assume-the-cross-account-role).
  An existing credential can be reused; set Role Session Name to
  `n8n-depositing-players-export` to tell the workflows apart in CloudTrail.
- **Google Sheets OAuth2 API** (`googleSheetsOAuth2Api`) on `Clear Sheet` and
  `Write Rows to Sheet`. The Google account behind it needs edit access to the spreadsheet.

### 5. Schedule and activate

```text
0 8 * * *       # 08:00 daily, Europe/Warsaw
```

The workflow timezone is `Europe/Warsaw`, set in the workflow settings, so the cron fields are
Warsaw local time. The query reads complete days only: registrations and deposits up to
yesterday.

Imported workflows arrive inactive.

## Athena

The SQL lives in `Execute Athena Query` and decides who is exported and in which order.

| Rule | Where |
|---|---|
| Registered in the last 10 full days (today excluded) | `reg_from` in `params`, final `WHERE` |
| No successful deposit since `2026-01-01` (effectively all time) | `depositors` CTE, `d.player_hk IS NULL` |
| Brands `NORMCASINO`, `LACASINO`, `VOLKCASINO`, `IKRACASINO` | `depositors` CTE and the final `WHERE` |
| Test accounts and deleted players excluded | `is_test = false`, `player_deleted_date IS NULL` |
| Phone number present and not blank | final `WHERE` |
| Streamers (risk tag `39`) excluded | `NOT EXISTS` on `d_risk_tags_summary` |
| Order: brand, then days since registration, then user id | `ORDER BY` |

`2026-01-01` only bounds the deposit lookup so Athena can prune partitions; move it if older
data is ever loaded. The `depositors` CTE does not filter `is_test` — it only has to find who
deposited, and the player filter already excludes test accounts.

## Sheet columns

| Column | Source | Notes |
|---|---|---|
| Brand | `agg_player_summary.brand` | |
| User ID | `agg_player_summary.player_id` | Written as text, so leading zeros survive. |
| First Name | `d_player.first_name` | |
| Phone Number | `d_player.phone` | Written as text, so `+` and leading `0` survive. |
| Days Since Registration | `date_diff('day', registration date, today)` | 1 to 10. |

Headers and column order are defined once, in `Collect Export Rows`. Adding a column means
adding it to the SQL, to `COLUMNS` in `Parse Athena Page` and to `COLUMNS` in
`Collect Export Rows`.

## Failure behaviour

| Situation | Result |
|---|---|
| Athena returns `FAILED` / `CANCELLED` / an unexpected state | `Athena Query Failed` stops the run. Sheet untouched. |
| Query still running after 120 polls (~10 min) | `Athena Poll Timeout` stops the run. Sheet untouched. |
| Result above 100 pages (100,000 rows) | `Parse Athena Page` throws. Sheet untouched. |
| A required column missing from the result | `Parse Athena Page` throws. Sheet untouched. |
| HTTP error on any Athena or Sheets call | Node retries 3 times, 5 s apart, then the run fails. |
| `Write Rows to Sheet` fails after `Clear Sheet` succeeded | The tab is left empty until the next successful run. |
| Zero players | Header row only. Run succeeds. |

## Testing

Run it manually with *Execute Workflow*. Point `googleSheetTab` at a scratch tab first. Do
not pin data on any node before exporting the workflow back to Git — it is real player data
with names and phone numbers.

## Data handling

The sheet holds names and phone numbers. Share the spreadsheet only with the people who work
the list, and remember that executions are saved with their data
(`saveDataSuccessExecution: all`), so the same rows sit in n8n's execution history.

## Known limitations

- **One Sheets write.** All rows go in a single `values.update` call. That is comfortable at
  tens of thousands of rows; near the 100-page cap, the request gets large enough that it
  should be split.
- **Clear, then write.** Not atomic: a failure between the two calls leaves an empty tab.
- **Brand list, the 10-day window and the start date are hardcoded in the SQL**, not in `Config`.
- **The name is historical.** The workflow is still called Depositing Players Export, but it
  now exports players who have *not* deposited.
