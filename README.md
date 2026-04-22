# Multi-Container Runtime — OS-Jackfruit

 lightweight container runtime built in C for Linux. This project implements a multi-container supervisor with kernel-level memory monitoring, similar to how Docker works but much simpler.
 
## 1. Team Information

| Name | SRN |
|------|-----|
| Moksha S   | PES1UG24AM167 |
| Lakshya budhauliya | PES1UG24AM147 |

---

## 2. Build, Load, and Run Instructions

the user-space runtime, test workloads, and a kernel module for memory enforcement

### Prerequisites

Ubuntu 22.04 or 24.04 VM with Secure Boot **OFF**.

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
```

### Prepare Alpine rootfs

```bash
mkdir rootfs
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs
```

### Build Everything

```bash
make
```

This builds `engine`, `cpu_hog`, `io_pulse`, `memory_hog`, and `monitor.ko`.

### Copy workloads into rootfs (so they run inside containers)

```bash
cp cpu_hog io_pulse memory_hog ./rootfs/
```

### Load Kernel Module

The kernel module creates a character device and will enforce memory limits on our containers.

```bash
sudo insmod monitor.ko
ls -l /dev/container_monitor   # should appear
dmesg | tail -3                # confirm: "container_monitor: loaded"
```

### Start the Supervisor

The supervisor is now running. It manages multiple containers, handles logging through a bounded buffer, and communicates via a UNIX socket.

Open **Terminal 1**:

```bash
sudo ./engine supervisor ./rootfs
```

### Launch Containers (Terminal 2)

```bash
# Background container
sudo ./engine start alpha ./rootfs /bin/sh

# Another background container
sudo ./engine start beta ./rootfs /bin/sh

# Foreground (blocks until container exits)
sudo ./engine run gamma ./rootfs /cpu_hog 10 gamma-label
```

### List Containers

```bash
sudo ./engine ps
```

### View Logs

```bash
sudo ./engine logs alpha
```

### Stop a Container

```bash
sudo ./engine stop alpha
```

### Run Memory Limit Test

```bash
# Start a container that will grow beyond the 50MB soft / 100MB hard limits
sudo ./engine start memtest ./rootfs /memory_hog 20 200 2
# Watch kernel logs
dmesg -w
```

### Run Scheduling Experiment

```bash
# Experiment 1: Two CPU-bound containers at different priorities
sudo nice -n 0  ./engine run high-prio ./rootfs /cpu_hog 10 high &
sudo nice -n 19 ./engine run low-prio  ./rootfs /cpu_hog 10 low  &
wait

# Experiment 2: CPU-bound vs I/O-bound concurrently
sudo ./engine run cpu-c ./rootfs /cpu_hog   10 cpu-c &
sudo ./engine run io-c  ./rootfs /io_pulse  10 io-c  &
wait
```

### Stop Supervisor and Unload Module

```bash
# In Terminal 2:
sudo kill -SIGTERM $(pgrep -f "engine supervisor")

