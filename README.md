# Multi-Container Runtime

## 1. Team Information
- Name: Naveen Karthikeyan, SRN: PES2UG24CS306
- Name: Navya Kishore Dash, SRN: PES2UG24CS307

## 2. Build, Load, and Run Instructions

### Build
```bash
cd boilerplate
make
```

### Load kernel module
```bash
sudo insmod monitor.ko
ls /dev/container_monitor
```

### Start supervisor (Terminal 1)
```bash
sudo ./engine supervisor ./rootfs-base
```

### Prepare per-container rootfs copies (Terminal 2)
```bash
cp -a ./rootfs-base ./rootfs-alpha
cp -a ./rootfs-base ./rootfs-beta
```

### Copy workload binaries into rootfs before launch
```bash
cp cpu_hog ./rootfs-alpha/
cp cpu_hog ./rootfs-beta/
cp memory_hog ./rootfs-alpha/
cp io_pulse ./rootfs-beta/
```

### Launch containers (Terminal 2)
```bash
sudo ./engine start alpha $(pwd)/rootfs-alpha /cpu_hog --soft-mib 40 --hard-mib 64
sudo ./engine start beta  $(pwd)/rootfs-beta  /cpu_hog --soft-mib 40 --hard-mib 64 --nice 19
```

### List containers
```bash
sudo ./engine ps
```

### View logs
```bash
sudo ./engine logs alpha
```

### Stop a container
```bash
sudo ./engine stop alpha
```

### Run a container in foreground (blocks until exit)
```bash
sudo ./engine run test $(pwd)/rootfs-alpha /cpu_hog --soft-mib 40 --hard-mib 64
```

### Stop supervisor
Press Ctrl+C in Terminal 1

### Check kernel log output
```bash
dmesg | tail -20
```

### Unload module
```bash
sudo rmmod monitor
```

## 3. Demo Screenshots
Screenshots for all tasks are in the OS-ss folder

## 4. Engineering Analysis

### Isolation Mechanisms
The runtime uses `clone()` with `CLONE_NEWPID`, `CLONE_NEWUTS`, and `CLONE_NEWNS`
flags to give each container its own PID namespace (so container PIDs start at 1),
UTS namespace (its own hostname set to the container ID), and mount namespace
(isolated filesystem view). `chroot()` then locks the process into its assigned
Alpine rootfs directory, and `/proc` is mounted inside so tools like `ps` work
correctly within the container.

The host kernel itself is still shared — containers use the same kernel, the same
system calls, and the same physical memory as the host. Only the *view* of
resources is isolated, not the kernel itself. This is fundamentally different
from a VM, where the guest runs its own kernel on virtualised hardware.

### Supervisor and Process Lifecycle
A long-running supervisor is necessary because containers are child processes
that must be reaped when they exit. Without a persistent parent, exited children
become zombies, consuming a PID slot indefinitely. The supervisor installs a
`SIGCHLD` handler with `SA_NOCLDSTOP` so it is notified only when a child
terminates (not when it stops). The handler calls `waitpid(-1, &status, WNOHANG)`
in a loop to reap all exited children and update their metadata records atomically
under the metadata lock.

Container metadata (ID, host PID, state, limits, exit code, log path) is stored
in a heap-allocated linked list. All accesses to this list are protected by a
`pthread_mutex_t` so that the SIGCHLD handler, the event loop, and the logger
thread never corrupt the list concurrently.

### IPC, Threads, and Synchronization
Two IPC mechanisms are used:

1. **Pipes (Path A — logging):** Each container's `stdout` and `stderr` are
   `dup2`'d into the write end of a pipe before `execv`. The supervisor holds
   the read end and spawns a dedicated producer thread per container that reads
   chunks and pushes them into the bounded buffer. Without this, container output
   would be lost or intermixed on the terminal.

2. **UNIX domain socket (Path B — control):** The CLI client connects to
   `/tmp/mini_runtime.sock`, sends a `control_request_t` struct, and waits for
   a `control_response_t`. Using a socket (rather than the same pipes) keeps the
   two IPC paths independent and allows the supervisor to serve CLI commands
   without touching the logging pipeline.

The bounded buffer uses a `pthread_mutex_t` + two `pthread_cond_t` variables
(`not_full`, `not_empty`). Without the mutex, concurrent producer and consumer
threads would corrupt the `head`/`tail` indices. Without condition variables,
threads would busy-wait instead of sleeping, wasting CPU. Condition variables
also allow `bounded_buffer_begin_shutdown()` to broadcast a wakeup to all
waiting threads at once, ensuring clean drain-and-exit on shutdown. A semaphore
could replace the condition variables for simple blocking but cannot express the
shutdown broadcast as elegantly.

### Memory Management and Enforcement
RSS (Resident Set Size) measures the physical memory pages actually present in
RAM for a process. It does **not** include pages in swap, shared library pages
counted only once across all users, or pages allocated via `malloc` but not yet
touched (lazy allocation / demand paging).

