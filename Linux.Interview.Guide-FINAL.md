# Linux — Complete Interview Guide

> This is Document 1 of your DevOps interview series: **Linux → Networking → Docker → Kubernetes → CI/CD → AWS → Terraform.**
> Built to cover the Linux knowledge, troubleshooting patterns, scripting, and interview angles most relevant to 3–5 YOE DevOps roles — not an exhaustive catalog of everything Linux can do.
> Final QA edition: hands-on labs sit right next to the concept they belong to, research-backed additions are folded into the Interview Corner, and a small catch-all Practical Exercises section remains at the end for anything that didn't fit one spot. This document is intentionally finite rather than exhaustive.

**How the callouts work (color-coded on GitHub/VS Code):**
- 🔵 `NOTE` = **Anchor Fact** — the exact thing to memorize verbatim, no derivation needed
- 🟡 `WARNING` = **Failure Scenario** — what breaks, how you'd actually find out why
- 🟢 `TIP` = **Interview Answer** — a tight, sayable-out-loud answer
- 🔧 `IMPORTANT` (HANDS-ON) = **Practical Lab** — a realistic operational task, done right where the concept is taught. Do it on a throwaway VM or container, not by reading. Every lab follows: **Concept → Implementation → Hands-on → Failure → Investigation → Fix → Verification → Interview Explanation.**
- 🟣 `IMPORTANT` (LIVE-CODING) = **Interview Task** — the kind of thing you're asked to write live in a shared-terminal/CoderPad round.

---

## Quick Jump Index

