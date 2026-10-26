# Players 12M Export (Norm, LA and Volk)

Three scheduled workflows, one per brand, that each run one Athena query and rewrite a tab in
the **players spreadsheet** with every depositor of the brand and their last-12-month metrics:
deposits, withdrawals, casino turnover/GGR/NGR, bonuses, e-mail engagement and key dates.

| Workflow | File | Brand | Output tab | Schedule (Europe/Warsaw) |
|---|---|---|---|---|
| Players 12M Export - Norm | [players-12m-export-norm.json](../../workflows/players-12m-export-norm.json) | `NORMCASINO` | `NORMCASINO` | 06:00 |
| Players 12M Export - LA | [players-12m-export-la.json](../../workflows/players-12m-export-la.json) | `LACASINO` | `LACASINO` | 06:20 |
| Players 12M Export - Volk | [players-12m-export-volk.json](../../workflows/players-12m-export-volk.json) | `VOLKCASINO` | `VOLKCASINO` | 06:40 |

Output spreadsheet (ships in `Config`): `1x9DxNS-AEjB3jjsegrd1yRflHwoKvaUMwZsi1WtyAYM`, one tab per brand.

The three files are generated from one template and differ only in the brand filter of the
SQL, the output tab, the name, the cron expression and the `ClientRequestToken` prefix. Fix a
bug in one and apply it to the other two.

Verified against n8n `2.20.7`.

Built on the same Athena transport as [depositing-players-export](depositing-players-export.md):
the `awsAssumeRole` credential, a `Config` node, the five-state Switch and the bounded poll
loop. The difference is on the Sheets side: the result can be tens of thousands of rows, so
**every Athena page (1000 rows) is appended to the sheet as soon as it is read**. No node ever
holds the whole result in one item, and no single Sheets request is larger than one page.

## Portability

The workflows are **self-contained**. They carry their own settings in a `Config` node and
read nothing from the host: no `$env`, no `$vars`, no dependency on this repository's
`docker-compose.yml` or `.env`.

## Node chain

```text
Schedule Trigger
  -> Config                        (deployment settings)
  -> Execute Athena Query          (StartQueryExecution)
  -> Wait 15s
  -> Poll Athena Query Status      (GetQueryExecution)
  -> Athena Query State            (Switch)
       SUCCEEDED  -> Clear Sheet                 (output tab: values:clear A:AZ)
                     -> Get Athena Results Page  (GetQueryResults, 1000 rows)
                     -> Parse Athena Page        (rows -> 2D array, header on page 0)
                     -> Append Page to Sheet     (output tab: values:append, RAW)
                     -> More Pages?              (IF, NextToken present)
                          true  -> back to Get Athena Results Page
                          false -> Export Summary (pages, rowCount — for the execution log)
       QUEUED     -> Poll Attempts Remaining? -> back to Wait 15s
       RUNNING    -> Poll Attempts Remaining? -> back to Wait 15s
       FAILED     -> Athena Query Failed   (Stop and Error)
       CANCELLED  -> Athena Query Failed   (Stop and Error)
       fallback   -> Athena Query Failed   (Stop and Error)
```

Pagination is an explicit loop rather than the HTTP Request node's built-in pagination:
Athena answers with `Content-Type: application/x-amz-json-1.1`, which n8n's paginator does
not parse as JSON, so `$response.body.NextToken` would never resolve.

### How the page loop keeps its place

`Get Athena Results Page` is entered twice: run 0 from `Clear Sheet` (no token) and run N from
`More Pages?`. Run N sends the `NextToken` that `Parse Athena Page` run N-1 returned, read
back explicitly as `$('Parse Athena Page').all(0, $runIndex - 1)`. `More Pages?` reads the
token of its own run the same way. Nothing depends on the item that happens to flow through
the Sheets response.

### Why values:append, not values.update

A tab created in the UI has 1000 rows. `values.update` refuses a range beyond the grid, so a
long export would need the grid resized first, and the final size is unknown until the last
page. `values:append` finds the table starting at `A1`, writes below its last filled row and
grows the grid as needed. After `Clear Sheet` the tab is empty, page 0 lands on row 1 with the
header, every later page lands below the previous one. `insertDataOption=OVERWRITE` keeps the
write from inserting rows in the middle of anything.

`valueInputOption=RAW`: strings stay strings, so phone numbers keep their `+` or leading `0`,
account ids are never reformatted and dates stay exactly as Athena printed them. Numeric
columns are sent as JSON numbers, so the sheet can sum, sort and filter them.

