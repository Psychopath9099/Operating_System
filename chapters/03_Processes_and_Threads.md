# Chapter 3 — Processes and Threads

> *"A process is a program in execution. A process is the unit of work in most systems."*
> — Silberschatz

---

## 3.1 What Is a Process?

A **process** is a running instance of a program. The same program can have multiple processes (open two terminals — two separate `bash` processes). Each process has its own:

- **Memory space** (private virtual address space)
- **File descriptor table** (open files, sockets)
- **Process ID (PID)**
- **State** (running, waiting, stopped, zombie)
- **Credentials** (UID, GID on Linux; SID on Windows)

### The Process Memory Layout

```
High Addresses ─────────────────────────────
              │   Command Line Args / Env   │
              │───────────────────────────── │
              │           Stack             │ ← grows downward
              │           ↓                 │   (local vars, return addrs)
              │                             │
              │           ↑                 │
              │          Heap               │ ← grows upward
              │───────────────────────────── │   (malloc'd memory)
              │   BSS (uninitialized data)  │
              │───────────────────────────── │
              │   Data (initialized data)   │
              │───────────────────────────── │
              │   Text (code segment)       │ ← read-only
Low Addresses ─────────────────────────────
```

This layout is critical for security — **buffer overflows**, **heap exploits**, and **stack smashing** all target specific regions here.

---

## 3.2 Process States

A process doesn't just run continuously. The OS moves it between states:

```
                 ┌──────────────────┐
                 │      New         │ ← Process created
                 └────────┬─────────┘
                          │ admitted
                          ▼
              ┌──────────────────────┐
         ┌────│       Ready          │◄────┐
         │    └──────────────────────┘     │
interrupt│        │ scheduler               │ I/O complete
or yield │        │ dispatch                │ or event
         │        ▼                         │
         │    ┌──────────────────────┐      │
         └───►│      Running         │──────┘
              └──────────┬───────────┘
                         │
                    I/O or event wait
                         │
                         ▼
              ┌──────────────────────┐
              │       Waiting        │
              └──────────────────────┘
                         │
                    process ends
                         ▼
              ┌──────────────────────┐
              │     Terminated       │
              └──────────────────────┘
```

### Linux Process States (from `ps` output)

