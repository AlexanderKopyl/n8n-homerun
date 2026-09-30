# Depositing Players Export (Norm and LA)

Two scheduled workflows, one per brand group, that each run one Athena query and overwrite a
tab in the **output spreadsheet** with the day's calling list: every player with at least one
successful deposit, a phone number and no streamer tag — **minus the players the managers
called in the last three days**, read from that brand's **GR Base spreadsheet**.

| Workflow | File | Brands | Output tab | Call history read from |
|---|---|---|---|---|
| Depositing Players Export - Norm | [depositing-players-export-norm.json](../../workflows/depositing-players-export-norm.json) | `NORMCASINO`, `VOLKCASINO`, `IKRACASINO` | `Norm` | GR Base Norm |
| Depositing Players Export - LA | [depositing-players-export-la.json](../../workflows/depositing-players-export-la.json) | `LACASINO` | `LA` | GR Base LA |

Three spreadsheets are involved, and the ids ship in `Config`:

| Role | Spreadsheet id | Used by |
|---|---|---|
| Output (written) | `1Ic93YmdJA-upT-asuF_ehrkkXCNnkhL-_m1w9Bt3HmY` | both, different tabs |
| GR Base Norm (read) | `1ZIpxoMRcI4hMvncKedGpAKC2wSRZHw4fsaa4HkYvJRE` | Norm |
| GR Base LA (read) | `1UnpFp9HxEgiTnisCBn9JUWjo8Zrl8C_jgHfdlTE2ghU` | LA |

The two files are generated from one template and differ only in the brand filter of the
SQL, the node ids and the name. Fix a bug in one and apply it to the other.

Verified against n8n `2.20.7`.

Built on the same Athena transport as [vip-deposit-alert](vip-deposit-alert.md): the
`awsAssumeRole` credential, a `Config` node, the five-state Switch and the bounded poll loop.

## Portability

The workflows are **self-contained**. They carry their own settings in a `Config` node and
read nothing from the host: no `$env`, no `$vars`, no dependency on this repository's
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
                                   -> List Sheet Tabs        (GR Base: tab names)
                                   -> Select Recent Tabs     (date-named tabs in the cooldown window)
                                   -> Recent Tabs Found?     (IF)
                                        true  -> Read Recent Tabs   (GR Base: values:batchGet)
                                                 -> Build Sheet Payload
                                        false -> Build Sheet Payload
                                   -> Clear Sheet            (output sheet: values:clear)
                                   -> Write Rows to Sheet    (output sheet: values.update)
       QUEUED     -> Poll Attempts Remaining? -> back to Wait 5s
       RUNNING    -> Poll Attempts Remaining? -> back to Wait 5s
       FAILED     -> Athena Query Failed   (Stop and Error)
       CANCELLED  -> Athena Query Failed   (Stop and Error)
       fallback   -> Athena Query Failed   (Stop and Error)
