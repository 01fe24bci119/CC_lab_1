# Performance Analysis: Type-1 (Proxmox VE) vs Type-2 (VMware Workstation) Hypervisors

## 1. Executive Summary

This study evaluates and contrasts the CPU computation performance and virtualization overhead of a **Type-1 Bare-Metal Hypervisor (Proxmox VE / KVM)** against a **Type-2 Hosted Hypervisor (VMware Workstation)** under identical hardware configurations and workloads.

Benchmarking was conducted using `sysbench` prime-number computation (`--cpu-max-prime=20000`) on **Ubuntu 22.04 LTS** virtual machines configured with **2 vCPUs**, **2 GB RAM**, and **20 GB disk**.

The results demonstrate that **Type-1 Hypervisor (Proxmox VE)** outperforms **Type-2 Hypervisor (VMware Workstation)** across all key computing metrics:
- **Throughput Advantage**: Proxmox VE achieved **1,716.69 events/sec** compared to **875.72 events/sec** on VMware Workstation, delivering **+96.03% higher computational throughput**.
- **Event Capacity**: In the standard 10-second test window, Proxmox VE processed **17,169 events** versus **8,760 events** on VMware Workstation.
- **Latency Reduction**: Average task latency was **0.58 ms** on Proxmox VE versus **1.14 ms** on VMware Workstation—a **49.12% reduction in processing latency**.
- **Tail Latency Stability**: 95th percentile latency remained tightly bounded at **0.65 ms** on Type-1, whereas Type-2 experienced **1.82 ms** (and maximum spikes up to 8.83 ms), highlighting the superior determinism of bare-metal scheduling.

---

## 2. Experimental Environment & Standard VM Configuration

To ensure strict fairness and scientific reproducibility, both virtual machines were configured with identical resource allocations:

| Configuration Parameter | Type-1 Hypervisor (Proxmox VE) | Type-2 Hypervisor (VMware Workstation) |
| :--- | :--- | :--- |
| **Hypervisor Role** | Bare-Metal Hypervisor (KVM/QEMU) | Hosted Hypervisor (Application over Host OS) |
| **Hypervisor Version** | Proxmox VE 8.3.0 | VMware Workstation 17.x |
| **Host Environment** | Dedicated Linux Kernel (pve-kernel) | Windows 11 Enterprise / Pro Host |
| **Guest OS** | Ubuntu 22.04 LTS (x86_64) | Ubuntu 22.04 LTS (x86_64) |
| **Virtual CPUs (vCPU)** | 2 Cores (1 Socket, 2 Cores/Socket) | 2 Cores (1 Socket, 2 Cores/Socket) |
| **Memory Allocation** | 2048 MiB (2.0 GiB RAM) | 2048 MiB (2.0 GiB RAM) |
| **Virtual Disk Size** | 20.00 GiB SCSI (`local-lvm:20`) | 20.00 GiB Virtual Disk (NVMe/SCSI) |
| **Benchmark Tool** | Sysbench 1.0.x | Sysbench 1.0.x |
| **Benchmark Workload** | `sysbench cpu --cpu-max-prime=20000 run` | `sysbench cpu --cpu-max-prime=20000 run` |

---

## 3. Quantitative Benchmark Results

The benchmark output parameters collected from the respective virtual machine runs are summarized below:

| Metric / Parameter | Proxmox VE (Type-1) | VMware Workstation (Type-2) | Absolute Delta | Relative Gain / Difference |
| :--- | :--- | :--- | :--- | :--- |
| **Total Execution Time** | `10.0004 s` | `10.0008 s` | -0.0004 s | Equal duration window (~10s) |
| **Total Events Processed** | `17,169` | `8,760` | +8,409 events | **+96.03%** on Type-1 |
| **Events per Second** | `1,716.69 ev/s` | `875.72 ev/s` | +840.97 ev/s | **+96.03% Throughput** |
| **Average Latency** | `0.58 ms` | `1.14 ms` | -0.56 ms | **49.12% Faster (Lower)** |
| **Minimum Latency** | `0.57 ms` | `0.94 ms` | -0.37 ms | **39.36% Lower Baseline** |
| **Maximum Latency** | `2.78 ms` | `8.83 ms` | -6.05 ms | **68.52% Lower Peak Spike** |
| **95th Percentile Latency** | `0.65 ms` | `1.82 ms` | -1.17 ms | **64.29% Tighter Tail** |
| **Latency Sum** | `9,996.45 ms` | ~9,990.00 ms | - | Consistent test window |

