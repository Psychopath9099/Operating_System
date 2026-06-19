# Chapter 1 — Introduction to Operating Systems

> *"An operating system is a software layer that manages hardware resources and provides an environment in which application programs can execute."*
> — Silberschatz, *OS Concepts* 10th Ed.

---

## 1.1 What an OS Actually Does

An operating system is not one program. It's a collection of software that sits between your hardware and your applications, doing three things:

**1. Resource Manager**
The CPU has limited cores. RAM is finite. Disk bandwidth is shared. The OS is the referee that decides who gets what, when, and for how long.

**2. Extended Machine**
Without the OS, writing to disk means talking directly to hardware registers. With the OS, you call `write()` and it handles the rest. The OS hides ugly hardware details behind clean interfaces.

**3. Security Enforcer**
The OS decides which process can access which memory region, which user can read which file, and which syscall is allowed from where. This is the part that matters most for security.

---

## 1.2 The Kernel — What Lives in the Core

The **kernel** is the central part of the OS. It runs with full hardware privileges (called **kernel mode** or **ring 0** on x86). Everything else — browsers, terminals, your music player — runs in **user mode** (ring 3) with restricted access.

```
  ┌──────────────────────────────────────┐
  │         User Applications            │  ← Ring 3 (User Mode)
  │   (browsers, editors, daemons)       │
  ├──────────────────────────────────────┤
  │       System Libraries (glibc)       │  ← Still Ring 3
  │     (printf, malloc, pthread)        │
  ├──────────────────────────────────────┤
  │        System Call Interface         │  ← Boundary crossing
  ├──────────────────────────────────────┤
  │             THE KERNEL               │  ← Ring 0 (Kernel Mode)
  │  Process Mgmt | Memory | FS | Net    │
  ├──────────────────────────────────────┤
  │            Hardware                  │  ← CPU, RAM, Disk, NIC
  └──────────────────────────────────────┘
```

When a user-mode program needs the kernel (to open a file, allocate memory, send a network packet), it makes a **system call**. The CPU switches from ring 3 to ring 0, the kernel does its work, then switches back.

This ring transition is fundamental to OS security. An attacker who escapes user mode and gets kernel execution has full control of the machine.

---

## 1.3 Types of Operating Systems

Not all operating systems are built the same way. Understanding the architecture types helps you reason about attack surfaces.

### Monolithic Kernel
Everything runs in kernel space: file systems, drivers, memory management, networking. Linux is monolithic.

```
Advantages:   Fast (no IPC overhead between components)
Disadvantages: A buggy driver can crash the whole kernel
Security note: A vulnerable kernel module = game over
```

### Microkernel
Only the bare minimum runs in kernel space (IPC, basic scheduling, memory mapping). Everything else (file systems, drivers) runs as user-space servers. QNX and MINIX use this model.

```
Advantages:   A crashing file system doesn't take the kernel down
Disadvantages: Slower (needs message passing between components)
Security note: Smaller attack surface in kernel space
```

### Hybrid Kernel
Pragmatic middle ground. Windows NT is hybrid — it has a microkernel-like structure but runs many components in kernel space for performance.

```
  macOS/XNU: Mach microkernel + BSD components in kernel space
  Windows NT: Microkernel + HAL + executive components
  Linux: Monolithic but with loadable kernel modules (LKM)
```

### Exokernel / Library OS
Research territory mostly. The kernel exposes raw hardware and lets each application implement its own OS abstractions. Not common in production.

---

## 1.4 Linux vs. Windows — A High-Level Comparison

| Aspect | Linux (Kernel 6.x) | Windows 11 |
|---|---|---|
| Kernel type | Monolithic + LKM | Hybrid (NT Kernel) |
| Architecture | `arch/` subdirectory per CPU | HAL abstraction layer |
| Source | Open source (GPLv2) | Proprietary |
| Process model | fork/exec model | CreateProcess model |
| File system | ext4, Btrfs, XFS, etc. | NTFS (primary), ReFS |
| Security model | DAC + LSM (SELinux/AppArmor) | ACLs + Integrity Levels + Defender |
| User accounts | UID/GID based | SID based |
| Package management | apt, pacman, dnf, etc. | Windows Update + winget/MSIX |
| Kernel modules | `.ko` files (loadable) | `.sys` drivers (kernel space) |

---

## 1.5 OS Services — What Gets Offered

An OS provides a set of services that programs depend on:

```
┌─────────────────────────────────────────────────┐
│               OS Services                        │
├───────────────┬──────────────┬───────────────────┤
│ Process mgmt  │ File I/O     │ Network comms     │
│ Memory alloc  │ Device access│ Security/auth     │
│ Error handling│ Accounting   │ Protection        │
└───────────────┴──────────────┴───────────────────┘
```

**For cybersecurity:** Every one of these services is also an attack surface. Process creation can be abused (process injection). File I/O can leak data. Memory allocation bugs cause buffer overflows.

---

## 1.6 System Calls — The Only Door In

A **system call** (syscall) is the official mechanism for user-mode programs to request kernel services. On Linux, there are ~350+ syscalls. On Windows, there are several hundred NT system calls.

### Linux Syscall Example — Opening a File
```c
// What you write in C:
int fd = open("/etc/passwd", O_RDONLY);

// What actually happens:
// 1. glibc wraps this in a syscall instruction
// 2. CPU switches to kernel mode (ring 0)
// 3. Kernel's sys_openat() runs
// 4. Kernel returns file descriptor to user space
// 5. CPU returns to ring 3
```

### Seeing Syscalls in Action (Linux)
```bash
# Trace all system calls made by 'ls'
strace ls /tmp

# Count syscall frequency
strace -c ls /tmp

# Trace only specific syscalls
strace -e trace=open,read,write cat /etc/hostname
```

### Windows Equivalent
```powershell
# Use Sysinternals Process Monitor (ProcMon) or:
# Windows has NtOpenFile, NtReadFile, NtWriteFile etc. (NT syscalls)
# Accessed via ntdll.dll → kernel transition
```

> **Security Lens Preview:** Malware often uses direct syscalls (bypassing `ntdll.dll`) to evade EDR/antivirus hooks. Understanding the syscall boundary is key to understanding modern malware evasion.

---

## 1.7 OS as a Trust Boundary

The single most important security concept introduced here:

**The OS is the enforcer of trust boundaries.**

- It separates processes from each other (process isolation)
- It separates users from each other (user isolation)
- It separates user space from kernel space (privilege separation)
- It controls access to hardware devices (device protection)

Every significant attack on a computer system is, in some way, a violation of one of these boundaries. Privilege escalation means breaking the user-to-root boundary. Buffer overflows often break the user-to-kernel boundary. Container escapes break the process-isolation boundary.

Understanding how the OS enforces these boundaries — and where it struggles — is the foundation of system security.

---

## Summary

| Concept | What to Remember |
|---|---|
| OS role | Resource manager + extended machine + security enforcer |
| Kernel mode | Ring 0, full hardware access |
| User mode | Ring 3, restricted access |
| System call | Only official crossing from user → kernel |
| Kernel types | Monolithic (Linux), Hybrid (Windows), Microkernel (QNX) |
| Trust boundary | OS enforces isolation between users, processes, kernel |

---

## Key Terms

| Term | Definition |
|---|---|
| Kernel | Core OS component running in privileged mode |
| Ring 0 | Most privileged CPU execution level |
| Ring 3 | User-mode execution level |
| Syscall | Mechanism for user programs to request kernel services |
| HAL | Hardware Abstraction Layer (Windows) |
| LKM | Loadable Kernel Module (Linux) |
| Monolithic kernel | All OS services in one kernel space binary |
| Hybrid kernel | Mix of microkernel and monolithic design |

---

## 🔒 Security Lens

**Privilege rings as an attack model:**
The ring model is the foundation of most local privilege escalation attacks. Attacks like:
- **CVE-2021-3490** (Linux eBPF verifier bypass) — escaped to ring 0 via eBPF
- **CVE-2020-0796** (SMBGhost) — Windows kernel crash/execution via SMB

**Syscall filtering:**
Modern OS hardening uses syscall filtering (Linux: `seccomp`, Windows: `WDAG`) to restrict what syscalls a process can make. This is how sandboxes like Chrome restrict their renderer processes.

```bash
# See what seccomp filters a process has
cat /proc/$(pidof firefox)/status | grep Seccomp
# 2 = BPF filter active
```

**Lab Idea:**
Use `strace` on a simple C program, then on a suspicious binary. Learn to spot anomalous syscall patterns (e.g., `ptrace()`, `mprotect()`, `memfd_create()`) that indicate process injection.

---

**[← Preface](00_Preface.md)** | **[→ Chapter 2: OS Structures and Design](02_OS_Structures_and_Design.md)**