```

Pagination is an explicit loop rather than the HTTP Request node's built-in pagination:
Athena answers with `Content-Type: application/x-amz-json-1.1`, which n8n's paginator does
not parse as JSON, so `$response.body.NextToken` would never resolve.

The output sheet is touched only after every Athena page **and** the call history have been
read. An Athena failure, a poll timeout, a malformed page or a GR Base read error leaves
yesterday's list in place. The GR Base spreadsheets are never written.

## How the daily list is used

Managers work in their brand's **GR Base** spreadsheet. Each calling day has its own tab
named by date (`29/09`, `2909`, `30.09`, `08.09 (таск)`), where they fill in
`Статус звонка`, `Количество попыток дозвона`, `Менеджер` and `Комментарий` next to each
player.

The workflow writes the day's candidates into the **output spreadsheet** (tab `Norm` or
`LA`), overwritten every morning. Before writing, it reads GR Base and removes everyone the
managers had on a dated tab in the last `callCooldownDays` days.

### Cooldown rule

- A tab counts as a calling day when its name **starts with** a day and a month:
  `29/09`, `2909`, `29.09`, `9.07`, `08.09 (таск)`, `0809 ` all qualify. `Bonus`, `task`,
  `рассылка 24.06`, `Лист22` do not.
- The year is the current one, unless that would put the tab in the future — a `31/12` tab
  read on 1 January is 31 December of last year.
- With `callCooldownDays = 3`, the tabs dated **yesterday, 2 days ago and 3 days ago** are
  read. A player called yesterday is skipped today, tomorrow and the day after, and comes
  back on the fourth day. Today's own tab, if it already exists, is not read.
- The player id is taken from the column headed `User ID`, `UserID`, `Account` or `ID`
  (any case); when no such header exists, column A. `123`, `123.0` and `"123"` are the same id.

## Setup on a new n8n instance

### 1. Import

```text
Workflows -> Import from File -> depositing-players-export-norm.json
Workflows -> Import from File -> depositing-players-export-la.json
```

### 2. Prepare the output spreadsheet

In the output spreadsheet create the tabs `Norm` and `LA`. They must exist — the Sheets API
does not create them — and columns `A:Z` of each belong to the workflow: they are cleared
and the data is rewritten into `A:N` on every run. Nothing needs to change in GR Base.

### 3. Fill in the `Config` node (in both workflows)

| Field | Example | Notes |
|---|---|---|
| `athenaRegion` | `eu-central-1` | Keep it equal to the region set on the AWS credential. |
| `athenaDatabase` | `operator_ro_normcasino` | Query context. The SQL fully qualifies its tables. |
| `athenaCatalog` | `AwsDataCatalog` | Falls back to `AwsDataCatalog` when left empty. |
| `athenaWorkgroup` | `primary` | Falls back to `primary` when left empty. |
| `athenaOutputLocation` | `s3://bucket/prefix/` | Leave **empty** if the workgroup enforces a result location. |
| `outputSheetId` | `1Ic93...HmY` | The output spreadsheet: the long id between `/d/` and `/edit` in its URL. Same in both workflows. |
| `outputSheetTab` | `Norm` / `LA` | Tab to overwrite in the output spreadsheet. Quotes in the name are handled. |
| `historySheetId` | `1ZIpx...JRE` / `1UnpF...ghU` | The brand's GR Base spreadsheet, read only. |
| `callCooldownDays` | `3` | Whole number ≥ 1. How many days before today of dated tabs exclude a player. |

### 4. Credentials

