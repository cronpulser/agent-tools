# Linux Runner: install, configure, and write jobs

The Runner is a Go binary that runs as a systemd service under the unprivileged
`cronpulser` user. It opens one outbound WebSocket to `api.cronpulser.com:443`.
No inbound port, public IP, or SSH access is needed. CronPulser schedules; the
Runner executes only the job keys in its local `runner.yaml` and returns exit
code, stdout, stderr, and duration.

## Install

1. In the app: Runners → **Add Runner** → copy the one-time token (`cp_runner_…`).
2. On the machine, as root:

```bash
curl -fsSL https://downloads.cronpulser.com/runner/install-runner.sh | \
sudo sh -s -- \
  --server https://api.cronpulser.com \
  --token cp_runner_xxxxxxxxx \
  --version 0.3.1
```

The installer verifies the release SHA-256, creates the `cronpulser` user,
installs `/usr/local/bin/cronpulser-runner`, writes `/etc/cronpulser/runner.yaml`
(mode 0600), enables the systemd unit, and starts it. The Runner shows as
Online on the Runners page within seconds.

Supported: x86_64 (`linux-amd64`) and aarch64 (`linux-arm64`).

## Paths

| Purpose | Path |
|---|---|
| Binary | `/usr/local/bin/cronpulser-runner` |
| Config | `/etc/cronpulser/runner.yaml` |
| Duplicate-execution cache | `/var/lib/cronpulser/executions.json` |
| Convention for scripts | `/opt/cronpulser/scripts/` |
| Service | `cronpulser-runner` (systemd) |

## runner.yaml

```yaml
server: https://api.cronpulser.com
runnerId: generated-during-install
runnerSecret: generated-during-install
maxConcurrentJobs: 1
cachePath: /var/lib/cronpulser/executions.json
jobs:
  backup-postgres:
    command: /usr/bin/sudo
    arguments:
      - /opt/cronpulser/scripts/backup-postgres-docker.sh
    workingDirectory: /opt/cronpulser/scripts
    timeout: 30m
```

Rules the Runner enforces:

- `command` is an absolute path to an executable. No shell, no `$PATH` lookup, no pipes or redirects at this level.
- `arguments` is a YAML list. Each element is passed as one argv entry.
- `timeout` is required and positive. Accepts `30`, `90s`, `15m`, `2h`.
- `workingDirectory` is optional but recommended.
- Never touch `runnerId` or `runnerSecret`. Rotate from the app instead.
- `maxConcurrentJobs` caps parallel executions on this machine. Keep `1` for backups.

After every edit:

```bash
sudo -u cronpulser /usr/local/bin/cronpulser-runner validate
sudo systemctl restart cronpulser-runner
sudo journalctl -u cronpulser-runner -n 50 --no-pager
```

The job key appears under the Runner on the Runners page after restart. Only
then can a CronPulser job select it.

## Script rules

Every script the Runner executes should:

1. Start with `#!/usr/bin/env bash` and `set -Eeuo pipefail`.
2. Use absolute paths for every binary. The service environment has a minimal `PATH`.
3. Be non-interactive. No prompts, no `sudo` password, no TTY.
4. Write to a temporary `.partial` file and move it into place only on success.
5. Exit non-zero on any failure. `set -e` covers most cases; check `$?` for commands run inside `if`.
6. Print one summary line: what it did, where, how big. Never print secrets, environment variables, connection strings, or data rows.
7. Be owned by root, mode 0755, and not writable by `cronpulser`, especially when it runs through sudo.

Test as the service user before scheduling:

```bash
sudo -u cronpulser /opt/cronpulser/scripts/your-script.sh; echo "exit=$?"
```

## Privilege: the one allowed sudo pattern

The `cronpulser` user should not be in the `docker` group; that is root-equivalent.
Do not grant `cronpulser ALL=(ALL) NOPASSWD: ALL`. When a script genuinely needs
root or Docker access, allow exactly that script:

```bash
sudo install -o root -g root -m 0755 backup-postgres-docker.sh /opt/cronpulser/scripts/
sudo tee /etc/sudoers.d/cronpulser-backup >/dev/null <<'SUDO'
cronpulser ALL=(root) NOPASSWD: /opt/cronpulser/scripts/backup-postgres-docker.sh
SUDO
sudo chmod 0440 /etc/sudoers.d/cronpulser-backup
sudo visudo -c
```

Then set `command: /usr/bin/sudo` with the script path as the single argument.
Because the script is root-owned and not writable by `cronpulser`, the sudo
grant cannot be turned into arbitrary root access.

## Example 1: PostgreSQL backup from a Docker container

Scenario: PostgreSQL runs in a container named `pgdb`, database `app`, superuser
`postgres`. Backups go to `/var/backups/postgres` on the host. Uses the sudo
pattern above because `docker` requires root.

`/opt/cronpulser/scripts/backup-postgres-docker.sh`:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

container=pgdb
db_user=postgres
database=app
backup_dir=/var/backups/postgres
stamp=$(/usr/bin/date -u +%Y%m%dT%H%M%SZ)
partial="$backup_dir/$database-$stamp.dump.partial"
final="$backup_dir/$database-$stamp.dump"

umask 077
test -d "$backup_dir"
/usr/bin/docker inspect -f '{{.State.Running}}' "$container" | /usr/bin/grep -qx true

# Custom format (-Fc) is compressed and restorable with pg_restore.
/usr/bin/docker exec "$container" pg_dump -U "$db_user" -Fc --no-owner "$database" > "$partial"

# Prove the archive is readable before calling it done.
/usr/bin/docker exec -i "$container" pg_restore --list < "$partial" > /dev/null

/usr/bin/mv -- "$partial" "$final"
bytes=$(/usr/bin/stat -c %s "$final")
printf 'backup_complete database=%s file=%s bytes=%s\n' "$database" "$final" "$bytes"
```

Setup once, as an administrator:

```bash
sudo install -d -o root -g root -m 0700 /var/backups/postgres
sudo install -o root -g root -m 0755 backup-postgres-docker.sh /opt/cronpulser/scripts/
# sudoers entry from the pattern above
sudo -u cronpulser /usr/bin/sudo /opt/cronpulser/scripts/backup-postgres-docker.sh; echo "exit=$?"
```

`runner.yaml` entry:

```yaml
jobs:
  backup-postgres:
    command: /usr/bin/sudo
    arguments: [/opt/cronpulser/scripts/backup-postgres-docker.sh]
    workingDirectory: /opt/cronpulser/scripts
    timeout: 30m
```

In the app: New Job → Remote Runner → this Runner → `backup-postgres` →
schedule `0 2 * * *` in the customer's timezone → Save → **Run Now**.

Expected execution record:

```text
status: success   exit_code: 0   duration: 8.2s
stdout: backup_complete database=app file=/var/backups/postgres/app-20260921T020000Z.dump bytes=5120334
```

Restore test (on a scratch database, not scheduled):

```bash
sudo docker exec -i pgdb pg_restore -U postgres -d app_restore_test --no-owner < /var/backups/postgres/app-20260921T020000Z.dump
```

Container with Compose: replace `pgdb` with the service's container name from
`docker compose ps`. If the password is not trusted locally inside the container,
set `PGPASSWORD` in the container's own environment, never in this script.

## Example 2: MySQL or MariaDB backup from a Docker container

Credentials stay inside the container in a `my.cnf`-style file, so nothing
secret appears in the script, the YAML, or execution output.

Once, create `/run/secrets/backup.cnf` inside the container (or mount it, mode 0600):

```ini
[client]
user=backup
password=REPLACE_ME
```

Grant that MySQL user `SELECT, LOCK TABLES, SHOW VIEW, EVENT, TRIGGER, PROCESS`
on the databases to back up.

`/opt/cronpulser/scripts/backup-mysql-docker.sh`:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

container=mysql
database=shop
backup_dir=/var/backups/mysql
stamp=$(/usr/bin/date -u +%Y%m%dT%H%M%SZ)
partial="$backup_dir/$database-$stamp.sql.gz.partial"
final="$backup_dir/$database-$stamp.sql.gz"

umask 077
test -d "$backup_dir"

/usr/bin/docker exec "$container" mysqldump \
  --defaults-extra-file=/run/secrets/backup.cnf \
  --single-transaction --quick --routines --triggers --events \
  "$database" | /usr/bin/gzip -1 > "$partial"

# gzip -t fails if the stream was truncated.
/usr/bin/gzip -t "$partial"
/usr/bin/mv -- "$partial" "$final"
bytes=$(/usr/bin/stat -c %s "$final")
printf 'backup_complete database=%s file=%s bytes=%s\n' "$database" "$final" "$bytes"
```

