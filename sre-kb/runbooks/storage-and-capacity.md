# Runbook · Storage & Capacity

> Fires when a disk volume, database, or object store approaches/hits its capacity limit.

Template: [../runbook-template.md](../runbook-template.md) · Index: [../README.md](../README.md)

## Alert / Symptoms
- Disk usage > 80/90% and rising; `No space left on device` (`ENOSPC`) errors.
- DB storage nearing the provisioned/autogrow ceiling; WAL/redo directory filling.
- Object-store quota or bucket size limit approaching.
- Inode exhaustion (space free but "no space" errors) from many small files.
- Symptoms cascade: writes fail, DB goes read-only, logs stop, app crashes.

## Impact
- **Users:** write failures, uploads fail, and — if the DB fills — a full outage (many engines halt writes when storage is exhausted).
- **Severity:** **SEV-1** if a database or primary volume is about to hit 100% (imminent hard outage / risk of corruption); **SEV-2** for an app/log volume with headroom.

## Quick checks
1. **What's full and how fast:** `df -h` (space) and `df -i` (inodes); plot fill rate to estimate time-to-full.
2. **Biggest consumers:** `du -xh --max-depth=1 /path | sort -h` — logs, temp files, uploads, DB data, backups?
3. **Sudden vs gradual:** a cliff = a runaway process/log/dump; a ramp = organic growth or a retention gap.
4. **DB-specific:** WAL/redo/binlog accumulation (replication lag or disabled archiving prevents cleanup), bloat, unvacuumed tables, or a huge temp/sort spill.
5. **Retention:** are log/backup/soft-deleted-data retention jobs actually running?

## Diagnosis
1. **Runaway log growth:** a crash loop or debug logging can fill a volume in minutes. Check log dir size and rotation config.
2. **WAL/redo pileup:** replication lag or a stuck replica prevents WAL cleanup, and it grows until the volume fills — check replica health and archiving status.
3. **Unpurged data:** retention/purge job failed or was never configured; soft-deleted rows, old exports, orphaned temp files, or old backups accumulate.
4. **Inode exhaustion:** many tiny files (sessions, cache, temp) exhaust inodes while bytes look fine.
5. **Organic growth:** legitimate data growth simply outran provisioning — a capacity-planning miss, not a leak.

## Mitigation
_Safe / reversible first — but move fast if a DB volume is near 100%._
1. **Reclaim obvious safe space first:** rotate + compress logs, clear temp/scratch dirs, remove already-shipped log segments. Buys time immediately.
2. **Throttle/stop the runaway producer:** silence a crash-loop's debug logging, pause the job dumping data.
3. **Fix WAL/redo pileup:** repair or drop the lagging replica / re-enable archiving so the DB can recycle segments.
4. **Grow the volume / raise the quota:** most cloud disks and object stores expand online without downtime — the cleanest fix when growth is legitimate. Expand *before* 100% if possible.
5. **Run retention purge:** execute the retention/cleanup job to delete expired data (verify the policy first — irreversible).
6. **Archive cold data** to cheaper/object storage and free the primary volume.
7. **Delete backups/old data only with sign-off** — last resort, irreversible; confirm you're not destroying the only recovery copy.

## Escalation
- Page **infra/platform on-call** to expand volumes/quotas.
- Page the **DBA** for any DB-storage or WAL issue — DB-full can corrupt or hard-stop the database.
- Get **data-owner sign-off** before purging or deleting anything with retention/compliance implications.

## Prevention / follow-up
- Alert at 70/80/90% with time-to-full projection, not just at 90%.
- Enforce log rotation + size caps; ship logs off-box.
- Automate retention/purge jobs and monitor that they run; verify backup lifecycle policies.
- Capacity-plan with growth trends; enable storage autoscaling where available (watch the cost side).
- **Ties to:** [../../standards-kb/06-data-management.md](../../standards-kb/06-data-management.md) and [../../standards-kb/16-finops-cost.md](../../standards-kb/16-finops-cost.md).