Data Source: [benchmark_results.csv](benchmark_results.csv)

---

## 4. Architectural Analysis & Performance Drivers

The substantial performance divergence observed between Proxmox VE and VMware Workstation is explained by fundamental differences in hypervisor architecture:

### 4.1 Direct Hardware Execution vs Host OS Intermediary
```
  [Type-1: Proxmox VE]                   [Type-2: VMware Workstation]
+-------------------------+             +-----------------------------+
| Guest VM (Ubuntu 22.04) |             |   Guest VM (Ubuntu 22.04)   |
+-------------------------+             +-----------------------------+
| Proxmox Kernel / KVM    |             | VMware Workstation (App)    |
+-------------------------+             +-----------------------------+
|    Physical Hardware    |             | Host OS (Windows 11 Kernel) |
+-------------------------+             +-----------------------------+
                                        |      Physical Hardware      |
                                        +-----------------------------+
```

1. **Type-1 (Proxmox VE / KVM)**:
   - Operates directly on the bare-metal hardware. KVM transforms the Linux kernel itself into a hypervisor via kernel modules (`kvm.ko`, `kvm-intel.ko`).
   - CPU instructions in guest VMs are executed directly by the physical CPU using hardware virtualization extensions (Intel VT-x / AMD-V) with zero user-space host mediation.
   - The hypervisor's Completely Fair Scheduler (CFS) schedules vCPU threads directly onto physical CPU cores, avoiding multi-tiered context switching.

2. **Type-2 (VMware Workstation)**:
   - Runs as an application layer on top of a general-purpose host operating system (Windows 11).
   - CPU calls and hardware requests must traverse multiple abstraction layers: Guest VM &rarr; Virtual VMM &rarr; Host OS Drivers &rarr; Host Windows OS Kernel &rarr; Physical Hardware.
   - The host OS kernel constantly competes with the virtual machine for CPU time, scheduling host background tasks, UI rendering, system interrupts, and antivirus scanners.

### 4.2 Latency Distribution and Tail Predictability
- **Minimum Latency (0.57 ms vs 0.94 ms)**: Proxmox VE demonstrates the near-native processing capability of KVM, with minimal virtualization entry/exit (VM-exit) penalty.
- **Maximum Latency (2.78 ms vs 8.83 ms)**: Under VMware Workstation, unexpected execution pauses occur whenever the host Windows kernel preempts the VMware process to service high-priority host interrupts or DPCs (Deferred Procedure Calls). Proxmox VE maintains tight control over physical cores, preventing latency spikes.
- **95th Percentile (0.65 ms vs 1.82 ms)**: 95% of all events on Proxmox VE completed within 0.65 ms, ensuring predictable quality-of-service (QoS) critical for production cloud and server workloads.

---

## 5. Comparative Visualizations

Comprehensive multi-metric charts and tables are generated from the benchmark data:

- **Graphical Comparison**: [graphs/performance_comparison.png](../graphs/performance_comparison.png)  
  Displays isolated axes for throughput (ev/s), event count, latency (ms), and duration (s).
- **Comparison Evidence Table**: [screenshots/comparison/01-hypervisor-performance-comparison.png](../screenshots/comparison/01-hypervisor-performance-comparison.png)  
  Tabular reference capturing all primary and secondary latency percentiles.

---

## 6. Conclusion & Recommendations

| Workload Requirement | Recommended Hypervisor | Rationale |
| :--- | :--- | :--- |
| **Enterprise Cloud / Data Centers** | **Proxmox VE (Type-1)** | High computational throughput (+96%), lowest latency (0.58 ms), minimal resource overhead, bare-metal isolation. |
| **Production Database & HPC** | **Proxmox VE (Type-1)** | Tight latency percentiles, zero host OS preemption interference. |
| **Local Desktop Development** | **VMware Workstation (Type-2)** | Convenience, rapid desktop prototyping, cross-platform portability on desktop machines without dedicated hardware. |
