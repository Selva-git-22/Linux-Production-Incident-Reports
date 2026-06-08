# Incident 01 — SSH Inaccessible After Reboot

**Category:** Linux Production Outage
**Severity:** P1 Critical
**Status:** Resolved

---

## Scenario

- Server was rebooted (planned or unplanned).
- SSH connection attempts are failing or timing out.
- No response on port 22 from remote clients.
- Console / out-of-band access (cloud console or physical terminal) is still available.
- Server itself is up and running — only SSH is broken.

---

## 1. What is the symptom?

**Observed symptoms:**
- SSH connection refused or times out.
- Remote team unable to access the server.
- Automation scripts and CI/CD pipelines failing due to SSH errors.
- No response on port 22.
- Server is reachable via ping but SSH is dead.

**Verification:**

```bash
ssh -v user@<server-ip>
telnet <server-ip> 22
nc -zv <server-ip> 22
```

**Sample output:**

```
ssh: connect to host <server-ip> port 22: Connection refused
```
or
```
ssh: connect to host <server-ip> port 22: Connection timed out
```

**Conclusion:** SSH service is not accepting connections post reboot.

---

## 2. What logs will you check?

**SSH service logs:**

```bash
journalctl -u ssh -xe
journalctl -u sshd -xe
# Check for: "Failed to start OpenSSH Server Daemon"
```

**Auth logs:**

```bash
tail -100 /var/log/auth.log        # Debian/Ubuntu
tail -100 /var/log/secure          # RHEL/CentOS
# Look for: "error", "refused", "Permission denied", "Bad configuration"
```

**Syslog / System journal:**

```bash
journalctl -b -p err
journalctl --since "30 minutes ago"
# Check for: startup failures, port binding errors, config parse errors
```

**Firewall logs:**

```bash
tail -100 /var/log/ufw.log         # UFW
journalctl -u firewalld            # firewalld
# Look for: port 22 being blocked or dropped
```

---

## 3. What commands will you run?

**Step 1 — Check if SSH service is running:**

```bash
systemctl status ssh
systemctl status sshd
```

**Step 2 — Check if SSH is listening on port 22:**

```bash
ss -tlnp | grep 22
netstat -tlnp | grep 22
```

**Step 3 — Try to start SSH if it's stopped:**

```bash
systemctl start ssh
systemctl start sshd
```

**Step 4 — Check SSH configuration for syntax errors:**

```bash
sshd -t
sshd -T | head -30
```

**Step 5 — Check sshd_config for recent changes:**

```bash
cat /etc/ssh/sshd_config
grep -i "Port\|PermitRootLogin\|PasswordAuthentication\|ListenAddress" /etc/ssh/sshd_config
```

**Step 6 — Check if port 22 is blocked by firewall:**

```bash
ufw status                                                          # UFW
firewall-cmd --list-all                                             # firewalld
iptables -L -n | grep 22
```

**Step 7 — Check if the SSH port was changed:**

```bash
grep -i "Port" /etc/ssh/sshd_config
```

**Step 8 — Check if host keys exist:**

```bash
ls -lh /etc/ssh/ssh_host_*
```

**Step 9 — Check network interface is up:**

```bash
ip addr show
ip link show
```

**Step 10 — Check if server is reachable at all (from remote machine):**

```bash
ping <server-ip>
```

---

## 4. What is the likely root cause?

**Investigation findings:**

```bash
# Command
systemctl status sshd

# Output
"Failed to start OpenSSH Server Daemon"
```

```bash
# Command
sshd -t

# Output
/etc/ssh/sshd_config line 14: Bad configuration option: PermitRootlogin
```

```bash
# Command
ss -tlnp | grep 22

# Output
(no output — port 22 not listening)
```

**Analysis:**

- A config change was made to `/etc/ssh/sshd_config` before the reboot.
- The change introduced a syntax error in the configuration file.
- When the server rebooted, sshd attempted to load the config, failed, and never started.
- Since sshd never came up, port 22 was never opened.
- Result: All SSH connections refused.

**Other possible root causes (depending on findings):**

- Firewall rule added before reboot blocking port 22.
- SSH host keys missing or corrupted (rare, can happen after OS-level changes).
- Port changed in `sshd_config` without firewall being updated.
- `sshd` service disabled (`systemctl disable ssh` accidentally run).
- Network interface misconfigured — server got a different IP after reboot.
- SELinux/AppArmor policy blocking sshd from binding to port.

**Final root cause (in this scenario):**

> Syntax error introduced in `/etc/ssh/sshd_config` caused sshd to fail on startup after reboot, leaving port 22 unbound and all remote SSH access broken.

---

## 5. How would you fix it?

> Recovery performed via cloud console or physical terminal.

**Step 1 — Check and fix sshd_config syntax error:**

```bash
sshd -t
nano /etc/ssh/sshd_config
# Fix the offending line flagged by sshd -t
```

**Step 2 — Validate config after fix:**

```bash
sshd -t
# Expected: No output (clean = no errors)
```

**Step 3 — Start SSH service:**

```bash
systemctl start ssh
systemctl start sshd
```

**Step 4 — Verify SSH is now listening:**

```bash
ss -tlnp | grep 22
# Expected: sshd listening on 0.0.0.0:22 or :::22
```

**Step 5 — Enable SSH to start on boot (if it was disabled):**

```bash
systemctl enable ssh
systemctl enable sshd
```

**Step 6 — Check firewall is allowing port 22:**

```bash
ufw allow 22                                                                  # UFW
firewall-cmd --permanent --add-service=ssh && firewall-cmd --reload           # firewalld
```

