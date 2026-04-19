# Multi-Container Runtime

A lightweight Linux container runtime in C with a long-running supervisor and a kernel-space memory monitor.

Read [`project-guide.md`](project-guide.md) for the full project specification.

---
Team details:
| Name           |       SRN      | 
|----------------|----------------|
| Jagriti Mohan  | PES2UG24CS200  | 
| Joel William   | PES2UG24CS205  | 


# Multicontainer Runtime Project  
### Build, Load, Run, and Cleanup Guide (Ubuntu 22.04 / 24.04)

---

## Overview
This project implements a multicontainer runtime consisting of:
- A **kernel module** for low-level container handling  
- A **supervisor** (user-space controller)  
- A **CLI tool** for managing containers  

This guide provides **step-by-step instructions** to reproduce the setup from scratch.

---

##  1. System Requirements

- Ubuntu 22.04 / 24.04
- GCC, Make
- Linux Kernel Headers
- Root (sudo) access

---

### Architecture Overview

The runtime is a single binary (`engine`) used in two ways:

1. **Supervisor daemon** — started once with `engine supervisor ./rootfs-base`. It stays alive, manages containers, and owns the logging pipeline.
2. **CLI client** — each command like `engine start alpha ./rootfs-alpha /bin/sh` is a short-lived process that sends a request to the running supervisor and prints the response.

The CLI process connects to the supervisor over an IPC control channel, sends a command, receives a response, and exits. The supervisor, upon receiving a `start` command, calls `clone()` to create a new container child process with its own namespaces. Each container's stdout and stderr are connected back to the supervisor via pipes, which feed into the bounded-buffer logging pipeline.

There are two separate IPC paths in this project:

- **Path A (logging):** Container stdout/stderr → Supervisor, via pipes. Described in Task 3.
- **Path B (control):** CLI process → Supervisor, via a UNIX domain socket, FIFO, or shared-memory channel. Described in Task 2.

### CLI Contract

Use this exact command interface:

```bash
engine supervisor <base-rootfs>
engine start <id> <container-rootfs> <command> [--soft-mib N] [--hard-mib N] [--nice N]
engine run   <id> <container-rootfs> <command> [--soft-mib N] [--hard-mib N] [--nice N]
engine ps
engine logs <id>
engine stop <id>
```

### RESULTS
#### task1:  Multiple containers running under supervisor
#### task2: containers metadata using ps
#### task3: logging output of cpu_hog
#### task4: CLI to supervisor IPC communication
#### task5:  Soft and hard memory limit enforcement
#### task6: Scheduling Experiment results
#### task7: clean the teardown with no zombie processes


# Engineering Analysis

This section explains the design reasoning, tradeoffs, and observed system behavior, focusing on how operating system mechanisms are exercised in this project.

---

## 5. Design Decisions and Tradeoffs

### 5.1 Namespace Isolation

**Design Choice:**  
Used Linux namespaces (PID, mount, network) to isolate containers while sharing the host kernel.

**Tradeoff:**  
Namespaces are lightweight and provide efficient performance compared to full virtualization, but they offer weaker isolation since all containers share the same kernel.

**Justification:**  
Namespaces provide sufficient process-level isolation while maintaining high performance, which is suitable for a lightweight multicontainer runtime.

---

### 5.2 Supervisor Architecture

**Design Choice:**  
Implemented a centralized user-space supervisor to manage container lifecycle and coordinate operations.

**Tradeoff:**  
A centralized design simplifies control and monitoring but introduces a single point of failure.

**Justification:**  
The centralized supervisor reduces system complexity and improves debuggability, making it a practical choice for development and experimentation.

---

### 5.3 IPC and Logging

**Design Choice:**  
Used simple IPC mechanisms such as pipes, signals, or shared memory, along with kernel logs (`dmesg`) for communication and debugging.

**Tradeoff:**  
These mechanisms are easy to implement and debug but may not scale efficiently under heavy workloads.

**Justification:**  
For a prototype system, simplicity and observability are more important than scalability, making this approach appropriate.

---

### 5.4 Kernel Monitor

**Design Choice:**  
Implemented a kernel module to monitor container-related events and system-level activities.

**Tradeoff:**  
Kernel-level access allows detailed monitoring but increases the risk of system instability if errors occur.

**Justification:**  
Direct interaction with the kernel is necessary to observe low-level OS behavior, which is essential for understanding container execution.

---

### 5.5 Scheduling Experiments

**Design Choice:**  
Performed experiments using multiple processes with varying workloads to observe Linux scheduling behavior.

**Tradeoff:**  
This approach provides realistic insights but results may vary depending on system load and environment.

**Justification:**  
Practical experimentation offers better understanding of scheduler behavior compared to purely theoretical analysis.

---

## 6. Scheduler Experiment Results

### Experiment Setup

Multiple container processes were created and assigned CPU-intensive workloads. Observations were recorded using system monitoring tools such as `top`, `htop`, and kernel logs.

---

### Raw Observations

Processes with lower nice values (higher priority) consistently received a larger share of CPU time. As a result, these processes completed execution faster compared to processes with higher nice values.

Processes with higher nice values experienced reduced CPU allocation and longer execution times, especially under CPU contention.

---

### Comparative Analysis

When multiple processes competed for CPU resources, the scheduler favored higher-priority processes by allocating CPU time more frequently to them. Lower-priority processes were scheduled less often, leading to delayed completion.

---

### Key Insights on Linux Scheduling

The Linux scheduler, specifically the Completely Fair Scheduler (CFS), attempts to distribute CPU time fairly among processes. However, fairness is influenced by process priority, which is determined by the nice value.

The experiment demonstrates that fairness does not imply equal CPU distribution. Instead, the scheduler adjusts CPU allocation dynamically to balance system responsiveness and process priority.

---

## Summary

Namespace isolation enables lightweight containerization with efficient performance.  
The supervisor simplifies system management but introduces central dependency.  
Kernel monitoring provides deep visibility into system behavior.  
Scheduling experiments highlight how Linux balances fairness and priority in CPU allocation.

---



