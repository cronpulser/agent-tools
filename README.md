# CronPulser for AI agents

[CronPulser](https://cronpulser.com) is a hosted cron scheduler. It owns the
schedule and starts the work: it calls your HTTP endpoints or webhooks on a
cron expression with a timezone, runs commands on your own Linux or Windows
machines through CronPulser Runner, and records every execution with status,
timing and optional response logging. Alerts fire on failures, slow runs,
unexpected output, and schedules that stop running.

This repository holds the two ways an AI agent can work with CronPulser:

| | What it is | Account access | Plans |
| --- | --- | --- | --- |
| [Remote MCP server](#remote-mcp-server) | Tools to manage projects, HTTP, Runner and TLS jobs, alerts and Runners, and read run history and failure summaries | Scoped OAuth 2.1, as you | Pro and Scale |
| [Agent Skill](#agent-skill) | Markdown guidance that teaches a coding agent how to use CronPulser well | None | Free on every plan |

## Remote MCP server

Endpoint (Streamable HTTP, OAuth 2.1 with dynamic client registration and PKCE):

```
https://api.cronpulser.com/mcp
```

Claude Code:

```bash
claude mcp add --transport http cronpulser https://api.cronpulser.com/mcp
```

Any MCP client that supports remote HTTP servers with OAuth discovers the
authorization endpoints automatically. Sign in to CronPulser in the browser,
review the requested scopes, approve. The client receives short-lived access
and a rotating refresh token, never a tenant-wide API key.

Tools in this release:

| Tool | Operation | Scope |
| --- | --- | --- |
| `list_projects` | Read accessible projects | `projects:read` |
| `create_project`, `update_project` | Create or change a project (admins) | `jobs:write` |
| `delete_project` | Delete a project with its jobs (destructive, admins) | `jobs:write` |
| `list_jobs` | Read safe job metadata | `jobs:read` |
| `get_job` | Read one accessible job | `jobs:read` |
| `list_execution_logs` | Read execution status and timing | `logs:read` |
| `get_job_stats` | Read a job's success rate and latest failures | `logs:read` |
| `get_failure_summary` | Find failing jobs and jobs that did not run | `logs:read` |
| `create_http_job` | Create and schedule an HTTP job | `jobs:write` |
| `create_runner_job` | Schedule an allow-listed command on a Runner | `jobs:write` |
| `create_tls_job` | Schedule a TLS certificate check | `jobs:write` |
| `update_job` | Change only the fields you send | `jobs:write` |
| `delete_job` | Delete a job and its schedule (destructive) | `jobs:write` |
| `pause_job`, `resume_job` | Change scheduling state | `jobs:write` |
| `trigger_job` | Start one immediate execution | `jobs:trigger` |
| `list_alerts`, `get_alert` | Read alert rules | `jobs:read` |
| `create_alert`, `update_alert` | Add or change any alert type | `jobs:write` |
| `set_alert_active` | Turn an alert on or off | `jobs:write` |
| `delete_alert` | Delete an alert (destructive) | `jobs:write` |
| `list_alert_history` | Read when alerts fired | `jobs:read` |
| `list_runners`, `get_runner` | Read Runner status, version and upgrade command | `jobs:read` |
| `create_runner` | Create a Runner and its single-use install token (admins) | `jobs:write` |

Billing, team management and secret rotation are intentionally not exposed.
Write tools require an admin or member role; viewers stay read-only. Tool
results never include job secrets, header values or payloads, webhook header
values, response bodies, or Runner output; header values are write-only. The
only credential returned is the single-use Runner install token from
`create_runner`. Every mutation is written to the tenant audit log with an MCP marker.

Registry listings: [Official MCP Registry](https://registry.modelcontextprotocol.io/v0.1/servers?search=com.cronpulser/cronpulser)
(`com.cronpulser/cronpulser`, see [server.json](server.json)) and
[Smithery](https://smithery.ai/servers/info-lo62/cronpulser).

Documentation: https://cronpulser.com/docs/mcp/

## Agent Skill

The skill in [`skills/cronpulser/`](skills/cronpulser/) follows the
[Agent Skills](https://agentskills.io) format: a `SKILL.md` plus reference
files. Once installed, the agent loads it whenever CronPulser, Runner,
`runner.yaml` or `X-CronPulser-Secret` comes up.

It covers jobs in the app (HTTP, Runner and TLS certificate jobs), Linux
Runner scripts including PostgreSQL and MySQL backups from Docker containers
with least-privilege sudo, Windows Runner scripts including SQL Server backup
to a network share, verifying `X-CronPulser-Secret` on receiving endpoints
with Express and FastAPI examples, every alert rule type, and what each
status and error means.

It is guidance only. It never asks for an API key or job secret and never
touches your account.

Install into Claude Code (macOS and Linux):

```bash
curl -fsSL https://cronpulser.com/skills/install.sh | sh
```

Other agents take the skills directory as an argument:

```bash
# Codex
curl -fsSL https://cronpulser.com/skills/install.sh | sh -s -- ~/.codex/skills
# Cursor
curl -fsSL https://cronpulser.com/skills/install.sh | sh -s -- ~/.cursor/skills
# Project-scoped for a team repository
curl -fsSL https://cronpulser.com/skills/install.sh | sh -s -- .claude/skills
```

Windows PowerShell:

```powershell
irm https://cronpulser.com/skills/install.ps1 | iex
```

Or with the skills CLI:

```bash
npx skills add cronpulser/agent-tools
```

Re-run the installer to update. The skill is versioned with the product, so
it always describes current behavior.

Documentation: https://cronpulser.com/docs/agents/

## License

The contents of this repository are released under the [MIT License](LICENSE).
CronPulser itself is a hosted service; see https://cronpulser.com/pricing/.
