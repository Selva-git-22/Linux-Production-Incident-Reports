# Incident 3: LINUX PRODUCTION OUTAGE - Memory Usage Continuously Increasing

## Scenario

- Server's memory usage is growing steadily over time.
- Application response times are degrading progressively.
- The OOM (Out-Of-Memory) killer is firing and terminating processes.
- Monitoring dashboards show a **sawtooth pattern** - memory climbs, process is killed, restarts, and climbs again.
- The server is up and running, but the application is degraded or crashing.

---

## 1. Symptoms

**Observed symptoms:**
- RAM usage climbs steadily over hours/days and never comes back down, even during low-traffic periods.
- System becomes sluggish as the kernel starts using swap space; I/O spikes visible in `iostat`.
- OOM killer fires - processes are randomly terminated; `dmesg` shows `oom-kill` entries.
- Application response times degrade progressively; timeouts and 502/503 errors begin appearing.
- Monitoring shows sawtooth pattern - memory rises until OOM kills the process, which restarts and rises again.
- A specific process (Java, Node.js, Python, MySQL) is the top consumer in `top` or `ps`.
- Server may become completely unresponsive, requiring a hard reboot.

**Verification:**

```bash
free -h
top                                      # sort by %MEM
ps aux --sort=-%mem | head -20
dmesg | grep -i "oom\|killed\|out of memory"
```

**Sample output:**

```
PID    USER  PR  NI  VIRT    RES    SHR  %MEM  COMMAND
 3412  app   20   0  8.2g   7.4g   12m   93.1  java
```

```
[ 4823.112] Out of memory: Kill process 3412 (java) score 892 or sacrifice child
[ 4823.113] Killed process 3412 (java) total-vm:8388608kB, anon-rss:7812444kB
```

**Conclusion:** A process is leaking memory and not releasing it, causing progressive memory exhaustion and eventual OOM kill.

---

## 2. Logs to check

**OOM killer logs (most critical):**

```bash
dmesg | grep -i "oom\|killed\|out of memory"
journalctl -k | grep -i "oom\|memory"
grep -i "oom\|out of memory" /var/log/syslog
```
> Look for: process names, PIDs, and memory sizes at time of kill.

**Application logs:**

```bash
tail -500 /var/log/app/application.log | grep -i "heap\|memory\|gc\|OutOfMemory"
```
> Java → `java.lang.OutOfMemoryError`  
> Python → `MemoryError` tracebacks  
> Node.js → `ENOMEM`, `JavaScript heap out of memory`

**System memory history:**

```bash
sar -r 1 10                              # recent memory stats via sysstat
sar -r -f /var/log/sa/saXX              # historical from sysstat daily files
```
> Look for: when memory growth started, correlation with deployments.

**Swap usage logs:**

```bash
vmstat 5 20
```
> Look for: `si`/`so` columns (swap-in / swap-out rate increasing).

**Systemd service restart logs:**

```bash
journalctl -u your-app.service --since "6 hours ago" | grep -i "killed\|OOM\|restart"
systemctl show your-app.service | grep -i restart
```
> Look for: repeated restarts indicating OOM kills followed by auto-restart.

**Java GC logs (if enabled):**

```bash
cat /var/log/app/gc.log
```
> Look for: `GC overhead limit exceeded`, full GC firing every few seconds.

**Kubernetes logs (if containerised):**

```bash
kubectl describe pod <name> | grep -A5 "OOM\|Limits\|Requests"
kubectl get events --field-selector reason=OOMKilling
```
> Look for: `OOMKilled` in the Last State section.

---

## 3. Commands to run

**Step 1: Check current memory usage:**

```bash
free -h
cat /proc/meminfo
vmstat -s
```

**Step 2: Identify top memory-consuming processes:**

```bash
ps aux --sort=-%mem | head -20
top -b -n 1 | head -30
smem -r -s rss | head -20
pmap -x <PID> | tail -5
```

**Step 3: Check OOM kill history:**

```bash
dmesg | grep -i "oom\|killed\|out of memory"
journalctl -k --since "6 hours ago" | grep -i oom
```

**Step 4: Track memory growth of suspect process over time:**

```bash
watch -n 5 'ps aux --sort=-%mem | head -10'

while true; do
  date; ps -p <PID> -o pid,vsz,rss,%mem --no-headers
  sleep 30
done
```

**Step 5: Check swap usage and pressure:**

