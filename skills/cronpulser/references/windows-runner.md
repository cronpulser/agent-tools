# Windows Server Runner: install, configure, and write jobs

Supported: Windows Server 2019, 2022, 2025 on AMD64. The Runner installs as the
automatic `CronPulserRunner` service running as **LocalService**. It opens one
outbound connection to `api.cronpulser.com:443`; no inbound firewall rule. A job
that is already running keeps running if the connection drops; its result is
sent once the Runner reconnects.

The Runner is open source (MIT): `https://github.com/cronpulser/cronpulser-runner`.
A user-facing walkthrough with a Task Scheduler comparison is at
`https://cronpulser.com/guides/windows-job-scheduler/`.

## Install

1. In the app: **Runners → New Runner**, enter a name, **Create Runner**. In the
   dialog switch to **Windows Server** and use **Copy command** (one-time token,
   valid 24 hours, shown once).
2. In PowerShell opened as Administrator:

```powershell
Invoke-WebRequest https://downloads.cronpulser.com/runner/install-runner.ps1 -OutFile .\install-runner.ps1
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\install-runner.ps1 `
  -Server https://api.cronpulser.com `
  -Token cp_runner_xxxxxxxxx `
  -Version 0.3.5
```

The installer downloads the binary from the GitHub Release for that version
(0.3.1 and 0.3.2 come from `downloads.cronpulser.com`) and refuses to run it
unless its SHA-256 matches the release's `SHA256SUMS`.

## Upgrade

Upgrade with `-Upgrade`; it keeps the registration, credentials, the
`runner.yaml` jobs and the execution cache, and needs no token:

```powershell
Invoke-WebRequest https://downloads.cronpulser.com/runner/install-runner.ps1 -OutFile .\install-runner.ps1
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\install-runner.ps1 -Upgrade -Version 0.3.5
```

The new binary validates the existing `runner.yaml` first and changes nothing
if it is invalid. It stops the service (Windows cannot replace a running
executable), keeps the old binary as `cronpulser-runner.previous.exe`, starts
the service and restores the old binary automatically if the service does not
stay running. A job running during the upgrade is reported as failed, so
upgrade between runs. Without `-Upgrade` the installer stops because the
service already exists.

## First job: smoke test

In PowerShell as Administrator:

```powershell
New-Item -ItemType Directory -Force C:\ProgramData\CronPulser\scripts | Out-Null
Set-Content -Path C:\ProgramData\CronPulser\scripts\hello.ps1 -Value 'Write-Output "hello from $env:COMPUTERNAME at $((Get-Date).ToUniversalTime().ToString(''s''))Z"'
icacls C:\ProgramData\CronPulser\scripts /inheritance:r /grant:r "*S-1-5-32-544:(OI)(CI)F" "*S-1-5-18:(OI)(CI)F" "*S-1-5-19:(OI)(CI)RX"
notepad C:\ProgramData\CronPulser\runner.yaml   # replace `jobs: {}` with the block below
```

```yaml
jobs:
  hello:
    command: C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
    arguments: [-NoProfile, -NonInteractive, -File, C:\ProgramData\CronPulser\scripts\hello.ps1]
    workingDirectory: C:\ProgramData\CronPulser\scripts
    timeout: 30
```

Validate and restart (see below), create a **Remote Runner** job with the
`hello` key in the app, click **Run Now**, and expect status success, exit code
0, and `hello from <COMPUTERNAME> at <time>`.

## Paths

| Purpose | Path |
|---|---|
| Binary | `C:\Program Files\CronPulser\cronpulser-runner.exe` |
| Config | `C:\ProgramData\CronPulser\runner.yaml` |
| Cache | `C:\ProgramData\CronPulser\state\executions.json` |
| Scripts | `C:\ProgramData\CronPulser\scripts\` (Administrators and SYSTEM full, LOCAL SERVICE read and execute, no inheritance) |
| Service | `CronPulserRunner` |
| Events | Application log, provider `CronPulserRunner` |

## runner.yaml on Windows

PowerShell is the executable; the script is an argument. Never put a whole
command line in `command`. Only `.exe` and `.com` files are accepted as
`command`. A fresh install contains `jobs: {}`; replace that line, never add a
second `jobs:` key. Edit the file from an Administrator session.

```yaml
jobs:
  backup-orders-sqlserver:
    command: C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
    arguments: [-NoProfile, -NonInteractive, -File, C:\ProgramData\CronPulser\scripts\backup-orders-sqlserver.ps1]
    workingDirectory: C:\ProgramData\CronPulser\scripts
    timeout: 2h
