# Registered Players Without Deposits (Norm and LA)

Two scheduled workflows, one per brand, that each run one Athena query and overwrite a tab in
the **output spreadsheet** with every player who **registered in the last 10 days and has never
made a successful deposit**, has a phone number and is not a streamer — one row per player,
with the number of days since registration.

| Workflow | File | Brand | Output tab |
|---|---|---|---|
| Registered Players Without Deposits - Norm | [registered-players-without-deposits-norm.json](../../workflows/registered-players-without-deposits-norm.json) | `NORMCASINO` | `Reg Norm` |
| Registered Players Without Deposits - LA | [registered-players-without-deposits-la.json](../../workflows/registered-players-without-deposits-la.json) | `LACASINO` | `Reg LA` |

Both write to the same output spreadsheet as [Depositing Players Export](depositing-players-export.md),
`1Ic93YmdJA-upT-asuF_ehrkkXCNnkhL-_m1w9Bt3HmY`, on their own tabs. The two files are generated
from one template and differ only in the brand filter of the SQL, the node ids, the tab name
and the workflow name. Fix a bug in one and apply it to the other.

Verified against n8n `2.20.7`.

Built on the same Athena transport and Sheets write as Depositing Players Export, **without its
call-history step**: no GR Base spreadsheet is read, and every eligible player is written every day.

## Portability

The workflows are **self-contained**. They carry their own settings in a `Config` node and read
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
                                   -> Read Sheet Title     (properties.title)
                                   -> Stamp Sheet Title    (batchUpdate, title)
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

After the rows are written, the output spreadsheet is renamed to
`<name> (обновлено dd.MM HH:mm)` in Europe/Warsaw time, replacing an earlier stamp and keeping
the name. It is the last step, so a title with today's time means the data is today's. The
spreadsheet is shared with [Depositing Players Export](depositing-players-export.md), and all
four workflows stamp the same title.

## Setup on a new n8n instance

### 1. Import

```text
Workflows -> Import from File -> registered-players-without-deposits-norm.json
Workflows -> Import from File -> registered-players-without-deposits-la.json
```

### 2. Prepare the output spreadsheet

In the output spreadsheet create the tabs `Reg Norm` and `Reg LA`. They must exist — the
Sheets API does not create them — and columns `A:Z` of each belong to the workflow: they are
cleared and the data is rewritten into `A:I` on every run. Keep notes and formulas on another tab.

### 3. Fill in the `Config` node (in both workflows)

| Field | Example | Notes |
|---|---|---|
| `athenaRegion` | `eu-central-1` | Keep it equal to the region set on the AWS credential. |
| `athenaDatabase` | `operator_ro_normcasino` | Query context. The SQL fully qualifies its tables. |
| `athenaCatalog` | `AwsDataCatalog` | Falls back to `AwsDataCatalog` when left empty. |
| `athenaWorkgroup` | `primary` | Falls back to `primary` when left empty. |
| `athenaOutputLocation` | `s3://bucket/prefix/` | Leave **empty** if the workgroup enforces a result location. |
| `outputSheetId` | `1Ic93...HmY` | The output spreadsheet: the long id between `/d/` and `/edit` in its URL. Same in both workflows. |
| `outputSheetTab` | `Reg Norm` / `Reg LA` | Tab to overwrite. Quotes in the name are handled. |

### 4. Credentials

- **AWS (Assume Role)** on `Execute Athena Query`, `Poll Athena Query Status` and
  `Get Athena Results Page`. Identical to the analytics report — see
  [analytics-slack-report.md, section 3](analytics-slack-report.md#3-aws-credential--assume-the-cross-account-role).
  An existing credential can be reused.
- **Google Sheets OAuth2 API** (`googleSheetsOAuth2Api`) on `Clear Sheet` and
  `Write Rows to Sheet`. The Google account behind it needs edit access to the output
  spreadsheet. The same credential as Depositing Players Export works; one credential serves
  all four workflows.

### 5. Schedule and activate

```text
0 8 * * *       # 08:00 daily, Europe/Warsaw
```

The workflow timezone is `Europe/Warsaw`, set in the workflow settings, so the cron fields are
Warsaw local time. Deposits are read up to yesterday. Registrations are counted for complete
days only: a player who registered today appears from tomorrow's run.

Imported workflows arrive inactive.

## Athena

The SQL lives in `Execute Athena Query` and decides who is exported and in which order.

| Rule | Where |
|---|---|
| Registered in the last 10 complete days (today excluded) | `reg_from` in `params`, final `WHERE` |
| No successful deposit on **any** of the four brands since `2026-01-01`, up to yesterday | `depositors` CTE, `d.player_hk IS NULL` |
| Brand (Norm: `NORMCASINO`; LA: `LACASINO`) | final `WHERE` only |
| Test accounts and deleted players excluded | `is_test = false`, `player_deleted_date IS NULL` |
| Phone number present and not blank | final `WHERE` |
| Streamers (risk tag `39`) excluded | `NOT EXISTS` on `d_risk_tags_summary` |
| Order: brand, then days since registration, then user id | `ORDER BY` |

`2026-01-01` only bounds the deposit lookup so Athena can prune partitions; the billing data
itself starts later. The `depositors` CTE looks at all four brands on purpose: a NORMCASINO
registrant who deposited on VOLKCASINO is not a non-depositor. It does not filter `is_test` —
it only has to find who deposited, and the player filter already excludes test accounts.

## Sheet columns

| Column | Source | Notes |
|---|---|---|
| Brand | `agg_player_summary.brand` | |
| User ID | `agg_player_summary.player_id` | Written as text, so leading zeros survive. |
| First name | `d_player.first_name` | |
| Phone Number | `d_player.phone` | Written as text, so `+` and leading `0` survive. |
| Days Since Registration | `date_diff('day', registration date, today)` | 1 to 10. |
| Статус звонка | — | empty, for the managers |
| Количество попыток дозвона | — | empty, for the managers |
| Менеджер | — | empty, for the managers |
| Комментарий | — | empty, for the managers |

The SQL names its columns with the sheet headers (`AS "User ID"` and so on), and
`Parse Athena Page` checks for those exact names. Adding or renaming a column means changing
the SQL, `COLUMNS` in `Parse Athena Page` and `COLUMNS` in `Collect Export Rows` together —
in both workflows.

## Failure behaviour

| Situation | Result |
|---|---|
| Athena returns `FAILED` / `CANCELLED` / an unexpected state | `Athena Query Failed` stops the run. Sheet untouched. |
| Query still running after 120 polls (~10 min) | `Athena Poll Timeout` stops the run. Sheet untouched. |
| Result above 100 pages (100,000 rows) | `Parse Athena Page` throws. Sheet untouched. |
| A required column missing from the result | `Parse Athena Page` throws. Sheet untouched. |
| HTTP error on any Athena or Sheets call | Node retries 3 times, 5 s apart, then the run fails. |
| `Write Rows to Sheet` fails after `Clear Sheet` succeeded | The tab is left empty until the next successful run. |
| `Read Sheet Title` / `Stamp Sheet Title` fails after the rows were written | The run is marked failed; the data is fresh but the title keeps the previous stamp. |
| Zero players | Header row only. Run succeeds. |

## Testing

Run it manually with *Execute Workflow*. Point `outputSheetTab` at a scratch tab first. Do
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
- **Brand, the 10-day registration window and the start date are hardcoded in the SQL**, not
  in `Config`.
- **No call history.** Unlike Depositing Players Export, nobody is skipped for having been
  called recently; a registrant stays on the list for all 10 days unless they deposit.
