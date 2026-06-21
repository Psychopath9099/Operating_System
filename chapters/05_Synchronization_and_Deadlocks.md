# Chapter 5 — Synchronization and Deadlocks

> *"A race condition occurs when several processes access and manipulate the same data concurrently and the outcome depends on the particular order in which the access takes place."*
> — Silberschatz

---

## 5.1 The Concurrency Problem

When multiple threads or processes access shared data, results become unpredictable if accesses aren't controlled. This isn't a bug in one program — it's a fundamental property of concurrent execution.

### The Classic Race Condition

```c
// Two threads, both doing: counter++

// Thread 1              Thread 2
// READ counter (= 5)
//                       READ counter (= 5)
// counter = 5 + 1 = 6
//                       counter = 5 + 1 = 6  ← Wrong! Should be 7
// WRITE counter = 6
//                       WRITE counter = 6    ← Lost update
```

The problem: `counter++` is **not atomic**. It's three operations: read, increment, write. The scheduler can switch between them.

---

## 5.2 The Critical Section Problem

A **critical section** is a segment of code that accesses shared data. We need to ensure only one thread executes the critical section at a time.

```
Entry section   ─── Request permission to enter
Critical section ── Access shared resource
Exit section    ─── Release permission
Remainder section ─ Everything else
```

Any correct solution must satisfy:

**Mutual Exclusion:** At most one process in its critical section at a time.  
**Progress:** If no process is in its critical section, one of the waiting processes must be allowed in (no deadlock on entry).  
**Bounded Waiting:** A process won't wait forever (no starvation).

---

## 5.3 Mutex Locks

The simplest synchronization primitive. A **mutex** (mutual exclusion lock) can be in two states: locked or unlocked.

```c
#include <pthread.h>

pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
int counter = 0;

void *increment(void *arg) {
    for (int i = 0; i < 1000000; i++) {
        pthread_mutex_lock(&lock);    // Enter critical section
        counter++;                    // Safe now
        pthread_mutex_unlock(&lock);  // Exit critical section
    }
    return NULL;
}
```

**Spinlock vs. Sleeping mutex:**

A **spinlock** keeps checking ("spinning") in a busy loop until the lock is free. Fast for short waits, wastes CPU for long ones.

A **sleeping mutex** puts the thread to sleep and wakes it when the lock is available. Better for long waits, has overhead for short ones.

```bash
# Linux kernel uses both depending on context
# User space: pthreads mutex (sleeping)
# Kernel: spinlocks (for interrupt-safe code), mutexes (sleepable kernel code)
```

---

## 5.4 Semaphores

A **semaphore** is a generalized synchronization tool — essentially a counter with atomic increment/decrement operations.

**Binary semaphore (= mutex):** value is 0 or 1  
**Counting semaphore:** value ≥ 0, controls access to N instances of a resource

```
wait(S):   while S == 0: sleep;  S--
signal(S): S++;  wake up one sleeping process
```

```c
#include <semaphore.h>

sem_t sem;
sem_init(&sem, 0, 1);  // Binary semaphore

// Thread entry
sem_wait(&sem);         // wait() — decrement (blocks if 0)
// critical section
sem_post(&sem);         // signal() — increment
```

### Semaphore Use Cases

```c
// Producer-Consumer with counting semaphores
sem_t empty_slots;  // initialized to BUFFER_SIZE
sem_t full_slots;   // initialized to 0
sem_t mutex;        // initialized to 1

// Producer
sem_wait(&empty_slots);  // Wait for empty slot
sem_wait(&mutex);
// add item to buffer
sem_post(&mutex);
sem_post(&full_slots);   // Signal a new item

// Consumer
sem_wait(&full_slots);   // Wait for item
sem_wait(&mutex);
// remove item from buffer
sem_post(&mutex);
sem_post(&empty_slots);  // Signal freed slot
```

---

## 5.5 Monitors and Condition Variables