```

- `-NoProfile` avoids machine-specific profile side effects.
- `-NonInteractive` makes any prompt fail instead of hanging until the timeout.
- Use `-File`, not `-Command`, so the script path is one argument.
- For PowerShell 7 use `C:\Program Files\PowerShell\7\pwsh.exe` with the same arguments.

Validate and restart from an elevated PowerShell after every edit:

```powershell
& 'C:\Program Files\CronPulser\cronpulser-runner.exe' validate
Restart-Service CronPulserRunner
Get-Service CronPulserRunner
```

## LocalService: what it can and cannot do

- Minimal local privileges. Grant **LOCAL SERVICE** read and execute on each script and its folder, and modify on any output folder. Use the folder's Security tab or `icacls`:

```powershell
icacls C:\ProgramData\CronPulser\scripts /inheritance:r /grant:r "*S-1-5-32-544:(OI)(CI)F" "*S-1-5-18:(OI)(CI)F" "*S-1-5-19:(OI)(CI)RX"
icacls C:\Backups /grant "*S-1-5-19:(OI)(CI)M"
```

`*S-1-5-19` is LOCAL SERVICE by SID, which works on every Windows language;
the account name is localized. Keep scripts in
`C:\ProgramData\CronPulser\scripts` with inheritance removed as above: a
folder directly under `C:\` lets every signed-in user change its files, and
only Administrators and SYSTEM should be able to change scripts.

- Presents **anonymous credentials on the network**. It cannot write to a UNC share that requires authentication, and mapped drives (`Z:`) do not exist for services. For network destinations, let a service that has its own identity do the write (SQL Server's service account in Example 1), or copy with an approved transfer tool. Never put a share password in YAML, arguments, or scripts.
- Cannot reach Docker Desktop. Docker Desktop's engine runs in the logged-in user's session and its named pipe is restricted to Administrators and the `docker-users` group. For Docker-hosted databases, run the backup on the Linux host or use the database's native backup as in Example 1. Do not add LocalService to `docker-users` or Administrators.
- Runs with no interactive desktop. Anything that opens a window or prompt fails.

## PowerShell script rules

1. First lines: `$ErrorActionPreference = 'Stop'` and `Set-StrictMode -Version Latest`.
2. Wrap the body in `try { … exit 0 } catch { [Console]::Error.WriteLine("failed: $($_.Exception.Message)"); exit 1 }`. A script that throws without `exit` still exits 1, but be explicit.
3. After calling a native executable, check `$LASTEXITCODE`. PowerShell does not throw for native failures.
4. Keep all paths, database names, and hostnames fixed inside the script. Nothing arrives as remote input.
5. Print one summary line with `Write-Output`. Send errors to stderr with `[Console]::Error.WriteLine`.
6. Test in the same context: `psexec -s` or a scheduled task running as LocalService, never only as your admin account.

## Example 1: SQL Server backup to a network share

`BACKUP DATABASE` runs inside the SQL Server engine, so the **SQL Server service
account** writes the file, not LocalService. Grant that account write access to
the share and NTFS folder. LocalService needs only: read/execute on the script,
local integrated login to SQL Server, `db_backupoperator` on the database, and
`CREATE DATABASE` if `RESTORE VERIFYONLY` is kept. Never grant `sysadmin`.

Prerequisite: `sqlcmd.exe` installed and on the system `PATH`.

`C:\ProgramData\CronPulser\scripts\backup-orders-sqlserver.ps1`:

```powershell
$ErrorActionPreference = 'Stop'
Set-StrictMode -Version Latest

$Database = 'Orders'
$SqlInstance = 'localhost'
$RemoteDirectory = '\\backup01\sqlserver\orders-prod'

try {
    $SqlCmd = (Get-Command sqlcmd.exe -ErrorAction Stop).Source
    $Timestamp = (Get-Date).ToUniversalTime().ToString('yyyyMMddTHHmmssZ')
    $FileName = "$($Database.ToLowerInvariant())-$Timestamp.bak"
    $RemoteFile = "$RemoteDirectory\$FileName"

    $EscapedDatabase = $Database.Replace(']', ']]')
    $EscapedRemoteFile = $RemoteFile.Replace("'", "''")
    $BackupQuery = "BACKUP DATABASE [$EscapedDatabase] TO DISK = N'$EscapedRemoteFile' WITH COPY_ONLY, CHECKSUM, INIT, STATS = 10;"

    $BackupOutput = & $SqlCmd -S $SqlInstance -E -b -Q $BackupQuery 2>&1
    if ($LASTEXITCODE -ne 0) { throw "SQL Server backup failed: $($BackupOutput -join ' ')" }

    $VerifyQuery = "RESTORE VERIFYONLY FROM DISK = N'$EscapedRemoteFile' WITH CHECKSUM;"
    $VerifyOutput = & $SqlCmd -S $SqlInstance -E -b -Q $VerifyQuery 2>&1
    if ($LASTEXITCODE -ne 0) { throw "SQL Server backup verification failed: $($VerifyOutput -join ' ')" }

    Write-Output "backup_complete database=$Database file=$FileName verified=true destination=network-share"
    exit 0
}
catch {
    [Console]::Error.WriteLine("backup_failed: $($_.Exception.Message)")
    exit 1
}
```

`COPY_ONLY` keeps this backup out of an existing differential chain. `CHECKSUM`
validates page checksums. `-E` uses integrated auth; `-b` makes `sqlcmd` return
a failing exit code on SQL errors. For a local disk destination, set
`$RemoteDirectory = 'C:\Backups\SQL'` and grant the SQL Server service account
write access there.

Expected record:

```text
status: success   exit_code: 0   duration: 4m 18s
stdout: backup_complete database=Orders file=orders-20260921T021500Z.bak verified=true destination=network-share
```

Access denied on the share appears as
`backup_failed: SQL Server backup failed: Operating system error 5 (Access is denied.)`
and means the SQL Server service account, not LocalService, lacks share permission.

## Example 2: Service health report

Checks a fixed list of Windows services, writes a CSV, and fails when any is stopped.

`C:\ProgramData\CronPulser\scripts\service-report.ps1`:

```powershell
$ErrorActionPreference = 'Stop'

