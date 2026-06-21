# Chapter 2 — OS Structures and Design

> *"The structure of an OS reflects the design decisions made about the separation of concerns, the interfaces between components, and the mechanisms for protection."*

---

## 2.1 How an OS Is Organized

An OS is not a single program — it's a layered architecture of subsystems. Understanding this structure tells you where bugs live, where security controls are enforced, and where attackers look for weaknesses.

### The Layered Model (Conceptual)

```
┌─────────────────────────────────────────────┐
│  Layer 6  │       User Processes            │
├───────────┴─────────────────────────────────┤
│  Layer 5  │    System Libraries (libc)      │
├─────────────────────────────────────────────┤
│  Layer 4  │   System Call Interface         │
├─────────────────────────────────────────────┤
│  Layer 3  │   OS Services (VFS, scheduler)  │
├─────────────────────────────────────────────┤
│  Layer 2  │   Device Drivers                │
├─────────────────────────────────────────────┤
│  Layer 1  │   Hardware Abstraction Layer    │
├─────────────────────────────────────────────┤
│  Layer 0  │   Physical Hardware             │
└─────────────────────────────────────────────┘
```

In practice, operating systems don't follow a clean layered model. Linux's architecture is better described as a **set of subsystems** that interact through well-defined interfaces.

---

## 2.2 Linux Kernel Architecture (Modern)

The Linux kernel (as of 6.x) is organized into major subsystems:

```
┌──────────────────────────────────────────────────────────┐
│                    Linux Kernel 6.x                       │
├──────────┬──────────┬───────────┬────────────┬───────────┤
│ Process  │  Memory  │ Virtual   │  Network   │  Device   │
│ Scheduler│  Mgmt    │ File Sys  │  Stack     │  Drivers  │
│          │  (MM)    │  (VFS)    │  (netif)   │           │
├──────────┴──────────┴───────────┴────────────┴───────────┤
│              Arch-Specific Code (x86_64, ARM, etc.)       │
├──────────────────────────────────────────────────────────┤
│                System Call Interface (entry.S)            │
└──────────────────────────────────────────────────────────┘
```

### Key Subsystems:

**Process Scheduler (kernel/sched/)**
Decides which process runs next. Linux uses the **Completely Fair Scheduler (CFS)** as the default. For real-time tasks, there's `SCHED_FIFO` and `SCHED_RR`.

**Memory Management (mm/)**
Handles virtual memory, page allocation, the buddy allocator, slab/SLUB allocators, and OOM (Out of Memory) killing.

**Virtual File System (VFS) (fs/)**
An abstraction layer that lets Linux support dozens of file systems (ext4, Btrfs, XFS, FAT32, tmpfs) under one uniform interface.

**Network Stack (net/)**
Full TCP/IP implementation, Netfilter (iptables/nftables hooks), socket layer.

**Device Drivers (drivers/)**
The largest part of the kernel source. Hardware drivers for everything from USB to GPU.

### Viewing Kernel Info on Your System
```bash
# Kernel version
uname -r

# Loaded kernel modules
lsmod

# Kernel messages (dmesg)
dmesg | tail -50

# Kernel config (if available)
zcat /proc/config.gz | grep CONFIG_SECURITY
```

---

## 2.3 Windows NT Architecture (Modern)

Windows uses a **layered hybrid architecture** based on the NT kernel. The design separates user mode from kernel mode clearly, but places many high-level services in kernel space for performance.

```
User Mode:
┌─────────────────────────────────────────────┐
│  Win32 Apps │  .NET Apps │  UWP Apps        │
├─────────────────────────────────────────────┤
│  Subsystem DLLs (kernel32.dll, ntdll.dll)   │
├─────────────────────────────────────────────┤
│  Subsystem Processes (csrss.exe, lsass.exe) │
└─────────────────────────────────────────────┘
         ↕  System Call Boundary (ntdll → ntoskrnl)
Kernel Mode:
┌─────────────────────────────────────────────┐
│         NT Executive                         │
│  (Object Manager, I/O, Memory, Security)    │
├─────────────────────────────────────────────┤
│         NT Kernel (ntoskrnl.exe)             │
│   (Thread scheduling, IRQ handling)          │
├─────────────────────────────────────────────┤
│         HAL (hal.dll)                        │
│   (Hardware Abstraction Layer)               │
├─────────────────────────────────────────────┤
│         Hardware                             │
└─────────────────────────────────────────────┘
```

