# Preface — Why Another OS Reference?

---

## The Gap This Fills

If you've read Silberschatz, you already know about process scheduling and memory paging. If you've done the Tanenbaum book, you have a decent mental model of microkernels vs monolithic kernels. Both are excellent — but they were written before:

- Containers became the dominant deployment model
- Spectre and Meltdown forced kernel-level redesigns
- UEFI Secure Boot became the new normal
- Ransomware and kernel-level rootkits became common threats
- cgroups v2 and eBPF completely changed Linux internals
- Windows moved to a hybrid kernel with WSL2

This reference tries to bridge that gap. It covers the theory you need (because theory doesn't expire) and then grounds it in how things actually look on a **modern Linux system** or **Windows 11** in 2024–2025.

---

## What Makes This Different

Most OS textbooks explain what a page table is. This one also tells you:
- What happens when an attacker controls a page table entry
- How the OS defends against that
- How to look at page mappings in `/proc` yourself

Every concept is anchored to something you can **see** or **test**. Every chapter has a **Security Lens** section because, frankly, if you want to understand attack and defense, you need to understand the operating system first. Not the other way around.

---

## How to Read This

Think of it as a **layered book**:

**Layer 1 — Core Concepts** (what every chapter starts with)
Pure OS theory. Works for exam prep, interviews, or just understanding your machine.

**Layer 2 — Modern Implementation** (the bulk of each chapter)
How Linux or Windows actually implements this. Code snippets, kernel source references, and system call behavior.

**Layer 3 — Security Lens** (end of each chapter)
Attack surfaces, defense mechanisms, real CVEs, and lab ideas.

You don't have to read all three layers every time. Jump in at whatever depth the moment calls for.

---

## Prerequisites

You should be comfortable with:
- Basic C or any systems language (Python is fine too)
- Command line on Linux (even beginner level)
- What a process, thread, and file system are (roughly)

You do **not** need to have finished Silberschatz or any specific book. This stands alone.

---

## A Note on Commands

All Linux commands in this book are written for **Bash** on a modern Linux distribution (Arch or Debian-based). They're tested. Windows commands use **PowerShell** unless CMD is specified.

When a command requires root:
```bash
# This needs root / sudo
sudo cat /proc/1/maps
```

When it doesn't:
```bash
# Regular user is fine
cat /proc/cpuinfo
```

---

## Sources & Acknowledgments

This work heavily references:

| Source | Used For |
|---|---|
| Silberschatz et al., *OS Concepts* 10th Ed. | Core theory, process/memory/storage chapters |
| Tanenbaum, *Modern Operating Systems* 4th Ed. | Design philosophy, microkernel comparison |
| Robert Love, *Linux Kernel Development* 3rd Ed. | Linux-specific internals |
| Linux Kernel Documentation (kernel.org) | Current kernel behavior, eBPF, cgroups |
| Russinovich et al., *Windows Internals* 7th Ed. | Windows architecture, NTFS, registry |
| NIST SP 800-series | Security standards and hardening |
| CVE Database (cve.mitre.org) | Real-world vulnerability examples |

---

*Let's get into it.*

---

**[→ Chapter 1: Introduction to Operating Systems](01_Introduction_to_OS.md)**
