# Alert rules

Alerts belong to one job. Create them from the job's Alerts section
(New Alert with the job preselected) or from the Alerts page. Each job can have
a limited number of active rules. A rule has a type, a condition, recipients,
an optional webhook, and incident settings (reminders and recovery).

## Types

| Type | Executor | Condition fields | Fires when |
|---|---|---|---|
| Execution Failure | all | none | The execution failed: non-2xx status, transport error, timeout, non-zero exit code, Runner offline at dispatch, TLS check failure |
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
- **Custom webhook body** (optional, needs a webhook URL): a JSON template sent instead of the standard body, e.g. to open a ticket: `{"summary":"{{alert_name}}: {{job_name}}","affected_web_site":"{{job_host}}","details":"{{message}}"}`. Placeholders work inside string values: `alert_name`, `alert_type`, `message`, `triggered_at`, `job_id`, `job_name`, `job_url`, `job_host`, `execution_status`, `executed_at`, `status_code`, `latency_ms`, `error_kind`, `event` (`triggered`, `reminder`, `resolved`), `incident_started_at`, `resolved_at`. Unknown placeholders are rejected on save. **Send test webhook** in the form posts the unsaved body once with sample data and shows the receiver's HTTP status and response, which is the fastest way to debug a 4xx from a ticket API.
- **Webhook headers** (optional, up to 10, needs a webhook URL): extra request headers such as `X-Api-Key` or `Authorization` for receivers that require authentication. Values are encrypted and never shown again; leave a saved value blank to keep it. Changing the webhook host requires re-entering the values. `Content-Type`, `User-Agent`, `Host`, and `X-CronPulser-*` are reserved.

## Incidents, reminders, and recovery

An alert is OK or Firing. The first matching result opens an incident and sends
one notification; later matches stay quiet. Settings (`options`):

- `reminder_minutes`: repeat while still firing every N minutes, `0` (off, default) or 10 to 10080 (7 days). The app offers Off, 15 minutes, 1 hour, 4 hours, 24 hours, and Custom (minutes, hours, or days).
- `recovery_checks`: consecutive passing results that close the incident, 1 to 5 (default 1).
- `notify_on_recovery`: send a "Resolved" notification with the outage duration (default on).
- `trigger`: `{failures: N, window: M}` fires when N of the last M checks match, 1 ≤ N ≤ M ≤ 10. Default 1 of 1 (every match). The app offers "Immediately" and "Custom: N of the last M checks" (starting at 2 of 2; N of N means N in a row). Not used by No Execution. The former Consecutive Failures type is now Execution Failure with N of N.

Pausing the job, changing its URL, method, or Runner, disabling the rule, or
changing its condition closes the incident without a notice. **Mark resolved**
on the alert page (MCP `resolve_alert`) does the same.
With the job's Retry on, a failure followed by a passing retry 15 seconds later
does not alert.

## Delivery

Notifications are queued durably and retried. The Alerts page and each alert's
detail show delivery status and history. Alerts are evaluated after each
execution; No Execution rules are the only ones evaluated on the schedule
itself. Pausing a job stops its No Execution rule as well.

## Recommended baseline per job

1. Execution Failure, recovery notice on, email to the on-call person; turn on Retry for GET checks.
2. No Execution with grace 10 to 15 minutes for anything that must run.
3. For backups: Output Match on `stdout` contains `backup_complete` inverted with `not_contains`, so a script that exits 0 without finishing still alerts.
4. For Runner machines: Execution Failure with "3 in a row" to catch a Runner that is offline at every dispatch.
