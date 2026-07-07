# Incident 04:  Application Port Already in Use

**Category:** Linux Production Outage
**Severity:** P1 Critical
**Status:** Resolved

## Scenario

- Application fails to start after a deploy, restart, or reboot.
- Startup logs show a "bind" or "address already in use" error for the application's port.
- The server itself is healthy and reachable.
- Only the application process fails to come up. Everything else on the host is fine.

## 1. Symptom:

**Observed symptoms:**

- Application service fails to start or crashes immediately after starting.
- Health checks fail; load balancer marks the instance as unhealthy.
- Users get connection refused or 502/503 errors from the app.
- Restarting the service repeatedly fails with the same error.

**Verification:**

```bash
systemctl status myapp
journalctl -u myapp -xe
curl -I http://localhost:8080
```

**Sample output:**

```
Error: listen tcp :8080: bind: address already in use
```

or

```
OSError: [Errno 98] Address already in use
```

**Conclusion:** The application cannot bind to its configured port because something else already holds it.

## 2. Logs to check:

**Application logs:**

```bash
journalctl -u myapp -xe
tail -200 /var/log/myapp/app.log
# Look for: "address already in use", "bind failed", "EADDRINUSE"
```

**System journal around the failed start:**

```bash
journalctl -u myapp --since "10 minutes ago"
journalctl -b -p err
```

**Syslog:**

```bash
tail -200 /var/log/syslog        # Debian/Ubuntu
tail -200 /var/log/messages      # RHEL/CentOS
```

**Container logs (if containerized):**

```bash
docker logs <container_id> --tail 200
docker ps -a
# Check for repeated restart/crash loop caused by port conflict
```

## 3. Commands to run:

**Step 1: Identify what is already using the port:**

```bash
ss -tlnp | grep 8080
netstat -tlnp | grep 8080
lsof -i :8080
```

**Step 2: Identify the exact process holding the port:**

```bash
fuser 8080/tcp
ps -p <pid> -o pid,ppid,cmd
```

**Step 3: Check if a duplicate instance of the same app is already running:**

```bash
ps -ef | grep myapp
systemctl status myapp
```

**Step 4: Check for zombie/orphaned processes from a previous crash:**

```bash
ps -ef | grep defunct
ps -ef --sort=start_time | tail -20
```

**Step 5: Check if a container or another service is bound to the same host port:**

```bash
docker ps --format "table {{.ID}}\t{{.Ports}}\t{{.Names}}"
docker port <container_id>
```

**Step 6: Check the app's configured port for accidental duplication across services:**

```bash
grep -ri "port" /etc/myapp/config.yml
grep -ri "8080" /etc/*/*.conf 2>/dev/null
```

**Step 7: Check if the previous process didn't release the socket (TIME_WAIT / lingering socket):**

```bash
ss -tan | grep 8080
# Look for sockets stuck in TIME_WAIT or CLOSE_WAIT
```

**Step 8: Check systemd for stale or duplicate unit instances:**

```bash
systemctl list-units --type=service | grep myapp
systemctl list-units --state=failed
```

## 4. Likely root cause:

**Investigation findings:**

```bash
# Command
ss -tlnp | grep 8080

# Output
LISTEN 0 128 0.0.0.0:8080 0.0.0.0:* users:(("myapp",pid=4821,fd=6))
```

```bash
# Command
ps -p 4821 -o pid,ppid,cmd

# Output
4821  1  /usr/bin/myapp --config /etc/myapp/config.yml
```

```bash
# Command
systemctl status myapp

# Output
Active: failed (Result: exit-code)
Process: myapp.service: Failed with result 'exit-code'
```

**Analysis:**

- A previous instance of the application (PID 4821) was still running and holding port 8080.
- The deploy/restart script started a new instance without first stopping the old one.
- systemd's restart attempt for the new process failed because the port was already bound by the leftover process.
- The old instance likely survived because a prior `systemctl restart` did not fully terminate the process (e.g., app ignored SIGTERM, or a crash left an orphaned child).

**Other possible root causes (depending on findings):**

- A different, unrelated service was accidentally configured to use the same port.
- A container that was supposed to be stopped/removed still has the host port mapped.
- Two instances of the deployment pipeline ran concurrently, each trying to start the app.
- Application does not clean up its socket on crash, leaving it in a bound state.
- Port conflict introduced by a config change (e.g., port number typo matching another service).
- OS-level socket lingering in `TIME_WAIT` blocking rebind (less common, usually mitigated by `SO_REUSEADDR`).

**Final root cause (in this scenario):**

> The previous application process did not terminate cleanly during the last restart and continued holding port 8080 in the background. When the deployment attempted to start a new instance, it failed to bind to the already-occupied port, causing the service to enter a failed state.

## 5. The fix:

**Step 1: Identify and confirm the process holding the port:**

```bash
lsof -i :8080
```

**Step 2: Stop the stale process safely:**

```bash
kill <pid>
# If it does not terminate:
kill -9 <pid>
```

**Step 3: Confirm the port is now free:**

```bash
ss -tlnp | grep 8080
# Expected: no output
```

**Step 4: Start the application service:**

```bash
systemctl start myapp
```

**Step 5: Verify the new process is bound correctly:**

```bash
ss -tlnp | grep 8080
systemctl status myapp
```

**Step 6: Confirm the application responds correctly:**

```bash
curl -I http://localhost:8080
```

**Step 7:** Confirm the load balancer/health check marks the instance healthy again.

## 6. Recurrence prevention:

**a) Ensure the service is fully stopped before restart, not just restarted blindly:**

```bash
systemctl stop myapp && sleep 2 && systemctl start myapp
```

Prefer an explicit stop-then-start over relying solely on `systemctl restart` if the app is known to linger.

**b) Handle graceful shutdown properly in the application:**

Ensure the app releases its socket on `SIGTERM` and does not require `SIGKILL` to exit. Set `SO_REUSEADDR` where appropriate so a quick restart can rebind immediately.

**c) Add a pre-start check in the systemd unit or deploy script:**

```bash
# Pre-start script
if lsof -i :8080 >/dev/null; then
  echo "Port 8080 already in use, aborting deploy"
  exit 1
fi
```

**d) Prevent concurrent deploys:**

Add deployment locking (e.g., a lock file or CI/CD concurrency limit) so two deploy jobs cannot start the same service simultaneously.

**e) Container hygiene:**

Always run `docker stop` and `docker rm` before starting a replacement container, or use `docker run --rm` with orchestration that guarantees old containers are removed before new ones start.

**f) Monitoring alerts:**

| Condition | Severity |
|---|---|
| Application health check failing for over 1 min | Warning |
| Application health check failing for over 3 min | Critical |

## 7. Which monitoring alert should have detected it?

**Health check / uptime probe:**

```yaml
- alert: AppPortDown
  expr: probe_success{job="app_health_check"} == 0
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "Application on {{ $labels.instance }} is not responding on its port"
```

**systemd failure alert (via node_exporter/textfile collector):**

```yaml
- alert: SystemdUnitFailed
  expr: node_systemd_unit_state{name="myapp.service", state="failed"} == 1
  for: 0m
  labels:
    severity: critical
  annotations:
    summary: "myapp.service is in a failed state"
```

**Grafana alert:**

| Level | Condition |
|---|---|
| Warning | Health check failing > 1 min |
| Critical | Health check failing > 3 min |

**CloudWatch / Cloud Provider:**

- **AWS** — ALB/NLB target health check marking instance unhealthy, triggering CloudWatch alarm.
- **GCP** — Load balancer health check failure alert via Cloud Monitoring.
- **Azure** — Azure Monitor alert on App Service/VM health probe failure.

**Nagios / Zabbix / Datadog:**

- TCP/HTTP port check on the application port with a short interval.
- Alert on 2 consecutive failures, correlated with a systemd unit-failed check.

**Expected Detection:** Within 1–3 minutes of the failed restart, before the load balancer removes significant capacity or users notice errors.

## 8. RCA (Root Cause Analysis)

| Field | Details |
|---|---|
| Incident Title | Application Failed to Start Due to Port Already in Use |
| Incident Date | 08-Jun-2026 |
| Severity | P1 Critical |
| Duration | 22 minutes |
| Owner | DevOps Team |

**Summary:**

Following a routine restart, the application failed to come back online. Investigation showed a previous instance of the process had not fully terminated and was still holding the application's port, blocking the new instance from binding to it.

**Impact:**

- Application was unreachable, returning connection errors to users.
- Load balancer marked the instance unhealthy and removed it from rotation.
- On-call engineer had to manually intervene to restore service.

**Timeline:**

| Time | Event |
|---|---|
| T+00 min | Restart triggered as part of routine deploy |
| T+01 min | New application instance fails to start |
| T+02 min | Health check begins failing; load balancer marks instance unhealthy |
| T+05 min | On-call engineer paged |
| T+08 min | `journalctl -u myapp -xe` shows "address already in use" |
| T+12 min | `lsof -i :8080` identifies stale process still bound to port |
| T+15 min | Stale process terminated |
| T+18 min | Application restarted successfully and port confirmed bound |
| T+20 min | Health checks pass, instance returns to load balancer rotation |
| T+22 min | Incident closed, preventive actions logged |

**Root Cause:**

> A prior application process failed to shut down cleanly during the previous restart cycle and remained running, holding the service's TCP port. The subsequent deploy's new process could not bind to the same port and exited with an error, leaving the service down until the stale process was manually identified and killed.

**Resolution:**

- Identified the stale process using `lsof -i :8080`.
- Terminated the stale process.
- Confirmed the port was free using `ss -tlnp`.
- Restarted the application service successfully.
- Verified health checks passed and load balancer restored the instance.

**Preventive Actions:**

- Add a pre-start port-availability check to the deployment pipeline.
- Ensure the application handles `SIGTERM` gracefully and releases its socket promptly.
- Add deployment locking to prevent concurrent restarts of the same service.
- Add systemd unit-failed and health-check monitoring alerts.
- Document a standard "stop, verify, start" restart procedure instead of a single restart command.

## 9. Business Impact Estimate

**Technical impact:**

- Application unavailable for the duration of the incident.
- Load balancer capacity reduced by one instance during the outage.
- Manual intervention required to restore service, increasing MTTR.

**Customer impact:**

- Users attempting to access the application during the window received errors.
- Any in-flight requests to the affected instance were dropped or failed.

**Revenue impact example:**

| Parameter | Value |
|---|---|
| Users per hour | 1000 |
| Average revenue per user | Rs. 100 |
| Outage duration | 22 minutes |
| Affected users | 1000 × 0.37 = 370 users |
| Potential revenue loss | 370 × Rs. 100 = Rs. 37,000 |

Even a routine restart can cause a full outage if the previous process is not verified as fully stopped. The compounding risk is that this type of failure often recurs on every future deploy until the underlying shutdown/startup sequencing issue is fixed, not just patched once.