- **AWS (Assume Role)** on `Execute Athena Query`, `Poll Athena Query Status` and
  `Get Athena Results Page`. Identical to the analytics report — see
  [analytics-slack-report.md, section 3](analytics-slack-report.md#3-aws-credential--assume-the-cross-account-role).
  An existing credential can be reused.
- **Google Sheets OAuth2 API** (`googleSheetsOAuth2Api`) on `List Sheet Tabs`,
  `Read Recent Tabs`, `Clear Sheet` and `Write Rows to Sheet`. The Google account behind it
  needs **edit** access to the output spreadsheet and **view** access to both GR Base
  spreadsheets. One credential serves both workflows.

### 5. Schedule and activate

```text
0 8 * * *       # 08:00 daily, Europe/Warsaw
```

The workflow timezone is `Europe/Warsaw`, set in the workflow settings, so the cron fields
and the "today" used by the cooldown are Warsaw local time. Deposits are read for complete
days only, up to yesterday.

Imported workflows arrive inactive.

## Athena

The SQL lives in `Execute Athena Query` and decides who is a candidate and in which order.

| Rule | Where |
|---|---|
| At least one successful deposit since `2026-01-01` (effectively all time) | `payments` CTE, `deposit_count > 0` |
| Brand filter (Norm: three brands; LA: `LACASINO`) | `payments` CTE and the final `WHERE` |
| Test accounts excluded | `is_test = false` on billing and player |
| Phone number present and not blank | final `WHERE` |
| Streamers (risk tag `39`) excluded | `NOT EXISTS` on `d_risk_tags_summary` |
| Order: brand, then days since last deposit, then user id | `ORDER BY` |

`2026-01-01` only bounds the billing scan so Athena can prune partitions; the billing data
itself starts in June 2026. Move it if older data is ever loaded.

## Sheet columns

| Column | Source | Notes |
|---|---|---|
| Brand | `agg_player_summary.brand` | |
| User ID | `agg_player_summary.player_id` | Written as text, so leading zeros survive. |
| First name | `d_player.first_name` | |
| Phone Number | `d_player.phone` | Written as text, so `+` and leading `0` survive. |
| Total Deposit | sum of successful deposits, EUR | number |
| Average Deposit | Total Deposit / Deposit Count, 2 dp | number |
| Deposit Count | number of successful deposits | number |
| LDD | days since the last deposit | number |
| Hold % | (deposits − withdrawals) / deposits × 100, 2 dp | number; negative when withdrawals exceed deposits |
| Player Status | `agg_player_summary.risk_casino_segment` | |
| Статус звонка | — | empty, for the managers |
| Количество попыток дозвона | — | empty, for the managers |
| Менеджер | — | empty, for the managers |
| Комментарий | — | empty, for the managers |

The SQL names its columns with the sheet headers (`AS "User ID"` and so on), and
`Parse Athena Page` checks for those exact names. Adding or renaming a column means changing
the SQL, `COLUMNS` in `Parse Athena Page` and `COLUMNS` in `Build Sheet Payload` together —
in both workflows.

## Failure behaviour

| Situation | Result |
|---|---|
| Athena returns `FAILED` / `CANCELLED` / an unexpected state | `Athena Query Failed` stops the run. Sheet untouched. |
| Query still running after 120 polls (~10 min) | `Athena Poll Timeout` stops the run. Sheet untouched. |
| Result above 100 pages (100,000 rows) | `Parse Athena Page` throws. Sheet untouched. |
| A required column missing from the result | `Parse Athena Page` throws. Sheet untouched. |
| `callCooldownDays` not a whole number ≥ 1 | `Select Recent Tabs` throws. Sheet untouched. |
| GR Base unreachable / no access | `List Sheet Tabs` or `Read Recent Tabs` fails after retries. Sheet untouched. |
| HTTP error on any Athena or Sheets call | Node retries 3 times, 5 s apart, then the run fails. |
| `Write Rows to Sheet` fails after `Clear Sheet` succeeded | The tab is left empty until the next successful run. |
| No date tab in the window | Nobody excluded; the full list is written. |
| Every candidate already called | Header row only. Run succeeds. |

`Build Sheet Payload` outputs `rowCount`, `excludedCount` and `recentTabs` — check them in
the execution when a list looks too short or too long.

## Testing

Run it manually with *Execute Workflow*. Point `outputSheetTab` at a scratch tab first. Do
not pin data on any node before exporting the workflow back to Git — it is real player data
with names and phone numbers.

## Data handling

The output sheet holds names and phone numbers. Share it only with the people who work the
list, and remember that executions are saved with their data
(`saveDataSuccessExecution: all`), so the same rows sit in n8n's execution history.

## Known limitations

- **The cooldown depends on the managers' tab discipline.** Only GR Base tabs whose name
  starts with a date are read. A calling day without a dated tab is invisible to n8n.
- **One Sheets write.** All rows go in a single `values.update` call. That is comfortable at
  tens of thousands of rows; near the 100-page cap, the request gets large enough that it
  should be split.
- **Clear, then write.** Not atomic: a failure between the two calls leaves an empty tab.
- **Brand list and the start date are hardcoded in the SQL**, not in `Config`.
- **No row limit.** Every eligible player is written; on the first run this is the whole
  depositor base of the brand.