| State Code | Meaning |
|---|---|
| R | Running or runnable (on CPU or ready queue) |
| S | Sleeping (interruptible — waiting for event) |
| D | Sleeping (uninterruptible — usually I/O) |
| T | Stopped (signal SIGSTOP or traced) |
| Z | Zombie (terminated, parent hasn't collected exit status) |

```bash
# See process states
ps aux | head -20

# More detailed process view
ps -eo pid,ppid,stat,cmd | head -30

# See a specific process
cat /proc/$$/status   # $$ = current shell PID
```

---

## 3.3 Process Creation

### Linux: fork() and exec()

Linux uses a two-step model for creating new processes:

**fork()** — creates an exact copy of the current process (child = copy of parent)  
**exec()** — replaces the current process image with a new program

```c
#include <unistd.h>
#include <stdio.h>

int main() {
    pid_t pid = fork();

    if (pid == 0) {
        // Child process
        printf("I am the child, PID: %d\n", getpid());
        execlp("ls", "ls", "-la", NULL);   // Replace child with 'ls'
    } else if (pid > 0) {
        // Parent process
        printf("I am the parent, child PID: %d\n", pid);
        wait(NULL);  // Wait for child to finish
    }
    return 0;
}
```

**Copy-on-Write (CoW):** When fork() creates a child, it doesn't immediately copy all parent memory. Both parent and child share the same physical pages (marked read-only). Only when one of them writes to a page does the OS create a private copy for that process. This makes fork() very fast.

### Windows: CreateProcess()

Windows doesn't separate fork and exec. `CreateProcess()` does both in one call — it creates a new process and loads a new program immediately.

```c
// Windows
STARTUPINFO si = { sizeof(si) };
PROCESS_INFORMATION pi;
CreateProcess(
    L"C:\\Windows\\System32\\notepad.exe",
    NULL, NULL, NULL, FALSE,
    CREATE_NEW_CONSOLE, NULL, NULL,
    &si, &pi
);
```

```powershell
# PowerShell equivalent
Start-Process "notepad.exe"

# Or with arguments
Start-Process "cmd.exe" -ArgumentList "/c dir C:\"
```

---

## 3.4 The Process Control Block (PCB)

The OS tracks every process using a data structure called the **Process Control Block** (PCB). In Linux, this is the `task_struct` — defined in `include/linux/sched.h`.

```c
// Simplified from Linux kernel task_struct
struct task_struct {
    volatile long   state;        // TASK_RUNNING, TASK_INTERRUPTIBLE, etc.
    void            *stack;       // Kernel stack pointer
    pid_t           pid;          // Process ID
    pid_t           tgid;         // Thread group ID
    struct task_struct *parent;   // Pointer to parent process
    struct list_head children;    // List of child processes
    struct mm_struct *mm;         // Memory descriptor
    struct files_struct *files;   // Open file descriptors
    const struct cred *cred;      // Credentials (UID, GID, capabilities)
    // ... hundreds more fields
};
```

```bash
# Read process info directly from the kernel
cat /proc/1/status      # init/systemd process
cat /proc/$$/maps       # current shell's memory mappings
ls /proc/$$/fd          # current shell's open file descriptors
```

---

## 3.5 Threads

A **thread** is a unit of execution within a process. Multiple threads share the same process address space but each has its own:
- **Stack**
- **Registers** (including Program Counter)
- **Thread ID (TID)**

```
Process (PID 1234)
┌─────────────────────────────────────┐
│           Shared Resources           │
│  Code | Data | Heap | Open Files    │
├──────────┬──────────┬───────────────┤
│ Thread 1 │ Thread 2 │   Thread 3    │
│ (Stack)  │ (Stack)  │   (Stack)     │
│ (Regs)   │ (Regs)   │   (Regs)      │
└──────────┴──────────┴───────────────┘
```

### Why Threads?

- **Responsiveness:** A UI thread stays responsive while a worker thread does heavy computation
- **Parallelism:** On multi-core CPUs, threads can run genuinely in parallel
- **Shared Memory:** Threads communicate through shared memory (no expensive IPC needed)
- **Lower overhead:** Creating a thread is cheaper than creating a process

### Linux Threads (pthreads)

Linux implements threads using the `clone()` syscall. Unlike many systems, Linux threads are represented as `task_struct` entries — the kernel sees threads and processes almost identically.

```c
#include <pthread.h>
#include <stdio.h>

void *worker(void *arg) {
    printf("Thread running, ID: %lu\n", pthread_self());
    return NULL;
}

int main() {
    pthread_t tid;
    pthread_create(&tid, NULL, worker, NULL);
    pthread_join(tid, NULL);  // Wait for thread
    return 0;
}
// Compile: gcc -pthread thread_example.c -o thread_example
```

```bash
# See threads of a process
ps -eLf | grep firefox      # -L shows threads

# htop shows threads per process (press H)
htop
```

### Windows Threads

Windows uses `CreateThread()` and the native `NtCreateThread()` below it. The Windows thread model is similar conceptually but has more OS-level context (fiber, UMS, etc.).

```powershell
# See threads for a process in PowerShell
Get-Process -Name "chrome" | Select-Object -Property Threads
```

---

## 3.6 Context Switching

When the OS switches from one process/thread to another, it performs a **context switch**:

1. Save the current process's CPU state (registers, PC, stack pointer) into its PCB
2. Load the next process's CPU state from its PCB
3. Switch to the new process's memory map (flush TLB)
4. Resume execution

```
Process A         Kernel              Process B
    │                │                    │
    │ (running)      │                    │
    │── interrupt ──►│                    │
    │                │ save A's state     │
    │                │                    │
    │                │ restore B's state  │
    │                │──────────────────►│
    │                │                    │ (running)
```

**Context switches are not free.** Each one involves:
- CPU register save/restore (~100 ns)
- TLB flush (expensive — forces memory remapping)
- Cache warming time (new process's data not in CPU cache)

---

## 3.7 Inter-Process Communication (IPC)

Processes need to talk to each other. The OS provides IPC mechanisms:

| Mechanism | Speed | Persistence | Best For |
|---|---|---|---|
| Pipes | Medium | No | Parent-child communication |
| Named Pipes (FIFO) | Medium | Filesystem entry | Unrelated processes |
| Shared Memory | **Fast** | No | High-throughput data sharing |
| Signals | Instant | No | Notifications, killing processes |
| Message Queues | Medium | System-managed | Producer-consumer patterns |
| Sockets (Unix) | Fast | No | Local service communication |
| D-Bus | Medium | No | Desktop service communication (Linux) |

```bash
# Pipes
ls -la | grep "^d"              # stdout of ls → stdin of grep

# Named pipe
mkfifo /tmp/mypipe
cat /tmp/mypipe &               # Reader in background
echo "hello" > /tmp/mypipe     # Writer

# Shared memory segments
ipcs -m                         # List shared memory segments
```

---

## 3.8 Signals (Linux)

Signals are the Linux kernel's way of sending asynchronous notifications to processes.

| Signal | Number | Default Action | Meaning |
|---|---|---|---|
| SIGHUP | 1 | Terminate | Terminal hangup / reload config |
| SIGINT | 2 | Terminate | Ctrl+C |
| SIGKILL | 9 | Terminate (can't catch) | Force kill |
| SIGSEGV | 11 | Core dump | Segmentation fault |
| SIGTERM | 15 | Terminate | Graceful shutdown |
| SIGUSR1 | 10 | Terminate | User-defined signal 1 |
| SIGSTOP | 19 | Stop (can't catch) | Pause process |

```bash
# Send signal to a process
kill -SIGTERM 1234      # Graceful
kill -9 1234            # Force kill

# Send SIGUSR1 to reload nginx config
sudo kill -HUP $(cat /run/nginx.pid)

# Catch signals in bash
trap 'echo "Got SIGTERM"' SIGTERM
```

---

## Summary

| Concept | Key Point |
|---|---|
| Process | Running program instance with its own memory space |
| Thread | Execution unit within a process, shares process memory |
| fork()/exec() | Linux two-step process creation |
| CreateProcess() | Windows single-step process+program creation |
| PCB / task_struct | Kernel's data structure tracking each process |
| Context switch | OS saves/restores CPU state to switch processes |
| CoW (fork) | Pages shared until written — makes fork() fast |
| IPC | Pipes, shared memory, signals, sockets for process comms |

---

## Key Terms

| Term | Definition |
|---|---|
| PID | Process Identifier — unique number per process |
| TID | Thread Identifier |
| PCB | Process Control Block — kernel's process data |
| task_struct | Linux's PCB implementation |
| fork() | Linux syscall to clone a process |
| exec() | Linux syscall to replace process image with new program |
| CoW | Copy-on-Write — defer memory copy until write |
| Zombie | Process that finished but parent hasn't collected exit code |
| Signal | Asynchronous notification sent to a process |
| IPC | Inter-Process Communication |
| Context switch | Saving one process state and restoring another |

---

## 🔒 Security Lens

**Process injection — abusing process relationships:**
Process injection techniques exploit the trust relationships and shared APIs between processes. Common attack patterns:

- **DLL Injection** — Write a malicious DLL path into a target process's memory, then force it to load via `CreateRemoteThread()` + `LoadLibrary()`
- **Process Hollowing** — Create a legitimate process (like `svchost.exe`), unmap its memory, inject malicious code, then resume
- **ptrace injection (Linux)** — Use `ptrace()` to pause a process, write shellcode into its memory, and redirect execution

```bash
# On Linux: check if a process is being traced
cat /proc/1234/status | grep TracerPid
# Non-zero = process is being traced (could be debugger or attack)
```

**Fork bombs:**
```bash
# A fork bomb — crashes system by exhausting process table
# DO NOT RUN ON ANY REAL SYSTEM
# :(){ :|:& };:
# Prevent with ulimit:
ulimit -u 100    # Max 100 processes for this shell session
```

**Credential handling in lsass.exe (Windows):**
`lsass.exe` is a process like any other, but it holds credentials in memory. `MiniDumpWriteDump()` can dump its memory. Defense: Protected Process Light (PPL) makes lsass protected — only processes with an equally high integrity level can open it.

**CVE Highlight — CVE-2016-5195 (Dirty COW):**
A race condition in Linux's Copy-on-Write implementation. An attacker could write to any read-only file by winning a race condition during the CoW page fault handling. This allowed privilege escalation to root. Fixed in Linux 4.8.3. Classic example of a mechanism bug in the process memory model.

**Zombie processes as a detection indicator:**
A large number of zombie processes can indicate:
1. A parent process that's failing to call `wait()`
2. Malware that's creating and abandoning processes to evade monitoring
```bash
# Count zombie processes
ps aux | awk '{print $8}' | grep -c 'Z'
```

---

**[← Chapter 2](02_OS_Structures_and_Design.md)** | **[→ Chapter 4: CPU Scheduling](04_CPU_Scheduling.md)**