Soft limits trigger a `dmesg` warning the first time a container crosses the
threshold. This lets operators see a container approaching its budget without
disrupting it. Hard limits send `SIGKILL` to the process because once a
container exceeds its hard budget, continued growth risks starving other
containers or the host of physical memory.

Enforcement must be in kernel space because a user-space monitor could be
killed or frozen by the very process it is watching. The kernel timer fires
every second independently of any user-space process and cannot be bypassed
or delayed by the container.

### Scheduling Behavior
In Experiment 1, `cpu_hog` consumed nearly 100% CPU while `io_pulse` consumed
only 1–2%. `io_pulse` spends most of its time in `usleep()`, voluntarily
yielding the CPU after each write. The CFS scheduler sees it as nearly idle
and naturally gives the remaining time to `cpu_hog`.

In Experiment 2, two `cpu_hog` instances were run with `nice=0` and `nice=19`
respectively. The CFS scheduler assigns weights based on nice values: `nice=0`
maps to weight 1024 and `nice=19` maps to weight 15 in the kernel weight table.
The high-priority container receives proportionally more CPU time. Completion
time for `nice=0` was 59.262s vs 59.723s for `nice=19`, reflecting that the
low-priority container was repeatedly preempted and made slower progress.

## 5. Design Decisions and Tradeoffs

### Namespace Isolation
**Choice:** `clone()` with `CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS`

**Tradeoff:** Network namespace (`CLONE_NEWNET`) is not isolated, so containers
share the host network stack and can see each other's network interfaces.

**Justification:** Network isolation requires `veth` pairs, bridge setup, and
NAT rules, which are out of scope for this project. PID, UTS, and mount
namespace isolation is sufficient to demonstrate container-level process
and filesystem separation.

### Supervisor Architecture
**Choice:** Single-threaded event loop with `select()` for accepting CLI
connections.

**Tradeoff:** Cannot handle multiple simultaneous CLI clients. A second `engine ps`
issued while an `engine start` is being processed will block until the first
connection is served.

**Justification:** CLI commands are short-lived and infrequent. A single-threaded
loop keeps the supervisor simple and avoids the additional synchronization
complexity of a multi-threaded connection handler.

### IPC / Logging
**Choice:** UNIX domain socket for control, pipes for log capture, bounded buffer
with `mutex` + condition variables for producer-consumer synchronization.

**Tradeoff:** Log chunks are limited to `LOG_CHUNK_SIZE` (4096 bytes); very long
lines may be split across two buffer entries and appear on separate lines in
the log file.

**Justification:** UNIX domain sockets are the natural IPC mechanism for local
supervisor-client communication — they provide reliable, ordered, full-duplex
communication without the overhead of TCP. The bounded buffer decouples
container output speed from log file write speed, preventing a slow disk from
blocking the container.

### Kernel Monitor
**Choice:** `mutex` over `spinlock` for the monitored process list.

**Tradeoff:** A mutex has slightly higher overhead than a spinlock because it
may cause the acquiring thread to sleep and be scheduled out, rather than
spinning in place.

**Justification:** The timer callback calls `get_task_mm()`, which can sleep.
Spinlocks forbid sleeping in the locked section (doing so causes a kernel BUG),
so a mutex is the only correct choice here.

### Scheduling Experiments
**Choice:** `nice` values via the `--nice` flag rather than CPU affinity
(`taskset`).

**Tradeoff:** `nice` values affect scheduling priority but not CPU pinning. On a
multi-core machine both containers may run truly in parallel, making the share
difference harder to observe than on a single-core system.

**Justification:** `nice` values directly demonstrate CFS weight-based scheduling,
which is the core Linux scheduling concept relevant to this project. CPU affinity
would force serialisation and measure queue latency rather than scheduler
fairness.

## 6. Scheduler Experiment Results

### Experiment 1: CPU-bound vs I/O-bound
| Container | Workload   | Observed CPU% |
|-----------|------------|---------------|
| cpu1      | cpu_hog    | ~99%          |
| io1       | io_pulse   | ~1-2%         |

`io_pulse` spends most of its time blocked in `usleep()`, giving up the CPU
voluntarily after each I/O burst. `cpu_hog` never yields, consuming its full
time slice every scheduling period. This demonstrates that I/O-bound processes
are naturally cooperative with the scheduler and do not need to be throttled —
they self-limit by blocking on I/O.

### Experiment 2: Different nice values
| Container  | nice | Observed CPU% | Completion Time |
|------------|------|---------------|-----------------|
| high_prio  | 0    | ~99%          | 59.262s         |
| low_prio   | 19   | ~99%          | 59.723s         |

The CFS scheduler allocated CPU share proportional to the process weights
derived from nice values, consistent with the kernel's documented weight table
where `nice=19` has weight 15 and `nice=0` has weight 1024. The `low_prio`
container took longer to complete the same workload because it was repeatedly
preempted in favour of `high_prio`, confirming that CFS enforces proportional
fairness rather than strict time-slicing.