```bash
swapon --show
vmstat 2 10                              # watch si/so columns
cat /proc/sys/vm/swappiness
```

**Step 6: Check kernel slab cache for kernel-level leaks:**

```bash
cat /proc/slabinfo | sort -k3 -rn | head -20
slabtop -o
```

**Step 7: Check if tmpfs is being filled:**

```bash
df -h | grep tmpfs
du -sh /dev/shm /tmp /run
```

**Step 8: Java heap analysis:**

```bash
jstat -gcutil <PID> 5s 20
jmap -histo:live <PID> | head -30
jcmd <PID> GC.heap_info
jmap -dump:format=b,file=/tmp/heap.hprof <PID>
```

**Step 9: Python memory profiling:**

```bash
pip install memory-profiler
python -m memory_profiler app.py
mprof run app.py && mprof plot
```

**Step 10: Node.js heap snapshot:**

```bash
node --inspect app.js                    # attach Chrome DevTools
kill -USR1 <PID>                         # trigger V8 snapshot if configured
```

**Step 11: Kubernetes memory check:**

```bash
kubectl top pods --sort-by=memory
kubectl describe pod <name> | grep -A5 "OOM\|Limits\|Requests"
kubectl get events --field-selector reason=OOMKilling
```

---

## 4. Likely Root Cause

**Investigation findings:**

```bash
$ ps aux --sort=-%mem | head -5
# Output: order-service (java) consuming 93% of RAM

$ jmap -histo:live <PID> | head -10
# Output: 1,200,000 instances of ProductDTO consuming 4.2GB

$ dmesg | grep oom
# Output: OOM kill events every ~6 hours since last deployment

$ journalctl -u order-service | grep restart
# Output: Service auto-restarted 3 times in the last 24 hours
```

**Analysis:**
- A new product recommendation feature was deployed in v2.14.3.
- The feature introduced a `HashMap<Long, ProductDTO>` used as an in-memory cache.
- The cache had **no maximum size, no TTL, and no eviction policy**.
- Every unique product ID requested was stored and never removed.
- Under production traffic (~200 req/s), 1.2M objects accumulated over 6 hours, consuming 4.2GB of heap.
- JVM heap grew until the OOM killer terminated the process; systemd restarted it and the cycle repeated.

**Other possible root causes (depending on findings):**
- **Connection/thread pool leak** - DB or HTTP connections not returned to pool.
- **Event listeners not removed** - listeners added in loops, never cleaned up (Node.js, Java).
- **Large dataset loaded in memory** - full table/file loaded instead of streaming.
- **JVM heap misconfiguration** - `-Xmx` not set; JVM expands until the OS kills it.
- **Kernel slab cache leak** - visible in `/proc/slabinfo`, not tied to any user-space process.
- **tmpfs exhaustion** - `/dev/shm` or `/tmp` filled by runaway logs or temp files.

**Final root cause (in this scenario):**  
Unbounded in-memory `HashMap` introduced in v2.14.3 with no eviction policy caused heap to grow continuously until OOM kill, repeating on every restart.

---

## 5. The Fix

**Step 1: Capture a heap dump BEFORE restarting (for root cause analysis):**

```bash
# Java
jmap -dump:format=b,file=/tmp/heap_$(date +%s).hprof <PID>

# Python
python -c "import tracemalloc; tracemalloc.start()" app.py

# Node.js (if heapdump module installed)
kill -USR2 <PID>
```

**Step 2: Restart the leaking process to restore service:**

```bash
systemctl restart your-app.service
```

**Step 3: Configure systemd to auto-restart on OOM:**

```ini
# /etc/systemd/system/your-app.service
[Service]
Restart=on-failure
RestartSec=5s
OOMPolicy=restart
```

```bash
systemctl daemon-reload
```

**Step 4: Reduce swap pressure to protect system stability:**

```bash
sysctl -w vm.swappiness=10
echo "vm.swappiness=10" >> /etc/sysctl.conf
```

**Step 5: Analyse the heap dump to find the leaking object:**
- Open `heap.hprof` in **Eclipse MAT** or **VisualVM**.
- Check the **dominator tree** - identify the top retained objects.
- Find the **allocation stack trace** for the leaking class.

**Step 6: Fix the root cause in code:**

```java
// Replace unbounded HashMap with a bounded Guava cache
CacheBuilder.newBuilder()
  .maximumSize(10_000)
  .expireAfterWrite(30, TimeUnit.MINUTES)
  .build()

// Close resources properly using try-with-resources
try (Connection conn = dataSource.getConnection()) { ... }
```

