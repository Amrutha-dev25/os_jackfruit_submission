---

# Multi-Container Runtime (OS-Jackfruit)

---

## 1. Team Information

* Name: AMRUTHA KATTIMANI

* SRN: PES1UG24CS054

* Name: SAMHITHA

* SRN: PES1UG24CS050



---

## 2. Build, Load, and Run Instructions

### Setup (first time only)
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)


---

### Build the project
make


---

### Prepare base root filesystem
mkdir rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base


---

### Load kernel module
sudo insmod monitor.ko


Check if device is created:
ls -l /dev/container_monitor


---

### Start supervisor
sudo ./engine supervisor ./rootfs-base

---

### Create container rootfs copies
cp -a ./rootfs-base ./rootfs-alpha
cp -a ./rootfs-base ./rootfs-beta


---

### Start containers (run in another terminal)
sudo ./engine start alpha ./rootfs-alpha /bin/sh --soft-mib 48 --hard-mib 80
sudo ./engine start beta ./rootfs-beta /bin/sh --soft-mib 64 --hard-mib 96


---

### Check running containers
sudo ./engine ps

---

### View logs
sudo ./engine logs alpha


---

### Running workloads

If you have a test program:
cp workload_binary ./rootfs-alpha/


Then run it using the `run` command or inside the container shell.

---

### Scheduling experiments

* Run two containers at the same time
* Use different `--nice` values
* Try CPU-heavy and I/O-heavy programs
* Observe differences in execution

---

### Stop containers
sudo ./engine stop alpha
sudo ./engine stop beta


---

### Stop supervisor

Press `Ctrl + C` in the supervisor terminal or kill the process.

---

### Check kernel logs
dmesg | tail

---

### Unload module
sudo rmmod monitor


---

## 3. Demo with Screenshots

| # | What to Demonstrate         | What the Screenshot Must Show                             |
| - | --------------------------- | --------------------------------------------------------- |
| 1 | Multi-container supervision | At least two containers running under the same supervisor |
| 2 | Metadata tracking           | Output of `engine ps` showing container details           |
| 3 | Bounded-buffer logging      | Log file output showing captured logs                     |
| 4 | CLI and IPC                 | A command being sent and response from supervisor         |
| 5 | Soft-limit warning          | Warning message in `dmesg` or logs                        |
| 6 | Hard-limit enforcement      | Container killed after exceeding limit                    |
| 7 | Scheduling experiment       | Difference in execution between workloads                 |
| 8 | Clean teardown              | No zombie processes after stopping everything             |

---

## 4. Engineering Analysis

### Isolation Mechanisms

Each container runs in its own environment using namespaces.
PID namespace separates process IDs, UTS changes hostname, and mount namespace isolates filesystem.
`chroot()` is used so each container only sees its own root filesystem.
Even though containers are isolated, they still share the same Linux kernel.

---

### Supervisor and Process Lifecycle

Instead of launching containers directly, everything goes through a supervisor.
This makes it easier to manage multiple containers at once.
The supervisor keeps track of each container and handles signals properly.
It also makes sure no zombie processes are left behind.

---

### IPC, Threads, and Synchronization

Two types of communication are used:

* Pipes for logs (container → supervisor)
* Socket/FIFO for commands (CLI → supervisor)

Since multiple threads are involved, synchronization is needed.
Mutex locks are used to protect shared data, and condition variables help manage the buffer.

Without this, logs could get mixed up or the system could hang.

---

### Memory Management and Enforcement

Memory usage is tracked using RSS (resident set size).

Soft limit just gives a warning when exceeded.
Hard limit forcefully kills the container.

This is handled in kernel space because user programs cannot reliably control memory usage.

---

### Scheduling Behavior

Experiments showed how Linux scheduling works in practice.

Processes with higher priority (lower nice value) got more CPU time.
I/O-bound programs stayed responsive, while CPU-heavy ones used most of the CPU.

This matches how the Linux scheduler is designed.

---

## 5. Design Decisions and Tradeoffs

| Component      | Choice         | Tradeoff                    | Reason               |
| -------------- | -------------- | --------------------------- | -------------------- |
| Isolation      | chroot()       | Not as secure as pivot_root | Easier to implement  |
| Supervisor     | Single process | Central dependency          | Simpler control      |
| IPC            | Pipes + socket | Slight overhead             | Works reliably       |
| Logging        | Bounded buffer | More code complexity        | Prevents data loss   |
| Kernel Monitor | LKM            | Needs root privileges       | Better control       |
| Scheduling     | nice values    | Limited tuning              | Simple and effective |

---

## 6. Scheduler Experiment Results

### Experiment 1: Two CPU-bound processes

| Container | Nice Value | Result          |
| --------- | ---------- | --------------- |
| A         | 0          | Finished faster |
| B         | 10         | Took longer     |

This shows higher priority processes get more CPU time.

---

### Experiment 2: CPU-bound vs I/O-bound

| Type      | Observation      |
| --------- | ---------------- |
| CPU-bound | Uses most CPU    |
| I/O-bound | Responds quickly |

I/O-bound tasks get scheduled sooner after waiting, so they feel more responsive.

---

### Final Observation

The Linux scheduler balances fairness and responsiveness.
It does not give equal time blindly — it adjusts based on process behavior.

---

