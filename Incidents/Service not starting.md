# Incident 04 — Service Not Starting

**Category:** Linux Production Outage
**Severity:** P1 Critical
**Status:** Resolved

## Scenario

- A critical application/system service fails to start after a deploy, config change, patch, or reboot.
- `systemctl start` either fails immediately or the service enters a failed/crash-loop state.
- The server itself is healthy and reachable; only this specific service refuses to come up.
- Dependent services or downstream consumers begin failing because this service is down.

## 1. What is the symptom?

**Observed symptoms:**

- Service does not respond on its expected port/socket.
- `systemctl status` shows `failed`, `activating (auto-restart)`, or a crash loop.
- Application logs show immediate exit after startup, sometimes with no clear error visible to end users.
- Dependent services/health checks begin failing shortly after.

**Verification:**

```bash
systemctl status myservice
systemctl is-active myservice
systemctl is-enabled myservice
journalctl -u myservice -xe
```

**Sample output:**

```
Active: failed (Result: exit-code)
Main PID: 5210 (code=exited, status=1/FAILURE)
```

or

```
Active: activating (auto-restart) (Result: exit-code)
```

**Conclusion:** The service process is starting and immediately terminating (or never starting), and systemd is unable to bring it to a stable running state.

## 2. What logs will you check?

**Service unit logs:**

```bash
journalctl -u myservice -xe
journalctl -u myservice --since "15 minutes ago"
# Look for: "Failed with result", "core dumped", stack traces, config errors
```

**Application-level logs:**

```bash
tail -200 /var/log/myservice/app.log
tail -200 /var/log/myservice/error.log
```

**System journal for boot/startup context:**

```bash
journalctl -b -p err
journalctl --since "boot"
```

**Dependency logs (DB, cache, message queue, etc.):**

```bash
systemctl status postgresql
systemctl status redis
journalctl -u postgresql -xe
# Confirm dependencies the service relies on are actually up
```

**Kernel/OOM logs (in case the process was killed):**

```bash
dmesg | tail -50
grep -i "killed process" /var/log/syslog
# Look for Out-Of-Memory killer terminating the process
```

## 3. What commands will you run?

**Step 1: Check current service state in detail:**

```bash
systemctl status myservice -l --no-pager
```

**Step 2: Try starting the service manually and watch it fail in real time:**

```bash
systemctl start myservice
journalctl -u myservice -f
```

**Step 3: Run the binary/entrypoint directly to see the raw error (bypassing systemd):**

```bash
sudo -u myserviceuser /usr/bin/myservice --config /etc/myservice/config.yml
```

**Step 4: Validate the service's config file syntax:**

```bash
myservice --check-config
# or, for common formats:
python -m json.tool /etc/myservice/config.json
yamllint /etc/myservice/config.yml
```

**Step 5: Check file and directory permissions the service needs:**

```bash
ls -lh /etc/myservice/
ls -lh /var/lib/myservice/
ls -lh /var/log/myservice/
```

**Step 6: Check for a missing or corrupted dependency (library, runtime, DB connection):**

```bash
ldd /usr/bin/myservice
myservice --version
nc -zv <db-host> <db-port>
```

**Step 7: Check available system resources:**

```bash
free -h
df -h
ulimit -a
```

**Step 8: Check for a port conflict blocking startup:**

```bash
ss -tlnp | grep <port>
```

**Step 9: Check the systemd unit file itself for recent changes:**

```bash
systemctl cat myservice
cat /etc/systemd/system/myservice.service
```

**Step 10: Reload systemd in case the unit file was edited but not reloaded:**

```bash
systemctl daemon-reload
systemctl restart myservice
```

## 4. What is the likely root cause?

**Investigation findings:**

```bash
# Command
journalctl -u myservice -xe

# Output
myservice.service: Main process exited, code=exited, status=1/FAILURE
Error: could not connect to database at db-host:5432: connection refused
```

```bash
# Command
systemctl status postgresql

# Output
Active: inactive (dead)
```

```bash
# Command
nc -zv db-host 5432

# Output
nc: connect to db-host port 5432 (tcp) failed: Connection refused
```

