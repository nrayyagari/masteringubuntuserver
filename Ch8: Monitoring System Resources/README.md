# Chapter 8: Monitoring System Resources

## Table of Contents
1. [Why Monitor System Resources](#why-monitor-system-resources)
2. [Critical System Resources Overview](#critical-system-resources-overview)
3. [CPU: From Fundamentals to Production](#cpu-from-fundamentals-to-production)
   - [CPU Fundamentals](#cpu-fundamentals)
   - [Multi-Core Processors](#multi-core-processors)
   - [CPU Utilization Zones](#cpu-utilization-zones)
   - [Why 100% CPU is Bad](#why-100-cpu-is-bad)
4. [Memory (RAM & Swap)](#memory-ram--swap)
   - [How Memory is Allocated](#how-memory-is-allocated)
   - [When Swap Activates](#when-swap-activates)
   - [Memory Thresholds](#memory-thresholds)
5. [CPU Metrics & How to Check Them](#cpu-metrics--how-to-check-them)
6. [Understanding htop Output](#understanding-htop-output)
7. [Tools & Commands Reference](#tools--commands-reference)
8. [Common Misconceptions](#common-misconceptions)
9. [Real-World Troubleshooting Example](#real-world-troubleshooting-example)

---

## Why Monitor System Resources

In production systems, undetected resource exhaustion kills services predictably:

- **Memory leak** → OOM killer terminates processes → Service goes down at 3AM
- **Disk fills up** → Databases can't write → Data corruption
- **CPU maxes out** → Request queue grows → User-facing latency → Customers leave
- **Network saturates** → Packet loss → Application timeouts → Cascading failures

**Monitoring is your early warning system** that catches these issues before users notice. The goal is **reliable performance under load**, not theoretical maximum efficiency.

---

## Critical System Resources Overview

### 1. **CPU Utilization & Load Average**
- **Why**: CPU saturation means your server can't keep up
- **Metric**: Load average (should be ≤ 0.75 × number of cores)
- **Threshold**: Warn at 70-80%, critical at 90%+
- **Reality Check**: A webserver at 85% CPU for 2 hours means you need to scale

### 2. **Memory (RAM + Swap)**
- **Why**: When available memory hits near-zero, kernel swaps or kills processes
- **Metric**: Free/available memory (not just "used %"). Monitor swap usage.
- **Threshold**: Warn when available < 10-15%, critical when < 5%
- **Impact**: Process with memory leak silently consumes all RAM

### 3. **Disk I/O (Read/Write Latency)**
- **Why**: High disk latency means storage can't keep up → everything slows down
- **Metrics**: IOPS, read/write latency (ms), queue length
- **Threshold**: Average latency > 10-20ms suggests I/O contention
- **Reality**: Database writes suddenly take 500ms instead of 5ms → connection pool exhausts

### 4. **Disk Space**
- **Why**: Full disks break everything. Databases can't write, logs stop, apps crash.
- **Metric**: Used % per filesystem
- **Threshold**: Warn at 80%, critical at 90%
- **Problem**: Log files growing unchecked can fill a disk in hours

### 5. **Network Traffic & Connections**
- **Why**: Network saturation causes packet loss, retransmissions, timeouts
- **Metrics**: Bandwidth utilization, packet loss, open connections, connection states
- **Threshold**: Warn at 70-80% of link capacity
- **Cascading**: DDoS or runaway batch job saturates link → legitimate traffic dropped

### 6. **Process & Service Health**
- **Why**: Running OS ≠ running application. Process can hang, crash, or leak silently.
- **Metrics**: Is critical process running? Memory/CPU per process, zombie processes
- **Threshold**: Any unexpected process exit = alert

### 7. **System Temperature (Hardware-dependent)**
- **Why**: Overheating hardware fails without warning
- **Metric**: CPU/device temperature
- **Threshold**: Warn at 80-85°C, critical at 95°C+

---

## CPU: From Fundamentals to Production

### CPU Fundamentals

#### The Core Concept: CPU is Serial

A modern CPU core executes **one instruction at a time**. When multiple processes need CPU time, the **kernel scheduler** decides: "Who runs next?"

#### Time-Slicing: The Illusion of Parallelism

```
Single-core execution (Real Hardware):
Time →  [Process A] [Process B] [Process C] [Process A] [Process B] ...
        ^50ms     ^50ms       ^50ms

What processes see (Illusion):
Process A: =============== (50ms) ======= (50ms) ======= (50ms)
Process B:           =============== (50ms) ======= (50ms)
Process C:                    =============== (50ms)
```

The kernel divides CPU time into **time slices** (1-10ms). Each process runs, then gets context-switched off so another can run.

**Key Insight**: At 50% CPU utilization, the CPU is idle 50% of the time. Processes run, complete work, then sleep. The scheduler finds other work to do.

---

### Multi-Core Processors

#### What Multi-Core Really Means

```
Single-core CPU:
  [Core 1] ← One process at a time

Dual-core CPU:
  [Core 1] ← Process A (TRULY PARALLEL)
  [Core 2] ← Process B (at same instant)

4-core CPU:
  [Core 1] ← Process A
  [Core 2] ← Process B
  [Core 3] ← Process C
  [Core 4] ← Process D
```

**Single-core with time-slicing** = Illusion of parallelism (context switching)
**Multi-core** = True parallelism (simultaneous execution on different cores)

#### Load Average on Multi-Core

Load average = Average number of processes in run queue (waiting for CPU)

```
2-core system:
  Load 1.0 = 100% utilized (all cores busy)
  Load 2.0 = 100% utilized (all cores busy, no queue)
  Load 3.0 = 150% utilized (all cores busy, 1 waiting)

4-core system:
  Load 1.0 = 25% utilized (1 core busy, 3 idle)
  Load 4.0 = 100% utilized (all cores busy)
  Load 5.0 = 125% utilized (all cores busy, 1 waiting)

Formula: Utilization % = (Load Average / Number of Cores) × 100
```

**Critical Rule**: Load 3.0 is "high" on 2-core, but "healthy" on 8-core.

---

### CPU Utilization Zones

#### **Zone 1: 0-60% Utilization** — The Safe Zone
- Scheduler easily finds runnable processes
- Response times stay low and predictable
- Run queue: Short or empty

#### **Zone 2: 60-80% Utilization** — The Saturation Zone
- More processes ready to run than CPU can handle
- Scheduler **queues** processes waiting for their turn
- User-facing latency starts increasing

#### **Zone 3: 80-100% Utilization** — The Degradation Zone
- Run queue grows, every process must wait longer
- Latency degrades linearly (90% utilization = 2x latency vs 50%)
- Context switching overhead increases

#### **Zone 4: 100%+ Utilization** — The Overload/Collapse Zone
- Run queue builds indefinitely
- **Latency degradation becomes exponential**
- Cascading failure begins: timeouts → retries → more load

#### The Performance Curve

```
Latency vs CPU Utilization:

   30ms |                           /
        |                        /
   20ms |                     /
        |                  /
   10ms |              /
        |            /
    5ms |       ___/
        |   ___/
    1ms | /
        |_________________________> CPU %
        0   20   40   60   80  100

50% utilization:  1-2ms latency (safe)
70% utilization:  3-5ms latency (acceptable)
80% utilization:  5-15ms latency (noticeable)
90% utilization:  15-50ms latency (degraded)
100%+ utilization: 50ms-seconds (broken)
```

#### System Collapse at 100%+ CPU

When CPU exceeds capacity (more work than CPU can handle):

**Physical Effects**:
1. **Listen queue fills** → Connections refused (SYN drop)
2. **Client timeouts trigger** → Users get errors after 30-60 seconds
3. **Retry cascade** → Timeouts cause retries → MORE load arrives
4. **Exponential degradation** → Response time goes from 100ms → seconds
5. **Context switching kills performance** → CPU wasted on scheduling, not work
6. **Memory pressure** → Queued connections consume RAM

**The Collapse Loop**:
```
100% CPU → High latency → Client timeouts → Retries → More load
→ CPU stays 100% → More timeouts → MORE retries → Exponential failure
```

**Paradox**: System becomes SLOWER and LESS productive despite maxing CPU.

---

### Why 100% CPU is Bad

**Wrong thinking**: "CPU should be maxed out. That's good utilization."

**Reality**: The goal is **reliable performance under load**, not theoretical maximum efficiency.

**Business perspective**:
- Max-out CPU: More hardware failures, constant on-call, frequent outages, lost revenue
- Keep CPU at 70%: Slightly more hardware, fewer outages, less operational stress, higher reliability

**The sweet spot**: 60-75% CPU utilization during normal peak load
- Leaves 25-40% headroom for spikes
- Spikes can be absorbed without cascading failure
- One server failure doesn't cascade to others
- Predictable, consistent latency

---

## Memory (RAM & Swap)

### How Memory is Allocated

```
Physical RAM (16GB):
├─ Applications: 8GB (various running programs)
├─ Cache/Buffers: 6GB (filesystem cache, helps disk I/O)
├─ Kernel: 1GB (OS internals)
└─ Free: 1GB (available for new allocations)
```

**Important distinction**:
- **Used** ≠ Unavailable (includes caches that can be freed)
- **Available** = Actually free for new allocations

---

### When Swap Activates

**Common misconception**: Swap only activates when RAM is 100% full.

**Reality**: Linux starts swapping around 70-85% RAM usage, before hitting 100%.

**Why?** To prevent emergency situations at 100%:

```
Memory pressure curve:

0-70% RAM:        No swap (plenty of free memory)
70-85% RAM:       Kernel starts deciding to swap (memory pressure rising)
85-95% RAM:       Active swapping (freeing up RAM for new allocations)
95-100% RAM:      Heavy swapping (system struggling)
100%+ RAM:        OOM killer (killing processes to survive)
```

**The trigger point**: When the kernel needs free memory to allocate to new processes, it looks at what's using RAM and moves idle pages to swap to make room.

**If kernel waited until 100%**:
- New process arrives → Nowhere to allocate → System stalls
- Emergency mode → OOM killer starts killing processes
- Cascading failure

---

### Memory Thresholds

```
Your system:
- Total RAM: 15.9G
- Used: 4.2G (26%)
- Swap used: 0B (0%)

At 26% RAM usage, swap isn't needed because plenty of memory is free.

If RAM grows to 12G (75%):
- Kernel might start swapping idle pages
- Swap: 100MB - 500MB (preventive)

If RAM hits 15G (94%):
- Heavy swapping happens
- Swap: 2-3GB (system struggling)
```

**Swap Usage Interpretation**:
- 0B = RAM has headroom, no pressure ✓
- < 500MB = Light pressure, OK
- > 1GB = Memory pressure high, system struggling
- > 3GB = System in trouble, consider adding RAM or killing processes

**What swap usage indicates**:
- Not a hard failure threshold, but a **warning signal** of memory pressure
- High swap = Disk I/O becomes bottleneck (swap is much slower than RAM)
- System entering degraded performance state

---

## CPU Metrics & How to Check Them

### The Golden Rule

**Never look at aggregate CPU %**

Aggregate CPU = Average across all cores (misleading for multi-core systems)

```
Example: 4-core server showing 25% aggregate CPU

Could mean:
A) 1 core at 100%, 3 cores at 0%          ← BOTTLENECKED
B) 4 cores at 6.25% each                  ← UNDERUTILIZED

Both show 25% but very different realities. Always check per-core.
```

### Commands Guide

#### **1. `uptime` — Load Average (Quick Check)**
```bash
$ uptime
load average: 2.45, 2.12, 1.89

Shows: 1-min, 5-min, 15-min load averages
Purpose: Quick utilization assessment
Action: Divide by nproc (2.45 / 4 cores = 61% utilization)
```

#### **2. `mpstat -P ALL 1` — Per-Core Breakdown (MOST IMPORTANT)**
```bash
$ mpstat -P ALL 1

CPU    %usr   %sys   %iowait  %idle
  0     85%     10%      2%      3%  ← MAXED
  1     92%      5%      1%      2%  ← MAXED
  2     15%     10%      5%     70%
  3      5%      2%      1%     92%

Shows: Utilization per individual core
Purpose: Find which core is the bottleneck
Use when: You need to know actual saturation (THIS IS THE REAL METRIC)
```

#### **3. `vmstat 1` — System-Wide View with I/O Wait**
```bash
$ vmstat 1

r   b  swpd   free   si   so  us sy id wa
2   0     0 2048M   0    0  40  5 55  0

r:  Run queue (processes waiting for CPU)
us: User CPU %
sy: System CPU %
id: Idle %
wa: I/O Wait % ← KEY: Shows if bottleneck is CPU or I/O

Purpose: Diagnose if bottleneck is CPU or I/O, see run queue
Use when: System is slow, need root cause
```

**Interpreting %iowait (wa)**:
- < 5% = CPU-bound (CPU is the bottleneck)
- > 10% = I/O-bound (disk/network is the bottleneck, CPU is idle waiting)

#### **4. `ps aux --sort=-%cpu | head -6` — Top CPU-Consuming Processes**
```bash
$ ps aux --sort=-%cpu | head -6

USER    PID   %CPU  %MEM  COMMAND
root    1234  25.0   2.5  nginx: worker
root    1235  24.8   2.4  nginx: worker
root    1236  24.5   2.3  nginx: worker
root    1237  23.9   2.2  nginx: worker

Shows: Top 5 processes by CPU usage (relative to one core)
Purpose: Find which process is consuming CPU
Note: 25% = using 1 full core out of 4
```

#### **5. `free -h` — Memory Usage**
```bash
$ free -h

              total        used        free      shared  buff/cache
Mem:          15.9G        4.2G        8.5G       123M       3.2G
Swap:          4.0G          0B        4.0G

Shows: RAM and swap usage
Purpose: Quick memory check
Key: focus on "available" (free + buff/cache that can be freed)
```

#### **6. `top` — Real-Time Overview**
```bash
$ top

Shows: Per-process CPU %, load average, memory, etc.
Purpose: Quick visual check, identify CPU-hogging processes
⚠️  Problem: Aggregate CPU % is misleading for multi-core
Use: Combined with mpstat for real understanding
```

#### **7. `iostat -x 1` — CPU + Disk I/O**
```bash
$ iostat -x 1

avg-cpu:  %user  %nice %system %iowait  %idle
           40      0      5       10       45

Shows: CPU breakdown + disk I/O metrics
Purpose: See CPU breakdown AND disk performance
Use when: Checking if I/O is the real bottleneck
```

### Quick Decision Tree

```
Q: "Is the system slow?"
└─ Run: vmstat 1
   ├─ High %wa? → Disk/Network bottleneck
   ├─ High %us/%sy & low %wa? → CPU bottleneck
   └─ High r (run queue)? → CPU saturation

Q: "Which core is maxed?"
└─ Run: mpstat -P ALL 1
   ├─ One core at 100%? → Single-threaded app
   ├─ All cores high? → Genuinely saturated
   └─ Most cores idle? → Bad workload distribution

Q: "Which process is consuming CPU?"
└─ Run: ps aux --sort=-%cpu | head -6
   Then: Cross-check with mpstat to confirm real bottleneck

Q: "Is system saturated?"
└─ Run: uptime, then divide by nproc
   └─ > 0.75? → Getting saturated, scaling needed
```

---

## Understanding htop Output

### Top Section: System Overview

When you run `htop`, the first few lines show:

```
Tasks: 53 total,   4 running,  49 sleeping,   0 zombie,   0 stopped
Thr: 87 total,    68 kthr,    19 user threads
Mem[████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░]  4.2G / 15.9G
Swp[░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░]  0B / 4.0G

Uptime: 10 days, 2:15:32     Load average: 0.45 0.38 0.32
```

#### **Line 1: Tasks**

```
Tasks: 53 total,   4 running,  49 sleeping,   0 zombie,   0 stopped
```

- **53 total** = Total number of processes running on the system
- **4 running** = Processes actively using CPU right now (at this moment)
- **49 sleeping** = Processes waiting (blocked on I/O, waiting for input, etc.)
- **0 zombie** = Dead processes not yet cleaned up by parent (should be 0, non-zero is a bug)
- **0 stopped** = Processes paused (suspended, not running)

**What it means**: "53 things happening, but only 4 are actually using CPU right now. The rest are waiting."

---

#### **Line 2: Thr (Threads)**

```
Thr: 87 total,    68 kthr,    19 user threads
```

- **87 total** = Total number of threads across all processes
- **68 kthr** = Kernel threads (background kernel work: memory management, disk I/O, network)
- **19 user threads** = Threads spawned by applications

**Why threads > tasks**: A single process (like Java) can spawn many threads internally.

**High kernel threads (68) is normal**:
- Light workload: 50-100 kthr ✓
- Medium workload: 100-200 kthr ✓
- Heavy I/O: 200-500+ kthr

---

#### **Line 3: Mem (Memory)**

```
Mem[████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░]  4.2G / 15.9G
```

- **4.2G used** = Memory currently being used
- **15.9G total** = Total system RAM
- **Bar shows percentage** = Visual representation (████ = filled, ░░░░ = empty)

In this example: 4.2 / 15.9 = ~26% RAM usage

**Important**: This includes caching. Actual "consumed by apps" is lower than shown.

---

#### **Line 4: Swp (Swap)**

```
Swp[░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░]  0B / 4.0G
```

- **0B used** = Swap space currently being used
- **4.0G total** = Total swap allocated
- **Bar empty** = Good sign (not using swap)

**What it means**: System has 4GB swap available but isn't using it.

**Red flag**: If shows `1.2G / 4.0G`, system is swapping to disk (very slow, memory pressure high).

---

#### **Uptime & Load Average**

```
Uptime: 10 days, 2:15:32     Load average: 0.45 0.38 0.32
```

**Uptime**: How long system has been running since last reboot.
- Short (< 1 day) = Recent reboot
- Long (> 30 days) = Stable, no crashes

**Load average**: Three numbers = 1-minute, 5-minute, 15-minute
- **0.45** = Last 1 min: 0.45 processes waiting (on average)
- **0.38** = Last 5 min: 0.38 processes waiting
- **0.32** = Last 15 min: 0.32 processes waiting

**Interpretation** (on 4-core system):
- Load 0.45 / 4 = 11% utilization ✓ Healthy
- Trend: 0.45 → 0.38 → 0.32 = Decreasing (getting better)

---

### Real Example Reading

```
Tasks: 53 total,   4 running,  49 sleeping,   0 zombie,   0 stopped
Thr: 87 total,    68 kthr,    19 user threads
Mem[████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░]  4.2G / 15.9G
Swp[░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░]  0B / 4.0G

Uptime: 10 days, 2:15:32     Load average: 0.45 0.38 0.32
```

**Translation**:
> System has been running 10 days. 53 processes running (4 active now, 49 waiting). 26% RAM used, no swap. Load is light at 0.45. System is healthy.

---

### When Things Look Bad

```
Tasks: 250 total,   10 running,  200 sleeping,   10 zombie,   0 stopped
Thr: 450 total,    100 kthr,     350 user threads
Mem[████████████████████░░░░░░░░░░░░░░░░░░░░░░]  12.5G / 15.9G
Swp[██████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░]  2.1G / 4.0G

Uptime: 2 days, 4:32:15     Load average: 8.5 7.2 6.1
```

**Red flags**:
- ✗ 10 zombie processes (parent didn't clean them up)
- ✗ 79% RAM used (12.5/15.9)
- ✗ 52% swap used (2.1/4.0) ← System thrashing!
- ✗ Load 8.5 on 4-core = 212% overloaded
- ✗ Load increasing over time (upward trend)

**What's happening**: System overloaded, memory pressure high, processes being swapped (slow).

---

## Tools & Commands Reference

### Built-in Tools (Always Available)

| Tool | Command | Use Case |
|------|---------|----------|
| uptime | `uptime` | Quick load average check |
| vmstat | `vmstat 1` | System view with I/O wait, run queue |
| iostat | `iostat -x 1` | CPU + disk I/O metrics |
| mpstat | `mpstat -P ALL 1` | **Per-core utilization (CRITICAL)** |
| ps | `ps aux --sort=-%cpu \| head -6` | Top CPU-consuming processes |
| free | `free -h` | Memory usage |
| df | `df -h` | Disk space |
| top | `top` | Real-time interactive (use mpstat for per-core) |

### Recommended Optional Tools

| Tool | Installation | Use Case | Why Better |
|------|---|---|---|
| **htop** | `sudo apt install htop` | Real-time process monitoring | Better UI than top, per-core aware |
| **btop** | `sudo apt install btop` | Beautiful real-time monitoring | Stunning visuals, intuitive |
| **glances** | `sudo apt install glances` | All-in-one monitoring | Everything in one tool, web UI available |
| **dstat** | `sudo apt install dstat` | Colorized real-time stats | Easy to read, configurable columns |
| **iotop** | `sudo apt install iotop` | Disk I/O by process | Like htop for disk I/O |
| **nethogs** | `sudo apt install nethogs` | Network usage by process | Like htop for network |
| **nmon** | `sudo apt install nmon` | Interactive all-in-one | Toggle between CPU/memory/disk/network views |

### Investigation Workflow

```
System is slow?
│
├─ Quick check (30 seconds)
│  ├─ uptime → Check load / cores
│  ├─ vmstat 1 → Check %wa (I/O wait)
│  └─ mpstat -P ALL 1 → Check per-core utilization
│
├─ If CPU bottleneck
│  ├─ ps aux --sort=-%cpu | head -6 → Find CPU hog
│  └─ htop → Real-time investigation
│
├─ If I/O bottleneck
│  └─ iotop → Find I/O-intensive process
│
├─ If Network bottleneck
│  └─ nethogs → Find network-intensive process
│
└─ If everything looks fine
   └─ Might be application-level issue, check logs/metrics
```

### Commands Cheat Sheet

**Quick Checks (30 seconds)**:
```bash
uptime                           # Load average
mpstat -P ALL 1                  # Per-core utilization (RUN THIS!)
vmstat 1                         # System overview with I/O wait
ps aux --sort=-%cpu | head -6    # Top CPU processes
free -h                          # Memory
df -h                            # Disk space
```

**Interactive Monitoring**:
```bash
htop                             # Best balance (need: sudo apt install htop)
btop                             # Beautiful visuals (need: sudo apt install btop)
top                              # Built-in, but use htop instead
```

**Specialized Investigation**:
```bash
iotop                            # Disk I/O (needs root: sudo iotop)
nethogs                          # Network by process (needs root: sudo nethogs)
iostat -x 1                      # CPU + disk I/O details
```

---

## Common Misconceptions

### **Wrong #1: "CPU is at 30%, so I have 70% capacity"**
**Reality**: Depends on per-core utilization. Aggregate hides the truth.
- Single-threaded app might have 1 core maxed (100%), others idle
- Multi-threaded app at 30% might genuinely have capacity

**Check**: Use `mpstat -P ALL 1` to see per-core reality

### **Wrong #2: "Load 8.0 is high"**
**Reality**: Context matters. On 2-core system = 400% (CRISIS). On 16-core system = 50% (FINE).

**Formula**: Utilization = Load / #Cores

### **Wrong #3: "Swap only activates when RAM is 100% full"**
**Reality**: Swap starts around 70-85% RAM usage to prevent emergency at 100%.

**The trigger**: Kernel needs free memory → Moves idle pages to swap → Continues running smoothly

**If waited until 100%**: New process arrives → Nowhere to allocate → OOM killer starts killing processes

### **Wrong #4: "My app uses 4 cores at 25% each, so it's fine"**
**Reality**: Could mean two different things:
- Single-threaded app: 1 core maxed, 3 idle ← BOTTLENECKED
- Multi-threaded app: 4 cores at 25% = 100% efficient load ← OK

**Check**: Use `ps -eLf` or `top -H` to see threads

### **Wrong #5: "I need to max out CPU for good utilization"**
**Reality**: 100% CPU is a failure state, not a goal.
- 100% CPU → No headroom for spikes
- Traffic spike → Cascading failure
- Business impact: Downtime costs >> slightly higher hardware costs

**Correct goal**: 60-75% utilization at normal peak load

### **Wrong #6: "High context switches = bad"**
**Reality**: Context switches are symptoms, not the problem.
- Few switches + High latency = Bad (users waiting)
- Many switches + Low latency = Maybe OK (fair scheduling)
- Many switches + Low throughput = Very Bad (thrashing)

**Check**: `vmstat 1` shows context switches (cs column). < 1000/sec is normal baseline.

### **Wrong #7: "My load is 2.5, let me add more cores"**
**Reality**: Before adding hardware, diagnose the real problem:
- How many cores do I have? (Load 2.5 on 32-core = 8% usage)
- Is one core maxed? (Single-threaded bottleneck)
- Is it CPU or I/O? (Check %iowait in vmstat)
- Is it sustained or spike? (Check 15-min average in uptime)

---

## Real-World Troubleshooting Example

### Scenario: Users Report Slow Response Times

**Step 1: Quick system check**
```bash
$ uptime
load average: 3.8, 3.9, 3.9

# Saturated? 3.8 / 4 cores = 95% utilization → YES, bottlenecked
```

**Step 2: Is it CPU or I/O?**
```bash
$ vmstat 1
wa = 1%   → NOT I/O wait
us = 85%  → CPU is the problem
r = 3     → Run queue has 3 processes waiting
```

**Step 3: Which core is maxed?**
```bash
$ mpstat -P ALL 1
CPU 0: 98%  ← Maxed
CPU 1: 92%  ← Maxed
CPU 2: 85%  ← High
CPU 3: 88%  ← High

All cores high → Genuinely saturated (multi-threaded load)
```

**Step 4: Which process is it?**
```bash
$ ps aux --sort=-%cpu | head -6
nginx:   1234  24.5%  nginx: worker
nginx:   1235  25.0%  nginx: worker
nginx:   1236  24.8%  nginx: worker
nginx:   1237  23.7%  nginx: worker

Yes, nginx workers consuming all CPU
```

**Conclusion**: Nginx is saturated. Options:
- Add more servers (scale horizontally)
- Add more CPU cores (scale vertically)
- Optimize nginx config (fewer connections, better caching)

---

## Summary: Key Metrics to Monitor

| Metric | Command | What It Means | Healthy Threshold |
|--------|---------|---|---|
| Load Average | `uptime` | Processes waiting for CPU (divide by cores) | Load ≤ cores × 0.75 |
| Per-Core % | `mpstat -P ALL 1` | Individual core utilization | Max core < 80% |
| Run Queue | `vmstat 1` (r column) | Processes waiting for CPU | r ≤ #cores |
| I/O Wait % | `vmstat 1` (wa column) | CPU idle waiting for disk/network | wa < 5% |
| RAM Used | `free -h` | Memory currently in use | < 80% |
| Swap Used | `free -h` | Memory pages on disk (slow!) | 0B or < 500MB |
| CPU-hogging process | `ps aux --sort=-%cpu` | Which app is consuming CPU | Check context with mpstat |

**Golden Rules**:
1. Never trust aggregate CPU %. Always check per-core with `mpstat -P ALL 1`
2. Divide load by #cores to get real utilization percentage
3. Swap usage indicates memory pressure, not necessarily failure
4. System is healthy at 60-75% normal peak load, not at 100%

---

## Next Steps

- Learn disk I/O monitoring (iostat, iotop, detailed latency analysis)
- Master memory management (OOM killer, memory pressure, mmap behavior)
- Understand network bottlenecks (netstat, ss, iftop, packet loss)
- Explore observability at scale (Prometheus, Grafana, distributed tracing)
