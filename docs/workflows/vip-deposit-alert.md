# VIP Deposit Alert

Scheduled workflow that runs one Athena query and posts a Slack Incoming Webhook message
listing the VIP players whose deposits are stuck: money the player tried to bring in during
the last 48 hours that nothing approved.

Workflow file: [workflows/vip-deposit-alert.json](../../workflows/vip-deposit-alert.json)

Verified against n8n `2.20.7`.

Adapted from a standalone VIP alert that talked to Athena with the generic `aws` credential
and posted through the Slack node. It now uses the same transport as
[analytics-slack-report](analytics-slack-report.md): the `awsAssumeRole` credential for
Athena and an Incoming Webhook for Slack.

## Portability

The workflow is **self-contained**. It carries its own settings in a `Config` node and reads
nothing from the host: no `$env`, no `$vars`, no dependency on this repository's
`docker-compose.yml` or `.env`. The file imports and runs unchanged on any n8n instance.

## Node chain

```text
Schedule Trigger
  -> Config                        (deployment settings)
  -> Execute Athena Query          (StartQueryExecution)
  -> Wait 5s
  -> Poll Athena Query Status      (GetQueryExecution)
  -> Athena Query State            (Switch)
       SUCCEEDED  -> Get Athena Results (GetQueryResults)
                     -> Normalize VIP Deposit Data
                     -> Stuck VIPs Found?            (IF, count > 0)
                          true  -> Build VIP Alert Payload
                          false -> Build All-Clear Payload
                     -> HTTP Request — Send VIP Alert to Slack
       QUEUED     -> Poll Attempts Remaining? -> back to Wait 5s
       RUNNING    -> Poll Attempts Remaining? -> back to Wait 5s
       FAILED     -> Athena Query Failed   (Stop and Error)
       CANCELLED  -> Athena Query Failed   (Stop and Error)
       fallback   -> Athena Query Failed   (Stop and Error)
```

One execution sends exactly one Slack message: either the alert digest or the all-clear.
Both branches feed the same HTTP Request node and the same `slackWebhookUrl`.

Zero candidates is a normal night, not a failure. To keep quiet nights silent, disconnect
`Build All-Clear Payload` from `Stuck VIPs Found?` — the false branch then ends there and the
execution still finishes green.

## Setup on a new n8n instance

### 1. Import

```text
Workflows -> Import from File -> vip-deposit-alert.json
```

Importing the same file twice creates two workflows. To update an existing one, open it and
import from inside its canvas. See [authoring.md](authoring.md).

### 2. Fill in the `Config` node

The same six fields as the analytics report, read downstream through
`$('Config').first().json.<field>`:

| Field | Example | Notes |
|---|---|---|
| `athenaRegion` | `eu-central-1` | Region of the Athena endpoint. Keep it equal to the region set on the AWS credential. |
| `athenaDatabase` | `operator_ro_normcasino` | Query context. The SQL fully qualifies its tables. |
| `athenaCatalog` | `AwsDataCatalog` | Falls back to `AwsDataCatalog` when left empty. |
| `athenaWorkgroup` | `primary` | Falls back to `primary` when left empty. |
| `athenaOutputLocation` | `s3://bucket/prefix/` | Leave **empty** if the workgroup enforces a result location — the field is then omitted from the request entirely. |
| `slackWebhookUrl` | `https://hooks.slack.com/services/...` | **Secret.** Ships as a placeholder. |

`slackWebhookUrl` is the one secret in this workflow. Fill it in on the n8n instance, never in
the file committed here, and clear it again before exporting the workflow back to Git.

### 3. AWS credential