**Analysis:**

- The service depends on a database connection at startup and exits immediately if it cannot establish one.
- The database service itself was down, so every connection attempt from `myservice` failed.
- systemd's restart policy kept retrying the service, but since the dependency never came up, every attempt failed identically, producing a crash loop.

**Other possible root causes (depending on findings):**

- Config file has a syntax error or invalid/missing required field introduced by a recent change.
- Service account lacks permission to read its config, write its log directory, or access a required socket/file.
- Required port is already in use by another process.
- Missing or incompatible shared library/runtime after an OS package upgrade.
- Out-of-memory killer terminated the process due to insufficient available memory.
- systemd unit file was edited but `daemon-reload` was never run, so systemd is using stale unit definitions.
- Disk full, preventing the service from writing logs, PID files, or temp data needed at startup.
- SELinux/AppArmor policy blocking the service from an action it needs to perform.

**Final root cause (in this scenario):**

> The service was configured to hard-depend on a database connection at startup with no retry/backoff logic. The database service was down at the time of the deploy/restart, causing every start attempt by `myservice` to fail immediately and systemd to enter a continuous crash-loop/backoff cycle.

## 5. How would you fix it?

**Step 1: Bring the missing dependency back up first:**

```bash
systemctl start postgresql
systemctl status postgresql
```

**Step 2: Confirm the dependency is actually reachable from the service host:**

```bash
nc -zv db-host 5432
```

**Step 3: Reset the failed state and start the service cleanly:**

```bash
systemctl reset-failed myservice
systemctl start myservice
```

**Step 4: Verify the service is now active and stable:**

```bash
systemctl status myservice
journalctl -u myservice --since "2 minutes ago"
```

**Step 5: Confirm the service is listening/responding as expected:**

```bash
ss -tlnp | grep <port>
curl -I http://localhost:<port>/health
```

**Step 6:** Confirm dependent services and health checks recover and traffic resumes normally.

## 6. How would you prevent recurrence?

**a) Add dependency ordering in the systemd unit:**

```
[Unit]
After=postgresql.service
Wants=postgresql.service
```

This does not guarantee the dependency is fully ready, but ensures correct start order rather than a race condition.

**b) Add startup retry/backoff logic in the application itself:**

Instead of exiting immediately on a failed dependency connection, the service should retry with exponential backoff for a bounded period before giving up, so transient dependency delays don't cause a hard failure.

**c) Add a pre-start health check for critical dependencies:**

```bash
# ExecStartPre in the systemd unit
ExecStartPre=/usr/local/bin/wait-for-it.sh db-host:5432 --timeout=30
```

**d) Validate config before restart in deploy pipelines:**

```bash
myservice --check-config || exit 1
```

**e) Always run `daemon-reload` after editing unit files:**

```bash
systemctl daemon-reload
systemctl restart myservice
```

**f) Set a sane systemd restart policy so crash loops don't hammer resources indefinitely:**

```
[Service]
Restart=on-failure
RestartSec=5
StartLimitIntervalSec=60
StartLimitBurst=5
```

**g) Monitoring alerts:**

| Condition | Severity |
|---|---|
| Service unit not active for over 1 min | Warning |
| Service unit not active for over 3 min | Critical |

## 7. Which monitoring alert should have detected it?

**systemd unit state alert (via node_exporter/textfile collector):**

```yaml
- alert: ServiceUnitFailed
  expr: node_systemd_unit_state{name="myservice.service", state="failed"} == 1
  for: 0m
  labels:
    severity: critical
  annotations:
    summary: "myservice.service is in a failed state on {{ $labels.instance }}"
```

**Dependency health alert:**

```yaml
- alert: DatabaseDown
  expr: pg_up == 0
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "PostgreSQL is down, dependent services may fail to start"
```

**Grafana alert:**

| Level | Condition |
|---|---|
| Warning | Service unit inactive/failed for > 1 min |
| Critical | Service unit inactive/failed for > 3 min |

**CloudWatch / Cloud Provider:**

