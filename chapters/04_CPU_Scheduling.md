# Chapter 4 — CPU Scheduling

> *"CPU scheduling is the basis of multiprogrammed operating systems. By switching the CPU among processes, the OS can make the computer more productive."*
> — Silberschatz

---

## 4.1 Why Scheduling Exists

A modern system runs hundreds of processes but has only a few CPU cores. The **CPU scheduler** decides which process runs next on a given core, for how long, and in what order.

Without scheduling:
- One process could hog the CPU forever
- Interactive applications would feel frozen
- I/O-bound and CPU-bound processes would fight unfairly

The scheduler runs every few milliseconds — it is one of the most performance-critical pieces of the OS.

---

## 4.2 Scheduling Goals (and Tensions)

No algorithm satisfies all goals simultaneously. Every scheduler makes trade-offs:

| Goal | Meaning | Who cares |
|---|---|---|
| **CPU Utilization** | Keep CPU as busy as possible | Server operators |
| **Throughput** | Max jobs completed per second | Batch systems |
| **Turnaround time** | Time from submit to completion | Batch users |
| **Waiting time** | Time spent in ready queue | All users |
| **Response time** | Time to first response | Interactive users |
| **Fairness** | Each process gets fair share | Shared systems |
| **Deadline** | Meet hard timing requirements | Real-time systems |

Linux's CFS prioritizes **fairness and interactive response**. Windows prioritizes **responsiveness for foreground processes**.

---

## 4.3 Classic Scheduling Algorithms

These are the building blocks — modern schedulers combine ideas from all of them.

### First-Come First-Served (FCFS)
Simple queue. Run processes in arrival order until done.

```
Process  Arrival  Burst
  P1       0       10ms
  P2       1        5ms
  P3       2        3ms

Timeline: [P1=0-10] [P2=10-15] [P3=15-18]

Average waiting time: (0 + 9 + 13) / 3 = 7.3ms
```

**Problem: Convoy effect** — one long process starves all short ones.

### Shortest Job First (SJF)
Run the process with the shortest estimated burst time next.

```
With same processes above, run P3 first (if we know burst times):
Timeline: [P3=0-3] [P2=3-8] [P1=8-18]

Average waiting time: (8 + 2 + 0) / 3 = 3.3ms  ← much better
```

**Problem:** We can't know actual burst time in advance. Use exponential averaging to predict it.

### Round Robin (RR)
Each process runs for a fixed time **quantum** (time slice), then goes to the back of the queue.

```
Quantum = 4ms
Timeline: [P1=0-4] [P2=4-8] [P3=8-11] [P1=11-15] [P2=15-16]
```

**Trade-off:** Small quantum = more responsive but more context switch overhead. Large quantum = approaches FCFS.

### Priority Scheduling
Each process has a priority number. Higher priority runs first.

**Problem: Starvation** — low-priority processes might never run.  
**Solution: Aging** — increase priority of waiting processes over time.

### Multilevel Queue
Separate ready queues for different process categories (foreground/interactive, background/batch). Each queue can have its own algorithm.

---

## 4.4 Linux CFS — Completely Fair Scheduler

Linux (since kernel 2.6.23) uses **CFS** as the default scheduler. It doesn't use fixed time slices. Instead, it tracks how much CPU time each process has had and always runs the process that has had the **least** CPU time (most "deserving").

### The Virtual Runtime (vruntime)

CFS maintains a **virtual runtime (vruntime)** per process — a measure of how much CPU time it has consumed, normalized by its weight (nice value).

```
vruntime increases while running
vruntime does NOT increase while sleeping
```

The scheduler picks the process with the **smallest vruntime** — stored in a red-black tree for O(log N) access.

```
          Red-Black Tree (sorted by vruntime)
          
                    [P3: vr=10]
                   /            \
          [P1: vr=5]            [P5: vr=20]
          /         \
    [P2: vr=2]   [P4: vr=8]
    
    ← Leftmost node (P2) runs next
```

### Nice Values and Weights

Linux processes have a **nice value** from -20 (highest priority) to +19 (lowest priority).

```bash
# Run a process with lower priority
nice -n 10 ./cpu_intensive_task

# Change priority of running process
renice -n 5 -p 1234

# See nice values
ps -eo pid,ni,cmd | head -20
top   # NI column
```

| Nice Value | Relative Weight | CPU share (2 processes) |
|---|---|---|
| -20 | 88761 | ~97% |
| 0 | 1024 | 50% |
| +19 | 15 | ~1.5% |

### Scheduler Classes

Linux has multiple scheduler classes stacked in priority order:

```
stop_sched_class      ← Highest (migration, watchdog)
dl_sched_class        ← Deadline tasks (SCHED_DEADLINE)
rt_sched_class        ← Real-time (SCHED_FIFO, SCHED_RR)
fair_sched_class      ← CFS (SCHED_OTHER, SCHED_BATCH) ← default
idle_sched_class      ← Lowest (runs when nothing else wants CPU)
```

```bash
# See scheduling policy of a process
chrt -p 1234

# Run process with real-time FIFO policy (requires root)
sudo chrt --fifo 50 ./realtime_task

# Set deadline scheduling
sudo chrt --deadline --sched-runtime 5000000 \
         --sched-deadline 10000000 \
         --sched-period 10000000 ./task
```

---

## 4.5 Windows Scheduler

Windows uses a **priority-based preemptive scheduler** with 32 priority levels (0–31).

```
Priority Levels:
31  ← Realtime (reserved for kernel, time-critical apps)
...
16
15  ← High (above normal processes)
...
 8  ← Normal (default for most apps)
...
 1  ← Idle (runs only when nothing else wants CPU)
 0  ← Reserved (zero page thread)
```