The sheet is touched only after Athena reports `SUCCEEDED`. A failed query or a poll timeout
leaves yesterday's data in place.

## Setup on a new n8n instance

### 1. Import

```text
Workflows -> Import from File -> players-12m-export-norm.json
Workflows -> Import from File -> players-12m-export-la.json
Workflows -> Import from File -> players-12m-export-volk.json
```

### 2. Prepare the output spreadsheet

Create the tabs `NORMCASINO`, `LACASINO` and `VOLKCASINO`. They must exist — the Sheets API
does not create them — and columns `A:AZ` of each belong to the workflow: they are cleared and
the data is rewritten into `A:AF` on every run. Keep anything else (notes, pivots, formulas)
on other tabs.

### 3. Fill in the `Config` node (in all three workflows)

| Field | Example | Notes |
|---|---|---|
| `athenaRegion` | `eu-central-1` | Keep it equal to the region set on the AWS credential. |
| `athenaDatabase` | `operator_ro_normcasino` | Query context. The SQL fully qualifies its tables; all three brands live in this schema. |
| `athenaCatalog` | `Athena-bebohuex` | Falls back to `AwsDataCatalog` when left empty. |
| `athenaWorkgroup` | `primary` | Falls back to `primary` when left empty. |
| `athenaOutputLocation` | `s3://bucket/prefix/` | Leave **empty** if the workgroup enforces a result location. |
| `outputSheetId` | `1x9Dx...yAYM` | The players spreadsheet: the long id between `/d/` and `/edit` in its URL. Same in all three. |
| `outputSheetTab` | `NORMCASINO` / `LACASINO` / `VOLKCASINO` | Tab to overwrite. Quotes in the name are handled. |

### 4. Credentials