Athena is reached through n8n's predefined **AWS (Assume Role)** credential (`awsAssumeRole`),
selected on `Execute Athena Query`, `Poll Athena Query Status` and `Get Athena Results`. The
setup is identical to the analytics report — including the mandatory External ID field — so it
is documented once, in
[analytics-slack-report.md, section 3](analytics-slack-report.md#3-aws-credential--assume-the-cross-account-role).

A credential already created for the analytics report can be selected here as is. Only the
Role Session Name is worth changing (`n8n-vip-deposit-alert`) if you want the two workflows
told apart in CloudTrail.

### 4. Schedule and activate

```text
30 4 * * *      # 04:30 daily, Europe/Kyiv
```

The time is deliberate: the query reads complete days only (`create_date < CURRENT_DATE`), so
it has to run after the nightly billing load has settled. The workflow timezone is
`Europe/Kyiv`, set in the workflow settings, so the cron fields are Kyiv local time.

Imported workflows arrive inactive.

## Athena

The SQL lives in the `Execute Athena Query` node, in the JSON body of the
`StartQueryExecution` request, and is the single source of truth for who gets alerted and in
which order. The code nodes only reshape and format what it returns.

Source table: `operator_ro_normcasino.agg_billing_report_hyper`, deposits only.

**Who counts as VIP** is one line:

```sql
AND r.risk_casino_segment IN ('VIP-Silver', 'VIP-Gold', 'VIP-Platinum')
```

**The alert condition**, evaluated per player x brand over a rolling 48 hours:

| Condition | Meaning |
|---|---|
| `new_cnt_cur > 0` | at least one deposit is sitting in `New` |
| `success_cnt_cur = 0` | nothing was approved in that window |
| `NOT (new_cnt_prev > 0 AND success_cnt_prev = 0)` | yesterday's window did not already satisfy the first two |

The third condition is the deduplication: it compares the current window `[D-2, today)` with
yesterday's `[D-3, D-1)` and reports a player only on the day they became stuck, instead of
every morning until they give up. It is why the query reads three days of data to alert on
two.

`Fail` transactions are **context, not a trigger**. A player who only ever got declined is not
reported; a reported player's last provider decline is attached so the manager knows what to
say. Ordering is by pending plus declined amount together — a VIP with 32 declines on 835 EUR
is a more urgent call than one with a single small pending attempt.

Excluded brands (`SBNORM`, `VOLKCASINO`, `IKRACASINO`) match the analytics report's list, in
one `NOT IN` near the top of the query. `is_test` is read as `COALESCE(is_test, false)`, so a
missing flag never hides a real VIP.

Results are read as a single page of up to 1000 rows. `Normalize VIP Deposit Data` throws if
Athena returns a `NextToken`: at that point the alert condition, not the pagination, is what
needs looking at.

## Message layout

One digest message per execution:

```text
🚨 VIP deposits stuck — 3 players
22 Sep 2026 • Europe/Kyiv • Updated 04:31

*4,120 EUR pending* and *836 EUR declined* across 3 VIP players in the last 48h, none of it approved.

┌──────────┬────────────┬──────────┬─────────┬───────────┬──────────┬────────────┬───────────────┬──────────────┐
│ Player   │ Brand      │ Segment  │ Pending │ Pending € │ Declined │ Declined € │ Method        │ Last try     │
└──────────┴────────────┴──────────┴─────────┴───────────┴──────────┴────────────┴───────────────┴──────────────┘

*Last provider decline*
• P-100001 (NORMCASINO) — 32 declined · Issuer decline: Do not honour — contact card issuer…
```

Table rows keep Athena's order, which is the call order. Decline reasons are free provider
text that does not fit a table cell, so they sit below it — at most 10 lines, each truncated
to 120 characters, with a "…and N more" line when the list is longer.

Slack's table limits (10 columns, 100 rows) are asserted in `Build VIP Alert Payload`: a
layout change that adds an eleventh column fails loudly in n8n rather than silently in Slack.

## Failure behaviour

| Situation | Result |
|---|---|
| Athena returns `FAILED` / `CANCELLED` / an unexpected state | `Athena Query Failed` stops the execution with the state and Athena's `StateChangeReason`. No Slack message. |
| Query still running after 120 polls (~10 min) | `Athena Poll Timeout` stops the execution. No Slack message. |
| HTTP error on any Athena call or on Slack | Node retries 3 times, 5 s apart, then the execution fails. |
| Athena succeeds, zero candidates | All-clear message. Execution succeeds. |
| Athena response malformed, or a required column missing | `Normalize VIP Deposit Data` throws with what drifted. No Slack message. |

A failed execution is visible in *Executions*; nothing pages anyone by itself. `retryOnFail`
on `Execute Athena Query` is safe because `ClientRequestToken` is derived from the execution
id — a retry rejoins the same Athena query instead of starting a second one.

## Testing

Run it manually with the *Execute Workflow* button; the Schedule Trigger is skipped and the
rest of the chain runs. To see the alert branch on a quiet day, pin data on
`Normalize VIP Deposit Data` with a handful of candidates, or temporarily relax the
`success_cnt_cur = 0` condition in the SQL. Remember to remove pinned data before exporting —
it is real player data and must not reach Git.

## Known limitations

- **One page of results.** Up to 1000 candidates; beyond that the run throws rather than
  reporting a partial list.
- **Segment list is hardcoded in the SQL.** Adding a VIP tier means editing the query, not the
  `Config` node.
- **No per-player thread.** A digest cannot be assigned per player in Slack. If the team wants
  to work the list by claiming players, this needs to become one message per player again.
- **Decline reasons are capped** at 10 players per message; the table still lists everyone.
