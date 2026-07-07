# Incident 03: Cron Job Not Running

**Category:** Linux Production Outage
**Severity:** P2 High
**Status:** Resolved

## Scenario

- A scheduled cron job (e.g., backup script, log rotation, data sync, report generation) did not execute at its scheduled time.
- No output, no logs, no error notification and the job simply never ran.
- The script works fine when run manually.
- Server itself is up and running only the cron-scheduled execution is broken.

---

## 1. Symptom:

**Observed symptoms:**

- Scheduled task did not run at expected time.
- No new output file / log entry / DB update generated.
- Downstream jobs or reports depending on this cron job are missing or stale.
- Manual execution of the same script works without issues.
- No alert or failure notification received (silent failure).

**Verification:**

```bash
crontab -l
crontab -l -u <username>
systemctl status cron
systemctl status crond
grep CRON /var/log/syslog | tail -50
```

**Sample output:**

```
no crontab for <username>
```

or

```
cron.service - could not be found / inactive (dead)
```

or (in syslog):

```
CRON[1234]: (username) CMD (/opt/scripts/backup.sh)
CRON[1234]: (username) MAIL (mailed 2 bytes of output but got status 0x7f00)
```

**Conclusion:** Cron either did not trigger the job, or triggered it but the job failed silently in the cron environment.

---

## 2. Logs to check:

**Cron service logs:**

```bash
journalctl -u cron -xe
journalctl -u crond -xe
# Check for: "cron.service: Failed", restarts, or missing entries at scheduled time
```

**Syslog / cron log:**

```bash
grep CRON /var/log/syslog | tail -100        # Debian/Ubuntu
tail -100 /var/log/cron                      # RHEL/CentOS
# Look for: job entry at scheduled time, exit status, "MAIL" lines indicating errors
```

**Script's own log/output (if it logs to a file):**

```bash
tail -100 /opt/scripts/backup.log
ls -lh /opt/scripts/*.log
# Check last modified timestamp - did it update at the scheduled time?
```

**Mail log (cron mails job output by default):**

```bash
mail -u <username>
tail -100 /var/log/mail.log
# cron emails stdout/stderr if MAILTO is set - check for silent errors here
```

**System journal around scheduled time:**

```bash
journalctl --since "<scheduled-time>" --until "<scheduled-time+5min>"
```

---

## 3. Commands to run:

**Step 1: Confirm the cron entry actually exists:**

```bash
crontab -l
crontab -l -u <username>
cat /etc/crontab
ls /etc/cron.d/
ls /etc/cron.daily/ /etc/cron.hourly/ /etc/cron.weekly/
```

**Step 2: Check cron service status:**

```bash
systemctl status cron
systemctl status crond
```

**Step 3: Restart cron if it's stopped/dead:**

```bash
systemctl restart cron
systemctl restart crond
```

**Step 4: Validate the cron schedule syntax:**

```bash
# Confirm expression is valid, e.g. using crontab.guru logic manually
echo "0 2 * * * /opt/scripts/backup.sh" | crontab -
```

**Step 5: Check if the script has execute permissions:**

```bash
ls -lh /opt/scripts/backup.sh
# Expected: -rwxr-xr-x
chmod +x /opt/scripts/backup.sh
```

**Step 6: Check for environment differences (cron runs with a minimal environment):**

```bash
crontab -l
# Look for missing PATH, missing env vars used by the script
env -i /bin/sh -c '/opt/scripts/backup.sh'   # simulate cron-like minimal env
```

**Step 7: Check disk space and permissions on output directories:**

```bash
df -h
ls -ld /opt/scripts/output/
```

**Step 8: Check if the user account running cron is locked/disabled:**

```bash
passwd -S <username>
chage -l <username>
```

**Step 9: Check system time / timezone (common silent cause):**

```bash
timedatectl
date
```

**Step 10: Check if cron itself is running at all (process level):**

```bash
ps -ef | grep cron
```

---

## 4. Likely root cause:

**Investigation findings:**

```bash
# Command
crontab -l -u <username>

# Output
no crontab for <username>
```

```bash
# Command
systemctl status cron

# Output
Active: active (running)
```

```bash
# Command
grep CRON /var/log/syslog | tail -20

# Output
(no entries for this job at scheduled time)
```

**Analysis:**

- The cron daemon itself was running fine (ruled out service-level failure).
- No crontab entry existed for the user it had been silently wiped.
- This typically happens when: a deployment script overwrote the crontab, a package update reset it, or a manual `crontab -r` was accidentally run instead of `crontab -e`.
- Since there was no entry, cron never attempted to trigger the job hence zero logs and no error/failure notification.

