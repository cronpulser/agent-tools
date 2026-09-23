# Alert rules

Alerts belong to one job. Create them from the job's Alerts section
(New Alert with the job preselected) or from the Alerts page. Each job can have
a limited number of active rules. A rule has a type, a condition, recipients,
an optional webhook, and a cooldown.

## Types

| Type | Executor | Condition fields | Fires when |
|---|---|---|---|
| Execution Failure | all | none | The execution failed: non-2xx status, transport error, timeout, non-zero exit code, Runner offline at dispatch, TLS check failure |
| Consecutive Failures | all | `count` | The last N executions all failed |
| Status Code | HTTP | `operator` (`gte gt eq neq lte lt`), `value` | The HTTP status matches, e.g. `gte 400` |
| Latency Threshold | HTTP, Runner | `threshold_ms` | Duration exceeds the threshold |
| Output Match | HTTP, Runner | `source`, `mode`, `operator`, `value`, `json_path` | Output text or a JSON path value matches |
| No Execution | all | `grace_minutes` | No execution started within grace minutes after a scheduled time |
| TLS Certificate | TLS | `days_before_expiry`, `include_failures` | Certificate expires within N days, optionally on check failure |

Output Match details:

- `source`: `response_body` for HTTP; `stdout` or `stderr` for Runner.
- `mode`: `text` compares the whole output; `json_path` extracts a value first (for example `$.stopped` or `$.result.count`).
- `operator`: `contains`, `not_contains`, `eq`, `neq`; with `json_path` also `gt`, `gte`, `lt`, `lte`.
- Evaluation uses the live output regardless of the response logging mode, so `never` does not disable Output Match. HTTP bodies are read up to 4 KB; a JSON body longer than that is cut and fails to parse, so keep summaries small. Runner stdout and stderr are evaluated in full up to 1 MiB.

## Recipients and channels

- **Email**: pick team members who have access to the job's project. Included on every plan. New rules preselect nobody; the user must tick recipients.
- **Webhook URL**: an HTTPS endpoint that receives a JSON POST per notification. Paid plans only.

## Cooldown

`cooldown_minutes`, 1 to 1440, default 30. After a notification, the same rule
stays silent for that long even if every execution keeps matching. Then it
notifies again while the condition still holds. There is no built-in
"recovered" notification for most rules; suggest a short cooldown plus the
success-rate chart for that.

## Delivery

Notifications are queued durably and retried. The Alerts page and each alert's
detail show delivery status and history. Alerts are evaluated after each
execution; No Execution rules are the only ones evaluated on the schedule
itself. Pausing a job stops its No Execution rule as well.

## Recommended baseline per job

1. Execution Failure, cooldown 30, email to the on-call person.
2. No Execution with grace 10 to 15 minutes for anything that must run.
3. For backups: Output Match on `stdout` contains `backup_complete` inverted with `not_contains`, so a script that exits 0 without finishing still alerts.
4. For Runner machines: Consecutive Failures 3 to catch a Runner that is offline at every dispatch.