- **AWS (Assume Role)** on `Execute Athena Query`, `Poll Athena Query Status` and
  `Get Athena Results Page`. Identical to the analytics report — see
  [analytics-slack-report.md, section 3](analytics-slack-report.md#3-aws-credential--assume-the-cross-account-role).
  An existing credential can be reused.
- **Google Sheets OAuth2 API** (`googleSheetsOAuth2Api`) on `Clear Sheet` and
  `Append Page to Sheet`. The Google account behind it needs **edit** access to the players
  spreadsheet. One credential serves all three workflows.

### 5. Schedule and activate

```text
0 6 * * *       # Norm  06:00 daily, Europe/Warsaw
20 6 * * *      # LA    06:20
40 6 * * *      # Volk  06:40
```

Staggered so the three Athena queries do not compete for the workgroup and the three writes
do not share the Sheets per-user write quota. The workflow timezone is `Europe/Warsaw`, set in
the workflow settings. The 12-month window ends yesterday (complete days only).

Imported workflows arrive inactive.

## Athena

The SQL lives in `Execute Athena Query`. It is the `players_12m_daily_<brand>.sql` query
embedded as-is; the only brand-specific lines are the four `brand = '<BRAND>'` filters.

| Rule | Where |
|---|---|
| Depositors only: at least one successful deposit since `2026-01-01` (effectively all time) | `payments` CTE, `last_deposit_date is not null` |
| 12-month metrics: `current_date - 12 months` up to yesterday | `params` CTE, `date_from` / `date_to_excl` |
| Brand filter | `payments`, `casino`, `mail` CTEs and the final `WHERE` |
| Test accounts excluded | `is_test = false` on billing, dashboard and player |
| Streamers (risk tag `39`) excluded | `NOT EXISTS` on `d_risk_tags_summary` |
| Last withdrawal converted to EUR by the rate on its date | `rates` CTE, `d_fx_rate` |
| Order: brand, then GGR Casino 12M descending, then account | `ORDER BY` |

## Sheet columns

32 columns, `A:AF`, headed with the Athena column names (the SQL aliases). `number` columns are
written as numbers, everything else as text.

| Column | Type | Column | Type |
|---|---|---|---|
| Brand | text | Withdrawal count 12M | number |
| Account | text | Registration date | text |
| Risk business segment | text | First Deposit date | text |
| Risk casino segment | text | First Bet Casino date | text |
| Phone | text | Last Bet Casino date | text |
| Email | text | Bonus rate Casino 12M, % | number |
| Email verified | text (`true`/`false`) | Last Withdrawal date | text |
| Aff id | text | LDD | number |
| Last Deposit date | text | Hold % | number |
| Deposit count 12M | number | Emails sent 12M | number |
| Deposit amount 12M | number | Emails opened 12M | number |
| Withdrawal amount 12M | number | Email open rate 12M, % | number |
| Average bet Casino 12M | number | Last email open date | text |
| Average deposit 12M | number | | |
| GGR Casino 12M | number | | |
| Last withdrawal amount | number | | |
| Margin Casino 12M, % | number | | |
| NGR Casino 12M | number | | |
| Total Casino Clear bonus amount 12M | number | | |

`Parse Athena Page` checks that every column in its `COLUMNS` list exists in the Athena result
and maps by name, so the SQL may reorder columns freely. Adding or renaming a column means
changing the SQL and `COLUMNS` in `Parse Athena Page` together — in all three workflows.

## Failure behaviour

| Situation | Result |
|---|---|
| Athena returns `FAILED` / `CANCELLED` / an unexpected state | `Athena Query Failed` stops the run. Sheet untouched. |
| Query still running after 120 polls (~30 min at 15 s) | `Athena Poll Timeout` stops the run. Sheet untouched. |
| A required column missing from the result | `Parse Athena Page` throws on page 0, right after the clear. Tab left empty. |
| Result above 100 pages (100,000 rows) | `Parse Athena Page` throws. The tab holds the first 100,000 rows. |
| HTTP error on an Athena call | Node retries 3 times, 5 s apart, then the run fails. |
| HTTP error on a Sheets call (incl. 429 rate limit) | Node retries 5 times, 5 s apart, then the run fails. The tab holds the pages written so far. |
| Append succeeded server-side but the response timed out | The retry appends the same page again (duplicate rows). Rare; 120 s timeout on the node. |
| Zero depositors | Header row only. Run succeeds. |

`Export Summary` outputs `pages`, `rowCount` and `lastUpdatedRange` — check them in the
execution when a tab looks too short or too long.

## Observed sizes (first runs, 2026-10-26)

| Brand | Rows | Pages | Athena | Whole run |
|---|---|---|---|---|
| LACASINO | 62,466 | 63 | 26 s (24 GB scanned) | ~4.5 min |
| VOLKCASINO | 3 | 1 | 15 s (6.5 GB scanned) | ~35 s |
| NORMCASINO | 77,718 | 78 | 27 s (52 GB scanned) | ~6 min |

Roughly 3.5–4.5 s per 1000-row page end to end (GetQueryResults + parse + append). The three tabs
together hold about 4.5M cells, under half of the spreadsheet's 10M-cell limit — watch this as the
player base grows.

## Size limits to keep in mind

- A Google spreadsheet holds at most **10,000,000 cells** across all tabs. At 32 columns that is
  about 310,000 rows for the whole file, so roughly 100,000 depositors per brand if the three
  tabs are of similar size. `MAX_PAGES = 100` in `Parse Athena Page` is set to that order of
  magnitude.
- Each run stores every page in n8n's execution data (`saveDataSuccessExecution: all`): a
  100,000-row export is a few tens of MB per execution. Prune executions or switch the setting
  to `none` if the database grows.
- Sheets allows 60 write requests per minute per user. One page is one request and a loop
  iteration takes a couple of seconds, so a run stays well under the limit; the retries on
  `Append Page to Sheet` absorb an occasional 429.

## Testing

Run it manually with *Execute Workflow*. Point `outputSheetTab` at a scratch tab first. Do
not pin data on any node before exporting the workflow back to Git — it is real player data
with phone numbers and e-mails.

## Data handling

The output sheet holds phone numbers and e-mail addresses. Share it only with the people who
work the list, and remember that executions are saved with their data, so the same rows sit in
n8n's execution history.

## Known limitations

- **Clear, then append, page by page.** Not atomic: a failure mid-way leaves a partial tab
  until the next successful run. The tab is never left with stale rows below fresh ones, since
  it is cleared first.
- **No title stamp.** Unlike the depositing-players exports, the spreadsheet title is not
  renamed with a timestamp: three workflows share the file and would overwrite each other's
  stamp. Use the execution list (or `Export Summary`) to see when a tab was last written.
- **Brand and the `2026-01-01` lifetime bound are hardcoded in the SQL**, not in `Config`.
- **No row limit.** Every depositor of the brand is written on every run.