$OutputDirectory = 'C:\ProgramData\CronPulser\reports'
$Services = @('CronPulserRunner', 'W32Time', 'MSSQLSERVER')
$Timestamp = (Get-Date).ToUniversalTime().ToString('yyyyMMddTHHmmssZ')
$OutputFile = Join-Path $OutputDirectory "service-status-$Timestamp.csv"

New-Item -ItemType Directory -Path $OutputDirectory -Force | Out-Null

$Rows = foreach ($Name in $Services) {
    $Service = Get-Service -Name $Name -ErrorAction Stop
    [pscustomobject]@{ Name = $Service.Name; Status = $Service.Status.ToString(); CapturedAtUtc = (Get-Date).ToUniversalTime().ToString('o') }
}

$Rows | Export-Csv -Path $OutputFile -NoTypeInformation -Encoding UTF8
$StoppedCount = @($Rows | Where-Object Status -ne 'Running').Count
Write-Output "report_complete file=$OutputFile services=$($Rows.Count) stopped=$StoppedCount"

if ($StoppedCount -gt 0) {
    [Console]::Error.WriteLine("$StoppedCount monitored service(s) are not running")
    exit 2
}
```

`runner.yaml`:

```yaml
jobs:
  windows-service-report:
    command: C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
    arguments: [-NoProfile, -NonInteractive, -File, C:\ProgramData\CronPulser\scripts\service-report.ps1]
    workingDirectory: C:\ProgramData\CronPulser\scripts
    timeout: 5m
```

Pair it with an **Execution Failure** alert; the non-zero exit is the signal.

## Example 3: Local retention cleanup

Deletes backups older than a fixed number of days from a local folder. Keep it
as a separate job from the backup itself.

`C:\ProgramData\CronPulser\scripts\prune-sql-backups.ps1`:

```powershell
$ErrorActionPreference = 'Stop'
Set-StrictMode -Version Latest

$BackupDirectory = 'C:\Backups\SQL'
$KeepDays = 14

try {
    if (-not (Test-Path -LiteralPath $BackupDirectory -PathType Container)) { throw "missing directory $BackupDirectory" }
    $Cutoff = (Get-Date).AddDays(-$KeepDays)
    $Old = @(Get-ChildItem -LiteralPath $BackupDirectory -File -Filter '*.bak' | Where-Object LastWriteTime -lt $Cutoff)
    foreach ($File in $Old) { Remove-Item -LiteralPath $File.FullName -Force }
    $Remaining = @(Get-ChildItem -LiteralPath $BackupDirectory -File -Filter '*.bak').Count
    Write-Output "retention_complete removed=$($Old.Count) remaining=$Remaining keep_days=$KeepDays"
    exit 0
}
catch {
    [Console]::Error.WriteLine("retention_failed: $($_.Exception.Message)")
    exit 1
}
```

Grant LOCAL SERVICE modify on `C:\Backups\SQL`. Timeout `10m`.

## Operations

The installer sets the service to start at boot (Automatic) and to restart 5
seconds after a crash. While it is stopped, scheduled runs fail immediately and
are not replayed; stopping it also stops a running job.

```powershell
Get-Service CronPulserRunner
Stop-Service CronPulserRunner
Start-Service CronPulserRunner
Restart-Service CronPulserRunner                      # after editing runner.yaml
Set-Service CronPulserRunner -StartupType Disabled    # Automatic to undo; required before Start-Service
Get-WinEvent -LogName Application | Where-Object ProviderName -eq CronPulserRunner | Select-Object -First 20
& 'C:\Program Files\CronPulser\cronpulser-runner.exe' validate
& 'C:\Program Files\CronPulser\cronpulser-runner.exe' version
& 'C:\Program Files\CronPulser\cronpulser-runner.exe' rotate -token cp_runner_xxxxxxxxx
```

Use CronPulser execution history for the script's result and the Application
event log for the service itself.

## Uninstall

In the app, delete or reassign the jobs that use the Runner first (a Runner
with scheduled jobs cannot be deleted), then delete the Runner. Then in
PowerShell as Administrator:

```powershell
Stop-Service CronPulserRunner
sc.exe delete CronPulserRunner
Remove-EventLog -Source CronPulserRunner
Remove-Item -Recurse -Force 'C:\Program Files\CronPulser', 'C:\ProgramData\CronPulser'
```

This also deletes `C:\ProgramData\CronPulser\scripts`; copy scripts elsewhere
first to keep them.