**Other possible root causes (depending on findings):**

- Cron syntax error in the schedule line (job silently ignored/skipped).
- Script lacks execute permission (`+x`)  cron logs the attempt but fails to exec.
- `PATH` or environment variables missing  script depends on env not present under cron's minimal shell.
- Script depends on another service (DB, network mount) not yet up at the scheduled time.
- System clock/timezone drift job scheduled at "2 AM" but server timezone changed after an OS update.
- Disk full job attempted but failed to write output, then errored out silently since `MAILTO` wasn't configured.
- User account expired/locked cron refuses to run jobs for locked accounts.
- Cron daemon itself was down/crashed.

**Final root cause (in this scenario):**

> The user's crontab was accidentally cleared during a recent deployment (a script ran `crontab -r` instead of appending to the existing file), removing the scheduled job entirely. Since there was no crontab entry, cron never attempted execution and produced no logs, making the failure completely silent.

---

## 5. The fix:

**Step 1: Recreate the crontab entry:**

```bash
crontab -e -u <username>
# Add back:
# 0 2 * * * /opt/scripts/backup.sh >> /opt/scripts/backup.log 2>&1
```

**Step 2: Validate the entry was saved:**

```bash
crontab -l -u <username>
```

**Step 3: Ensure script has execute permission:**

```bash
chmod +x /opt/scripts/backup.sh
```

**Step 4: Test-run the job manually with cron-like minimal environment:**

```bash
env -i /bin/bash -c '/opt/scripts/backup.sh'
```

**Step 5: Restart cron service (if needed):**

```bash
systemctl restart cron
```

**Step 6: Manually trigger a dry run to confirm output is generated correctly:**

```bash
/opt/scripts/backup.sh
ls -lh /opt/scripts/output/
```

**Step 7: Wait for or simulate the next scheduled window and confirm execution:**

```bash
grep CRON /var/log/syslog | tail -10
# Confirm new entry appears at the next scheduled trigger
```

**Step 8:** Confirm downstream systems/reports are back to receiving fresh data.

---

## 6. Recurrence prevention:

**a) Never use `crontab -r` in scripts or deployment pipelines:**

Always use `crontab -l > backup && edit && crontab new_file`, never a blind `-r`.

**b) Version-control crontabs:**

Store crontab content in git (as a file, applied via `crontab file.txt` during deployment) instead of editing in place. Any change goes through PR/review.

**c) Always set `MAILTO` in crontab so failures are not silent:**

```
MAILTO=devops-alerts@example.com
0 2 * * * /opt/scripts/backup.sh
```

**d) Add explicit logging and exit-code checks inside the script:**

```bash
/opt/scripts/backup.sh >> /opt/scripts/backup.log 2>&1 || echo "FAILED $(date)" >> /opt/scripts/backup.log
```

**e) Use a job scheduler with visibility instead of raw cron where possible:**

Tools like `healthchecks.io`, `cronitor`, or `systemd` timers with `OnFailure=` units provide built-in alerting for missed/failed runs.

**f) Pre-deployment checklist:**

Before any deployment or package update, snapshot and diff existing crontabs to detect accidental wipes:

```bash
crontab -l -u <username> > /backup/crontab_<username>_$(date +%F).bak
```

**g) Monitoring alerts:**

| Condition | Severity |
|---|---|
| Expected job output/log not updated within schedule + 15 min | Warning |
| Expected job output/log not updated within schedule + 1 hr | Critical |

---

## 7. Which monitoring alert should have detected it?

> **Note:** Cron failures are inherently silent unless explicitly monitored. "No news" must be treated as bad news for scheduled jobs.

**Healthchecks.io / Cronitor style dead-man's-switch:**

Script pings a monitoring endpoint at the end of a successful run. Alert if no ping received within the expected window.

```bash
curl -fsS -m 10 --retry 5 https://hc-ping.com/<uuid> > /dev/null
# Added at the end of backup.sh
```

**Prometheus + Blackbox/Textfile Exporter:**

Job writes a success timestamp to a metrics file; alert if timestamp is stale.

```yaml
- alert: CronJobMissed
  expr: time() - cron_last_success_timestamp{job="backup_job"} > 5400
  for: 0m
  labels:
    severity: critical
  annotations:
    summary: "Cron job backup_job has not completed successfully in over 90 minutes"
```

