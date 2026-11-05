# VIP Deposit Alert

Scheduled workflow that runs one Athena query and posts a Slack Incoming Webhook message
listing the VIP players whose deposits are stuck: deposit attempts from the last 48 hours
that are still pending and came after the player's last successful deposit.

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

The time is deliberate: the query reads complete days only (`action_date < current_date`), so
it has to run after the nightly load has settled. The workflow timezone is
`Europe/Kyiv`, set in the workflow settings, so the cron fields are Kyiv local time.

Imported workflows arrive inactive.

## Athena

The SQL lives in the `Execute Athena Query` node, in the JSON body of the
`StartQueryExecution` request, and is the single source of truth for who gets alerted and in
which order. The code nodes only reshape and format what it returns.

Source tables (reworked with the VIP team on 2026-10-13, replacing
`agg_billing_report_hyper`):

| Table | Role |
|---|---|
| `operator_ro_normcasino.agg_player_summary` | who is VIP: `risk_casino_segment`, `is_test`, brand |
| `operator_ro_normcasino.f_payment` | deposit events with `action_ts`, status, amounts, decline reason, method, paygroup, provider |
| `operator_ro_normcasino.d_fx_rate` | conversion of `send_amount` to EUR on the event date |

**Who counts as VIP** is one line in the `vip` CTE:

```sql
and s.risk_casino_segment in ('Good guys-X', 'VIP-Silver', 'VIP-Gold', 'VIP-Platinum')
```

**The alert condition**, evaluated per player x brand, with both thresholds in the `params`
CTE (`min_stuck_attempts`, currently 1, and `min_episode_eur`, currently 100, agreed with the
PSP team on 2026-10-14):

| Condition | Meaning |
|---|---|
| `pending_cnt_cur >= min_stuck_attempts` | at least one `New` deposit in `[D-2, today)` that happened **after** the player's last `Success` |
| `pending_eur_cur + declined_eur_cur >= min_episode_eur` | the episode is worth a call: pending plus declined EUR after that success reaches the threshold |
| `not (pending_cnt_prev >= min_stuck_attempts and pending_eur_prev + declined_eur_prev >= min_episode_eur)` | yesterday's window `[D-3, D-1)` did not already satisfy both |

Counting only attempts after the last success is what changed versus the first version: a
VIP who got stuck, then deposited successfully, then got stuck again is reported on the new
episode, and an old pending attempt that was later followed by a success no longer counts.

The third condition is the deduplication: it reports a player only on the day the episode
first crossed both thresholds, instead of every morning until they give up. It is why the query reads four days of
events to alert on two.

`Fail` transactions are **context, not a trigger**. A player who only ever got declined is not
reported; a reported player's last provider decline *after the same last success* is attached
so the manager knows what to say. Ordering is by pending plus declined amount together — a VIP
with 32 declines on 835 EUR is a more urgent call than one with a single small pending attempt.

Only `SBNORM` is excluded, in the `vip` CTE. Test players are filtered on both tables
(`is_test` on the summary, `player_is_test` on the events) as `coalesce(..., false)`, so a
missing flag never hides a real VIP.

**Result columns**, in the order `Normalize VIP Deposit Data` expects them:

| Column | Type | Notes |
|---|---|---|
| `player_id`, `brand`, `vip_segment` | text | |
| `pending_attempts`, `pending_amount_eur` | number | `New` attempts after the last success, current window |
| `last_attempt_ts` | timestamp | last pending attempt, with time |
| `hours_since_attempt` | number | computed by Athena at query time |
| `last_success_ts` | timestamp | NULL when the player had no success in the four-day window |
| `currency`, `country`, `payment_method`, `paygroup`, `provider`, `method_type` | text | from the last pending attempt |
| `declined_attempts`, `declined_amount_eur` | number | `Fail` attempts after the same last success |
| `decline_group`, `decline_message` | text | last such decline, NULL when there was none |

Renaming or dropping any of these makes `Normalize VIP Deposit Data` throw with the list of
missing columns. Adding a column is harmless until the table in `Build VIP Alert Payload` wants
it.

**No backslashes in the SQL.** The query lives inside an n8n expression, and the expression
engine collapses `\\` to `\` before the HTTP Request node parses the body as JSON, so a
backslash anywhere in the query, even in a comment such as a Windows path, fails the node with
"The value in the JSON Body field is not valid JSON". Use forward slashes.

Results are read as a single page of up to 1000 rows. `Normalize VIP Deposit Data` throws if
Athena returns a `NextToken`: at that point the alert condition, not the pagination, is what
needs looking at.

## Message layout

One digest message per execution:

```text
🚨 VIP deposits stuck — 3 players
22 Sep 2026 • UTC • Updated 01:31

*4,120 EUR pending* and *836 EUR declined* across 3 VIP players, attempts on 20–21 Sep (UTC), all of them after the player's last successful deposit.

┌──────────┬────────────┬──────────┬─────────┬───────────┬──────────┬────────────┬────────────────────────┬────────────────────────┬──────────────┐
│ Player   │ Brand      │ Segment  │ Pending │ Pending € │ Declined │ Declined € │ Method                 │ Last try               │ Last success │
│ P-100001 │ NORMCASINO │ Gold     │       3 │     1,251 │        2 │        300 │ card / TPay / nc_tpay  │ 12 Oct 14:05 · 18h ago │ 10 Oct 09:12 │
└──────────┴────────────┴──────────┴─────────┴───────────┴──────────┴────────────┴────────────────────────┴────────────────────────┴──────────────┘

*Last provider decline*
• P-100001 (NORMCASINO) — 32 declined · Issuer decline: Do not honour — contact card issuer…
```

Every time in the message is UTC, including the header. Athena returns `last_attempt_ts` and
`last_success_ts` in UTC and the query's windows are UTC days, so the message is labelled UTC
rather than converted: that keeps "Last try", "Nh ago" and "Updated" consistent with each
other. The window is named by its dates ("20–21 Sep") instead of "last 48h", which drifts as
the day goes on.

Table rows keep Athena's order, which is the call order. `Method` folds `payment_method`,
`provider` and `paygroup` into one cell, skipping the parts that are NULL. `Last try` carries
`hours_since_attempt` as "· Nh ago"; `Last success` is "—" for a player with no success in the
window. Decline reasons are free provider text that does not fit a table cell, so they sit
below it — at most 10 lines, each truncated to 120 characters, with a "…and N more" line when
the list is longer.

Slack's table limits (10 columns, 100 rows) are asserted in `Build VIP Alert Payload`. The
table is now exactly at the 10-column limit: adding another column means folding two together
first, otherwise the node fails loudly in n8n rather than silently in Slack.

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
`Normalize VIP Deposit Data` with a handful of candidates, or temporarily drop the
`pending_cnt_prev < p.min_stuck_attempts` deduplication condition in the SQL. Remember to remove pinned data before exporting —
it is real player data and must not reach Git.

## Known limitations

- **One page of results.** Up to 1000 candidates; beyond that the run throws rather than
  reporting a partial list.
- **Segment list and thresholds are hardcoded in the SQL.** Adding a VIP tier or changing
  `min_stuck_attempts` / `min_episode_eur` means editing the query, not the `Config` node.
- **No per-player thread.** A digest cannot be assigned per player in Slack. If the team wants
  to work the list by claiming players, this needs to become one message per player again.
- **Decline reasons are capped** at 10 players per message; the table still lists everyone.
