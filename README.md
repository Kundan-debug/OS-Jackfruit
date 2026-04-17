# Mini Container Runtime

Course: Operating Systems – Mini Container Runtime Mini Project

---

## 1. Team Information
- **Name:** Kundan V  
  **SRN:** PES1UG24CS243
- **Name:** Vikas R  
  **SRN:** PES1UG24CS915

---

## 2. Project Summary
This project implements a lightweight Linux container runtime in C with a long-running parent supervisor and a Linux kernel module for memory monitoring.

The project contains two integrated parts:

### User-space Runtime (`engine.c`)
- Long-running supervisor daemon
- CLI client commands (`start`, `run`, `ps`, `logs`, `stop`)
- Multi-container lifecycle management
- Per-container metadata tracking
- UNIX domain socket IPC
- Bounded-buffer producer-consumer logging
- Safe cleanup and zombie prevention

### Kernel-space Monitor (`monitor.c`)
- Linux kernel module
- Character device `/dev/container_monitor`
- `ioctl` PID registration
- Kernel linked list tracking
- Timer-based RSS checks
- Soft-limit warning
- Hard-limit kill enforcement
- Safe unload and cleanup

This project demonstrates process isolation, filesystem isolation, IPC, synchronization, memory enforcement, and scheduling behavior.

---

## 3. Build, Load, and Run Instructions

### Build project
```bash
make
```

### Load kernel module
```bash
sudo insmod monitor.ko
ls -l /dev/container_monitor
sudo dmesg | tail
```

### Start supervisor
```bash
sudo ./engine supervisor rootfs-alpha
```

### Create writable rootfs copies
```bash
cp -a ./rootfs-base ./rootfs-alpha
cp -a ./rootfs-base ./rootfs-beta
```

### Start containers
```bash
./engine start c1 rootfs-alpha /bin/sh
./engine start c2 rootfs-beta /bin/sh
```

### List metadata
```bash
./engine ps
```

### View logs
```bash
./engine logs c1
```

### Stop containers
```bash
./engine stop c1
./engine stop c2
```

### Unload kernel module
```bash
sudo rmmod monitor
```

---

## 4. Demo with Screenshots
### Screenshot 1 — Multi-container supervision
Two containers running under one long-running supervisor process.

### Screenshot 2 — Metadata tracking
Output of `./engine ps` showing:
- container ID
- PID
- state
- memory limits
- log path

### Screenshot 3 — Bounded-buffer logging
Output of `./engine logs c1` showing captured stdout/stderr through producer-consumer pipeline.

### Screenshot 4 — CLI and IPC
CLI command issued from client terminal and response from supervisor using UNIX socket IPC.

### Screenshot 5 — Soft-limit warning
`dmesg` output showing warning when container exceeded soft RSS limit.

### Screenshot 6 — Hard-limit enforcement
`dmesg` output showing SIGKILL after hard RSS limit crossed.

### Screenshot 7 — Scheduling experiment
Two workloads running with different `nice` values and measurable runtime difference.

### Screenshot 8 — Clean teardown
Evidence of:
- no zombie processes
- exited logger threads
- supervisor cleanup
- successful module unload

---

## 5. Engineering Analysis

### A. Isolation Mechanisms
The runtime uses **PID, UTS, and mount namespaces** to isolate processes. Each container gets its own process tree and hostname namespace. Filesystem isolation is achieved using **separate writable rootfs copies** with `chroot`, giving each container an independent `/` view.

The host kernel is still shared among all containers.

### B. Supervisor and Process Lifecycle
A long-running supervisor simplifies:
- container tracking
- signal forwarding
- log ownership
- metadata consistency
- zombie reaping using `SIGCHLD`

The supervisor acts as the parent of all container children and updates state transitions such as:
- starting
- running
- stopped
- killed
- exited

### C. IPC, Threads, and Synchronization
The project uses **two IPC paths**:

1. **UNIX domain socket**
   - CLI → supervisor control channel
2. **Pipes**
   - container stdout/stderr → supervisor logging

The logging system uses:
- producer thread(s)
- consumer thread(s)
- bounded shared buffer
- mutex
- condition variables

Without synchronization, races could cause:
- corrupted logs
- dropped messages
- deadlock
- metadata inconsistency

The bounded buffer avoids these using proper producer-consumer signaling.

### D. Memory Management and Enforcement
The kernel module monitors **RSS (Resident Set Size)**, which measures physical memory pages currently resident in RAM.

- **Soft limit:** warning only
- **Hard limit:** forceful termination using `SIGKILL`

Kernel-space enforcement is necessary because only the kernel has reliable access to process memory statistics and signal authority.

### E. Scheduling Behavior
The runtime was used to compare workloads with different priorities using `nice` values.

Observed behavior:
- lower nice value → more CPU share
- higher nice value → slower completion

This demonstrates Linux scheduler goals:
- fairness
- responsiveness
- throughput balancing

---

## 6. Design Decisions and Tradeoffs

### Namespace isolation
- **Choice:** PID + UTS + mount namespaces
- **Tradeoff:** simpler than full container runtime
- **Reason:** enough isolation for course project

### Supervisor architecture
- **Choice:** single long-running daemon
- **Tradeoff:** one failure point
- **Reason:** easier lifecycle tracking

### IPC and logging
- **Choice:** UNIX socket + pipe logging
- **Tradeoff:** more synchronization complexity
- **Reason:** clear separation of control and logs

### Kernel monitor
- **Choice:** timer-based RSS polling
- **Tradeoff:** periodic, not instant
- **Reason:** simpler and safe for LKM

### Scheduling experiment
- **Choice:** compare `nice 0` vs `nice 10`
- **Tradeoff:** simple experiment
- **Reason:** clearly observable scheduler behavior

---

## 7. Scheduler Experiment Results

| Workload | Nice | Completion Behavior |
|---|---:|---|
| cpu1 | 0 | Faster |
| cpu2 | 10 | Slower |

### Observation
The Linux scheduler gave higher CPU preference to the lower nice value workload, improving throughput for `cpu1` while maintaining fairness.

This aligns with Linux CFS scheduling behavior.