**Grafana alert:**

| Level | Condition |
|---|---|
| Warning | No successful run logged within schedule + 15 min |
| Critical | No successful run logged within schedule + 1 hr |

**CloudWatch / Cloud Provider:**

- **AWS** — CloudWatch Events/EventBridge scheduled rule with Lambda failure alarm (if using cloud-native scheduling instead of cron).
- **GCP** — Cloud Scheduler with Cloud Monitoring alert on missed invocation.
- **Azure** — Azure Monitor alert on Automation Account job failure/non-execution.

**Nagios / Zabbix / Datadog:**

- Check for freshness of an expected output file / log timestamp.
- Alert if file mtime exceeds expected schedule interval by a defined threshold.

**Expected Detection:** Within 15–90 minutes of the missed run, instead of discovering it hours or days later when a downstream report or backup is found missing.

---

## 8. RCA (Root Cause Analysis)

| Field | Details |
|---|---|
| Incident Title | Scheduled Cron Job Silently Not Running |
| Incident Date | 08-Jun-2026 |
| Severity | P2 High |
| Duration | Undetected for ~18 hours; 25 minutes to fix once identified |
| Owner | DevOps Team |

**Summary:**

A scheduled backup cron job stopped running without any error or alert. Investigation revealed the user's crontab had been accidentally cleared during a deployment, removing the job entry entirely. Since cron had nothing to trigger, no logs or failure notifications were ever generated.

**Impact:**

- Nightly backup did not run for one full cycle.
- No alert was raised, so the gap went unnoticed until a downstream report was found stale the next morning.
- Increased risk exposure. No recovery point available for that period.
- Team spent time investigating before identifying the missing crontab entry as the root cause.

**Timeline:**

| Time | Event |
|---|---|
| T-18h | Scheduled cron trigger time passes with no job execution |
| T-17h55m | No log entry, no output file update (undetected) |
| T+00 min | Team notices stale backup file during morning check |
| T+05 min | `crontab -l -u <username>` confirms crontab is empty |
| T+10 min | Deployment logs reviewed, `crontab -r` found in a deploy script |
| T+15 min | Crontab entry restored and validated with `crontab -l` |
| T+18 min | Manual dry-run of script confirms it executes correctly |
| T+22 min | `MAILTO` added to crontab for future failure visibility |
| T+25 min | Incident review completed, monitoring dead-man's-switch added |

**Root Cause:**

> A deployment script executed `crontab -r` for the service user instead of safely updating the crontab, wiping all scheduled jobs including the nightly backup. Because no crontab entry existed, cron never attempted to run the job, and no failure signal (log, mail, or alert) was ever produced, making the outage completely silent until manually discovered.

**Resolution:**

- Confirmed cron service itself was healthy.
- Identified missing crontab entry via `crontab -l`.
- Traced root cause to a deployment script using `crontab -r`.
- Restored crontab entry and validated with a manual dry run.
- Added `MAILTO` and a dead-man's-switch ping for future visibility.

**Preventive Actions:**

- Ban `crontab -r` from deployment scripts; use safe file-based updates instead.
- Version-control crontab files in git, applied via `crontab file` during deploys.
- Add `MAILTO` to all crontabs to surface silent failures.
- Add dead-man's-switch monitoring (e.g., healthchecks.io) for all critical scheduled jobs.
- Maintain daily crontab backups/snapshots for quick diffing and recovery.

---

## 9. Business Impact Estimate

**Technical impact:**

- One full backup cycle missed, reducing recovery point objective (RPO) coverage.
- Downstream reporting/data sync jobs relying on prior job's output ran on stale data.
- No proactive alerting — failure only surfaced via manual observation.
- Increased detection-to-resolution time compared to an actively monitored failure.

**Customer impact:**

- Delayed or stale reports delivered to internal/external stakeholders.
- Reduced disaster-recovery confidence for the affected time window.
- If backup had been needed during the gap, data loss risk would have increased significantly.

**Revenue impact example:**

| Parameter | Value |
|---|---|
| Reports/dependent processes affected | 1 nightly backup cycle |
| Estimated recovery point gap | ~18 hours |
| Risk-adjusted potential impact | Data loss window if disaster occurred during gap (non-linear, high-severity risk) |

Even though no immediate revenue was lost, the real risk is a compounding one: had a disaster or data corruption event occurred during the missed backup window, the organization would have had no valid recovery point, turning a silent scheduling gap into a potentially unrecoverable data loss event.