Mutexes and semaphores are low-level and error-prone. **Monitors** are a higher-level abstraction. In C/POSIX, monitors are implemented as **mutex + condition variable** pairs.

```c
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t  cond  = PTHREAD_COND_INITIALIZER;
int data_ready = 0;

// Waiter thread
pthread_mutex_lock(&mutex);
while (!data_ready) {
    pthread_cond_wait(&cond, &mutex);  // Atomically release mutex and sleep
}
// process data
pthread_mutex_unlock(&mutex);

// Signaler thread
pthread_mutex_lock(&mutex);
data_ready = 1;
pthread_cond_signal(&cond);   // Wake up one waiter
pthread_mutex_unlock(&mutex);
```

---

## 5.6 Modern Atomics

For simple counters and flags, heavy-weight mutexes are overkill. Modern CPUs provide **atomic instructions** that guarantee read-modify-write happens without interruption.

```c
#include <stdatomic.h>

atomic_int counter = 0;

// These are guaranteed atomic — no mutex needed
atomic_fetch_add(&counter, 1);       // counter++
atomic_load(&counter);               // read
atomic_store(&counter, 42);          // write
atomic_compare_exchange_strong(...); // CAS — Compare and Swap
```

**Compare-and-Swap (CAS)** is the foundation of lock-free programming:
```
CAS(variable, expected, new_value):
    if variable == expected:
        variable = new_value
        return true
    return false
```

Used extensively in kernel data structures for performance.

---

## 5.7 Deadlocks

A **deadlock** occurs when a set of processes are each waiting for resources held by others in the set — a circular wait.

### The Classic Example

```
Process 1 holds: Lock A       Wants: Lock B
Process 2 holds: Lock B       Wants: Lock A

P1 ──── waits for ────► Lock B ◄──── holds ──── P2
P1 ──── holds ─────► Lock A ◄──── waits for ── P2
                    DEADLOCK
```

### Four Necessary Conditions (Coffman Conditions)

All four must hold simultaneously for deadlock:

1. **Mutual Exclusion** — Resources can't be shared (only one process at a time)
2. **Hold and Wait** — Processes hold resources while waiting for more
3. **No Preemption** — Resources can't be forcibly taken from a process
4. **Circular Wait** — P1 waits for P2, P2 waits for P3, P3 waits for P1

Break any one of these → no deadlock possible.

---

## 5.8 Deadlock Prevention and Avoidance

### Prevention (Break a Coffman Condition)

**Break Hold and Wait:**
Require processes to acquire all resources before starting. Or: release all resources before requesting new ones.

**Break Circular Wait:**
Impose a total ordering on resource types. Always acquire resources in order. If you need Lock A and Lock B, always acquire A before B — circular wait becomes impossible.

**Break No Preemption:**
If a process can't get what it needs, release what it holds and try again later. Works for some resources (CPU, memory) but not others (printers, exclusive files).

### Avoidance — Banker's Algorithm

The **Banker's Algorithm** (Dijkstra) checks before granting any resource request whether the resulting state is **safe** (a sequence exists where all processes can eventually complete).

```
Safe state: ∃ sequence [P1, P2, ..., Pn] such that each Pi can
            obtain its remaining resources from current available +
            resources held by all Pj where j < i
```

Not widely used in real OSes (too much overhead, need to know max resource demands in advance) but the concept underpins resource management thinking.

---

## 5.9 Deadlock Detection and Recovery

Many real systems (including Linux) don't prevent or avoid deadlocks — they just let them happen and deal with them:

**Detection:**
- Maintain a resource allocation graph
- Look for cycles in the graph

```bash
# Linux: kernel detects some deadlocks
# lockdep — kernel lock dependency validator
# Enable in kernel config: CONFIG_PROVE_LOCKING=y
# Kernel will dump a warning when potential deadlock is detected
dmesg | grep -i deadlock
dmesg | grep -i "possible circular locking"
```

**Recovery:**
- **Kill a process:** Choose a victim to kill (minimize cost, rollback)
- **Preempt resources:** Roll back processes to a safe state (checkpoint/rollback)