**Step 7 — Test SSH from a remote machine:**

```bash
ssh user@<server-ip>
# Expected: Successful login prompt
```

**Step 8 — Verify service is running and stable:**

```bash
systemctl status ssh
# Expected: Active (running)
```

---

## 6. How would you prevent recurrence?

**a) Always validate sshd_config before reloading or rebooting:**

```bash
sshd -t && systemctl restart ssh
```
Never restart sshd without running `sshd -t` first. Make this a habit.

**b) Use sshd config test in deployment pipelines:**

If config is managed via Ansible, Chef, or Puppet — add a `sshd -t` check as a pre-task before applying changes.

**c) Keep a backup SSH port open:**

Run a secondary sshd instance on port 2222 as a failsafe:

```bash
# /etc/ssh/sshd_config.d/backup.conf
Port 2222
```

This provides a backdoor if port 22 breaks.

**d) Version-control SSH config:**

Track `/etc/ssh/sshd_config` in git or a config management tool. Any change must go through a PR/review process.

**e) Firewall rules audit before reboot:**

Before any planned reboot, verify:

```bash
ufw status
firewall-cmd --list-all
```

Confirm port 22 is allowed before rebooting.

**f) Monitoring alerts:**

| Condition | Severity |
|---|---|
| SSH port 22 unreachable > 1 min | Warning |
| SSH port 22 unreachable > 3 min | Critical |

---

## 7. Which monitoring alert should have detected it?

> Note: You can link this with the observability project and mention that after facing this, SSH port checks were added to the monitoring stack.

**Prometheus + Blackbox Exporter:**

Probe SSH port 22 every 30 seconds. Alert if probe fails for more than 1 minute.

```yaml
- alert: SSHPortDown
  expr: probe_success{job="ssh_probe"} == 0
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "SSH port 22 unreachable on {{ $labels.instance }}"
```

**Grafana alert:**

| Level | Condition |
|---|---|
| Warning | SSH port unreachable > 1 min |
| Critical | SSH port unreachable > 3 min |

**CloudWatch / Cloud Provider:**

- AWS — Route 53 Health Check on port 22.
- GCP — Uptime Check targeting TCP port 22.
- Azure — Azure Monitor with TCP port probe.

**Nagios / Zabbix / Datadog:**

- TCP port 22 check with 1-minute interval.
- Alert on 2 consecutive failures.

**Expected Detection:**

Within 1–3 minutes of server coming up with SSH broken — before any human would even notice the server has rebooted.

---

## 8. RCA — Root Cause Analysis

| Field | Details |
|---|---|
| **Incident Title** | SSH Inaccessible After Server Reboot |
| **Incident Date** | 08-Jun-2026 |
| **Severity** | P1 Critical |
| **Duration** | 35 minutes |
| **Owner** | DevOps Team |

**Summary:**

SSH access to the production server was completely lost following a reboot. The server itself remained operational, but sshd failed to start due to a syntax error in `/etc/ssh/sshd_config` introduced prior to the reboot.

**Impact:**
- All remote SSH access was unavailable.
- DevOps team unable to log in remotely.
- Deployment pipelines and automation scripts failed.
- Recovery required out-of-band console access (cloud console / physical terminal).

**Timeline:**

| Time | Event |
|---|---|
| T+00 min | Server reboot initiated |
| T+02 min | Server comes back online |
| T+03 min | SSH connection attempts from remote clients begin failing |
| T+05 min | Team confirms server is pingable but SSH is down |
| T+08 min | Console access established via cloud/physical terminal |
| T+12 min | `sshd -t` reveals config syntax error |
| T+18 min | `sshd_config` corrected, SSH service started |
| T+20 min | SSH confirmed working from remote client |
| T+35 min | Full incident review completed, monitoring alert added |

**Root Cause:**

A syntax error was introduced in `/etc/ssh/sshd_config` during a manual config change before the reboot. The sshd daemon failed to parse the config and exited on startup. No pre-reboot validation of the SSH config was performed.

**Resolution:**
- Accessed server via cloud console.
- Identified config error using `sshd -t`.
- Fixed the syntax error in `sshd_config`.
- Started sshd service manually.
- SSH access restored.

**Preventive Actions:**

1. Enforce `sshd -t` validation before any SSH config change.
2. Add SSH port 22 probe to monitoring stack (Blackbox Exporter).
3. Version-control `/etc/ssh/sshd_config` in git.
4. Configure a secondary SSH port (2222) as emergency fallback.
5. Add pre-reboot checklist: verify SSH config, firewall rules, and service status.

---

## 9. Business Impact Estimate

**Technical impact:**
- Complete loss of remote SSH access.
- DevOps and on-call team locked out of server.
- Deployment pipelines broken.
- Automation and monitoring agents unable to connect.
- Manual console access required, slowing recovery.

**Customer impact:**
- Users unable to access services if downstream application was affected.
- Delayed deployments and hotfixes during SSH downtime.
- Increased MTTR (Mean Time To Recover) due to lack of remote access.

**Revenue impact example:**

| Parameter | Value |
|---|---|
| Users per hour | 1000 |
| Average revenue per user | Rs. 100 |
| Outage duration | 35 minutes |
| Affected users | 1000 × 0.58 = **580 users** |
| Potential revenue loss | 580 × Rs. 100 = **Rs. 58,000** |

> Even if the application itself stayed up during the SSH outage, the inability to SSH in means any active incident during that window cannot be responded to remotely. The real risk is compounded impact — if a second issue had arisen during those 35 minutes, recovery time would have been significantly longer.