# Unload module
sudo rmmod monitor
dmesg | tail   # confirm: "container_monitor: unloaded"
```

---

## 3. Demo with Screenshots

> **Replace each placeholder below with an annotated screenshot.**

### Screenshot 1 — Multi-container supervision
*Two containers (alpha, beta) running under a single supervisor process.*

   ![Uploaded image](https://github.com/MokshaSomashekhar/OS-Jackfruit/blob/main/screenshots/1-1.png)
      ![Uploaded image](https://github.com/MokshaSomashekhar/OS-Jackfruit/blob/main/screenshots/1-2.png)

### Screenshot 2 — Metadata tracking (`ps`)
*Output of `sudo ./engine ps` showing name, PID, state, and log path.*

![Uploaded image](https://github.com/MokshaSomashekhar/OS-Jackfruit/blob/main/screenshots/2.png)

### Screenshot 3 — Bounded-buffer logging
*Contents of a container log file and supervisor output showing producer/consumer activity.*

![Uploaded image](https://github.com/MokshaSomashekhar/OS-Jackfruit/blob/main/screenshots/3.png)

### Screenshot 4 — CLI and IPC
*A `stop` command being issued in one terminal and the supervisor responding in another.*

![Uploaded image](https://github.com/MokshaSomashekhar/OS-Jackfruit/blob/main/screenshots/4-1.png)
![Uploaded image](https://github.com/MokshaSomashekhar/OS-Jackfruit/blob/main/screenshots/4-2.png)

### Screenshot 5 — Soft-limit warning
*`dmesg` output showing `[SOFT LIMIT]` warning for the memory_hog container.*

![Uploaded image](https://github.com/MokshaSomashekhar/OS-Jackfruit/blob/main/screenshots/5.png)

### Screenshot 6 — Hard-limit enforcement
*`dmesg` showing `[HARD LIMIT]` kill, and `sudo ./engine ps` showing state `hard-killed`.*

![Uploaded image](https://github.com/MokshaSomashekhar/OS-Jackfruit/blob/main/screenshots/6-1.png)
![Uploaded image](https://github.com/MokshaSomashekhar/OS-Jackfruit/blob/main/screenshots/6-2.png)

### Screenshot 7 — Scheduling experiment
*Terminal output comparing iterations/cycles of high-priority vs low-priority `cpu_hog`.*

![Uploaded image](https://github.com/MokshaSomashekhar/OS-Jackfruit/blob/main/screenshots/7-a.png)
![Uploaded image](https://github.com/MokshaSomashekhar/OS-Jackfruit/blob/main/screenshots/7-b.png)

### Screenshot 8 — Clean teardown
*`ps aux | grep engine` showing no zombies after supervisor exits.*

![Uploaded image](https://github.com/MokshaSomashekhar/OS-Jackfruit/blob/main/screenshots/8.png)

---

## 4. Engineering Analysis

### 4.1 Isolation Mechanisms

Linux namespaces provide the kernel-level isolation that makes containers possible without a hypervisor. Our runtime uses `clone()` with three namespace flags:

- **`CLONE_NEWPID`** — gives the container its own PID number space. The first process inside appears as PID 1, so `init`-style reaping applies. The host kernel retains full visibility via the global PID table; the container only sees its own subtree.
- **`CLONE_NEWUTS`** — isolates the hostname and domain name. We call `sethostname("container", ...)` inside the child so each container has its own identity visible to `uname`.
- **`CLONE_NEWNS`** — gives the container a private mount table. Any mounts made inside (e.g., `/proc`) do not propagate to the host.

`chroot()` then replaces the filesystem root with Alpine's `rootfs`, so container processes cannot traverse to the host filesystem. However, `chroot` does not isolate network, IPC, time, or user namespaces — the host kernel still serves all system calls, and the container shares the host network stack.

What the host kernel still shares: the same scheduler run queue, the same physical page allocator, and the same network stack. Containers are isolated in namespace but not in resource contention — which is exactly why the kernel memory monitor is necessary.

### 4.2 Supervisor and Process Lifecycle

A long-running parent supervisor is necessary because in Unix, when a parent exits before its children, the orphaned children are reparented to PID 1 (init). Init reaps them automatically, but we lose all metadata — container state, exit code, limits. By keeping the supervisor alive, it remains the parent of all containers and can:

- Receive `SIGCHLD` and call `waitpid(WNOHANG)` to reap each exited container without blocking.
- Update per-container metadata (exit status, termination signal) immediately.
- Distinguish a graceful stop (`SIGTERM` → exit 0) from a forced kill (`SIGKILL` from the hard limit).

The `SIGCHLD` handler sets a flag; the supervisor's `select()` loop calls `reap_children()` on each iteration to avoid race conditions in the handler itself (signal handlers should do minimal work).

### 4.3 IPC, Threads, and Synchronization

The project uses two IPC mechanisms:

**Logging pipe (container → supervisor):** Each container's `stdout`/`stderr` is redirected into a `pipe()`. The supervisor's reader thread polls the read end and inserts data into the bounded buffer. Race condition without synchronization: two reader threads could simultaneously update `head`/`tail`/`count`, corrupting the buffer. We protect these with a `pthread_mutex_t`. `pthread_cond_t not_empty` and `not_full` implement blocking back-pressure without busy-waiting.

**Command socket (CLI → supervisor):** A UNIX domain socket (`AF_UNIX, SOCK_STREAM`) is used. This is the second IPC mechanism (distinct from pipes). The socket is chosen because it supports connection-oriented semantics — the CLI knows whether the supervisor is actually running (`connect()` fails immediately if not), and the supervisor can read exactly one command per accepted connection.

**Container table (`containers[]`):** Protected by `containers_mutex`. Any thread or signal handler that reads or writes a `Container` struct must hold this mutex. Without it, the SIGCHLD handler (running asynchronously) could partially update a struct while the CLI handler reads it — a classic TOCTOU race.

### 4.4 Memory Management and Enforcement

RSS (Resident Set Size) measures the number of physical RAM pages currently mapped into a process's address space. It does not count:
- Pages swapped out to disk
- Pages in shared libraries counted once per process (partial double-counting)
- Memory-mapped files that are not yet faulted in

RSS is the right metric for container memory enforcement because it represents actual physical memory pressure on the system.

**Soft vs hard limits serve different purposes.** A soft limit is a warning threshold — it lets the supervisor (or an operator) know that a container is growing, while giving the application a chance to respond (e.g., free caches). A hard limit is a policy ceiling — exceeding it is grounds for termination because the system cannot permit unconstrained memory growth that would starve other containers or the host.

**Kernel-space enforcement is necessary** because a user-space monitor is subject to scheduling delays (it may be preempted for seconds between checks), and a misbehaving or compromised container process could simply ignore a `SIGUSR1` advisory signal. The kernel module runs in a timer callback that the scheduler cannot be tricked by container code, and it has direct access to `task_struct->mm` without copying data through system call boundaries.

### 4.5 Scheduling Behavior

Linux uses the Completely Fair Scheduler (CFS) for normal (SCHED_OTHER) processes. CFS maintains a virtual runtime (`vruntime`) per task and always picks the task with the smallest `vruntime` to run next. The `nice` value scales how fast `vruntime` accumulates — a `nice 19` process's `vruntime` grows much faster per wall-clock second than a `nice 0` process, so it gets fewer CPU cycles.

In our Experiment 1 (two CPU-bound containers, nice 0 vs nice 19), the high-priority container consistently accumulated more iterations per second. CFS gave it roughly 5–10× more CPU time under load, consistent with the nice-value weighting tables in the kernel.

In Experiment 2 (CPU-bound vs I/O-bound), the I/O-bound container voluntarily surrendered the CPU on every `read()`/`write()` call (blocking I/O → context switch). CFS rewarded it with lower `vruntime`, so when I/O completed, it was scheduled promptly. The CPU-bound container received the remaining CPU time. This demonstrates CFS's fairness: I/O-bound processes are not starved, and CPU-bound processes utilize idle cycles.

---

## 5. Design Decisions and Tradeoffs

### Namespace Isolation
**Choice:** `clone()` with `CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS` + `chroot()`.  
**Tradeoff:** We do not use `CLONE_NEWNET`, so containers share the host network stack. This makes container networking simpler for testing but means a container can bind to any host port.  
**Justification:** Network isolation would require `veth` pairs and `iptables` rules — significant extra complexity outside the project scope. The three namespaces chosen cover the required isolation goals.

### Supervisor Architecture
**Choice:** Single-process supervisor with a `select()` loop and detached reader threads per container.  
**Tradeoff:** A multi-threaded supervisor is harder to debug than a pure event-loop (e.g., `epoll`-based) design, but `epoll` would require non-blocking I/O on pipes and more complex state machines.  
**Justification:** Using threads simplifies the per-container reader logic — each reader is a simple blocking `read()` loop rather than a state machine.

### IPC and Logging
**Choice:** UNIX domain socket for CLI↔supervisor; pipes + bounded buffer for logging.  
**Tradeoff:** The UNIX socket is process-local only (no network access to the supervisor from another host). A TCP socket would allow remote management but adds authentication concerns.  
**Justification:** The project runs on a single VM; local IPC is sufficient and simpler.

### Kernel Monitor
**Choice:** Periodic timer (2-second interval) for RSS checks; `dmesg` only for event reporting.  
**Tradeoff:** A 2-second polling interval means a container could exceed its hard limit for up to 2 seconds before being killed. A shorter interval increases kernel overhead.  
**Justification:** For the project's memory workloads (which grow in 10MB steps with 2-second delays), a 2-second check catches violations promptly without unnecessary overhead.

### Scheduling Experiments
**Choice:** `nice` values via the shell and `sched_setaffinity` for CPU pinning.  
**Tradeoff:** `nice` affects CFS weight but does not guarantee real-time priority. A `SCHED_FIFO` experiment would show starker priority differences but could freeze the VM if misconfigured.  
**Justification:** `nice` is the standard user-space knob for CFS priority, safe to use without risking VM stability.

---

## 6. Scheduler Experiment Results

### Experiment 1: Two CPU-bound containers at different `nice` values

| Container | nice | Duration (s) | Iterations |
|-----------|------|-------------|------------|
| high-prio | 0    | 10          | 8,500  |
| low-prio  | 19   | 10          | 1,200  |

**Observation:** The `nice 0` container completed significantly more iterations, demonstrating CFS's weighted fair scheduling.

### Experiment 2: CPU-bound vs I/O-bound at the same priority

| Container | Type    | Duration (s) | Metric       | Value     |
|-----------|---------|-------------|--------------|-----------|
| cpu-c     | CPU     | 10          | Iterations   | 7700 |
| io-c      | I/O     | 10          | I/O cycles   | 450 |

**Observation:** The I/O-bound container maintained responsive throughput even while the CPU-bound container saturated the core, because CFS rapidly reschedules I/O-bound processes after their blocking calls return.

**Conclusion:** CFS balances fairness (equal `vruntime` growth for equal-priority tasks) with responsiveness (I/O-bound tasks are not starved). `nice` values provide a simple, safe mechanism to influence this balance.
