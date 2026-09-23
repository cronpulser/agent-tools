# Managing jobs in the CronPulser app

App: `https://app.cronpulser.com`. Sidebar pages: Dashboard, Projects, Jobs,
Runners, Alerts, API Keys, Audit Logs, Team, Settings.

## Roles

| Role | Can |
|---|---|
| admin | Everything, in every project |
| member | Create, edit, run, pause, delete jobs and alerts in assigned projects |
| viewer | Read only in assigned projects. No action buttons appear |

If the user says "I see no Edit button", ask which role they have (Team page)
and whether the job's project is assigned to them.

## Where actions live

Index tables (Jobs, Alerts, Runners) are for status and navigation only.
**Run Now, Edit, Pause/Activate, Delete** live on the job's detail page.
Tell the user to click the job name first.

## Create a job

Jobs → **New Job**.

1. **Project**: every job belongs to one project. A project must exist first.
2. **Executor**: HTTP, Remote Runner, or TLS Certificate. Fixed after creation.
3. **Job Name**.
4. Executor-specific fields:
   - HTTP: method, URL (`https://` or `http://`), optional JSON headers, optional JSON payload, retry on/off, response logging mode.
   - Remote Runner: pick an online Runner, then one of its discovered job keys. Keys come from that machine's `runner.yaml`; if the key is missing, the Runner has not been restarted after editing the file.
   - TLS Certificate: a public hostname such as `example.com`. No URL, path, or port.
5. **Cron Schedule**: the builder offers presets and a custom five-field expression, plus a timezone. The editor previews the next five occurrences before saving. Use it to confirm the timezone is right.
6. Save. The job is active immediately unless the plan limit is reached.

## Edit a job

Job detail → **Edit**. All executor types are editable. You can change:

- name, project, cron schedule and timezone
- HTTP: URL, method, headers, payload, retry, response logging
- Runner: which Runner and which discovered job key
- TLS: hostname. Changing it hides results from the previous hostname until the next check.

You cannot change the executor type. Create a new job for that.

Moving a job to another project keeps its history and alert rules. A member
can only move it into a project they are assigned to.

## Pause, activate, run, delete

- **Pause** removes the schedule and frees an active-job slot. History is kept.
- **Activate** re-registers the schedule. It fails if the plan's active-job limit is reached.
- **Run Now** dispatches one execution immediately and works for every executor type. It is the right first test after creating or editing a job.
- **Delete** removes the job, its alert rules, and its history. Ask the user to pause instead if they might need the history.

## Reading a job's detail page

- Header shows executor method (`GET`, `POST`, `RUNNER`, `TLS`) and status dot.
- A yellow **schedule synchronization pending** banner means the job is saved but its schedule registration is being repaired. It resolves on its own; see `troubleshooting.md`.
- **Execution history** table: status, start time, duration, exit code or HTTP status, and a link to the execution detail with stdout/stderr or response body (subject to response logging mode).
- **Success rate** chart over a selectable range.
- **Alerts** section lists rules for this job and links to create one.
- HTTP jobs show a **Secret Key** panel for admins and members. TLS jobs show the last certificate check.
- Logs can be exported to Excel from the history section.

## Projects

Projects → **New Project**. Members and viewers are assigned to projects on the
Team page. A job's project controls who can see it, its logs, and its alerts.

## What is metered

Do not quote numbers. Point to `https://cronpulser.com/pricing/`. The metered
things are: active jobs (HTTP, Runner, and TLS all count; paused jobs do not),
number of projects, number of registered Runners, execution-log retention days,
webhook alerts, API keys and REST API, and MCP. Email alerts, team members,
roles, audit log, CSV export, and Runner execution are on every plan.

When the free plan's active-job limit is hit, new jobs save as inactive with
reason "plan limit". Pausing another job or upgrading frees the slot.

## Timezones

The schedule builder lists UTC plus common zones. A schedule with a timezone is
stored as `CRON_TZ=Europe/Istanbul 0 2 * * *`. Daylight saving changes follow
that zone. Recommend UTC for machine-to-machine work and a local zone only when
a human expects "2 AM local".