`--single-transaction` gives a consistent InnoDB snapshot without locking
writes. For MariaDB the binary is `mariadb-dump`; the flags are the same.
Because `pipefail` is set, a failing `mysqldump` fails the script even though
`gzip` succeeds. Use the same sudoers pattern and a `runner.yaml` entry with
`command: /usr/bin/sudo` and `timeout: 30m`.

## Example 3: File archive backup without root

Scenario: uploads under `/srv/example/uploads`, destination `/var/backups/example`.
Runs directly as `cronpulser`, no sudo.

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

source_dir=/srv/example/uploads
backup_dir=/var/backups/example
stamp=$(/usr/bin/date -u +%Y%m%dT%H%M%SZ)
partial="$backup_dir/uploads-$stamp.tar.gz.partial"
final="$backup_dir/uploads-$stamp.tar.gz"

umask 077
test -d "$source_dir"
test -d "$backup_dir"
/usr/bin/tar -C /srv/example -czf "$partial" uploads
/usr/bin/mv -- "$partial" "$final"
bytes=$(/usr/bin/stat -c %s "$final")
printf 'backup_complete file=%s bytes=%s\n' "$final" "$bytes"
```

Permissions: `cronpulser` needs read on the source and write on the destination.

```bash
sudo install -d -o cronpulser -g cronpulser -m 0700 /var/backups/example
sudo setfacl -R -m u:cronpulser:rX /srv/example/uploads   # or add to the owning group
```

`runner.yaml`:

```yaml
jobs:
  backup-uploads:
    command: /opt/cronpulser/scripts/backup-uploads.sh
    workingDirectory: /opt/cronpulser/scripts
    timeout: 15m
```

## Example 4: Retention as a separate job

Never delete inside the backup script; a successful backup must not hide a
failed cleanup. Schedule this weekly.

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
backup_dir=/var/backups/postgres
keep_days=14
test -d "$backup_dir"
removed=$(/usr/bin/find "$backup_dir" -maxdepth 1 -type f -name '*.dump' -mtime +"$keep_days" -print -delete | /usr/bin/wc -l)
remaining=$(/usr/bin/find "$backup_dir" -maxdepth 1 -type f -name '*.dump' | /usr/bin/wc -l)
printf 'retention_complete removed=%s remaining=%s keep_days=%s\n' "$removed" "$remaining" "$keep_days"
```

## Example 5: Prevent overlap with flock

For jobs that may run longer than their interval:

```yaml
jobs:
  sync-orders:
    command: /usr/bin/flock
    arguments: [-n, /var/lib/cronpulser/sync-orders.lock, /opt/cronpulser/scripts/sync-orders.sh]
    timeout: 20m
```

`flock -n` exits 1 immediately when the previous run still holds the lock, so
the skipped run shows as failed in history. If a skip should not fail, wrap it:
`if ! flock -n ...; then echo "skipped_still_running"; exit 0; fi` inside the script.

## Operations

```bash
sudo systemctl status cronpulser-runner --no-pager
sudo journalctl -u cronpulser-runner -f
sudo -u cronpulser /usr/local/bin/cronpulser-runner validate
/usr/local/bin/cronpulser-runner version
sudo cronpulser-runner rotate --token cp_runner_xxxxxxxxx   # token from Runners page
```

Rotation revokes the old secret immediately and keeps local jobs and cache.