- **AWS** — CloudWatch agent alarm on systemd unit state, or ALB health check alarm.
- **GCP** — Cloud Monitoring alert on VM/managed instance health probe failure.
- **Azure** — Azure Monitor alert on VM/service health probe failure.

**Nagios / Zabbix / Datadog:**

- Direct systemd unit state check (`check_systemd`) with alert on failed/inactive state.
- Correlated dependency check (e.g., database reachability) to distinguish root cause from symptom.

**Expected Detection:** Within 1–3 minutes of the service entering a failed/crash-loop state, well before dependent systems or users are meaningfully impacted.

## 8. RCA (Root Cause Analysis)

| Field | Details |
|---|---|
| Incident Title | Critical Service Failing to Start Due to Unavailable Dependency |
| Incident Date | 08-Jun-2026 |
| Severity | P1 Critical |
| Duration | 28 minutes |
| Owner | DevOps Team |

**Summary:**

A critical service entered a continuous crash-loop after a restart because its required database dependency was down. The service had no retry/backoff logic and exited immediately on every failed connection attempt, and systemd's default retry policy was insufficient to recover once the dependency became available again without manual intervention.

**Impact:**

- Service was completely unavailable for the duration of the incident.
- Downstream consumers and health checks failed, cascading into partial degradation of dependent workflows.
- On-call engineer required to manually diagnose and restart the service after fixing the dependency.

**Timeline:**

| Time | Event |
|---|---|
| T+00 min | Service restart triggered as part of routine maintenance |
| T+01 min | Service fails to start; systemd begins auto-restart loop |
| T+03 min | Health check failures trigger paging |
| T+06 min | On-call engineer confirms service is crash-looping via `systemctl status` |
| T+10 min | `journalctl -u myservice -xe` reveals database connection refused error |
| T+14 min | `systemctl status postgresql` confirms the database service is down |
| T+18 min | Database service restarted and confirmed reachable |
| T+22 min | `systemctl reset-failed` and `systemctl start myservice` run |
| T+25 min | Service confirmed active and responding to health checks |
| T+28 min | Incident closed, preventive actions logged |

**Root Cause:**

> The service had a hard startup dependency on the database with no retry/backoff logic, so when the database was unavailable at restart time, every start attempt failed immediately. systemd's crash-loop backoff eventually exhausted its restart attempts, requiring manual intervention to reset the failed state once the dependency was restored.

**Resolution:**

- Identified the crash-loop via `systemctl status` and `journalctl`.
- Traced the failure to a database connection error in the service logs.
- Restarted the database dependency and confirmed reachability.
- Reset the service's failed state and started it manually.
- Verified the service was active and health checks passed.

**Preventive Actions:**

- Add retry/backoff logic in the application for dependency connections at startup.
- Add `After=`/`Wants=` ordering and an `ExecStartPre` dependency wait check in the systemd unit.
- Tune `Restart=`, `RestartSec=`, and `StartLimitBurst=` so transient failures self-heal without needing a manual `reset-failed`.
- Add separate monitoring/alerting for the database dependency so its outage is caught independently and earlier.
- Document a standard triage runbook for "service won't start" incidents, starting with dependency checks.

## 9. Business Impact Estimate

**Technical impact:**

- Full service outage for the duration of the incident.
- Cascading health check failures across dependent systems.
- Manual intervention required to fully restore service, increasing MTTR beyond what auto-restart could handle.

**Customer impact:**

- Users were unable to use features backed by the affected service during the outage window.
- Any in-flight requests during the crash-loop period failed or timed out.

**Revenue impact example:**

| Parameter | Value |
|---|---|
| Users per hour | 1000 |
| Average revenue per user | Rs. 100 |
| Outage duration | 28 minutes |
| Affected users | 1000 × 0.47 = 470 users |
| Potential revenue loss | 470 × Rs. 100 = Rs. 47,000 |

A single missing dependency check at startup turned a transient, easily fixable database blip into a full service outage. The real risk is that without retry/backoff logic, every future dependency hiccup — however brief — will trigger the same manual-intervention cycle instead of self-healing.