### Key Windows Components:

**ntoskrnl.exe** — The Windows kernel. Contains the NT Executive + NT Kernel layer.

**hal.dll** — Hardware Abstraction Layer. Abstracts platform-specific hardware details.

**ntdll.dll** — The bridge between user mode and kernel mode. All syscalls go through here (unless malware bypasses it with direct syscalls).

**csrss.exe** — Client/Server Runtime SubSystem. Manages the Win32 console and process/thread management.

**lsass.exe** — Local Security Authority SubSystem. Handles authentication. **This is why it's a prime target for credential theft tools like Mimikatz.**

---

## 2.4 OS Interfaces — APIs, ABIs, and Syscalls

### API (Application Programming Interface)
The set of functions exposed by a library or OS to developers. On Linux: POSIX API. On Windows: Win32 API.

```c
// POSIX API (Linux/macOS)
fd = open("file.txt", O_RDONLY);
bytes = read(fd, buffer, 1024);

// Win32 API (Windows)
HANDLE hFile = CreateFile(L"file.txt", GENERIC_READ, ...);
ReadFile(hFile, buffer, 1024, &bytesRead, NULL);
```

### ABI (Application Binary Interface)
How compiled code interacts with the OS at the binary level — calling conventions, register usage, system call numbers. This is why you can't run a Linux ELF binary on Windows without translation.

### Syscall Numbers
Every syscall has a number. On Linux x86_64:
```
sys_read  = 0
sys_write = 1
sys_open  = 2
sys_close = 3
```

```bash
# See system call table on Linux
ausyscall --dump | head -20

# Or directly
grep 'define __NR_' /usr/include/asm/unistd_64.h | head -20
```

---

## 2.5 Policy vs. Mechanism

This is one of the most important design principles in OS theory (Silberschatz emphasizes this heavily):

**Mechanism:** *How* to do something  
**Policy:** *What* to do (and when, and for whom)

**Example — CPU Scheduling:**
- Mechanism: Timer interrupts, context switches, run queues
- Policy: Which algorithm? Priority? Fairness? Real-time guarantees?

**Example — File Access:**
- Mechanism: Permission bits, ACLs, MAC labels
- Policy: Who owns this file? Who should be able to read it?

The separation of policy from mechanism is what allows Linux to be reconfigured (different schedulers, different security policies via LSMs) without changing the core mechanism code.

> **Security Lens:** When an OS hardening guide tells you to enable SELinux or AppArmor, they're talking about adding a new *policy layer* on top of existing *mechanisms*. The mechanism (LSM hooks in the kernel) is always there. The policy is what defines the actual rules.

---

## 2.6 OS Design Goals

Different OS designs optimize for different goals. Understanding this explains design decisions that might otherwise seem arbitrary.

| Goal | Linux Focus | Windows Focus |
|---|---|---|
| Performance | High (especially server) | Good (desktop + server) |
| Stability | Critical | Critical |
| Security | Configurable (SELinux, etc.) | Integrated (Defender, Smart App Control) |
| Compatibility | POSIX standard | Win32 backward compat |
| Developer friendliness | Open source, hackable | SDK, tooling-rich |
| Hardware support | Community-driven | OEM partnerships |

---

## 2.7 User Space vs. Kernel Space — The Hard Boundary

This is worth repeating and going deeper:

**Kernel space:**
- Direct hardware access
- No memory protection between kernel components
- A crash here crashes the entire system
- Code runs at ring 0

**User space:**
- Restricted hardware access (must use syscalls)
- Memory protected by the kernel (process isolation)
- A crash here only kills that process
- Code runs at ring 3

```bash
# On Linux: see what's in kernel space right now
cat /proc/modules          # loaded kernel modules
cat /proc/kallsyms         # kernel symbol table (requires root or kptr_restrict=0)
cat /proc/iomem            # memory-mapped I/O regions
```

---

## 2.8 Modern Design Additions — eBPF (Linux)