**Part 1 — Foundations**
1. [What an OS Actually Does, and Why Linux Owns Servers](#1-what-an-os-actually-does-and-why-linux-owns-servers)
2. [Kernel Space vs User Space](#2-kernel-space-vs-user-space)
3. [Processes](#3-processes)
4. [Memory Management](#4-memory-management)
5. [File System & Permissions](#5-file-system--permissions)
6. [Users, Groups & Sudo](#6-users-groups--sudo)
7. [Services & systemd](#7-services--systemd)
8. [Package Management](#8-package-management)
9. [Shell & Environment Variables](#9-shell--environment-variables)
10. [Storage & Disk](#10-storage--disk)

*Hands-on labs (🔧) and live-coding tasks (🟣) are embedded directly in the sections above — look for them right after the concept they belong to, rather than in a separate list.*

**Part 2 — Deeper Mechanics**
- [Signals & Process Control](#signals--process-control) — includes a hands-on D-state lab
- [File Descriptors & the Open File Table](#file-descriptors--the-open-file-table) — includes the "too many open files" lab
- [The Boot Process](#the-boot-process) — includes the fstab failure lab
- [Logging: journald & syslog](#logging-journald--syslog) — includes the journal-disk-usage lab
- [The Performance Troubleshooting Toolkit](#the-performance-troubleshooting-toolkit) — the "server is slow" method, plus a bottleneck-generation lab

**Part 3 — Interview Corner**
- [Processes & Signals — full 9-angle drill](#processes--signals-drill)
- [Permissions & Security — full 9-angle drill](#permissions--security-drill)
- [Disk & Troubleshooting — full 9-angle drill](#disk--troubleshooting-drill)
- [systemd & Services — full 9-angle drill](#systemd--services-drill)
- [Scripting & Live-Coding — full 9-angle drill](#scripting--live-coding-drill)
- [Performance & "Server Is Slow" — full 9-angle drill](#performance--server-is-slow-drill)
- [Quick-Coverage Round: Memory, Packages, Shell, Boot, Permissions](#quick-coverage-round)
- [Hot Topics Right Now](#hot-topics-right-now)
- [Research-Backed Additions (Current Market, 2025–2026)](#research-backed-additions-current-market-20252026)

**Part 4 — Real Incident / Story Bank** *(your real stories only — see the Practice Scenario note in Storage & Disk for a rehearsal-only worked example)*
- [Story 1: Gitea Without sudo or systemd](#story-1-gitea-without-sudo-or-systemd)
- [Story 2: GitLab CE and the Undersized /var Partition](#story-2-gitlab-ce-and-the-undersized-var-partition)

**Part 5 — Master Glossary (Active Recall)**
- [A–Z Quick Recall](#master-glossary-active-recall)

**Part 6 — Additional Practical Exercises** *(only what didn't fit neatly into one section above)*
- [1: Resource Threshold Monitor With Alerting](#exercise-1-resource-threshold-monitor-with-alerting)
- [2: Find Recently Modified Files in a Date Range](#exercise-2-find-recently-modified-files-in-a-date-range)

---

# Part 1 — Foundations

## 1. What an OS Actually Does, and Why Linux Owns Servers

**Why it exists:** A computer's hardware — CPU, memory, disk — can only do one raw thing at a time and has no concept of "fairness" or "safety" between programs. Without something managing it, two programs would step on each other's memory, fight over the CPU, and one crashing program could take the whole machine down. The operating system exists to sit between raw hardware and every program you run, handing out CPU time, memory, and device access in a controlled, safe way.

**Step-by-step flow:**
1. Hardware powers on — CPU, RAM, disk are just physical circuits with no organization yet.
2. The kernel (the core of the OS) loads first and takes control of the hardware directly.
3. Every other program that runs afterward — your shell, Jenkins, a database — talks to hardware only *through* the kernel, never directly.
4. The kernel decides, moment to moment, which program gets the CPU next (scheduling), which program's request for memory gets approved, and which program is allowed to read/write which files.

**Components involved:** CPU, RAM, disk controller, the kernel, and every user program (called a "process" once running) sitting on top of it.

**Plain example:** Two people (Jenkins and a database) both want to use the same CPU at the exact same moment. The kernel is the referee that decides who goes first, gives each a tiny slice of time, and switches between them so fast it looks like both are running simultaneously.

> [!NOTE]
> **Anchor Fact:** The kernel is the one piece of software with direct, unrestricted access to hardware. Everything else — including your shell — is a "user space" program that must ask the kernel for anything it needs (memory, files, network).

> [!WARNING]
> **Failure Scenario:** A server becomes completely unresponsive, not just one app. **Symptom:** SSH won't even connect. **Hypothesis:** something at the kernel/hardware level, not an app crash — a single app crashing shouldn't take the whole box down. **Evidence:** check console/serial access if SSH is dead, look for OOM-killer activity or a kernel panic in logs from before it died. **Root cause (commonly):** the kernel itself ran out of memory to manage and killed processes indiscriminately, or a kernel-level driver crashed. **Fix:** reboot, then dig into `journalctl -k` (kernel-only logs) after recovery. **Prevention:** memory limits/monitoring so no single process can starve the whole kernel.

> [!TIP]
> **Interview Answer:** "The OS — specifically the kernel — is the layer between raw hardware and every program running on the machine. It decides which process gets CPU time, approves or denies memory requests, and mediates every file and device access. Nothing touches hardware directly except through it."

**Why Linux specifically dominates servers, not desktops:** It's free with no per-server licensing cost, it's extremely stable for processes that need to run for months without a restart, you can strip it down to only what's needed (smaller attack surface, less overhead), and its scripting/automation support is unmatched — which is exactly what large-scale infrastructure needs. Windows dominates desktops because of application/driver compatibility for end users; that trade-off doesn't matter for a server that just needs to run one service reliably.

---

## 2. Kernel Space vs User Space

**Why it exists:** If every program could directly poke at hardware or other programs' memory, one buggy or malicious program could crash the entire system or steal another program's data. Splitting the world into two zones — a privileged kernel zone and a restricted user zone — is what makes a multi-program system safe.

**Step-by-step flow:**
1. Kernel space is where the kernel itself runs — full, unrestricted access to hardware and memory.
2. User space is where every regular program runs — Jenkins, your shell, a Python script — with restricted access, enforced by the CPU itself (via hardware-level privilege rings).
3. When a user-space program needs something only the kernel can do (read a file, open a network socket, allocate memory), it makes a **system call** — a formal, controlled request into kernel space.
4. The kernel validates the request, performs it safely, and hands control back to the user-space program.

**Components involved:** CPU privilege rings (hardware-enforced), system call interface, kernel, user-space processes.

**Plain example:** Think of kernel space as the back office of a bank that only bank staff can enter, and user space as the public lobby. As a customer (a user-space program), you can't walk into the vault yourself — you fill out a request slip (a system call) and hand it to a teller (the kernel), who goes in on your behalf and brings back only what you're authorized to have.

> [!NOTE]
> **Anchor Fact:** The mechanism a user-space program uses to ask the kernel to do something on its behalf is called a **system call** (e.g., `open()`, `read()`, `write()`, `fork()`).

> [!WARNING]
> **Failure Scenario:** An application seems to hang doing something simple like reading a file. **Symptom:** the process shows as running but makes no progress. **Investigate:** `strace -p <pid>` shows you the actual system calls the process is making in real time — if it's stuck in a `read()` or `connect()` call, the problem is downstream (a slow disk, a slow/dead network peer), not the application's own logic. **Root cause:** often a slow or unresponsive resource the syscall is waiting on. **Fix:** address the underlying resource (network timeout config, disk health). **Prevention:** timeouts on any blocking call so a stuck syscall doesn't hang the whole application forever.

> [!TIP]
> **Interview Answer:** "User-space programs never touch hardware directly — they make system calls, formal requests into the kernel, which is the only code with full hardware access. This separation, enforced at the CPU level, is what stops one misbehaving program from crashing or reading another program's memory."

---

## 3. Processes

**Why it exists:** You need a way to run more than one program at a time, track each one's own memory and state separately, and be able to start, stop, or clean up any one of them without affecting the others. A process is the kernel's answer: a fully isolated, trackable unit of "a program that is currently running."

**Step-by-step flow:**
1. You type a command in the shell (e.g., `python app.py`).
2. The shell calls `fork()` — this creates a near-exact copy of the shell itself as a new process (the "child").
3. That child process immediately calls `exec()` — which replaces its own code with the new program's code (`python`), while keeping the same process identity.
4. The kernel assigns this new process a unique **PID** (Process ID) and starts scheduling CPU time for it.
5. The process runs, using its own private memory space, its own open files, and its own state — invisible to other processes unless explicitly shared.
6. When it finishes, it exits with a status code; the parent process (the shell) is expected to "reap" it — read that exit status so the kernel can fully free its resources.

**Components involved:** PID, PPID (parent PID), process state (running/sleeping/stopped/zombie), the process table (kernel's internal bookkeeping of every process).

**Exact identifiers:** Every process has a PID (a positive integer, starting from 1 — PID 1 is always the very first process started at boot, traditionally `init` or `systemd`). Every process also tracks its PPID — who created it.

**Decision point:** Fork-then-exec is two separate steps on purpose — `fork()` alone is useful when you just want a duplicate worker (no new program), and `exec()` alone would just replace the current process instead of creating a new one. Splitting them gives you the flexibility to do either.

**Plain example:** Imagine photocopying yourself (fork) — you now have two identical copies. Then one copy walks into a costume booth and comes out looking and acting completely different (exec) — same badge number (PID) as before the costume change, totally different job.

> [!NOTE]
> **Anchor Fact:** PID 1 is the first process in the initial host PID namespace, started by the kernel at boot; on most modern Linux systems it is `systemd`. If the initial host PID 1 terminates unexpectedly, Linux cannot continue normal operation and will panic.

> [!WARNING]
> **Failure Scenario:** `ps aux` shows a process stuck in state `Z` (zombie) that never disappears. **Symptom:** a defunct process lingering. **Hypothesis:** the parent process never read its exit status. **Evidence:** check the PPID with `ps -ef`, see if that parent is itself unresponsive or buggy. **Root cause:** a parent process that spawned children but has a bug where it never calls `wait()` on them. **Fix:** restarting the parent process reaps all its zombie children at once (since when a parent dies, its children are "adopted" by PID 1, which does clean up properly). **Prevention:** well-written services always reap their child processes; a persistent pattern of zombies signals a real bug worth reporting upstream.

> [!TIP]
> **Interview Answer:** "A process is an isolated, running instance of a program — its own memory, its own file descriptors, tracked by a unique PID. New processes are created by forking an existing one and then exec-ing the target program into that forked copy. If a process exits and its parent never reads the exit status, it becomes a zombie — harmless in small numbers, but a sign of a bug if it keeps happening."

> [!IMPORTANT]
> 🔧 **HANDS-ON LAB — Find and handle zombie processes**
> **Concept:** Tests whether you actually understand the process lifecycle above, not just the command.
> **Implementation:**
> ```bash
> #!/bin/bash
> # find_zombies.sh — lists zombie processes and their parent PIDs
> ps -eo pid,ppid,stat,cmd | awk '$3 ~ /Z/ {print}'
> ```
> **Hands-on:** Run this against a live system. Empty output is expected and healthy — zombies are usually transient.
> **Failure:** The script finds a persistent zombie that never clears.
> **Investigation:** Check the PPID column — that parent is the actual problem. Is it still alive and healthy?
> **Fix:** You cannot kill a zombie directly — it's already dead, there's nothing to signal. The fix is restarting or fixing the parent, which causes the kernel to reap it.
> **Verification:** Re-run the script; the zombie clears once its parent is restarted or fixed.
> **Interview Explanation:** "A zombie means a process exited but its parent hasn't read the exit status yet — you can't kill something that's already dead, so the fix targets the parent, not the zombie entry itself."

---

## 4. Memory Management

**Why it exists:** Physical RAM is limited and shared across every running process. Without management, one process could allocate all of it, starving everything else, or two processes could accidentally write into the same physical memory and corrupt each other's data. Linux solves this with **virtual memory** — every process believes it has its own private, full address space, and the kernel quietly maps that illusion onto real physical RAM behind the scenes.

**Step-by-step flow:**
1. A process asks for memory (e.g., allocating a large array).
2. The kernel doesn't necessarily hand over real RAM immediately — it gives the process a range of **virtual addresses** and only maps them to actual physical RAM pages when they're first touched (this is called lazy allocation).
3. If physical RAM starts running low, the kernel can move rarely-used pages out to disk (**swap**), freeing physical RAM for something more active.
4. The kernel also uses any free RAM to cache recently-read disk data (**page cache**) — this isn't wasted memory, it's reclaimed instantly the moment a real application needs it.

**Components involved:** Virtual address space (per process), physical RAM, swap space (on disk), the page cache.

**Exact identifiers:** `free -h` shows `total`, `used`, `free`, `buff/cache`, and `available`. The one that actually matters for "am I under memory pressure" is `available` — not `free` — because `available` accounts for cache the kernel will happily give back to an app on demand.

**Decision point:** When physical RAM is genuinely tight, the kernel has to choose what to evict: reclaim page cache first (cheap, no cost to reclaim, since it can just be re-read from disk later) before ever touching swap (expensive, since swapping means slow disk I/O to bring memory back later). This is why "available" memory being low, not "free," is the real warning sign.

**Plain example:** Physical RAM is a small desk, and virtual memory is giving everyone the illusion of an infinitely large desk — the kernel is the one secretly shuffling papers to a filing cabinet in the corner (swap) or the trash bin they can grab back from anytime (page cache) so nobody notices the desk is actually much smaller than it looks.

> [!NOTE]
> **Anchor Fact:** Watch `available` in `free -h`, not raw `free`. Low `available` together with climbing swap activity is a much stronger signal than either alone — but a full diagnosis also weighs PSI (`/proc/pressure/memory`, if available on your kernel), actual OOM-kill events in `dmesg`/`journalctl -k`, whether the application itself is behaving abnormally, and any cgroup memory limits in play. No single number tells the whole story on its own.

> [!WARNING]
> **Failure Scenario:** A server is "slow" and `top` shows high memory usage. **Symptom:** sluggish everything. **Hypothesis:** could be genuine memory pressure, or could just be a healthy, full page cache doing its job. **Evidence:** run `free -h` and look specifically at `available` and swap usage (`swapon --summary` or the `Swap` line), not the raw `used` number. **Root cause (if genuine pressure):** an application with a memory leak, or too many processes for the box's actual RAM. **Fix:** identify the top consumer with `ps aux --sort=-%mem`, investigate that specific process. **Prevention:** memory limits per service (cgroups) so one runaway process can't starve the whole machine.

> [!TIP]
> **Interview Answer:** "Linux gives every process the illusion of its own large, private memory space through virtual memory, and uses any spare physical RAM as disk cache rather than leaving it idle. That's why 'used' memory looks high but is often totally healthy — 'available' plus climbing swap activity is a much better signal than 'used,' though I'd also check for actual OOM events and, on newer kernels, PSI before calling it real memory pressure."

---

## 5. File System & Permissions

**Why it exists:** A server needs an organized way to store and retrieve files, and — critically in a multi-user environment — a way to control exactly who can read, write, or execute each one. Without permissions, any user or process could read or modify any file on the system, which is an obvious security disaster.

**Step-by-step flow:**
1. Every file on disk has an **inode** — a data structure holding its metadata (owner, permissions, size, timestamps, and pointers to where its actual data blocks live on disk). The filename itself is just a label pointing at an inode, not the file's real identity.
2. When you try to access a file, the kernel checks the requesting process's user/group identity against that file's permission bits before allowing the operation.
3. Permissions are split three ways: what the **owner** can do, what the **group** can do, and what **everyone else** can do — each with **r**ead, **w**rite, and e**x**ecute bits.
4. Numeric shorthand: read=4, write=2, execute=1, summed per category — so `750` means owner=7 (rwx), group=5 (r-x), other=0 (nothing).

**Components involved:** Inode, filename (a separate pointer to an inode — this is why a "hard link" is possible: two different filenames pointing at the same inode), permission bits, owner UID, group GID.

**Exact identifiers:** `chmod 750 file` sets owner rwx, group r-x, other none. `chown user:group file` changes ownership. `ls -l` displays permissions as `rwxr-x---`.

**Decision point:** A hard link and a symlink solve the same-sounding problem completely differently — a hard link is a second filename pointing at the exact same inode (deleting one name leaves the file intact under the other name, since the inode itself isn't gone until every hard link to it is removed). A symlink is a separate, tiny file that just contains a path to another file — deleting the target breaks the symlink (it becomes "dangling"), because a symlink doesn't share the target's inode at all.

**Plain example:** An inode is like a locker's actual contents and access rules. The filename is just a sticky note with a name on it, pointing at that locker. A hard link is a second sticky note pointing at the *same* locker — remove one note, the locker's still there under the other note. A symlink is a note that just says "the real locker is over at desk #12" — if desk #12's locker gets removed, your note now points at nothing.

> [!NOTE]
> **Anchor Fact:** `chmod 750` = owner rwx (7), group r-x (5), other none (0). Read=4, write=2, execute=1 — always sum these three per category.

> [!WARNING]
> **Failure Scenario:** `df -h` shows a filesystem at 95% full, but manually adding up file sizes with `du` only accounts for 60%. **Symptom:** disk usage mismatch. **Hypothesis:** a process is holding a file open that's already been deleted from the directory listing. **Evidence:** `lsof +L1` or `lsof | grep deleted` shows open file handles pointing at unlinked (deleted) files. **Root cause:** deleting a file only removes its directory entry (the filename); the inode and its actual disk blocks aren't freed until every process holding it open closes that file descriptor. **Fix:** restart or signal the specific process holding it open. **Prevention:** log rotation tools that properly signal applications to reopen log files instead of just deleting them out from under a running process.

> [!TIP]
> **Interview Answer:** "A file's real identity is its inode — permissions, ownership, and where its data actually lives on disk. The filename is just a pointer to that inode, which is exactly why deleting a file a process still has open doesn't free the disk space immediately — the inode survives until the last open handle to it closes."
## 6. Users, Groups & Sudo

**Why it exists:** A server is almost never used by just one person or one purpose — multiple humans and multiple services need access, but not all of them should be equally trusted. Users and groups let the system say precisely who can do what, and `sudo` lets a trusted user temporarily borrow elevated privileges without ever actually knowing the root password.

**Step-by-step flow:**
1. Every user has a UID (numeric user ID); UID 0 is always `root` — the one account with unrestricted access to everything.
2. Every user belongs to at least one primary group, and optionally more secondary groups — file permissions check group membership, not just individual users.
3. When a permitted user runs `sudo <command>`, the system checks `/etc/sudoers` (or files under `/etc/sudoers.d/`) to see if that user is explicitly allowed to run that command as another user (usually root).
4. If allowed, the command runs with root's privileges for that one invocation only — the user's own session stays a normal, unprivileged user afterward.

**Components involved:** UID/GID, `/etc/passwd` (user records), `/etc/group` (group records), `/etc/sudoers`.

**Exact identifiers:** UID 0 = root, always. `id` shows your current UID, GID, and group memberships.

**Decision point:** Giving a service account full, unrestricted `sudo` versus scoping it to specific commands (`NOPASSWD: /usr/bin/systemctl restart myapp`) is a real, deliberate security tradeoff — full sudo is simpler to set up but means a compromised or buggy automation script can do absolutely anything as root; scoped sudo limits the blast radius to exactly the commands that account actually needs.

**Plain example:** Think of `sudo` as a hotel key card that, for the ten seconds it takes to open one specific door, briefly acts like a master key — but only for that door, and only because the front desk (the sudoers file) explicitly authorized you for that exact door.

> [!NOTE]
> **Anchor Fact:** UID 0 is always root, no matter what the account is named. Never assume an account is unprivileged just because it's not literally named "root."

> [!WARNING]
> **Failure Scenario:** An automation script suddenly can't run a command it always could. **Symptom:** `sudo: a command is not allowed` error. **Hypothesis:** a sudoers rule scoped that account to specific commands, and this one isn't on the list — or someone tightened the rule recently. **Evidence:** `sudo -l` as that user shows exactly which commands it's allowed to run with sudo. **Root cause:** either the command changed slightly (a new flag, a different path) so it no longer matches the exact allowed pattern, or the sudoers rule was intentionally narrowed. **Fix:** update the sudoers rule to match the actual command needed, as narrowly as still works. **Prevention:** document exactly why each sudoers rule exists so future changes don't blindly widen or accidentally break it.

> [!TIP]
> **Interview Answer:** "Every account has a UID, with 0 always meaning root regardless of the account's name. Sudo lets a trusted, explicitly-listed user run specific commands with elevated privileges for just that one command, based on rules in the sudoers file — which is why scoping sudoers to exact commands, rather than granting full access, meaningfully limits what a compromised account can do."

> [!IMPORTANT]
> 🔧 **HANDS-ON LAB — Provision a new team member with scoped access**
> **Concept:** Ties users/groups/sudo together into one realistic onboarding task.
> **Implementation:**
> ```bash
> # INSPECT — does this user or group already exist?
> id newuser 2>/dev/null; getent group deploy-team
>
> # EXECUTE
> sudo useradd -m -G deploy-team newuser
> sudo passwd newuser
> echo "newuser ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart myapp" | sudo tee /etc/sudoers.d/newuser
> sudo chmod 440 /etc/sudoers.d/newuser     # sudoers.d files must be 440 or sudo refuses to read them
>
> # VERIFY
> sudo -l -U newuser
> sudo visudo -c                            # validate sudoers syntax before trusting it
> ```
> **Failure:** `sudo -u newuser sudo systemctl restart myapp` is rejected.
> **Investigation:** `visudo -c` reveals a syntax error, or the file's permissions weren't exactly `440`.
> **Fix:** Sudo silently refuses sudoers.d files with incorrect permissions/ownership as a security measure — correct and re-validate.
> **Verification:** Re-run `sudo -l -U newuser` and confirm the exact command is listed.
> **Interview Explanation:** "I always validate sudoers syntax with visudo -c before trusting it, and check file permissions specifically — sudo enforces strict permissions on sudoers.d files and will silently refuse to honor a misconfigured one."

---

## 7. Services & systemd

**Why it exists:** A server runs many long-lived background programs — a database, a web server, a monitoring agent — that need to start automatically at boot, restart if they crash, and be manageable (start/stop/status) in a consistent way. `systemd` is the modern init system that does exactly this, replacing older, less consistent startup scripts.

**Step-by-step flow:**
1. Each managed service has a **unit file** (e.g., `/etc/systemd/system/myapp.service`) describing how to start it, what user to run it as, and what to do if it crashes.
2. `systemctl start myapp` tells `systemd` to launch the process according to that unit file.
3. `systemd` tracks the process's state continuously — if it crashes and the unit file specifies `Restart=on-failure`, systemd relaunches it automatically.
4. `journalctl -u myapp` reads that service's logs directly from `systemd`'s own logging system (`journald`), without needing a separate log file.

**Components involved:** Unit files, `systemctl` (the control command), `journald` (the logging component), targets (systemd's replacement for old-style "runlevels" — a target is a named group of units meant to be active together, like `multi-user.target`).

**Exact identifiers:** `systemctl status myapp` shows current state (active/inactive/failed), the main PID, and a tail of recent log lines. `systemctl enable myapp` makes it start automatically at boot; `systemctl start myapp` starts it right now — these are two independent actions.

**Decision point:** `enable` vs `start` trips people up constantly — enabling a service only sets it up to start at the *next* boot, it does not start it immediately; starting a service runs it right now, but if the box reboots and it wasn't enabled, it won't come back on its own. Production services almost always need both.

**Plain example:** `enable` is like scheduling an alarm for tomorrow morning — it will go off next time, but nothing happens right now. `start` is like the alarm going off immediately, right now, one time. You usually want both: it should be running now, and it should keep running after every future reboot.

> [!NOTE]
> **Anchor Fact:** `enable` = starts automatically on future boots. `start` = starts right now. They are independent — you often need both.

> [!WARNING]
> **Failure Scenario:** A service was working fine, but after a routine server reboot, it's down. **Symptom:** service isn't running post-reboot. **Hypothesis:** it was started manually at some point but never enabled. **Evidence:** `systemctl is-enabled myapp` — if it says `disabled`, that confirms it. **Root cause:** someone ran `systemctl start` without also running `systemctl enable` when they first set it up. **Fix:** `systemctl enable --now myapp` starts it immediately and enables it for future boots in one command. **Prevention:** always pair enable and start for any service meant to be permanent, and verify with `is-enabled` as part of setup, not just `status`.

> [!TIP]
> **Interview Answer:** "systemd manages services through unit files describing how to start them, restart them on failure, and what user to run as. `enable` and `start` are separate actions — enable schedules it for future boots, start runs it immediately — which is exactly why a service can be running fine today but silently fail to come back after a reboot if it was only ever started manually."

**Your real, honest constraint worth knowing here:** Not every environment gives you the option to use systemd. On a RHEL host without sudo access, you can't create unit files at all — you're stuck managing a process manually with `nohup <command> &`, under a dedicated OS user, with no automatic restart-on-crash and no `journalctl` integration. That's a real operational constraint, not a mistake — see the Story Bank below for exactly this situation.

---

## 8. Package Management

**Why it exists:** Installing software by manually downloading, compiling, and placing files is slow, error-prone, and makes it nearly impossible to track what's installed, update it safely, or remove it cleanly later. A package manager tracks installed software as a database, handles dependencies automatically, and gives you one consistent way to install/update/remove anything.

**Step-by-step flow:**
1. You run something like `dnf install nginx` (RHEL/Fedora family) or `apt install nginx` (Debian/Ubuntu family).
2. The package manager checks its local metadata cache to find where that package lives and what other packages it depends on.
3. It resolves the full dependency tree — nginx might need three other libraries first — and downloads all of them from a configured repository.
4. It installs everything, records exactly what was installed (versions, files placed) in its own database, so future updates or a clean uninstall are both possible.

**Components involved:** The package manager itself (`dnf`/`yum` on RHEL family, `apt` on Debian family), repositories (remote servers hosting packages), the local package database.

**Exact identifiers:** RHEL/CentOS/Fedora use `.rpm` packages via `dnf`/`yum`. Debian/Ubuntu use `.deb` packages via `apt`/`dpkg`. These two families are NOT cross-compatible — an `.rpm` won't install on Ubuntu.

**Decision point:** Installing from a package manager's repository versus manually compiling from source is a real, ongoing tradeoff — the package manager gives you tracked, easily-updatable, dependency-resolved software, but the version available might lag behind the newest release; compiling from source gets you exactly the version/build options you want, but you're now personally responsible for tracking updates and security patches yourself.

**Plain example:** A package manager is like ordering furniture from a store that also handles delivery and tells you exactly which screws and tools you'll need before it ships — versus building the furniture yourself from raw lumber, where you get exact control but you're on your own for sourcing every part.

> [!NOTE]
> **Anchor Fact:** RHEL family = `.rpm` + `dnf`/`yum`. Debian family = `.deb` + `apt`/`dpkg`. They are incompatible package formats — know which family you're on before assuming a command works.

> [!WARNING]
> **Failure Scenario:** `dnf install somepackage` fails, saying it can't find the package. **Symptom:** package not found. **Hypothesis:** either a typo, or the repository containing it isn't configured on this system. **Evidence:** `dnf repolist` shows which repositories are currently enabled. **Root cause:** many packages (especially newer or third-party ones) live in "extra" repositories (like EPEL on RHEL) that aren't enabled by default. **Fix:** enable the correct repository, then retry the install. **Prevention:** document which non-default repositories a server depends on, so a rebuild doesn't silently miss them.

> [!TIP]
> **Interview Answer:** "A package manager tracks installed software as a database, resolving dependencies automatically instead of you sourcing them manually — dnf and rpm on RHEL-family systems, apt and deb on Debian-family systems, and the two formats aren't interchangeable."

---

## 9. Shell & Environment Variables

**Why it exists:** You need a way to type commands and have them interpreted and executed, and a way to pass configuration (like where to find executables, or a database URL) into programs without hardcoding it into their source code. The shell is that command interpreter; environment variables are that configuration-passing mechanism.

**Step-by-step flow:**
1. You type a command; the shell (commonly `bash`) parses it, resolves any variables, and figures out which actual program to run.
2. For a bare command name (like `python`), the shell searches every directory listed in the `PATH` environment variable, in order, until it finds a matching executable.
3. When the shell starts a new program, that program inherits a copy of the shell's current environment variables — this is how configuration like `DATABASE_URL` reaches an application without being written into its code.
4. Environment variables set in one shell session do NOT automatically appear in a different session or a script run separately, unless explicitly exported (`export VAR=value`) or defined in a file that session reads on startup (like `.bashrc`).

**Components involved:** The shell process itself, the `PATH` variable, the broader environment variable table each process carries.

**Exact identifiers:** `echo $PATH` shows the search order for executables. `env` or `printenv` lists all current environment variables. `export VAR=value` makes a variable available to child processes; without `export`, it's only visible in the current shell.

**Decision point:** A plain shell variable (`VAR=value`) versus an exported one (`export VAR=value`) looks identical when you `echo $VAR` in the same session — the difference only shows up when a *child process* (like a script you run) tries to read it. This is a very common source of "it works when I run it directly but not from a script" confusion.

**Plain example:** An exported variable is like writing a note and photocopying it for everyone who works under you (child processes) — they each get their own copy. An unexported variable is a sticky note only on your own desk — nobody working for you can see it, even though you can.

> [!NOTE]
> **Anchor Fact:** A variable must be `export`-ed to be visible to any child process (including a script you run from that shell). Setting it without `export` keeps it local to the current shell only.

> [!WARNING]
> **Failure Scenario:** A deployment script fails, saying a required config value is missing, even though you can `echo` that exact variable successfully in your terminal. **Symptom:** works interactively, fails in a script. **Hypothesis:** the variable was set but never exported, or it was set in one shell session and the script runs in a separate process that never inherited it. **Evidence:** add `env | grep VARNAME` inside the script itself to check what it actually sees. **Root cause:** almost always a missing `export`, or the variable was only defined in an interactive shell config file that non-interactive script execution doesn't read. **Fix:** export the variable properly, or set it explicitly wherever the automation actually runs (a systemd unit's `Environment=` line, a CI pipeline's secrets config). **Prevention:** never rely on your personal interactive shell environment for something a script/service needs — define it explicitly in the script's own execution context.

> [!TIP]
> **Interview Answer:** "The shell resolves commands by searching the PATH variable in order, and passes its current environment variables down to any program it starts. But a variable only reaches a child process if it was exported — which is exactly why something can work fine when you type it directly but fail inside a script that runs as a separate process."

> [!IMPORTANT]
> 🔧 **HANDS-ON LAB — Cron job silently failing (environment mismatch)**
> **Concept:** The hands-on version of the exact failure mode above — a commonly reported real scripting failure.
> **Implementation:**
> ```bash
> cat > /home/user/check.sh << 'EOF'
> #!/bin/bash
> echo "PATH is: $PATH"
> some_custom_tool --version
> EOF
> chmod +x /home/user/check.sh
> ./check.sh                     # works manually
> # * * * * * /home/user/check.sh >> /home/user/cron.log 2>&1
> ```
> **Failure:** The cron-run version fails with "command not found" even though it worked manually.
> **Investigation:** Check `/home/user/cron.log` — cron's `$PATH` is its own minimal default, not your interactive shell's.
> **Fix:** Use absolute paths inside cron scripts, or explicitly set `PATH` at the top rather than relying on inherited shell config.
> **Verification:** Re-run and confirm the log shows success.
> **Interview Explanation:** "Cron runs with a minimal environment, not your interactive shell's — so a script that works when you run it directly can fail under cron purely from a missing PATH entry. I always redirect cron output to a log file specifically so this class of failure is diagnosable, not silent."

> [!IMPORTANT]
> 🟣 **LIVE-CODING LAB — The scripting round, part 1: an interview-grade disk alert script**
> Live-coding rounds for 3–5 YOE roles are reported to sometimes include "write a script that alerts when a disk is almost full." What separates candidates is *not* the loop — it's error handling, quoting, and exit codes.
> ```bash
> #!/usr/bin/env bash
> # disk_alert.sh — alert when any mounted filesystem exceeds a threshold
> set -euo pipefail          # -e: exit when a command fails where errexit applies; -u: error on unset vars; pipefail: a pipeline fails if any stage fails
>
> THRESHOLD="${1:-80}"       # arg or default; braces+quotes: always
> LOG_TAG="disk-alert"
>
> main() {
>   local usage pct mount
>   while read -r usage pct mount; do
>     pct="${pct%\%}"        # strip the % sign
>     if (( pct >= THRESHOLD )); then
>       logger -t "$LOG_TAG" "CRITICAL: $mount at $pct% (threshold ${THRESHOLD}%)"
>       echo "ALERT: $mount is ${pct}% full" >&2
>     fi
>   done < <(df -H --output=source,pcent,target | tail -n +2)
> }
>
> main "$@"
> ```
> **What interviewers are actually scoring:** `set -euo pipefail` present; variables quoted; a sensible exit code; no silent failures. Be ready to explain each of those three `set` flags out loud — they are follow-up questions almost every time. Also be ready for: "now make it alert only once per hour" (add state), "now run it every 5 minutes" (cron/systemd timer — and mention cron's minimal environment).

> [!IMPORTANT]
> 🟣 **LIVE-CODING LAB — The scripting round, part 2: log triage**
> "Here's an access log — give me the top 5 client IPs" is a staple because it tests whether you compose tools or memorize one-liners.
> 1. `awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -5` — the canonical answer.
> 2. Depth probes: "only 5xx errors": `awk '$9 >= 500 {print $1}' … | sort | uniq -c | sort -rn | head`. Know which *field number* holds status, IP, timestamp in a standard combined log format — that's the real test.
> 3. **Interview connection:** say the shape of your answer out loud first ("group, count, sort — that's the top-N pattern"), then type. Interviewers consistently rate structured narration above typing speed.

---

## 10. Storage & Disk

**Why it exists:** Data needs to persist somewhere physical, and a server needs a reliable way to see how full its storage is, where space is actually being used, and diagnose when usage doesn't add up. This is a distinct skill from the file-permissions topic above — this is about capacity and layout, not access control.

**Step-by-step flow:**
1. Physical or virtual disks are divided into **partitions**, each of which is **mounted** at a specific directory path (e.g., a separate disk mounted at `/var` so logs don't fill up the same space as the OS itself).
2. `df -h` reports usage at the filesystem/mount level — how full each partition is.
3. `du -sh *` reports usage at the directory/file level — what's actually taking up that space within a given folder.
4. These two commands answer genuinely different questions, which is exactly why they can appear to "disagree" (see the deleted-but-open-file scenario back in Section 5).

**Components involved:** Partitions, mount points, the `df` and `du` commands, `lsof` for open file handles.

**Exact identifiers:** `df -h` (human-readable sizes, whole filesystem view), `du -sh /path/*` (per-item breakdown inside a path), `mount` (shows what's currently mounted where).

**Decision point:** Splitting `/var` (logs, variable data) onto its own partition, separate from the root `/` partition, is a deliberate design choice — it means runaway log growth fills up `/var` and can't take down the entire OS by filling the root partition too, at the cost of needing to actively manage that partition's size separately.

**Plain example:** `df` tells you how full each storage room in a building is. `du` tells you what's actually sitting inside one specific room, item by item. If a room shows 95% full by the building's meter (df) but you can only account for 60% of that by walking through and adding up what's visibly there (du), something is taking up space that isn't showing up in your walkthrough — usually because it's technically "removed" from the room's inventory list but still physically occupying space (the deleted-but-open-file situation).

> [!NOTE]
> **Anchor Fact:** `df` = filesystem-level (how full is the partition). `du` = directory-level (what's inside a specific path). They answer different questions and can legitimately disagree.

> [!WARNING]
> **Failure Scenario:** An application server suddenly refuses new connections or fails to write files. **Symptom:** "no space left on device" errors, even though the app itself hasn't grown. **Hypothesis:** logs on a separate `/var` partition may have silently filled it up. **Evidence:** `df -h` immediately shows which specific partition is full, not just overall disk. **Root cause:** commonly a log file that isn't being rotated, growing unbounded. **Fix:** clear or rotate the offending log, and set up `logrotate` if it isn't already configured. **Prevention:** disk usage alerts on every mounted partition individually, not just the root filesystem, plus enforced log rotation on anything that writes continuously.

> [!IMPORTANT]
> 🔧 **HANDS-ON LAB — The disk-full emergency (a frequently reported scenario in current research)**
> **Concept:** A production server is nearly or completely out of disk space — one of the most commonly referenced real-world Linux scenarios across the interview-prep material and candidate reports checked for this guide (a sample, not a universal ranking).
> **Implementation — inspect → validate → execute → verify, never delete first:**
> ```bash
> # INSPECT
> df -h                                                          # which filesystem is full
> du -sh /var/* /home/* /tmp/* 2>/dev/null | sort -rh | head -10 # biggest consumers
> lsof +L1 2>/dev/null | grep -i deleted                         # deleted-but-open files
>
> # VALIDATE — before removing anything: is it actively written by a running service?
> #            is it referenced by anything currently running? is it inside a path
> #            logrotate or the app already expects to manage?
>
> # EXECUTE — for genuinely one-off cleanup, act narrowly on what you've validated
> #           (old package caches, confirmed-orphaned temp files). For LOGS specifically,
> #           don't hand-roll find+gzip/find+delete as your production mechanism — that's
> #           an ad-hoc habit that's easy to point at the wrong file under pressure.
> #           The production-correct fix is a proper logrotate policy (see the systemd &
> #           Services and Scripting sections) so rotation, compression, and retention
> #           happen on a schedule you've validated with `logrotate -d` (dry-run), not
> #           improvised during an incident. Use `find` here for INSPECTION — e.g.
> #           `find /var/log -name "*.log" -mtime +7` to see what a rotation policy
> #           would catch — not as the deletion mechanism itself.
>
> # VERIFY
> df -h                                     # space actually freed?
> systemctl status <affected-service>       # systemd/process state
> curl -fsS http://127.0.0.1:<port>/health  # application health, if the service exposes one
> ```
> **Failure:** You delete what looks like an old log, but the app immediately throws write errors.
> **Investigation:** The file wasn't rotated out — it was the active file a running process still had open (the file-descriptor mechanics from Part 2).
> **Fix:** Restore from backup if possible, or send the process a reload signal (many daemons reopen their log on SIGHUP) instead of deleting a file it's actively writing to.
> **Verification:** Confirm the service resumes writing logs normally.
> **Interview Explanation:** "My first move is never deletion — it's identifying what's actually consuming space with du, and checking whether anything open via lsof is holding a deleted file, since that's a common cause of a df/du mismatch. Once I know what's safe to remove, I act narrowly, then verify both that space was freed and the affected service is still healthy."

> [!IMPORTANT]
> 🔧 **HANDS-ON LAB — Investigate and safely unmount a busy filesystem (destructive — inspect/validate/execute/verify)**
> **Concept:** Unmounting a filesystem still in use fails or, worse, corrupts data if forced blindly — a genuinely destructive operation.
> **Implementation:**
> ```bash
> # INSPECT — is anything using this mount right now?
> fuser -vm /mnt/data
> lsof +D /mnt/data
>
> # VALIDATE — for each process found: stale handle, or a live service that
> #            should be stopped cleanly first rather than force-killed?
>
> # EXECUTE — normal path first, force only as an informed last resort
> umount /mnt/data
> umount -l /mnt/data     # lazy unmount — only once you've validated it's safe
>
> # VERIFY
> mount | grep /mnt/data
> df -h
> ```
> **Failure:** `umount` fails with "target is busy."
> **Investigation:** `fuser -vm` shows a process you don't recognize holding it open.
> **Fix:** Identify what that process actually is (`ps -p <pid>`) before killing anything — stop it cleanly if it's a legitimate service.
> **Verification:** Retry the unmount after a clean stop and confirm success.
> **Interview Explanation:** "I never force an unmount blind. I check what's holding it open with fuser or lsof, confirm it's safe to stop, and only reach for a lazy unmount as a last resort once I've already validated that — a forced unmount during an active write risks real data corruption."

> [!NOTE]
> **Practice Scenario (rehearsal only — not a personal account):** The following is a worked example for practicing the incident-narration format, not something to present as your own history. Use it to internalize the pattern, then tell your *real* stories (Story Bank, below) the same way.
>
> *A `/var` partition hits 98% at 2am. The obvious suspect is a runaway app log. `df -h` confirms 98% used, but `df -i` shows inodes are fine — this is bytes, not file count. `du -xh --max-depth=1 /var | sort -rh | head` points at `/var/log/myapp`, where a single 40GB log sits. Someone already tried `rm`-ing it an hour ago — `df` barely moved. `lsof +L1` reveals why: the deleted inode is still held open by a long-running worker that never reopened its log handle. Immediate fix: if a safe reload/restart is available, reload or restart the owning process so it closes the deleted file descriptor and releases the space; if service interruption is unacceptable, treat descriptor-level recovery as an advanced emergency technique rather than the default. After recovery, verify `df -h` and service health. Real fix: a proper `logrotate` policy with an application-supported reopen mechanism when available, or `copytruncate` when the application cannot reopen logs; validate it with `logrotate -d` before trusting it.*
>
> The lesson worth keeping regardless of whose story it is: **`rm` unlinks a directory entry; disk space held by an already-open deleted file is not released until the final open reference is closed.** That distinction — not the incident itself — is what's worth having ready. `truncate -s 0` is a separate technique that changes an existing file in place and destroys its current contents, so it should be used deliberately rather than treated as a universal emergency fix.

> [!TIP]
> **Interview Answer:** "df shows how full each mounted partition is; du breaks down usage inside a specific directory. Splitting things like /var onto their own partition is a deliberate safety measure — so a runaway log file fills up its own dedicated space instead of taking down the entire root filesystem."
---

# Part 2 — Deeper Mechanics

## Signals & Process Control

**Why it exists:** Sometimes you need to communicate with a running process from the outside — ask it to shut down cleanly, tell it to reload its config, or force-kill it if it's unresponsive. Signals are the kernel's mechanism for delivering these short, standardized messages to a process.

**Step-by-step flow:**
1. You (or another process, or the kernel itself) send a signal to a target process by PID — e.g., `kill -15 1234`.
2. The kernel delivers that signal to the target process.
3. If the process has registered a "handler" for that signal, its own code runs in response (e.g., saving state, closing connections cleanly before exiting).
4. If it hasn't registered a handler, the kernel applies that signal's default behavior instead (for most signals, that default is simply terminating the process).

**Components involved:** Signal numbers/names, the `kill` command (despite the name, it just *sends a signal* — which one determines what actually happens), signal handlers inside application code.

**Exact identifiers:** `SIGTERM` (15) — "please exit gracefully," the default and polite request. `SIGKILL` (9) — immediate, unconditional termination; cannot be caught, blocked, or handled by the process at all. `SIGHUP` (1) — traditionally "the controlling terminal hung up," commonly repurposed by daemons today to mean "reload your config file." `SIGINT` (2) — what Ctrl+C sends.

**Decision point:** `SIGTERM` versus `SIGKILL` is the single most important distinction here — `SIGTERM` gives a process a chance to clean up (close database connections, finish writing a file) before exiting; `SIGKILL` gives it zero chance, the kernel just removes it from existence instantly. This is exactly why `kill` (which defaults to SIGTERM) is tried first, and `kill -9` is the last resort when a process refuses to respond to the polite request.

**Plain example:** SIGTERM is knocking on someone's door and asking them to finish up and leave. SIGKILL is having security physically remove them right now, mid-sentence, with zero warning — they don't even get a chance to grab their coat.

> [!NOTE]
> **Anchor Fact:** `SIGTERM` = 15, catchable, graceful. `SIGKILL` = 9, uncatchable, instant, no cleanup possible. Always try SIGTERM first.

> [!WARNING]
> **Failure Scenario:** You run `kill <pid>` (SIGTERM) on a hung process, and it just... stays alive, doing nothing differently. **Symptom:** process ignores termination request. **Hypothesis:** either it's stuck in an uninterruptible kernel-level wait (commonly disk I/O), or its code has a signal handler that's supposed to clean up but is itself hanging. **Evidence:** `ps -o stat` shows process state — a `D` state specifically means "uninterruptible sleep," usually waiting on disk I/O, and it genuinely cannot respond to any signal except by that I/O completing. **Root cause:** either a slow/failing disk, or a buggy shutdown handler. **Fix:** for a true `D`-state hang, you often have to wait it out or address the underlying storage issue — `kill -9` won't even work here, since the process can't respond to any signal while stuck in that kernel state. **Prevention:** timeouts on all I/O operations in application code, and disk health monitoring.

> [!TIP]
> **Interview Answer:** "Signals are how you communicate with a running process from the outside — SIGTERM asks it to exit gracefully and can be caught and handled; SIGKILL terminates it instantly with zero chance to clean up and can't be caught at all. Best practice is always to try SIGTERM first and only escalate to SIGKILL if the process genuinely won't respond."

> [!IMPORTANT]
> 🔧 **HANDS-ON LAB — The kill -9 that won't kill**
> **Concept:** The pressure-question above ("SIGTERM didn't work and neither did SIGKILL — now what?") becomes trivial once you've *seen* a D-state process.
> **Implementation:** Observe one during heavy disk load: `ps -eo pid,stat,comm | awk '$2 ~ /^D/'` while running `stress-ng --io 4` on a throwaway VM.
> **Hands-on:** For a candidate PID, `cat /proc/<pid>/stack` (as root) shows the kernel function it's sleeping in — often `io_schedule` or an NFS wait. That's proof it's parked in the kernel, not ignoring you.
> **Failure:** `kill -15`, then `kill -9` — the process stays. `kill -0 <pid>` still succeeds; state stays `D`.
> **Investigation/Fix:** It leaves D-state only when the underlying I/O completes or errors out. Real levers: fix/restart the storage, unmount the stalled filesystem, or reboot. `kill -9` was never going to work.
> **Verification:** Once the I/O resolves, the process either completes or is reapable normally.
> **Interview Explanation:** "D-state means the process is inside a kernel code path that can't be interrupted — no signal, including 9, is delivered until that path returns. So the answer is never 'try harder signals'; it's 'find out what I/O it's waiting on' — /proc/pid/stack, iostat, storage logs."

---

## File Descriptors & the Open File Table

**Why it exists:** A running process needs a lightweight way to refer to any open file, network socket, or pipe it's using, without repeatedly looking up the full path or address every time. A file descriptor is just a small integer that a process uses as a handle to something it has open — this same mechanism underlies files, network connections, and even keyboard/screen I/O.

**Step-by-step flow:**
1. When a process opens a file (or a network socket), the kernel doesn't hand back the file itself — it hands back a small integer, the file descriptor, that refers to an entry in the kernel's own open-file table.
2. Every subsequent read/write from that process references the file only by that integer.
3. Three file descriptors are opened automatically for every process at startup: `0` = standard input, `1` = standard output, `2` = standard error.
4. When the process closes the descriptor (explicitly, or by exiting), the kernel's table entry is released — and if that was the last reference to that file, the file's actual disk resources are freed too (this is the exact mechanism behind the deleted-but-open-file disk space situation from Section 5).

**Components involved:** File descriptor table (per-process), the kernel's global open-file table, `lsof` as the tool that inspects this.

**Exact identifiers:** FD `0`, `1`, and `2` conventionally represent stdin, stdout, and stderr. Most processes inherit them by default, but they can be closed or redirected. `lsof -p <pid>` lists every file descriptor a specific process currently holds open.

**Decision point:** This is exactly why "delete a log file to free space" sometimes silently doesn't work — if a running process still holds that file open (has its file descriptor pointing at it), deleting the filename doesn't touch the kernel's underlying file table entry; the disk space stays occupied until that process closes the descriptor or exits.

**Plain example:** A file descriptor is like a coat-check ticket — the ticket itself is just a small number, but it's your only way to reference which coat (file/socket) you're actually dealing with. Throwing away the coat rack's label (deleting the filename) doesn't make the coat (the disk space) disappear if someone is still holding a valid ticket for it.

> [!NOTE]
> **Anchor Fact:** FD 0/1/2 conventionally mean stdin/stdout/stderr, and every process gets them by default — but they're just convention plus default inheritance, not an unbreakable guarantee: a process can close them or have them redirected before it even starts. This convention is the basis of shell redirection (`2>&1` means "send stderr wherever stdout is currently going").

> [!WARNING]
> **Failure Scenario:** Redirecting output with `command > out.log 2>&1` doesn't seem to capture everything you expect. **Symptom:** some output missing from the log file. **Hypothesis:** ordering of redirections matters — `2>&1` needs to come *after* `> out.log` to correctly redirect stderr to wherever stdout was just pointed. **Evidence:** compare `command > out.log 2>&1` (correct — both go to out.log) versus `command 2>&1 > out.log` (stderr goes to the terminal, only stdout goes to the file, because at the moment `2>&1` executed, stdout was still pointing at the terminal). **Root cause:** shell redirections are applied left to right, and `2>&1` duplicates wherever FD 1 currently points *at that moment*, not wherever it ends up pointing later. **Fix:** always redirect stdout to the file first, then redirect stderr to follow it. **Prevention:** understand redirection order rather than pattern-matching it.

> [!TIP]
> **Interview Answer:** "A file descriptor is a small integer a process uses to reference an open file, socket, or pipe — every process starts with FDs 0, 1, and 2 for stdin, stdout, and stderr. This is also why deleting a file doesn't always free its disk space immediately: the filename is separate from the kernel's actual open-file entry, which only goes away once every process holding a descriptor to it closes it."

> [!IMPORTANT]
> 🔧 **HANDS-ON LAB — The "too many open files" incident**
> **Concept:** A common real production failure for 3–5 YOE rounds: a service starts failing with "Too many open files" after hours of uptime.
> **Implementation:**
> ```bash
> ulimit -n                                  # soft limit (shell) — does NOT apply to systemd services
> cat /proc/<pid>/limits | grep "open files"  # the limit that actually applies to this process
> ls /proc/<pid>/fd | wc -l                   # actual usage right now
> lsof -p <pid> | awk '{print $5}' | sort | uniq -c | sort -rn   # what KIND dominates
> ```
> **Hands-on:** A growing socket count is a connection leak; one specific file opened thousands of times is a missing `close()` in application code. Note systemd services get their limit from the unit's `LimitNOFILE=`, not the shell's `ulimit`.
> **Failure:** The service crashes with `OSError: [Errno 24]` (EMFILE) after running fine for hours.
> **Investigation:** `lsof -p` grouped by type shows the leak growing steadily over time, not a sudden spike.
> **Fix:** Immediate — raise `LimitNOFILE=` and restart. Real fix — the application must close what it opens.
> **Verification:** `lsof` count flat-lines over time instead of climbing.
> **Interview Explanation:** "EMFILE is frequently a descriptor leak rather than a misconfigured limit — the limit is just where you hit the wall. lsof -p grouped by type tells you what's actually leaking, and that's what I'd fix, not just the ceiling."

---

## The Boot Process

**Why it exists:** Getting from "power is applied" to "a fully running, usable Linux system" involves handing control through several distinct stages, each responsible for a specific part of bringing the machine to life — understanding this sequence is what lets you diagnose exactly *where* a machine is failing to start.

**Step-by-step flow:**
1. **Firmware (BIOS/UEFI)** runs first — performs a basic hardware check (POST) and locates a bootable device.
2. **Bootloader** (commonly GRUB on Linux) loads from that device — its job is simply to locate and load the actual Linux kernel into memory, and hand control to it.
3. **Kernel initializes** — sets up memory management, detects hardware, mounts an initial temporary root filesystem, then mounts the real root filesystem.
4. **init/systemd (PID 1) starts** — the kernel hands off control to PID 1, which then starts every other configured service in the correct dependency order, eventually reaching a fully running system (a systemd "target," like `multi-user.target` or `graphical.target`).

**Components involved:** Firmware/BIOS/UEFI, the bootloader (GRUB), the kernel image, `initramfs` (temporary early filesystem), systemd targets.

**Exact identifiers:** `multi-user.target` = normal, full multi-user system without a GUI (typical for a server). `graphical.target` = multi-user plus a graphical desktop. `systemctl get-default` shows which target a system boots into by default.

**Decision point:** A failure at the bootloader stage (can't find/load a kernel) looks completely different from a failure after the kernel loads but before systemd finishes bringing services up — the first typically shows GRUB-level errors before Linux even starts printing anything; the second shows kernel boot messages that then stall or drop into an emergency/rescue shell partway through startup. Correctly identifying *which* stage failed tells you where to actually look.

**Plain example:** Think of boot like a relay race with four runners: firmware (hardware self-check, hands off to) → bootloader (finds and loads the kernel, hands off to) → kernel (sets up the core system, hands off to) → systemd (brings every actual service online). If the race stalls, knowing which runner is currently holding the baton tells you exactly where to look — a GRUB error means runner two failed before ever passing to runner three.

> [!NOTE]
> **Anchor Fact:** Boot order is always: Firmware/BIOS-UEFI → Bootloader (GRUB) → Kernel → init/systemd (PID 1) → target services. Each stage fully hands off to the next; a failure in one stage never lets you see what would've happened in the next.

> [!WARNING]
> **Failure Scenario:** A server doesn't come back up after a routine reboot. **Symptom:** SSH unreachable post-reboot. **Hypothesis:** could be failing anywhere in the boot chain — need console/serial access to actually see, since SSH itself depends on the system having fully booted. **Evidence:** console access shows exactly which stage it's stuck at — stuck at a GRUB menu (bootloader-stage problem, often a bad kernel entry), a kernel panic message (kernel-stage problem, often hardware or a bad kernel update), or dropped into an emergency shell with a specific failed unit named (systemd-stage problem, e.g., a filesystem that failed to mount). **Root cause:** varies entirely by which stage it's stuck at — this is exactly why identifying the stage first matters more than guessing at a fix. **Fix:** stage-specific — e.g., booting the previous kernel version from the GRUB menu if a recent kernel update is the culprit. **Prevention:** never let a routine change (kernel update, fstab edit) go untested on a non-critical system first.

> [!TIP]
> **Interview Answer:** "Boot happens in a strict handoff chain — firmware finds and loads a bootloader, the bootloader loads the kernel, the kernel sets up the core system and starts PID 1, and systemd brings every service up according to its target. When a boot fails, the first diagnostic step is figuring out which stage it actually got stuck at, because a GRUB-stage failure and a systemd-stage failure need completely different fixes."

> [!IMPORTANT]
> 🔧 **HANDS-ON LAB — fstab: a frequently self-inflicted boot failure**
> **Concept:** A bad `/etc/fstab` entry (for example, a device or UUID that no longer exists) is a recurring real-world boot-failure scenario worth practicing for both interviews and production. Practice on a VM you can snapshot.
> **Implementation:**
> ```bash
> # The trap: add a line for a device that won't exist
> echo "UUID=deadbeef-dead-beef-dead-beefdeadbeef /data ext4 defaults 0 0" | sudo tee -a /etc/fstab
>
> # Validate BEFORE rebooting — the habit that saves you
> sudo mount -a          # mounts everything in fstab NOW; the error shows immediately
> findmnt --verify        # checks fstab syntax directly
> ```
> **Failure:** Reboot the VM — systemd drops to **emergency mode**, naming the failed mount unit. This is a stage-4 (systemd) boot failure.
> **Investigation:** The emergency shell is your workspace — it tells you exactly which unit failed.
> **Fix:** Comment out the bad line (remount root read-write first if needed: `mount -o remount,rw /`), then `systemctl daemon-reload` and `exit` — boot resumes.
> **Verification:** Reboot again cleanly and confirm normal startup.
> **Interview Explanation:** "Before any fstab change on a production box: mount -a to validate live, and use nofail or x-systemd.device-timeout on non-critical mounts so a missing disk degrades the boot instead of killing it. Recovery is console access → emergency shell → fix or comment the line → continue boot."

---

## Logging: journald & syslog

**Why it exists:** Every service and the kernel itself produce log output constantly, and you need a consistent, searchable way to collect, store, and query all of it — rather than every application inventing its own separate logging scheme with no common way to search across them.

**Step-by-step flow:**
1. On a modern systemd-based system, `journald` collects log output directly from every systemd-managed service, the kernel, and anything writing to the traditional syslog socket.
2. It stores this centrally in a structured, indexed binary format (not plain text files) — which is what makes precise filtering fast.
3. `journalctl` is the query tool — you can filter by service (`-u`), by time (`--since`), by severity (`-p err`), or follow it live (`-f`).
4. Many systems also still run traditional syslog-style plain-text logs (e.g., `/var/log/messages`) either alongside or forwarded from journald, especially where older tooling expects plain text files.

**Components involved:** `journald` (the collector/storage daemon), `journalctl` (the query tool), traditional syslog files under `/var/log/`.

**Exact identifiers:** `journalctl -u myapp -f` = live-follow one service's logs. `journalctl -p err` = only error-severity and above, across everything. `journalctl --since "10 min ago"` = time-window filtering. `journalctl -k` = kernel messages only.

**Decision point:** journald's structured, indexed storage makes precise queries (by service, by severity, by time range) fast without needing external tools — but it's a binary format, not readable with plain `cat`/`grep` directly on its files the way old-style flat log files were, which is exactly why `journalctl` exists as the required interface to it.

**Plain example:** Old-style syslog files are like a single, ever-growing paper notebook everyone scribbles into — searchable only by physically flipping through it. journald is like a proper searchable database of the same information — you can instantly ask "just the errors from this one service in the last ten minutes" instead of scrolling through everything.

> [!NOTE]
> **Anchor Fact:** `journalctl -u <service> --since "10 min ago" -p err` combines service filter, time filter, and severity filter in one line — this exact combination is worth having memorized cold.

> [!WARNING]
> **Failure Scenario:** A service crashed hours ago, and you need to know exactly why, but `systemctl status` only shows the last few lines. **Symptom:** insufficient context from a quick status check. **Hypothesis:** the real error is further back in the log, before the final crash message. **Evidence:** `journalctl -u myapp --since "2 hours ago"` pulls the full window, not just the tail; combined with `-p err` to cut noise. **Root cause:** varies, but the discovery method is what matters here — the fix is usually visible once you see the actual error, not just the final "process exited" line. **Prevention:** for anything mission-critical, ship logs to a centralized system (not covered in this doc — see the Observability topic later in the series) so historical logs survive even if the local journal is size-capped and rotates away old entries.

> [!TIP]
> **Interview Answer:** "journald centrally collects logs from every systemd-managed service and the kernel into a structured, indexed store, queried through journalctl — filterable by service, time window, and severity in a single command. It replaces having to manually grep through scattered plain-text log files, though traditional syslog-style files often still exist alongside it for compatibility."

> [!IMPORTANT]
> 🔧 **HANDS-ON LAB — The investigation triad: logs, disk, and journal rotation**
> **Concept:** Logging rounds often blend into disk rounds: "the journal is filling /var — what do you do?" is a real production scenario testing whether you understand journald's storage model.
> **Implementation:**
> ```bash
> journalctl --disk-usage                                                    # journal's own footprint
> grep -E 'SystemMaxUse|SystemKeepFree|MaxRetentionSec' /etc/systemd/journald.conf  # retention caps
> df -h /var                                                                  # is /var actually filling?
> ```
> **Failure:** Journal pinned at its cap plus a chatty service means old entries get evicted within minutes — the crash evidence from 3 hours ago is already gone when you look.
> **Investigation:** `journalctl --since "3 hours ago" -u <chatty-service>` on a busy machine shows the gap directly.
> **Fix:** Adjust `SystemMaxUse`/`SystemKeepFree` AND fix the chatty logging at the app level — size caps alone only treat the symptom.
> **Verification:** `journalctl --disk-usage` after a journald restart, watching the rate with `journalctl -f`.
> **Interview Explanation:** "journald is a database with a retention policy, not an infinite archive. For anything serious I'd ship logs off-box, because the local journal is exactly what a full disk or an eviction policy destroys first — usually right when you need it most."

---

## The Performance Troubleshooting Toolkit

**Why it exists:** "The server is slow — walk me through what you do" is one of the most reliable questions in 3–5 YOE DevOps/SRE rounds, and interviewers are usually looking for a *method*, not a favorite command. The widely-taught method is the **USE method** (Brendan Gregg): for every resource — CPU, memory, disk, network — check **U**tilization, **S**aturation, **E**rrors. A structured checklist keeps you from flailing during a real incident.

**The first 60 seconds — baseline triage, run in order, say what each rules in or out:**

| Command | What it tells you | First red flags |
|---|---|---|
| `uptime` | load average (1/5/15 min) | load >> core count, and still climbing |
| `dmesg -T \| tail` | kernel errors right now | I/O errors, OOM kills, NIC resets |
| `vmstat 1` (5 rows) | runnable/blocked procs, memory, swap in/out, CPU | `r` high + `b` high; `si/so` > 0 |
| `mpstat -P ALL 1` | per-CPU breakdown | one hot core (locking); all cores sys-heavy |
| `pidstat 1` | per-process CPU by default | one process eating everything |
| `iostat -xz 1` | per-disk utilization, await, queue depth | `%util` ~100 + high `await` |
| `free -m` | memory truth | `available` low + swap active |
| `sar -n DEV 1` | per-NIC throughput | interface at wire speed |
| `sar -n TCP,ETCP 1` | TCP retransmits, resets | retransmit % > 0 |
| `top` / `htop` | the interactive synthesis | everything above, live |

> [!NOTE]
> **Anchor Fact:** The 60-second checklist: `uptime`, `dmesg | tail`, `vmstat 1`, `mpstat -P ALL 1`, `pidstat 1`, `iostat -xz 1`, `free -m`, `sar -n DEV 1`, `sar -n TCP,ETCP 1`, `top`. Saying this checklist out loud is itself a passing answer to "how do you approach a slow server."

> [!TIP]
> **Interview Answer:** "I don't start guessing — I run a fixed 60-second triage: uptime and dmesg for the lay of the land, then vmstat/mpstat/pidstat for CPU, iostat for disk, free for memory, sar for network. Each tool answers one question — utilization, saturation, or errors per resource — and by the end of the minute I know which resource is the bottleneck and which process is holding it."

> [!IMPORTANT]
> 🔧 **HANDS-ON LAB — Generate each bottleneck and diagnose it (the highest-value lab in this guide)**
> **Concept:** Recognizing the *signature* of each bottleneck in the tools above is the actual skill being tested.
> **Implementation** (throwaway VM with `stress-ng`):
> 1. **High CPU:** `stress-ng --cpu 4`. Signature: `vmstat` → `r` ≈ 4, `us` ~100%; `pidstat` names `stress-ng`; all cores busy evenly. Load is high but the box stays *responsive*.
> 2. **High I/O:** `stress-ng --io 4 --hdd 2`. Signature: `iostat -xz 1` → `%util` pegged, `await` in the hundreds of ms; `vmstat` → `b` climbing; load explodes *without* high CPU.
> 3. **High memory + swap:** `stress-ng --vm 2 --vm-bytes 90%`. Signature: `free -m` → `available` collapsing, `si/so` active in vmstat; ends with an OOM kill in `dmesg`.
> 4. **Network (observation-only):** `iperf3 -s` on a second host, `iperf3 -c <host>` on the target. Signature: `sar -n DEV 1` shows the NIC at capacity.
> **Verification:** For each run, note the *three* tools that identified it fastest — until you can glance at `vmstat` and call the bottleneck class in under 5 seconds.
> **Interview Explanation:** Close every such question with the loop: triage (checklist) → isolate (which resource + which PID) → root cause (strace/lsof/logs for that process) → fix → verify (same tool showing the metric back to normal) → prevent (limits/alerts).
---

# Part 3 — Interview Corner

Each "drill" below hits all nine question angles: Basic, Why, How, Scenario, Troubleshooting, Architecture, Comparison, Follow-up, Pressure. These four topics get the full treatment because they're the ones interviewers push hardest on. The Quick-Coverage Round after gives solid coverage on everything else without the full nine angles each.

## Processes & Signals Drill

1. **Basic:** What is a process? — An instance of a running program, with its own memory, file descriptors, and a unique PID.
2. **Why:** Why does Linux separate `fork()` and `exec()` into two steps instead of one "start new program" call? — Because `fork()` alone (duplicating the current process) is useful on its own for creating worker processes, and having them separate gives more flexibility than a single combined operation would.
3. **How:** How does the kernel actually decide which process gets CPU time next? — Through its scheduler, which assigns time slices based on priority and fairness algorithms, switching between processes fast enough that multiple programs appear to run simultaneously even on a single CPU core.
4. **Scenario:** A process's parent has already exited — what happens to it? — It becomes an "orphan." Normally it gets re-parented to PID 1 (systemd/init), which reaps it when it eventually exits — though if the original parent had designated a "subreaper" (via `prctl(PR_SET_CHILD_SUBREAPER)`, which some process supervisors set up deliberately), that subreaper receives the orphan first instead of PID 1. Either way, it won't become a permanent zombie.
5. **Troubleshooting:** `top` shows a process consuming 100% CPU constantly — what's your process? — Identify it precisely with `ps aux --sort=-%cpu`, check what it's actually doing with `strace -p <pid>` (is it stuck in a tight loop, or legitimately working), check its logs for repeated errors that might indicate a retry loop, and only then decide whether to restart it or let it finish.
6. **Architecture:** How would you design a system so one runaway process can't take down an entire server? — Use cgroups to cap CPU/memory per process or service, run risky workloads in containers (which use cgroups and namespaces under the hood for exactly this isolation), and monitor with alerts on sustained high resource usage.
7. **Comparison:** `kill` vs `pkill` vs `killall` — what's the actual difference? — `kill` targets one specific PID. `pkill` matches processes by name/pattern and can signal multiple matches at once. `killall` also matches by exact name but historically behaves differently across Unix variants — on Solaris, `killall` used to kill ALL processes, which is why it's worth being cautious and explicit about which tool you're using in scripts.
8. **Follow-up:** If you `kill -9` a process, does its parent get notified? — Yes — the kernel still delivers a `SIGCHLD` to the parent, and the parent can still read the exit status showing it was killed by SIGKILL, even though the process itself never got to run any cleanup code.
9. **Pressure:** You've sent SIGTERM, waited, and the process still hasn't exited — SIGKILL also doesn't seem to remove it from `ps`. What now? — Check its state with `ps -o stat` — a `D` state (uninterruptible sleep, usually disk I/O) cannot respond to *any* signal, including SIGKILL, until the underlying I/O operation completes; at that point the fix is addressing the stuck I/O (failing disk, unresponsive NFS mount), not the process itself.

**Additional angles worth having ready:**

10. **Deep-dive — "What happens when you type `ls` and press enter?"** — A real attention-to-detail question. Expected arc: shell tokenizes the line → checks aliases/builtins → resolves `ls` via `PATH` → `fork()`s a child → child `execve()`s the binary → kernel's loader maps libraries → `ls` issues syscalls against directory inodes → results print to FD 1 → `_exit()` → parent reaps it via `wait()` (the next prompt appearing IS that reap). Bonus: mention you've actually run `strace ls`.
11. **Process vs thread** — A thread shares its process's address space and file descriptors (cheap to create, but one bad write can corrupt siblings); a process is fully isolated. Real anchor: why a runaway thread is harder to contain than a runaway process — cgroups cap the whole process/container, not one thread within it.
12. **Live task — "Find the process using the most memory and kill it politely."** — `ps aux --sort=-%mem | head`, confirm with `pidstat -r 1`, `kill -TERM` first, escalate to `-9` only if needed, verify with `ps -p <pid>`. The politely-first ordering is what's actually being scored.

---

## Permissions & Security Drill

1. **Basic:** What do the three permission groups (owner/group/other) control? — Who can read, write, or execute a given file, evaluated separately for the file's owner, its group, and everyone else.
2. **Why:** Why does Linux check group membership at all, instead of just owner vs. everyone else? — Because real systems need a middle tier — a team of trusted users who need shared access to something, without opening it to literally everyone on the box.
3. **How:** How does the kernel actually decide whether to allow a file operation? — It checks the requesting process's effective UID/GID against the file's owner UID, group GID, and permission bits, in a specific order — owner check first, then group, then other — stopping at the first applicable match.
4. **Scenario:** A user is in the correct group for a file, but still gets "permission denied" — why might that happen? — The user's current login session may predate being added to that group; group membership is loaded at login, so they need to log out and back in (or run `newgrp`) for the change to take effect in their active session.
5. **Troubleshooting:** A cron job that runs fine when you execute it manually fails silently when run by cron — where do you look? — Cron jobs run with a minimal environment (no interactive shell config loaded), so check for hardcoded assumptions about `PATH` or environment variables in the script, and check cron's own logs (`grep CRON /var/log/syslog` or `journalctl -u cron`) for the actual failure reason rather than assuming it's a permissions issue at all.
6. **Architecture:** How would you set up a shared directory for a team where everyone in the group can read/write files, but files aren't accidentally left owner-only? — Set the directory's group ownership to the team's group, apply the `setgid` bit on the directory (so new files automatically inherit the directory's group instead of the creating user's primary group), and set group permissions to read/write.
7. **Comparison:** SUID vs SGID vs the sticky bit — what does each actually do? — SUID on an executable makes it run with the *file owner's* privileges rather than the invoking user's (classic example: `passwd`, which needs root privileges to edit `/etc/shadow` even when a regular user runs it). SGID on a directory makes new files inherit the directory's group. The sticky bit on a directory (like `/tmp`) means users can only delete their own files, even if they technically have write access to the whole directory.
8. **Follow-up:** Why is SUID considered a security risk if misused? — Because it means running that binary temporarily grants the invoking user the file owner's privileges — if that binary has a bug (like a buffer overflow) that lets an attacker execute arbitrary code, they inherit root's privileges too, not just the attacker's own limited account.
9. **Pressure:** You find a script somewhere on a production server with SUID set and owned by root, and you don't know who put it there or why. What do you do? — Treat it as a potential security concern rather than something to fix quietly — check its contents and purpose before touching it, check file timestamps and any audit/access logs to see when it appeared, and remove the SUID bit (or the file entirely, if unjustified) only after understanding what depends on it, escalating if the origin can't be explained.

**Additional angles worth having ready:**

10. **Scenario — "SSH says my key is ignored: 'Permissions 0644 for id_rsa are too open.'"** — An extremely common real failure. Cause: OpenSSH refuses private keys readable by anyone but the owner. Fix: `chmod 700 ~/.ssh`, `chmod 600` on private keys and `authorized_keys`, `644` on public keys. The underlying principle: ssh enforces *minimum* permission requirements on secrets — a concrete illustration of permissions as enforced security.
11. **Implementation — "Audit this server for permission problems."** — Expected toolkit: `find / -xdev \( -perm -4000 -o -perm -2000 \)` (SUID/SGID inventory), `find / -xdev -perm -0002` (world-writable), `find / -xdev \( -nouser -o -nogroup \)` (files whose recorded owner/group no longer resolves), `awk -F: '$3==0' /etc/passwd` (inventory all UID-0 accounts and investigate any unexpected ones).
12. **Troubleshooting — "chmod 750 on a directory, but someone in the group still can't cd into it."** — Execute on a directory means traversal, and `750` does grant the group that — unless a *parent* directory lacks execute for them; permission checks apply at every path component, not just the final one. Also check for the reverse classic: a script that "won't run" due to Windows line endings (`cat -A script.sh` reveals `^M$`), not permissions at all.

---

## Disk & Troubleshooting Drill

1. **Basic:** What's the difference between `df` and `du`? — `df` reports space usage at the whole-filesystem/partition level; `du` reports usage broken down by directory or file.
2. **Why:** Why would you put `/var` on its own separate partition instead of leaving everything under one root partition? — So runaway growth in logs or variable data (which live under `/var`) fills up only that dedicated partition, instead of consuming the same space the OS itself needs to keep running.
3. **How:** How does Linux actually reclaim disk space when a file is deleted? — Deleting a file removes its directory entry (its name), decrementing the inode's link count; if that count hits zero AND no process still has it open, the kernel frees the inode and its underlying disk blocks — otherwise, the space stays occupied until both conditions are true.
4. **Scenario:** You need to add capacity to a server that's completely full, right now, with no time to provision new storage — what's the safe way to find something to clear? — Never blindly delete without checking what's actually consuming space — sort with `du -sh /path/* | sort -rh` to find the biggest directories, check for obviously safe targets (old rotated logs, package manager caches, temp files), and confirm nothing important still references them before removing anything.
5. **Troubleshooting:** `df` and `du` disagree — `df` says a partition is 90% full, `du` on that same mount only accounts for 50%. Walk through it. — Check for a process holding a deleted-but-open file with `lsof +L1` first (the classic cause); if that comes up empty, also check for another filesystem mounted inside a subdirectory of this one that `du` might be traversing into unexpectedly, or reserved blocks that `df` counts but aren't visible as regular files at all (like ext4's reserved root-only space).
6. **Architecture:** How would you design disk monitoring for a fleet of servers so you're never caught by surprise? — Alert on percentage-full thresholds per mount point individually (not just root), alert on rate-of-growth (a slowly filling disk is different urgency than one filling in minutes), and separately monitor inode exhaustion — a filesystem can run out of inodes (available file "slots") even with plenty of raw byte space left, if it's storing huge numbers of tiny files.
7. **Comparison:** Traditional partitioning vs LVM (Logical Volume Manager) — why would you choose LVM? — Traditional partitions are fixed in size once created without significant hassle; LVM adds a flexible layer that lets you resize volumes, add new physical disks into an existing volume group, and take snapshots — at the cost of a bit more complexity to understand and manage.
8. **Follow-up:** If inodes run out but there's plenty of free byte space left, what does that actually mean happened? — The filesystem has an enormous number of very small files — each file consumes one inode regardless of its size, so a directory tree with millions of tiny files can exhaust the fixed number of inodes a filesystem was formatted with, long before the raw disk space itself is full.
9. **Pressure:** A production database server's disk fills up mid-transaction and writes start failing. What's your immediate action, and what do you deliberately NOT do? — Immediately check what can be safely freed without touching database files themselves (old logs, temp files, package caches) to buy breathing room; deliberately do NOT delete or truncate anything inside the database's own data directory, and do NOT restart the database blindly, since an abrupt restart mid-write on a full disk risks data corruption — stabilize disk space first, then let the database recover cleanly.

**Additional angles worth having ready:**

10. **Scenario (a frequently reported storage question) — "Disk hits 100% on a production app server. Walk me through your investigation."** — Score comes from *sequencing*: triage (`df -h` which partition, `df -i` inodes) → locate (`du -xh --max-depth=1` largest-first, `-x` to stay on the filesystem; `lsof +L1` for the deleted-but-open ghost) → mitigate safely (free only obviously-safe space, one action at a time, verifying with `df` after each) → stabilize (truncate or rotate a live log properly rather than `rm` under a running writer) → prevent (logrotate plus per-mount alerts at 75–80%). Ending at "I deleted the big file" misses the prevention half — mentioning `lsof +L1` unprompted stands out.
11. **Comparison — `truncate -s 0 file` vs `rm file` on a live log.** — `rm` unlinks the name; the space stays held until every writer closes its file descriptor (and the app may break logging). `truncate` zeroes the existing file in place: the inode and open descriptors remain valid, and the space used by that file is reclaimed immediately, but the existing contents are destroyed. Long-term fix is still logrotate — with `copytruncate` for apps with no reopen signal, worth knowing its trade-off: a small amount of log data written between the copy and the truncate can be lost, since it's not an atomic operation.
12. **Hands-on probe — "Add 5GB of space to /var right now."** — With LVM: `df -h /var` → `lsblk`/`pvs`/`vgs` (free space in the volume group?) → `lvextend -L +5G /dev/mapper/vg-var` → `xfs_growfs /var` (XFS, online) or `resize2fs` (ext4, online after lvextend) → verify with `df -h`. Worth mentioning unprompted: XFS cannot be shrunk, and both grow operations are online/safe.

---

## systemd & Services Drill

1. **Basic:** What's a unit file? — A configuration file telling systemd how to start, stop, and manage a specific service — what command to run, as which user, and what to do on failure.
2. **Why:** Why does systemd support automatic restart-on-failure as a built-in feature instead of leaving that to the application itself? — Because not every application is written to supervise and restart itself, and centralizing this in the init system means consistent, predictable recovery behavior across every service on the box, configured the same way regardless of what the application itself does internally.
3. **How:** How does `systemctl status` figure out whether a service is actually healthy, not just running? — By default, it primarily reports whether the main process is still alive (based on the PID it's tracking) — it does NOT automatically know if the *application* is functionally healthy (e.g., responding correctly to requests) unless that service specifically implements a more thorough readiness check systemd is configured to use.
4. **Scenario:** You `enable`d a service but after a reboot, it's still not running — what's the first thing to check? — `systemctl status` on it directly — it may have tried to start and immediately failed (a startup error), which is a completely different situation from never having been told to start in the first place; `enable` alone doesn't guarantee success, only that systemd attempted it.
5. **Troubleshooting:** A service shows `active (running)` in `systemctl status`, but the application isn't actually responding to any requests. — This is exactly the gap in question 3 — the process is alive, but not functionally working. Check the application's own logs (`journalctl -u service`) for internal errors, check if it's listening on the expected port at all (`ss -tulnp | grep <port>`), and consider whether the unit file needs a proper health-check mechanism rather than relying on "process exists" as the only signal of health.
6. **Architecture:** How would you set up a service so it only starts after its database dependency is actually ready, not just "started"? — Unit file dependency ordering (`After=`, `Requires=`) controls *start order*, but does NOT wait for the dependency to be functionally ready — for true readiness, you typically need the application itself to retry its connection on startup, or use a wrapper/init script that actively waits for the dependency to respond before launching the main process.
7. **Comparison:** `Restart=on-failure` vs `Restart=always` — what's the practical difference? — `on-failure` restarts on "failure," which is broader than just a non-zero exit code — it also covers the process being terminated by an unclean signal, a watchdog timeout, or an operation timeout, depending on `SuccessExitStatus=` config. `always` restarts regardless of how it exited, even a clean, intentional exit — meaning `always` on a service that's supposed to run once and stop would create an infinite restart loop.
8. **Follow-up:** What happens if a service is stuck in a restart loop because of `Restart=always` on a persistently broken configuration? — systemd has built-in rate-limiting for restarts (`StartLimitBurst`/`StartLimitIntervalSec`) — after too many restart attempts in too short a window, systemd will stop trying and mark the unit as failed, rather than restart-looping forever and burning CPU.
9. **Pressure:** You need to change a systemd unit file's restart behavior on a production service right now, but you're worried about the service being briefly unavailable during the change. What's the safe sequence? — Edit the unit file, then run `systemctl daemon-reload` (which reloads systemd's understanding of unit files WITHOUT restarting any running services) — the running service is completely unaffected by the reload itself; only a subsequent `systemctl restart` would actually apply the new configuration to the running process, so you control exactly when the brief interruption happens, separately from when the config change is loaded.

**Additional angles worth having ready:**

10. **Implementation — "Make this Python app a systemd service. Now tell me why it didn't start."** — A live-coding staple. Expected flow: unit file (`ExecStart` with an absolute path, `User=`, `Restart=on-failure`, `WorkingDirectory=`) → `daemon-reload` → `enable --now` → `status` → on failure, `journalctl -u app -b` to read the actual error — most often a relative path in ExecStart, a missing WorkingDirectory, or a missing environment variable (solved with `Environment=`/`EnvironmentFile=` in the unit, connecting back to the export/environment failure mode in Shell & Environment Variables).
11. **Troubleshooting — "`systemctl start` says 'Unit not found' but the file exists in /etc/systemd/system/."** — You added or edited a unit and forgot `systemctl daemon-reload` — systemd caches unit definitions in memory. `systemctl cat <unit>` shows the *effective* config across all fragments/overrides, which is useful follow-up depth.
12. **Depth — "How do you safely deploy a unit-file change across a fleet?"** — `systemctl cat` to confirm effective config → `systemd-analyze verify unitfile` to validate syntax without installing it → `daemon-reload` (zero service impact) → staged `restart` during a window → verify via `status`/a health endpoint. The separation of "reload config" from "restart process" is the entire safety story — say it explicitly.

---

## Scripting & Live-Coding Drill

Hands-on scripting rounds are a commonly reported stage in DevOps hiring loops — screening, fundamentals, live problem-solving, scenario, managerial — typically 20–30 minutes, shared-terminal or CoderPad, writing and running Bash or Python live. What candidate reports describe getting scored on: error handling (`set -euo pipefail`, `trap`), quoting, exit codes, and readability over cleverness — plus whether you narrate your approach before typing.

1. **Basic:** What does `set -euo pipefail` do, flag by flag? — `-e`: exit immediately when a command returns non-zero — but with real exceptions worth knowing: it does NOT trigger for a command inside an `if`/`while`/`until` condition, a command before `&&`/`||`, or (in the last element of) a pipeline unless `pipefail` is also set — these gaps are exactly why relying on `-e` alone gives false confidence. `-u`: treat an unset variable reference as an error instead of silently expanding to empty. `-o pipefail`: a pipeline's exit status becomes the *rightmost failing* command's status, not just the last one — without it, `grep foo file | head` "succeeds" even when grep found nothing.
2. **Why:** Why do senior engineers insist on error handling in "simple" ops scripts? — A script that silently half-fails is worse than no script — it reports success while leaving production in an unknown state. Exit-on-error forces failure into the open where a human or alert can see it.
3. **How:** How do you make a script fail loudly and safely at the first sign of trouble? — `set -euo pipefail` at the top, explicit `exit 1` with a message to stderr at decision points, validate inputs before using them, and use `trap` for best-effort cleanup on exit paths where the shell can execute the trap.
4. **Scenario:** Write a script that checks whether nginx is healthy and restarts it if not, narrating your design decisions as you go. — `systemctl is-active --quiet nginx || { logger ...; systemctl restart nginx; }` is the base. What scores beyond that: retry/backoff instead of a hot restart loop, logging every action, a lock against concurrent runs (`flock -n`), and an exit code reflecting the outcome.
5. **Troubleshooting:** A script "works" interactively but fails under cron with "command not found." — Cron runs with a minimal environment (bare PATH, no aliases, no exported vars). Diagnose by having the script log its own view (`env > /tmp/cronenv.log` as the first line). Fix: absolute paths, or set `PATH=` explicitly at the top. This is a frequently-reported scripting failure in real interviews.
6. **Architecture:** Automate log rotation across 40 servers with no existing tooling — design it. — A small script via config management/cron, or better, a systemd *timer* + oneshot unit (the modern take on "cron with logs and dependency handling"), writing a dated archive, compressing, pruning by retention count, and signaling the app to reopen (or using `copytruncate`, accepting its small window of possible log loss between copy and truncate if the app has no reopen signal), with `flock` to prevent overlap. Mentioning you'd *prefer* standard `logrotate` unless there's a specific reason not to shows production judgment.
7. **Comparison:** `"$@"` vs `"$*"`; single vs double quotes; `$(...)` vs backticks. — `"$@"` expands each argument as its own quoted word (typically what you want for iterating arguments); `"$*"` merges all args into one word. Single quotes are literal; double quotes allow expansion but preserve as one word. `$(...)` nests cleanly; backticks can't nest without escaping pain.
8. **Follow-up:** What does `trap 'cleanup' EXIT` give you, and when is it essential? — Runs `cleanup` whenever the shell reaches a point where it can execute the trap — normal exit, an error under `set -e`, or most signals. It is NOT an absolute guarantee: it cannot protect against `SIGKILL` (which gives no chance for any handler to run), a power loss, a kernel panic, or the machine dying outright. Still essential whenever a script leaves state behind on ordinary failure: temp files, half-written backups, held locks. Pair with `mktemp -d` for a temp workspace `trap` removes in the cases it *can* catch.
9. **Pressure:** 15 minutes, an interviewer watching: "Parse this nginx log, give me the top 5 IPs by request count, then the top 5 URLs returning 500s." — Narrate the pattern before typing: "group, count, sort — sort | uniq -c | sort -rn | head." Then: `awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -5` and `awk '$9 >= 500 {print $7}' access.log | sort | uniq -c | sort -rn | head -5`. State your field assumptions (IP is `$1`, status `$9`, path `$7` in combined log format) out loud.

---

## Performance & "Server Is Slow" Drill

"A server is slow / high load — how do you troubleshoot?" is a frequently reported scenario question in current DevOps and SRE interview accounts, across companies of varying sizes. The differentiator is having a method, not a favorite command.

1. **Basic:** What is load average, and what's a "bad" value? — The exponentially-decayed average of runnable plus uninterruptible (D-state) tasks over 1/5/15 minutes. "Bad" is relative to core count — but load also counts D-state tasks, so a high load with idle CPU means I/O wait, not CPU pressure. Reading it correctly is the first test.
2. **Why:** Why run `dmesg` early in any performance triage? — Kernel-level failures (I/O errors, NIC resets, OOM kills) change every downstream interpretation — a box with disk errors looks exactly like "a slow app" until you check. It's the cheapest way to avoid misdiagnosing a hardware problem as an application problem.
3. **How:** Walk through your first 60 seconds on a slow production box. — The Toolkit checklist, said out loud in order: `uptime` → `dmesg | tail` → `vmstat 1` → `pidstat 1` → `iostat -xz 1` → `free -m` → `sar -n DEV,TCP 1`. By the end of the minute: which resource, which process.
4. **Scenario:** `top` shows high load average but low CPU usage — what's going on? — Classic I/O wait or D-state accumulation: `vmstat` shows high `b` (blocked) and nonzero `wa`; `iostat -xz` shows a disk near 100% util with high await. Processes are queued on storage, not computing.
5. **Troubleshooting:** One CPU core is at 100% while the rest are idle — why? — A single-threaded hot process or a lock/serialization bottleneck: `mpstat -P ALL 1` shows the hot core; `pidstat -t -p <pid> 1` breaks it down to threads; `perf top` names the hot function for real depth.
6. **Architecture:** How would you make sure you never learn about a slow server from a user again? — Instrument utilization/saturation/errors per host and per service, alert on saturation *before* exhaustion (disk 75%, load-vs-cores ratio, memory available trend), and keep those dashboards as the first screen during any incident.
7. **Comparison:** `vmstat`, `iostat`, `mpstat`, `pidstat` — when do you reach for which? — vmstat: whole-system pulse. iostat: per-disk utilization/await. mpstat: per-CPU (skew, single-hot-core). pidstat: per-process CPU, memory, or I/O (`-d`) — the "which process is guilty" tool.
8. **Follow-up:** A server is slow only at 9am every day — how do you approach it? — Time-correlated triage: check cron/systemd timers firing at that hour (`systemctl list-timers --all`), pull archived metrics (`sar` keeps per-day archives under `/var/log/sa`) to see which resource spikes at 9am, then map it to the scheduled job.
9. **Pressure:** SSH into a barely-responsive production box takes 10 seconds per keystroke. What do you do first, before any diagnosis? — Stabilize access before investigating — try a lighter channel (serial/console, an existing session, out-of-band management), avoid commands that allocate heavily, and only kill an identified runaway once its blast radius is understood. On a dying box, every command costs something; sequencing matters.

---

## Quick-Coverage Round

**Memory**
- Why does "used" memory looking high not necessarily mean trouble? Because Linux uses free RAM as disk cache by default — reclaimable instantly, not wasted.
- What's the real warning sign of memory pressure? `available` dropping low together with climbing swap activity is a strong signal — but weigh it alongside PSI, OOM events, application behavior, and cgroup limits rather than reading one number in isolation.
- What does OOM-killer do? When the kernel truly runs out of memory, it picks a process to kill (based on a scoring heuristic) to free memory rather than let the whole system crash.

**Package Management**
- RPM/dnf family vs deb/apt family — know which OS family you're on before assuming a command exists.
- Why enable extra repositories (like EPEL)? Because not every useful package ships in the default, minimal repo set.

**Shell & Scripting**
- Why does `export` matter for scripts? Only exported variables are visible to child processes, including scripts you invoke.
- What does `$?` hold? The exit code of the last command — 0 means success, non-zero means some kind of failure.
- What's the difference between `$(command)` and backticks `` `command` ``? Functionally similar (command substitution), but `$()` nests cleanly for complex commands where backticks get awkward and error-prone.
- What are `set -euo pipefail` and `trap`? The error-handling and best-effort-cleanup pair that separates production scripts from demos — know them cold for live-coding rounds (see the Scripting Drill).
- Why do scripts behave differently under cron? Cron's environment is minimal (bare PATH, no exported vars, no tty) — define PATH explicitly in the script rather than depending on your interactive environment.

**Boot**
- What's the correct boot-stage order? Firmware → Bootloader → Kernel → init/systemd.
- Why does identifying which boot stage failed matter? Because each stage's failures look completely different and need completely different fixes.
- What one file commonly breaks boot after a reboot, and how do you validate changes to it beforehand? `/etc/fstab` — always run `mount -a` (and `findmnt --verify`) *before* rebooting; use `nofail` for non-critical mounts.

**Permissions**
- What are the correct SSH key permissions, and why does OpenSSH care? `~/.ssh` at 700, private keys and `authorized_keys` at 600, public keys at 644 — sshd refuses keys readable by others. This is a commonly reported real-world permissions failure among DevOps engineers.

---

## Hot Topics Right Now

- **cgroups v2** — the current standard for resource limiting (CPU/memory/IO caps per process group), and the exact mechanism containers rely on for their resource limits — worth being able to explain this connects directly into how Docker/Kubernetes resource limits actually work under the hood.
- **Immutable/read-only root filesystems** — an increasingly common hardening pattern (especially relevant to containers) where the root filesystem is mounted read-only and only specific, deliberate paths are writable — reduces the blast radius of a compromised process trying to persist changes.
- **eBPF** — a modern kernel technology letting you run sandboxed, safety-checked programs directly inside the kernel (used heavily in modern observability and security tooling) without writing traditional kernel modules — worth knowing the name and one-line purpose even without deep expertise here.
- **"How would you debug a production issue right now?" as an interactive exercise** — a growing round format (shared-terminal or verbal pair-debugging). Rehearse the triage → isolate → root-cause → fix → verify → prevent loop out loud on three stock scenarios: disk full, high load with low CPU, and a service that's "running" but dead. The loop matters more than any single command.
- **Live-coding expectations** — Bash (and often Python) scripting is assumed, not optional, at the 3–5 YOE bar: expect "write a disk-space alert script," "parse this log for top error counts," "check a service and restart if down." When this appears, the useful bar is error handling, quoting, exit codes, and narrated reasoning — not clever one-liners.

---

## Research-Backed Additions (Current Market, 2025–2026)

This section exists because generic question banks go stale. These points came from checking actual, dated interview reports and current DevOps hiring content — not assumptions about what "should" be asked.

> [!NOTE]
> **What's actually confirmed from recent candidate reports (India/Bengaluru market, 3–5 YOE DevOps roles):** "How does an OS boot?" is a real, verbatim-asked question at a Bengaluru DevOps interview — not a hypothetical. One candidate described a full interview at a Bengaluru company weighted 90% toward Linux administration and kernel-level questions for a role nominally titled "DevOps," despite the JD emphasizing containers/cloud. The lesson: don't assume Linux is a warm-up round — some interviewers treat it as the main event.

> [!NOTE]
> **Interview culture point worth internalizing:** Multiple recent Infosys Bengaluru candidate reports specifically describe interviewers who "expect a straight answer and just clearly say no if you don't know" rather than reward guessing. This matches exactly what the Master Positioning approach for this whole series already says — the "I don't know, but here's how I'd find out" answer pattern isn't just honest, it's what these specific interviewers are reported to actually want.

> [!NOTE]
> **Linux troubleshooting often doesn't arrive as a standalone question in current interviews.** Recent reports (Infosys, Mphasis, and others, 2025–2026) show Linux fundamentals questions frequently embedded INSIDE a CI/CD or deployment-failure scenario question — e.g., "walk me through troubleshooting a failed deployment" — rather than asked as an isolated "what is a process" question. Practice tracing a pipeline failure down to a Linux-level root cause (disk full, wrong permissions on a deploy script, a service not actually running) as part of one connected answer, not as separate topics.

- **What are your day-to-day activities as a DevOps engineer?** — a real, currently-used opening question (2026 Infosys report). Have a genuine, specific answer ready — this is a screening question, not small talk, and a vague answer here sets a weak tone for the rest of the interview.
- **"Describe a problem and how you solved it in your previous organization"** — reported as an actual standalone interview question, not just a behavioral-round filler. This is exactly what the Story Bank below is built for — have it ready in the Symptom → Root Cause → Fix shape, not as a vague summary.

---

# Part 4 — Real Incident / Story Bank

These are your actual stories, structured the way a strong interviewer wants to hear them: Symptom → Initial Hypothesis → Evidence → Narrowing → Root Cause → Fix → Verification → Prevention. Practice saying these out loud until they come out naturally, not memorized word-for-word.

## Story 1: Gitea Without sudo or systemd

**Symptom:** You needed to run a self-hosted Gitea instance reliably on a RHEL host, but you had no `sudo` access on that machine — meaning no ability to create systemd unit files, which is the normal, expected way to run a persistent Linux service.

**Initial hypothesis:** Without systemd, there's no standard way to keep Gitea running in the background, restart it on crash, or manage it cleanly.

**Evidence/investigation:** Confirmed that regular users without sudo genuinely cannot write to `/etc/systemd/system/`, so unit files were off the table entirely — not a workaround-able permissions quirk, a hard constraint of the environment.

**Narrowing:** The remaining option was running Gitea as a plain background process under your own user account, using `nohup` to keep it running after the terminal session ended, and a dedicated OS user to keep it isolated from your personal account's permissions.

**Root cause:** The real "root cause" here wasn't a bug — it was an environment constraint (restricted, no-sudo RHEL host) that ruled out the standard tooling.

**Fix:** Ran Gitea via `nohup ./gitea web &` under a dedicated service-style OS user, redirecting output to a log file manually since there was no `journalctl` integration without systemd.

**Verification:** Confirmed the process survived terminal disconnection (the whole point of `nohup`), and that logs were being captured correctly in the redirected output file instead of being lost.

**Prevention/what this taught you:** No automatic restart-on-crash, no `systemctl status` visibility, no `journalctl` integration — real, accepted tradeoffs of the constraint, not oversights. It's a genuinely good story because it shows you correctly diagnosing an environment's hard limits and choosing the right tool available within them, rather than trying to force systemd where it structurally couldn't work.

> [!TIP]
> **How to tell this in an interview:** "I had to run a self-hosted Gitea instance on a RHEL host where I didn't have sudo access — so systemd unit files, the normal way to manage a persistent service, weren't an option at all. I ran it under a dedicated OS user with nohup instead, redirecting output manually since I couldn't get journalctl integration without systemd. It meant no automatic restart-on-crash, which is a real limitation, but it was the correct tool for that specific, locked-down environment."

---

## Story 2: GitLab CE and the Undersized /var Partition

**Symptom:** Setting up a self-hosted GitLab CE instance, you hit a wall where GitLab's default installation path expects to store a large amount of data under `/var/opt/gitlab` — but the host's `/var` partition was small, sized for normal system logs, not for hosting an entire Git platform's repositories and data.

**Initial hypothesis:** GitLab would either fail to install, or would fill up `/var` and start affecting the rest of the system's stability once real usage began.

**Evidence/investigation:** Checked actual available space on `/var` with `df -h`, confirmed it was nowhere near sufficient for GitLab's expected data footprint at any real scale, while a separate, larger partition (mounted elsewhere, effectively under your home directory's storage) had plenty of room.

**Narrowing:** The real question was whether to fight GitLab's hardcoded default path expectations, or redirect its actual storage location to the larger partition while keeping the expected path structure intact for GitLab's own tooling to keep working correctly.

**Root cause:** A mismatch between where GitLab expects to store data by default and the actual partition layout of this specific host — not a GitLab bug, an environment-fit problem.

**Fix:** Used a bind mount — mounting the larger partition's directory at the exact path GitLab expects (`/var/opt/gitlab`) — so GitLab's own code and backup tooling continued working against the path it expects, while the actual bytes physically lived on the partition with real capacity.

**Verification:** Confirmed `df -h` showed the underlying larger partition's capacity reflected at the `/var/opt/gitlab` mount point, and that GitLab's backup process (which needs to read/write that path) worked correctly against the bind-mounted location.

**Prevention/what this taught you:** A bind mount versus a symlink was a deliberate choice, not an arbitrary one — GitLab's internal backup tooling and permission checks work more reliably against a true mount point than a symlink, since some tools treat symlinks differently for permission/traversal purposes. This is a strong story specifically because it shows evaluating two real options (symlink vs. bind mount) and picking correctly based on how the dependent tooling actually behaves, not just picking whichever was faster to set up.

> [!TIP]
> **How to tell this in an interview:** "I set up self-hosted GitLab CE on a host where the /var partition was too small for GitLab's expected storage footprint. Rather than a symlink, I used a bind mount to redirect GitLab's actual data path onto a larger partition, because GitLab's own backup tooling behaves more reliably against a true mount point than a symlink. It's a good example of not just finding *a* fix, but checking how the dependent tooling actually treats the two options before picking one."

---

# Part 5 — Master Glossary (Active Recall)

**How to use this:** Cover the right-hand answer with your hand or a piece of paper. Read only the term. Say the definition out loud from memory. Only then check if you got it right. Rereading this list passively will NOT build real recall — forcing yourself to answer first is what actually works.

| Term | Definition |
|---|---|
| Kernel | The core of the OS with direct, unrestricted hardware access; everything else goes through it via system calls |
| System call | A formal request from a user-space program into the kernel to do something only the kernel can do |
| Process | A running instance of a program, with its own memory, file descriptors, and PID |
| PID | Process ID — a unique integer identifying a running process; PID 1 is the first process in the initial host PID namespace at boot |
| PPID | Parent Process ID — identifies which process created a given process |
| fork() | Creates a near-exact duplicate of the calling process |
| exec() | Replaces a process's own code with a new program's code, keeping the same process identity |
| Zombie process | A process that has exited but whose exit status hasn't been read by its parent yet |
| Orphan process | A process whose parent exited before it did; normally re-parented to PID 1, unless a designated child subreaper claims it first |
| Virtual memory | The illusion each process gets of having its own full, private memory space |
| Swap | Disk space used to hold memory pages the kernel has temporarily moved out of physical RAM |
| Page cache | RAM used to cache recently-read disk data; reclaimed instantly when a real app needs that memory |
| Inode | The data structure holding a file's real metadata and disk location; the filename is just a pointer to it |
| Hard link | A second filename pointing at the exact same inode as another filename |
| Symlink | A small separate file containing a path to another file; breaks if the target is removed |
| chmod 750 | Owner rwx (7), group r-x (5), other none (0) |
| UID 0 | Always root, regardless of the account's actual name |
| sudo | Lets an explicitly authorized user run specific commands with another user's (usually root's) privileges, for one invocation |
| Unit file | A systemd configuration file describing how to start/manage a specific service |
| systemctl enable | Schedules a service to start automatically on future boots — does not start it now |
| systemctl start | Starts a service immediately — does not persist across reboots unless also enabled |
| journald | systemd's centralized, structured logging system |
| journalctl -u | Filters journald output to one specific service |
| SIGTERM (15) | Polite, catchable request to a process to exit gracefully |
| SIGKILL (9) | Immediate, uncatchable termination with zero chance to clean up |
| File descriptor | A small integer a process uses to reference an open file, socket, or pipe |
| FD 0/1/2 | Conventional stdin/stdout/stderr — inherited by default, but can be closed or redirected before a process starts |
| df | Reports disk usage at the filesystem/partition level |
| du | Reports disk usage at the directory/file level |
| SUID | Makes an executable run with the file owner's privileges rather than the invoking user's |
| SGID (on a directory) | New files created inside inherit the directory's group automatically |
| Sticky bit | On a shared directory, lets users delete only their own files even with broader write access |
| Bootloader | The stage (e.g. GRUB) that locates and loads the kernel into memory |
| initramfs | A temporary early root filesystem used during boot before the real root filesystem is mounted |
| cgroups | Kernel mechanism for limiting CPU/memory/IO resources per process group; the basis of container resource limits |
| Namespaces | Kernel mechanism for isolating what a process can see (PIDs, network, filesystem) — the basis of container isolation |
| export (shell) | Makes a variable visible to child processes, not just the current shell |
| PATH | The ordered list of directories the shell searches to resolve a bare command name |
| set -euo pipefail | Bash strict mode: exit on error, error on unset variables, pipeline fails if any stage fails |
| trap | Runs a command (e.g. cleanup) on script exit or most signals — cannot catch SIGKILL, power loss, or a kernel panic |
| `"$@"` | All script arguments, each preserved as its own quoted word (vs `"$*"`, which merges them) |
| visudo | The standard, recommended tool for editing sudoers — locks the file and validates syntax on save |
| lsof +L1 | Lists files with link count < 1 — i.e., deleted but still held open by a process |
| truncate -s 0 file | Zeroes a file in place without unlinking it; can reclaim space while preserving the pathname/open file, but destroys the existing contents and should be used deliberately |
| logrotate copytruncate | Rotation mode that copies then truncates the original — for apps with no reopen signal; can lose a small amount of log data written in the gap between the copy and the truncate |
| df -i | Shows inode usage — a filesystem can be "full" on inodes with bytes to spare |
| du -x | Stays on one filesystem — prevents wandering into other mounts and miscounting |
| Load average | Runnable + uninterruptible tasks averaged over 1/5/15 min; compare to core count, and remember D-state tasks inflate it |
| vmstat 1 | One-line system pulse: run queue, blocked, memory, swap in/out, CPU breakdown |
| iostat -xz 1 | Per-disk utilization, await, queue depth — the storage saturation verdict |
| pidstat 1 | Per-process CPU/memory/I/O — the "which process is guilty" tool |
| mpstat -P ALL 1 | Per-CPU breakdown — exposes single-hot-core / lock skew |
| sar -n DEV / TCP,ETCP | NIC throughput / TCP retransmits — the network slice of triage, with day archives under /var/log/sa |
| USE method | Utilization, Saturation, Errors — per-resource checklist for performance triage |
| stress-ng | Workload generator used to safely reproduce CPU/I/O/memory pressure for practice |
| journalctl --disk-usage | Shows how much disk the journal occupies — the journald side of a full-/var incident |
| systemd-analyze blame | Per-unit boot times — the first tool for "why does this server boot slowly" |
| mount -a | Mounts everything in fstab now — the mandatory pre-reboot validation for fstab edits |
| nofail (fstab) | Option so a missing device degrades boot instead of dropping to emergency mode |
| Exit code 137 | 128 + 9 — indicates the process was terminated by SIGKILL; it does not by itself identify who or what sent the signal |

---

# Part 6 — Additional Practical Exercises

Most hands-on labs now live right next to the concept they belong to (look for the 🔧 HANDS-ON callouts throughout Parts 1 and 2). This final section is deliberately kept small — just for the few genuinely useful exercises that don't map cleanly to one single section, so nothing important gets left out just because it didn't fit neatly earlier.

## Exercise 1: Resource Threshold Monitor With Alerting

**Concept:** A repeatedly cited practice scenario in current interview-prep material: monitor CPU, memory, and disk together, write results to a file in table format, and alert if any threshold is exceeded. This is a different angle from the single-purpose disk_alert.sh live-coding lab earlier — a combined, scheduled, multi-metric monitor rather than a one-shot live-coded script.

> [!NOTE]
> **Label this correctly:** The parser below is an interview/lab-level implementation, written to be readable and to demonstrate the pattern — it is deliberately simple, not a hardened production parser. Real production monitoring should use Prometheus's Node Exporter or similar rather than a hand-rolled `top`/`free` parser, precisely because output formats vary across distros, locales, and tool versions in ways this script doesn't handle.

**Implementation:**
```bash
#!/bin/bash
# resource_monitor.sh
THRESHOLD=80
REPORT="/var/log/resource_report.log"

CPU=$(top -bn1 | grep "Cpu(s)" | awk '{print 100 - $8}')
MEM=$(free | awk '/Mem/{printf "%.0f", $3/$2 * 100}')
DISK=$(df -h / | awk 'NR==2{print $5}' | tr -d '%')

printf "%-10s %-10s %-10s %-10s\n" "$(date +%T)" "CPU:${CPU}%" "MEM:${MEM}%" "DISK:${DISK}%" >> "$REPORT"

for pair in "CPU:$CPU" "MEM:$MEM" "DISK:$DISK"; do
  name="${pair%%:*}"; value="${pair##*:}"
  value=${value%.*}
  if [ "$value" -ge "$THRESHOLD" ] 2>/dev/null; then
    echo "ALERT: $name usage at ${value}% (threshold ${THRESHOLD}%)" | tee -a "$REPORT"
  fi
done
```

**Hands-on:** Schedule this with cron every 5 minutes (`*/5 * * * * /path/resource_monitor.sh`) and watch the report file grow.

**Failure scenario:** The script reports 0% CPU constantly, clearly wrong on a busy box. **Investigation:** the `top -bn1` parsing is fragile — output format differs slightly across distributions/locales. **Fix:** validate the actual `top` output on the target system first (`top -bn1 | head -5`) before trusting a parsing pattern copied from elsewhere — never deploy a monitoring script you haven't confirmed against the real output format of the box it'll run on. **Verification:** manually compare the script's reported values against `free -h` and `df -h` run at the same moment.

**Interview Explanation:** "I'd write a script that captures CPU, memory, and disk usage on a schedule, logs it in an easily-scannable format, and raises a clear alert line when any metric crosses a threshold — but I'd always validate the parsing against that specific system's actual command output first, since something as simple as a locale or top version difference can silently make a monitoring script lie to you."

---

## Exercise 2: Find Recently Modified Files in a Date Range

**Concept:** A directly cited real practice task: find the names and sizes of recently modified files, accepting a number of days (or a from/to date) as input.

**Implementation:**
```bash
#!/bin/bash
# recent_files.sh <days>
DAYS=${1:-1}
find /var/log /home -type f -mtime -"$DAYS" -printf "%TY-%Tm-%Td %10s  %p\n" | sort
```

**Hands-on:** Run with different values and confirm the output matches what you'd expect from `ls -la --time-style=full-iso` on a couple of sample files.

**Failure scenario:** The script misses a file you know was just created. **Investigation:** `-mtime` tracks *modification* time, not creation time — Linux doesn't universally track creation time the way some other systems do (ext4 has `crtime` but standard tools like `find` don't expose it directly). **Fix:** for most practical purposes, `-mtime` (or `-newer <reference-file>` for precise windows) is the right tool, but know this limitation exists rather than assuming "modified" and "created" are the same thing. **Verification:** cross-check a known file's timestamp with `stat <file>`.

**Interview Explanation:** "find with -mtime is based on modification time, not creation time — Linux doesn't expose creation time through standard tools the way Windows does, so any script relying on 'when was this created' actually needs to be built around modification time."

---

*Note: a firewall configuration exercise (ufw/firewalld) originally planned for this section has been moved to the Networking document, where it fits more naturally.*

---
# Part 7 — The Current-Market Interview Layer (2026)

This final layer is deliberately separate from the Interview Corner drills. The drills teach you a topic; this layer tells you **how the game is being played right now** at the 3–5 YOE DevOps bar, with the India/Bengaluru market called out where the evidence is specific. Two evidence tiers, kept distinct on purpose:

- **Tier A — verbatim candidate reports** (what an interviewer literally said, as reported by the candidate). Small sample, highest fidelity.
- **Tier B — loop analyses from recruiters and 2026-dated interview content** (what stages exist, what they test, what separates pass/fail). Larger sample, slightly lower fidelity — treat as "strong pattern," not "guaranteed."

## 7.1 The 2026 loop — and where Linux actually appears

Tier B sources converge on a consistent shape for 3–5 YOE DevOps roles (Simpliaxis's 2026 interview-guide analysis describes the same five-stage arc as multiple recruiter write-ups):

1. **Recruiter screen** — experience, notice period, salary. Pass bar: coherent story of what you actually do daily.
2. **Fundamentals round** — Linux, networking, Git, CI/CD basics. Linux appears here as direct questions.
3. **Hands-on / live problem-solving round** — scripting and/or live troubleshooting (see 7.2). This stage is where most candidates are filtered.
4. **Scenario / system round** — production incidents, trade-offs, sometimes light design.
5. **Managerial / culture** — behavioral, including the incident-story question (see 7.5).

**Where Linux appears in 2026:** not just stage 2. Current prep content and reports consistently embed Linux questions *inside* deployment-failure and CI/CD scenarios — "troubleshoot this failed deployment" often resolves to a Linux-level root cause (disk full, permissions on a deploy script, service not actually running). A Tier A report from a Bengaluru DevOps interview describes the round as weighted ~90% toward Linux administration and kernel-level questions for a role whose JD emphasized containers and cloud. The takeaway isn't that Kubernetes doesn't matter — it's that **some interviewers treat Linux as the main event**, and being surprised by that is the only real failure mode.

## 7.2 The live troubleshooting round — the biggest format shift

Tier B, strongly corroborated: a growing share of senior loops now include a **live troubleshooting stage** — shared screen or SSH into a deliberately broken environment, with the interviewer watching. One hiring-manager write-up covering recent loops states plainly that candidates fail this round through **silence, not inability** — they know the commands but say nothing, so the interviewer can't see the reasoning.

What this round actually scores:
- **Narration over silence.** Think aloud, constantly. The interviewer is scoring your reasoning process, not just the fix.
- **Ordered triage.** Triage → isolate → root cause → fix → verify → prevent, said out loud, matches exactly the structure of Lab 10.1 and the Performance Toolkit.
- **Safe habits.** Stating "inspect before execute" on anything destructive is a positive signal; `rm`-ing first is a fail even when it happens to work.

**Preparation protocol:** break your own VM on purpose (full disk via `dd`, stopped service with `systemctl disable`d surprise, wrong permissions on a script), then narrate the repair aloud to nobody. If you can keep a coherent sentence going while fixing it, you'll do fine in the real thing. Rehearsed narration phrases that work: *"First I want to see which resource is saturated — so df -h and free -h before I touch anything"* / *"That narrows it to this process, so now I'm going to look at what it's actually doing with strace"* / *"Before I delete anything, I want to know what's holding it open."*

## 7.3 Tier A — verbatim-reported India/Bengaluru patterns (candidate reports, 2025–2026)

- **"How does an OS boot?"** — asked verbatim in a Bengaluru DevOps interview. Not a warm-up — a genuine fundamentals probe. The boot-stage answer in this guide (firmware → GRUB → kernel → systemd, and why the stage matters more than the acronym) is exactly what this question wants.
- **"Interviewers expect a straight answer, and 'no' is an acceptable answer."** — repeated across Infosys Bengaluru candidate reports. Guessing confidently is scored *worse* than saying "I don't know, but here's how I'd find out." This is already this guide's TIP style on purpose.
- **Linux embedded in CI/CD scenario questions** — deployment-failure walkthroughs that resolve to Linux root causes (permissions, disk, dead service), rather than standalone definitions.
- **"What are your day-to-day activities as a DevOps engineer?"** — reported as a real opening question, not small talk. Have a specific, honest answer: what you monitor, what you deploy, what you've automated, one thing you fixed recently. Vagueness here sets the tone for everything after.
- **"Describe a problem you solved in your previous organization"** — a standalone question with real scoring weight. This is precisely what the Story Bank (Part 4) exists for — deliver one story in Symptom → Root Cause → Fix → Prevention shape, not a summary.

## 7.4 The 2026 Linux question set that keeps recurring

From dated 2026 prep content and reports (Tier B, high consistency — each maps to a section of this guide you now own):

| Recurring ask | What they're really testing | Your answer in one breath |
|---|---|---|
| "A service won't start after a config change — go." | journald-first reflex, config validation | `systemctl status x` → `journalctl -xeu x` → for executables: exit 203/EXEC = bad ExecStart path; for nginx-like apps: validate config *before* reload (`nginx -t`) |
| "Find what's listening on port 8443." | modern vs legacy tooling | `ss -ltnp` (or `lsof -nP -i :8443`); then *who* owns it, not just what |
| "Process died mysteriously overnight." | evidence-first thinking | `journalctl -u x -b --since "12 hours ago"`, `dmesg -T` for OOM, `$?` = 137 ⇒ killed by SIGKILL ⇒ check *who/what* sent it |
| "Disk is full but I can't find the big files." | the df/du/inode trio | `df -i` (inodes), `lsof +L1` (deleted-but-open), `du -xh --max-depth=1` (largest first, `-x` to stay on the fs) |
| "What happens when you type `ls`?" | fork/exec internals, attention to detail | shell parse → PATH resolution → fork → execve → loader → getdents syscalls → exit → parent reaps (Lab 1.1 is your proof) |
| "Server is slow — where do you start?" | method, not memory | the 60-second checklist (Performance Toolkit), narrated in order |

Notice what this list has in common: **every one of them is a workflow question, not a definition question.** Nobody is asking you to recite what a zombie process is; they're asking what you'd *do* about it.

## 7.5 The incident story — the behavioral round's real question

Tier B and Tier A agree that 3–5 YOE candidates are expected to narrate a real incident with engineering rigor — and recent interview content frames it as a **sev-1 walkthrough**: what broke, who was impacted, what you tried, what actually worked, what changed afterward. The structure that scores:

1. **Impact first** — what user-visible thing broke, and how big (this is what "sev" means).
2. **Timeline** — when detected, when mitigated, when fixed; detection lag is a fair thing to critique.
3. **Proximate cause vs root cause** — "the disk filled" is proximate; "no log rotation and a 98% alert threshold" is root. Naming both separately is the single strongest signal in the whole answer.
4. **Mitigation vs fix** — what stopped the bleeding vs what made it not happen again.
5. **The postmortem mindset** — no-blame, prevention-oriented. "We" language, not "someone."

This is exactly the shape of Stories 1–3 in Part 4. Prepare two of them cold: one environment/constraint story (Story 1 or 2) and one incident story (Story 3).

## 7.6 The judgment shift — from definitions to trade-offs

Tier B analysis of 2026 loops is blunt: memorized definitions no longer differentiate anyone, because AI-generated prep made everyone sound the same. What differentiates now is **judgment** — trade-offs, blast radius, and outcomes. The upgrade path for almost any answer in this guide is to append one of three things:

- **The trade-off:** "…and that's why we SIGTERM first — SIGKILL forfeits the app's chance to clean up its database connections."
- **The incident:** "…I learned this when a deleted-but-open log kept a partition at 98%." (Story 3)
- **The number:** "…we moved the alert threshold from 98% to 80%, which turned a 2am page into a daytime task."

If every core answer in your interview carries one of these three, you will sound like the 5-YOE engineer even on the 3-YOE questions.

## 7.7 Final self-check before you call this document "studied"

Answer these out loud, from memory, before moving to the Networking document:

1. **Foundation:** Can I explain, without notes, why `available` ≠ `free`, why deleting a file may not free space, and why PID 1 matters?
2. **Performance:** Can I run the 60-second triage on a slow box and say what each tool rules in or out? (Toolkit lab)
3. **Incident:** Can I walk the full disk-full incident — triage, locate, `lsof +L1`, safe cleanup, prevention — as one connected story? (Lab 10.1 + Story 3)
4. **Scripting:** Can I write the disk-alert script from scratch with `set -euo pipefail`, quoting, and a sane exit code — and explain each `set` flag? (Lab 9.1)
5. **systemd:** Can I create a unit file, reload safely without restarting, and diagnose a failed start from journald? (Lab 7.1)
6. **Signals/states:** Can I explain why `kill -9` sometimes can't kill — and what D-state means? (Signals lab)
7. **Live round:** Can I narrate a repair aloud while doing it, without long silences? (7.2 protocol)
8. **Stories:** Can I deliver two incident stories in sev-1 walkthrough shape, with proximate vs root cause named? (Part 4)
9. **Honesty:** Am I comfortable saying "I don't know, but here's how I'd find out" — and is that statement always followed by a real method?

If any answer is no, you know exactly which lab or drill to redo. That's the entire point of how this guide is built.

*End of Linux — Document 1 of 7. Next up: Networking.*
