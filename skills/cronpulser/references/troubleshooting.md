# Troubleshooting: what the app shows and what to ask

Work from what the user sees. Each entry: symptom, meaning, next question or fix.

## Job page and list

**Yellow banner "schedule synchronization pending"**
The job is saved but its schedule registration with the scheduler did not
complete. CronPulser repairs this automatically within minutes; runs are not
lost once repaired. Ask them to reload after a few minutes. If it persists for
more than 15 minutes, it is a platform issue: contact support with the job name.

**Status dot grey with title "Inactive: plan limit"**
The job was saved as inactive because the active-job allowance is used up.
Fix: pause another job or upgrade, then Activate.

**Status dot grey with "Inactive: Runner inactive"**
The Runner this job uses is disabled or over the plan's Runner limit. Check the
Runners page. Re-enable or free a Runner slot, then Activate the job.

**"I cannot edit the job" / no Edit button**
Ask: which role (Team page)? Viewers see no actions. Are they on the job detail
page? Index tables never show actions.

**"Edit fails with 'runner job key has not been discovered'"**
The selected key is not in that Runner's current discovered list. The Runner
must be restarted after `runner.yaml` changes, and `validate` must pass.

**"Edit fails with 'runner is unavailable'"**
They are activating a job whose Runner is disabled or inactive.

**"Project not found" when moving a job**
A member tried to move it into a project they are not assigned to.

## Executions

**Failed immediately, duration ~0, Runner job**
The Runner was offline at dispatch. CronPulser does not queue missed runs.
Check `systemctl status cronpulser-runner` or `Get-Service CronPulserRunner`
and the Runners page online state.

**Failed, stderr "Permission denied" / "Access is denied"**
The service identity (`cronpulser` on Linux, LocalService on Windows) lacks
access to a path. On Windows with SQL Server backups, the SQL Server service
account is the one writing the file. See the Runner references.

**Failed, exit code -1, message "execution timed out after 15m0s"**
The job exceeded its `timeout` in `runner.yaml` and the Runner killed it.
Raise the timeout only if the run legitimately takes longer; otherwise find
what hung. Non-interactive flags (`-NonInteractive`, no `sudo` prompts) prevent
most hangs. Exit code -1 with a different message means the process could not
be started at all: check the path and execute permission.

**Failed, "command not found" or exit 127**
A binary inside the script was not on the service's minimal `PATH`. Use
absolute paths.

**Runner validate says the command must be absolute / not executable**
`command` must be an absolute path to an executable file. On Windows point it
at `powershell.exe` and pass the script with `-File`.

**HTTP job: 401 or 403**
The endpoint rejected the request. Ask whether it verifies `X-CronPulser-Secret`
and whether the expected secret matches the job's Secret Key shown on the job
detail page. Secrets are per job; a copied job has a different secret.

**HTTP job: status 5xx then "retry"**
Retry is enabled. CronPulser waited at least five seconds and sent the request
once more. Confirm the endpoint is safe to call twice (idempotent).

**HTTP job: "destination must resolve only to public IP addresses; use Runner for private services"**
CronPulser only calls public destinations. Private IPs, localhost, and internal
hostnames are refused. Use a Runner on the private network instead.

**HTTP job: "cross-origin redirects are blocked; configure the final destination URL"**
The endpoint redirected to another host. Put the final URL in the job.

**HTTP job: succeeds but the response body is empty in history**
Response logging is `never` or `on_failure`. Set it to `always` if they need
the body, and note it is stored for the retention window. Alert rules still
evaluated the body; only the stored copy is affected.

**TLS job: results disappeared after editing**
Changing the hostname hides previous results until the next check. Run Now.

## Alerts

**Alert did not fire although runs fail**
Ask: does the rule have at least one recipient ticked? Is the rule active? Is
it inside its cooldown (default 30 minutes)? For Output Match with a JSON path,
is the output valid JSON, under 4 KB for HTTP bodies, and does the path exist?
Missing output, invalid JSON, or a missing path currently count as no match.

**Alert fires repeatedly**
Expected while the condition holds; the cooldown sets the interval. Raise the
cooldown or fix the job.

**No Execution alert fired but the job ran**
Grace minutes are shorter than queue delay plus run time. Raise `grace_minutes`.

**Webhook option greyed out**
Webhooks are on paid plans only. Email works on every plan.

## Runner machine

**Runner shows Offline**
On the machine: service stopped, outbound 443 blocked, or credentials rotated
elsewhere. Check the service status and logs. If the log shows an auth error,
generate a rotation token on the Runners page and run `rotate` on the machine.

**Job key does not appear on the Runners page**
`runner.yaml` was edited but the service was not restarted, or `validate` failed.
Run validate, fix the YAML, restart, then refresh the page.

**Duplicate executions after a machine restart**
Should not happen: the Runner keeps `executions.json` to deduplicate. If it was
deleted, one duplicate can occur once. Do not schedule deletion of that file.

## When to hand off

Give the user the job name, execution time in UTC, and the exact message, and
point them to support when: the pending banner persists, executions are missing
with the Runner online and the job active, or billing state looks wrong.
