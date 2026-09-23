# Windows Server Runner: install, configure, and write jobs

Supported: Windows Server 2019, 2022, 2025 on AMD64. The Runner installs as the
automatic `CronPulserRunner` service running as **LocalService**. It opens one
outbound connection to `api.cronpulser.com:443`; no inbound firewall rule.

## Install

1. In the app: Runners → **Add Runner** → copy the one-time token.
2. In an elevated PowerShell:

```powershell
Invoke-WebRequest https://downloads.cronpulser.com/runner/install-runner.ps1 -OutFile .\install-runner.ps1
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\install-runner.ps1 `
  -Server https://api.cronpulser.com `
  -Token cp_runner_xxxxxxxxx `
  -Version 0.3.1
```

## Paths

| Purpose | Path |
|---|---|
| Binary | `C:\Program Files\CronPulser\cronpulser-runner.exe` |
| Config | `C:\ProgramData\CronPulser\runner.yaml` |
| Cache | `C:\ProgramData\CronPulser\state\executions.json` |
| Convention for scripts | `C:\Scripts\` |
| Service | `CronPulserRunner` |
| Events | Application log, provider `CronPulserRunner` |

## runner.yaml on Windows

PowerShell is the executable; the script is an argument. Never put a whole
command line in `command`.

```yaml
jobs:
  backup-orders-sqlserver:
    command: C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
    arguments: [-NoProfile, -NonInteractive, -File, C:\Scripts\backup-orders-sqlserver.ps1]
    workingDirectory: C:\Scripts
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
icacls C:\Scripts /grant "LOCAL SERVICE:(RX)"
icacls C:\Backups /grant "LOCAL SERVICE:(OI)(CI)M"
```

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

`C:\Scripts\backup-orders-sqlserver.ps1`:

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

`C:\Scripts\service-report.ps1`:

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
    arguments: [-NoProfile, -NonInteractive, -File, C:\Scripts\service-report.ps1]
    workingDirectory: C:\Scripts
    timeout: 5m
```

Pair it with an **Execution Failure** alert; the non-zero exit is the signal.

## Example 3: Local retention cleanup

Deletes backups older than a fixed number of days from a local folder. Keep it
as a separate job from the backup itself.

`C:\Scripts\prune-sql-backups.ps1`:

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

```powershell
Get-Service CronPulserRunner
Get-WinEvent -LogName Application | Where-Object ProviderName -eq CronPulserRunner | Select-Object -First 20
& 'C:\Program Files\CronPulser\cronpulser-runner.exe' validate
& 'C:\Program Files\CronPulser\cronpulser-runner.exe' version
& 'C:\Program Files\CronPulser\cronpulser-runner.exe' rotate -token cp_runner_xxxxxxxxx
```

Use CronPulser execution history for the script's result and the Application
event log for the service itself.