### Priority Boost (Windows)

Windows dynamically **boosts priorities** for:
- Foreground window processes (UI responsiveness)
- Processes waiting for I/O (gets a boost after I/O completes)
- Processes that have waited too long (starvation prevention)

```powershell
# See process priorities
Get-Process | Select-Object Name, PriorityClass | Sort-Object PriorityClass -Descending

# Set process priority
$proc = Get-Process -Name "notepad"
$proc.PriorityClass = [System.Diagnostics.ProcessPriorityClass]::AboveNormal
```

---

## 4.6 Multiprocessor Scheduling

Modern systems have multiple cores. The scheduler must handle:

**Load Balancing:** Move threads between CPUs to keep all cores busy.

**CPU Affinity:** Bind a process/thread to specific CPU(s) for cache warmth.

**NUMA Awareness:** On NUMA systems (Non-Uniform Memory Access), it's faster to run a process on the CPU closest to its memory.

```bash
# Linux: bind a process to CPU 0 and 1
taskset -c 0,1 ./my_program

# Set affinity of running process
taskset -cp 0,1 1234

# NUMA topology
numactl --hardware

# Run on NUMA node 0
numactl --cpunodebind=0 --membind=0 ./my_program
```

```powershell
# Windows: set processor affinity
$proc = Get-Process notepad
$proc.ProcessorAffinity = 3   # Binary 11 = CPUs 0 and 1
```

---

## 4.7 Real-Time Scheduling

Real-time systems need **guaranteed** response times, not just fast average response.

**Hard real-time:** Missing a deadline is a system failure (airbags, pacemakers).  
**Soft real-time:** Missing a deadline degrades quality but doesn't fail (video streaming).

Linux supports real-time via:
- `SCHED_FIFO` — fixed priority, no preemption by same/lower priority
- `SCHED_RR` — round-robin among same-priority real-time tasks
- `SCHED_DEADLINE` — EDF-based, guarantees runtime within deadline

---

## 4.8 Observing the Scheduler

```bash
# CPU time breakdown per process
top          # real-time view
htop         # better view

# Scheduler statistics
cat /proc/schedstat         # global scheduler stats
cat /proc/1234/schedstat    # per-process stats

# Context switch rate
vmstat 1 5                  # cs column = context switches/sec

# CPU usage by core
mpstat -P ALL 1 5

# perf — the gold standard for scheduler profiling
sudo perf stat -e context-switches,migrations ./my_program
sudo perf record -g ./my_program && sudo perf report
```

---

## Summary

| Concept | Key Point |
|---|---|
| Scheduling goals | Utilization, throughput, response time, fairness |
| FCFS | Simple, convoy effect is the problem |
| SJF | Optimal but requires burst time knowledge |
| Round Robin | Fair, quantum size is the key trade-off |
| Linux CFS | vruntime-based fairness, red-black tree, O(log N) |
| Nice values | -20 to +19, controls weight in CFS |
| Windows scheduler | 32 priority levels + dynamic priority boost |
| CPU affinity | Binding processes to cores for cache performance |

---

## Key Terms

| Term | Definition |
|---|---|
| CPU burst | Period during which a process actively uses CPU |
| Time quantum | Fixed time slice in Round Robin |
| vruntime | Virtual runtime — CFS's fairness metric |
| Nice value | Linux process priority hint (-20 to +19) |
| Preemption | Forcibly removing a process from CPU before it finishes |
| Context switch | Saving one process state, restoring another |
| Load balancing | Distributing work across multiple CPUs |
| CPU affinity | Binding a thread to specific CPU cores |
| NUMA | Non-Uniform Memory Access — memory closer to some CPUs |
| Real-time | Tasks with hard timing guarantees |

---

## 🔒 Security Lens

**Scheduler as a side channel:**
The scheduler's timing decisions can leak information. **Flush+Reload** and **Spectre** attacks exploit CPU timing to read memory across process boundaries. These aren't scheduler bugs per se — they abuse the CPU's speculative execution, but the scheduler determines *when* attacker code runs relative to victim code.

**CPU resource exhaustion (DoS):**
An unprivileged attacker spawning many CPU-intensive processes can degrade system performance. Mitigations:
```bash
# Limit CPU usage of a cgroup (Linux)
# Create a cgroup and limit to 20% of one CPU
mkdir /sys/fs/cgroup/cpu/limited_group
echo 200000 > /sys/fs/cgroup/cpu/limited_group/cpu.cfs_quota_us
echo 1000000 > /sys/fs/cgroup/cpu/limited_group/cpu.cfs_period_us
echo $PID > /sys/fs/cgroup/cpu/limited_group/cgroup.procs
```

**Priority inversion — a real-world safety issue:**
In 1997, NASA's Mars Pathfinder mission experienced system resets due to **priority inversion**:
- A high-priority task waited for a resource held by a low-priority task
- A medium-priority task preempted the low-priority task
- The high-priority task starved
- Solution: **Priority inheritance** — temporarily boost the priority of the lock holder

**Process priority as persistence:**
Malware sometimes sets itself to high CPU priority to ensure it keeps running even when the system is under load. Monitoring for unexpected high-priority processes is a valid detection technique.

```bash
# Find high-priority real-time processes (suspicious if unexpected)
ps -eo pid,ni,cls,cmd | grep -E 'RR|FF'
# RR = SCHED_RR, FF = SCHED_FIFO
```

---

**[← Chapter 3](03_Processes_and_Threads.md)** | **[→ Chapter 5: Synchronization and Deadlocks](05_Synchronization_and_Deadlocks.md)**
