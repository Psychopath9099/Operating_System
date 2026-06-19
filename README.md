# 🖥️ Modern Operating Systems — A Practical Reference

> *From kernel space to cybersecurity implications — a no-nonsense guide to how operating systems really work.*

---

## What This Is

This is a personal reference book on modern operating systems, built from the ground up using:

- **Abraham Silberschatz, Peter Galvin & Greg Gagne** — *Operating System Concepts* (10th Ed.)
- **Robert Love** — *Linux Kernel Development* (3rd Ed.)
- **The Official Linux Kernel Documentation** — [kernel.org/doc](https://www.kernel.org/doc/html/latest/)
- **Microsoft Windows Internals** — Russinovich, Solomon & Ionescu (7th Ed.)
- **Tanenbaum** — *Modern Operating Systems* (4th Ed.)
- Supplementary: NIST, OWASP, Linux man pages, CVE database

It is **not** a beginner intro, and it is **not** a certification dump. It is a **deep but readable** guide to how operating systems are designed, how they behave in modern hardware environments, and — with special attention — how they behave (or break) under adversarial conditions.

---

## Who This Is For

Primarily **for me**. But also for:

- Anyone preparing for ethical hacking, bug bounty, or security research
- Developers who want to understand what's happening *under* their code
- People who read Silberschatz and felt it was a bit outdated in places

---

## Structure

```
OS-Fundamentals-Modern/
├── README.md                          ← You are here
├── CHANGELOG.md                       ← Version history
├── chapters/
│   ├── 00_Preface.md
│   ├── 01_Introduction_to_OS.md
│   ├── 02_OS_Structures_and_Design.md
│   ├── 03_Processes_and_Threads.md
│   ├── 04_CPU_Scheduling.md
│   ├── 05_Synchronization_and_Deadlocks.md
│   ├── 06_Memory_Management.md
│   ├── 07_Virtual_Memory.md
│   ├── 08_Storage_and_File_Systems.md
│   ├── 09_IO_Systems_and_Device_Drivers.md
│   ├── 10_Security_Fundamentals.md
│   ├── 11_Access_Control_and_Permissions.md
│   ├── 12_OS_Hardening.md
│   ├── 13_Virtualization_and_Containers.md
│   ├── 14_Distributed_Systems_Overview.md
│   ├── 15_Networking_in_the_OS.md
│   ├── 16_Boot_Process_and_Firmware.md
│   ├── 17_Modern_Kernel_Architectures.md
│   ├── 18_Performance_and_Profiling.md
│   ├── 19_Forensics_and_Incident_Response.md
│   └── 20_Case_Studies_Linux_and_Windows.md
├── case-studies/
│   └── (supplementary case study references)
└── assets/
    └── (diagrams, ASCII art, reference tables)
```

---

## Reading Order

You can jump around, but the intended flow is:

```
Chapters 1–2   →  Concepts & Design Philosophy
Chapters 3–5   →  Process Management
Chapters 6–7   →  Memory
Chapters 8–9   →  Storage & I/O
Chapters 10–12 →  Security (read these twice)
Chapters 13–15 →  Modern Environments
Chapters 16–18 →  Deep Internals
Chapter 19     →  Forensics & Response
Chapter 20     →  Case Studies (tie it all together)
```

---

## A Note on Cybersecurity Angle

This documentation is **not** a cybersecurity textbook. However, since a deep understanding of OS internals is the foundation of everything in security — from privilege escalation to memory forensics — I've added **"Security Lens"** sections in every chapter that highlight:

- Attack surfaces created by each concept
- How defenders use the same knowledge
- Real CVEs that relate to the topic
- Lab ideas for practice (mostly Kali/Linux-focused)

---

## Platform Coverage

| Topic | Linux (Primary) | Windows 10/11 |
|---|---|---|
| Kernel Architecture | ✅ Deep | ✅ Overview |
| Process Model | ✅ Full | ✅ Full |
| Memory Management | ✅ Full | ✅ Full |
| File Systems | ext4, Btrfs, XFS | NTFS, ReFS |
| Security Model | DAC + SELinux/AppArmor | ACLs + Defender + TPM |
| Boot Process | GRUB2 + UEFI | Secure Boot + UEFI |
| Forensics | `/proc`, `auditd` | Event Viewer, WinPrefetch |

---

## How to Use This

- Read chapters as needed
- Every chapter has a **Summary**, a **Key Terms** table, and a **Security Lens** section
- Commands shown are tested on **Arch Linux** and **Kali Linux** unless noted otherwise
- Windows examples are from **Windows 11 Pro (22H2/23H2)**

---

## Version

**v1.0** — Initial release, June 2026
Built and maintained by: Dear

---

> *"The most important skill for a security professional is understanding what normal looks like — so you recognize when it's not."*