---

## 5.10 Linux Kernel Synchronization Primitives

The kernel has its own set of synchronization tools (different from user space):

| Primitive | When Used |
|---|---|
| `spinlock_t` | Short critical sections, interrupt context (can't sleep) |
| `mutex` | Longer critical sections, process context (can sleep) |
| `rwlock_t` | Multiple readers OR one writer |
| `rcu` (Read-Copy-Update) | Read-heavy workloads, near-zero read overhead |
| `seqlock` | Read-heavy, writer shouldn't block readers |
| `atomic_t` | Simple counters, flags |

**RCU (Read-Copy-Update)** is particularly elegant — readers access data without any locking at all. Writers make a copy, update it, then atomically swap the pointer. Old data is freed only when no readers are using it.

---

## Summary

| Concept | Key Point |
|---|---|
| Race condition | Non-deterministic outcome from unsynchronized shared access |
| Critical section | Code region accessing shared data |
| Mutex | Binary lock for mutual exclusion |
| Semaphore | Counter-based synchronization, controls N resources |
| Condition variable | Thread sleeps waiting for a condition, mutex must be held |
| Atomic operations | Hardware-guaranteed read-modify-write, no mutex needed |
| Deadlock | Circular wait between processes holding resources |
| Coffman conditions | 4 conditions that must all hold for deadlock |
| Banker's algorithm | Deadlock avoidance via safe-state checking |

---

## Key Terms

| Term | Definition |
|---|---|
| Race condition | Bug where outcome depends on execution order |
| Mutual exclusion | At most one thread in critical section at a time |
| Spinlock | Busy-wait lock — spins until available |
| Semaphore | Integer + wait/signal operations |
| Monitor | Mutex + condition variable combined abstraction |
| CAS | Compare-and-Swap — foundation of lock-free algorithms |
| Deadlock | Processes blocked in circular resource dependency |
| Livelock | Processes keep changing state but no progress |
| Starvation | A process never gets the resource it's waiting for |
| RCU | Read-Copy-Update — lock-free reads in kernel |
| lockdep | Linux kernel deadlock detection tool |

---

## 🔒 Security Lens

**Race conditions as vulnerabilities:**
Race conditions in security-sensitive code are a major vulnerability class:

- **TOCTOU (Time of Check to Time of Use):** Check happens at time T1, use happens at time T2, attacker changes conditions in between
  ```c
  // VULNERABLE pattern
  if (access("file.txt", R_OK) == 0) {   // check: is readable?
      // attacker swaps file.txt with /etc/shadow here
      fd = open("file.txt", O_RDONLY);    // use: now opens shadow!
  }
  ```

- **CVE-2016-5195 (Dirty COW):** Linux CoW race condition — write to read-only files by winning a race in the kernel's page fault handler

- **Double-Fetch vulnerabilities:** Kernel reads user-space value twice. Attacker changes it between reads. Kernel uses inconsistent values.

**Mutex misuse → denial of service:**
A process that acquires a mutex and then crashes (or is killed) while holding it can leave the mutex permanently locked, causing other processes to block forever. POSIX provides **robust mutexes** that detect owner death.

```c
// Robust mutex — detects if owner dies while holding lock
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_setrobust(&attr, PTHREAD_MUTEX_ROBUST);
pthread_mutex_init(&mutex, &attr);
```

**Deadlock-as-DoS:**
Malicious or buggy kernel modules can introduce deadlocks in kernel lock sequences, causing a system hang. The `lockdep` tool in the Linux kernel is essential for catching these during development.

**Semaphore exhaustion:**
POSIX semaphores are a limited system resource. A process creating many semaphores without cleaning them up can exhaust the limit, causing a DoS.

```bash
# Check semaphore limits
ipcs -ls
cat /proc/sys/kernel/sem
```

---

**[← Chapter 4](04_CPU_Scheduling.md)** | **[→ Chapter 6: Memory Management](06_Memory_Management.md)**