One of the biggest changes to Linux in recent years is **eBPF (extended Berkeley Packet Filter)**. Originally for network packet filtering, it's now a general-purpose kernel extension mechanism.

eBPF lets you run sandboxed programs inside the kernel without writing a kernel module. The kernel's verifier checks them for safety before running.

```
User Space                    Kernel Space
    │                              │
    │  load eBPF program           │
    │ ──────────────────────────>  │
    │                     ┌────────┤
    │                     │ Verify │ ← Safety check
    │                     └────────┤
    │                     ┌────────┤
    │                     │  JIT  │ ← Compile to native
    │                     └────────┤
    │                     ┌────────┤
    │                     │ Attach │ ← Hook to kernel event
    │                     └────────┤
```

eBPF is used by:
- **Cilium** — Container networking
- **Falco** — Runtime security monitoring
- **bpftrace** — Kernel tracing and profiling
- **Tracee** — Security-focused event tracing

```bash
# Install bpftools (Arch)
sudo pacman -S bpf

# List attached eBPF programs
sudo bpftool prog list

# Trace kernel functions (requires bpftrace)
sudo bpftrace -e 'kprobe:do_sys_open { printf("%s opened %s\n", comm, str(arg1)); }'
```

---

## Summary

| Concept | Key Point |
|---|---|
| Linux architecture | Monolithic kernel, modular subsystems |
| Windows architecture | NT hybrid kernel, HAL, Executive |
| System call interface | Only crossing between user and kernel |
| Policy vs. mechanism | Separation enables flexibility |
| eBPF | Modern kernel extension without kernel modules |
| VFS | Abstraction that unifies all Linux file systems |
| ntdll.dll | Windows user→kernel bridge (bypassed by some malware) |

---

## Key Terms

| Term | Definition |
|---|---|
| VFS | Virtual File System — Linux's unified FS interface |
| HAL | Hardware Abstraction Layer (Windows) |
| NT Executive | High-level kernel services in Windows |
| ntoskrnl.exe | The Windows kernel binary |
| lsass.exe | Local Security Authority — manages credentials |
| eBPF | Extended Berkeley Packet Filter — programmable kernel hooks |
| CFS | Completely Fair Scheduler — Linux's default CPU scheduler |
| POSIX | Portable Operating System Interface — Unix standard |
| Win32 API | Windows programming interface for user applications |
| ABI | Application Binary Interface — binary-level calling conventions |

---

## 🔒 Security Lens

**Windows — Why lsass.exe matters:**
`lsass.exe` holds NTLM hashes, Kerberos tickets, and cleartext credentials in memory on older configurations. Tools like **Mimikatz** dump this process's memory to extract credentials. Modern Windows mitigations:
- **Credential Guard** (Virtualization-Based Security) isolates lsass in a separate VM
- **PPL (Protected Process Light)** prevents non-privileged access to lsass memory

```powershell
# Check if Credential Guard is enabled
Get-ComputerInfo | Select-Object -Property "DeviceGuard*"
```

**Linux — Kernel modules as a rootkit vector:**
Loadable Kernel Modules (.ko files) run in ring 0. A malicious LKM can hide processes, files, and network connections — this is how kernel rootkits work. Mitigations:
- **Secure Boot** + **Module signing** prevents unsigned modules from loading
- **Lockdown LSM** restricts module loading in kernel lockdown mode

```bash
# Check if module signing is enforced
cat /sys/kernel/security/lockdown
# Check module signature verification
grep CONFIG_MODULE_SIG /proc/config.gz
```

**eBPF as an attack tool:**
eBPF programs can be used by attackers *with root access* for stealthy kernel-level monitoring (network sniffing, keylogging) without writing a traditional rootkit. Defense: monitor eBPF program loading events.

**CVE Highlight — CVE-2022-0847 (Dirty Pipe):**
A Linux kernel bug in the pipe mechanism (pipe data structures) allowed unprivileged users to overwrite read-only files, including `/etc/passwd`. This was a mechanism bug — the pipe mechanism didn't properly enforce the read-only policy on memory pages.

---

**[← Chapter 1](01_Introduction_to_OS.md)** | **[→ Chapter 3: Processes and Threads](03_Processes_and_Threads.md)**