```javascript
// Remove event listeners when no longer needed (Node.js)
emitter.removeListener('event', handler)
```

**Step 7: Set hard memory limits to fail fast rather than degrade:**

```bash
# JVM flags — explicit heap bounds + auto heap dump on OOM
-Xms512m -Xmx2g -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/
```

```yaml
# Kubernetes resource limits
resources:
  requests:
    memory: "512Mi"
  limits:
    memory: "2Gi"
```

**Step 8: Deploy the fix and confirm flat memory profile:**
- Deploy patched version.
- Monitor RSS/heap usage for 30 minutes under load.
- Confirm memory is stable before closing the incident.

---

## 6. Recurrence Prevention

**a) Always capture heap dumps on OOM automatically:**

```bash
# Add to JVM startup flags
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/
```
> Ensures forensic data is always available without manual intervention.

**b) Set hard memory limits on all services:**
- JVM: always define `-Xms` and `-Xmx` explicitly.
- Node.js: use `--max-old-space-size` flag.
- Kubernetes: set both `requests` and `limits` for every container.
- Never allow a process to consume unbounded memory.

**c) Enforce cache eviction policies in code review:**
- Any in-process cache must have a maximum size limit AND a TTL.
- Treat raw unbounded collections used as caches as a **blocking PR comment**.

**d) Add memory profiling to CI/CD pipeline:**
- Run load tests with memory profilers (JProfiler, Py-Spy, Valgrind).
- Fail the build if RSS grows beyond a defined threshold during the test.

**e) Add 24-hour soak test for data-access changes:**
- For any PR touching caching, data-access, or object-creation code, run the service under realistic load in staging for 24 hours.
- Memory leaks invisible in 5-minute tests become obvious here.

**f) Monitoring — create alerts:**
- Memory growth rate > 3% per hour for 2+ consecutive hours → **Warning**
- Memory utilization > 85% for 5+ minutes → **Critical**
- OOM kill event detected → **Critical (immediate page)**

---

## 7. Which Monitoring Alert Should Have Detected It?

> **Note:** You can link this with the observability project and mention that after facing this incident, we added memory growth rate alerts to our monitoring stack.

**Prometheus + Alertmanager - memory utilisation threshold:**

```yaml
- alert: HighMemoryUsage
  expr: node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes < 0.15
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Memory usage above 85% on {{ $labels.instance }}"
```

**Prometheus + Alertmanager — OOM kill detection:**

```yaml
- alert: OOMKillDetected
  expr: increase(node_vmstat_oom_kill[5m]) > 0
  labels:
    severity: critical
  annotations:
    summary: "OOM kill detected on {{ $labels.instance }}"
```

**Prometheus + Alertmanager - memory growth rate (best early-warning alert):**

```yaml
- alert: MemoryGrowthAnomaly
  expr: deriv(node_memory_MemAvailable_bytes[1h]) < -0.03
  for: 2h
  labels:
    severity: warning
  annotations:
    summary: "Memory growing continuously on {{ $labels.instance }}"
```

**Grafana alert:**
- Warning  = Memory growth rate > 3%/hr for 2+ hours.
- Critical = Memory utilization > 85% for 5 minutes.
- Critical = OOM kill counter increments.

**JVM-specific (Prometheus JMX Exporter):**

```yaml
- alert: JVMHeapHigh
  expr: jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes > 0.85
  for: 2m
  labels:
    severity: warning
  annotations:
    summary: "JVM heap above 85% on {{ $labels.instance }}"
```

**Kubernetes:**

```bash
kubectl get events --field-selector reason=OOMKilling
```

```yaml
- alert: PodOOMKilled
  expr: kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} > 0
  labels:
    severity: critical
  annotations:
    summary: "Pod {{ $labels.pod }} was OOMKilled"
```

**Expected detection:**
- Growth rate alert fires **4–5 hours** before OOM kill.
- Heap usage alert fires **~30 minutes** before OOM kill.
- OOM kill alert fires **within seconds** of the event.

---

## 8. RCA (Root Cause Analysis)

**Incident title:** Memory Usage Continuously Increasing - Production API Outage

**Incident Date:** 08-Jun-2026

**Severity:** P1 Critical

**Duration:** 6 hours degraded performance; 23 minutes full outage

