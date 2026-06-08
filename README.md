# Linux Production Incident Reports

A personal collection of real-world Linux production incidents I've worked through — documented with full troubleshooting steps, root cause analysis, fixes, and prevention strategies.

---

## Why this exists

I started writing these down because I kept running into situations where I'd solved something before but couldn't remember exactly what I did or what the actual root cause was. So now I document every significant incident the same way, every time.

Each report follows the same structure so it's easy to scan and reuse.

---

## Report Structure

Every incident report covers:

1. **What is the symptom?** — What was observed, how to verify it
2. **What logs will you check?** — Where to look and what to look for
3. **What commands will you run?** — Step-by-step investigation commands
4. **What is the likely root cause?** — Findings and analysis
5. **How would you fix it?** — Recovery steps in order
6. **How would you prevent recurrence?** — Long-term fixes and hardening
7. **Which monitoring alert should have detected it?** — Prometheus, Grafana, CloudWatch etc.
8. **RCA (Root Cause Analysis)** — Formal write-up with timeline
9. **Business impact estimate** — Technical, customer, and revenue impact

---

## Incidents

| # | Title | Severity | Status |
|---|-------|----------|--------|
| 01 | Linux Root Filesystem Full (502 Bad Gateway) | P1 Critical | Resolved |
| 02 | SSH Inaccessible After Reboot | P1 Critical | Resolved |

---

## Stack / Environment

These incidents are based on Linux servers (Ubuntu / RHEL / CentOS).
Tools referenced: `systemctl`, `journalctl`, `sshd`, `ufw`, `firewalld`, `iptables`, `df`, `du`, `lsof`, `ss`, `netstat`.
Monitoring stack: Prometheus, Grafana, Blackbox Exporter, CloudWatch.

---

## Notes

- Incident dates and revenue numbers are illustrative examples used for RCA practice.
- This repo is meant to grow — every new incident I solve gets added here.
- If something helped you, feel free to use it.

---

*Maintained by [Your Name]*