**Summary:**  
Memory usage on the production `order-service` (Java 17, Spring Boot) grew continuously following the v2.14.3 deployment. After 6 hours of degraded performance, the OOM killer terminated the JVM process causing a 23-minute full outage. The root cause was an unbounded in-memory `HashMap` introduced in the new product recommendation feature with no eviction policy or size limit.

**Impact:**
- All order-service API endpoints unavailable for 23 minutes.
- 6 hours of elevated latency (120ms → 480ms p99) before full outage.
- Repeated OOM kills causing 3 unplanned service restarts in 24 hours.
- Deployment pipelines blocked during the outage window.
- Engineering time consumed: 4 engineers × 4 hours = 16 engineer-hours.

**Timeline:**

| Time | Event |
|------|-------|
| T+00 min | v2.14.3 deployed with new product recommendation feature. |
| T+120 min | Memory climbs from 38% to 55%. No alerts fire. |
| T+240 min | Memory at 75%. GC frequency doubles. Latency increases (120ms → 480ms). |
| T+330 min | Customer complaints received. Memory at 91%. P1 declared. |
| T+360 min | OOM killer terminates JVM. Service unavailable. systemd auto-restarts. |
| T+370 min | Heap dump captured. `RecommendationCache` identified as leak source. |
| T+383 min | Feature flag disabled. Memory stabilises at 41%. Latency returns to normal. |
| T+600 min | v2.14.4 deployed with bounded Guava cache. Soak test passed. Incident closed. |

**Root Cause:**  
The `RecommendationCache` class in v2.14.3 used a raw `HashMap<Long, ProductDTO>` with no maximum size, no TTL, and no eviction policy. Every unique product ID requested was inserted and never removed. Under production traffic (~200 req/s), 1.2 million `ProductDTO` objects accumulated over 6 hours, consuming 4.2GB of heap. No pre-deployment soak test was performed. No memory growth rate alert existed.

**Resolution:**
- Captured heap dump from live instance before restart.
- Identified leak source via `jmap -histo:live` (1.2M `ProductDTO` instances).
- Disabled recommendation feature via feature flag to stop memory growth immediately.
- Replaced unbounded `HashMap` with Guava `CacheBuilder` (max 5,000 entries, 30-min TTL).
- Deployed v2.14.4 and confirmed flat memory profile under load.

**Preventive Actions:**
1. Replace all unbounded in-memory collections with size-limited, TTL-based caches.
2. Add memory growth rate alert to monitoring stack (Prometheus + Alertmanager).
3. Add `-XX:+HeapDumpOnOutOfMemoryError` to all JVM startup configurations.
4. Enforce 24-hour soak test in staging pipeline for any caching or data-access changes.
5. Add PR review checklist item: every cache must document its eviction strategy.

**Owner:** DevOps and Backend Engineering team.

---

## 9. Estimated Business Impact

**Technical impact:**
- 23 minutes of complete API unavailability for the order-service.
- 6 hours of degraded performance (p99 latency 4× above baseline).
- Repeated OOM kills causing 3 unplanned service restarts in 24 hours.
- Deployment pipelines blocked during the outage window.
- Engineering time consumed: 4 engineers × 4 hours = 16 engineer-hours.

**Customer impact:**
- Users unable to place orders during the 23-minute full outage.
- Slow checkout experience (4× latency) leading to cart abandonment.
- Delayed deployments and hotfixes during the degraded window.
- Increased MTTR due to need for heap dump capture and code fix before recovery.

**Revenue impact example:**

| Parameter | Value |
|-----------|-------|
| Users per hour | 12,000 |
| Average revenue per user | Rs. 500 |
| Full outage duration | 23 minutes |
| Affected users (outage) | 12,000 × 0.38 = 4,600 users |
| **Potential revenue loss (outage)** | **4,600 × Rs. 500 = Rs. 23,00,000** |

**Degraded period (6 hrs, 30% conversion drop):**

| Parameter | Value |
|-----------|-------|
| Lost conversions per hour | 12,000 × 30% = 3,600 users/hr |
| Lost revenue per hour | 3,600 × Rs. 500 = Rs. 18,00,000 |
| **Over 6 hours** | **Rs. 1,08,00,000** |

> **Note:** Even during the degraded period before the OOM kill, slow response times directly increase cart abandonment rates. The real risk is compounded impact - if a second production issue had arisen during those 6 hours, the team's ability to respond would have been significantly hampered by the ongoing memory incident consuming all engineering attention. The cost of prevention (a bounded cache + a soak test) was less than 4 hours of engineering time.
