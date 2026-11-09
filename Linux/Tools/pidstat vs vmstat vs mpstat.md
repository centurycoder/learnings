# pidstat vs. vmstat vs. mpstat

**Use `vmstat` to classify system pressure, `mpstat` to inspect CPU balance, and `pidstat` to identify responsible processes or threads.**

## Which tool should I use?

| Question / symptom                                                    | Tool and command             | Fields to inspect                                  |
| --------------------------------------------------------------------- | ---------------------------- | -------------------------------------------------- |
| The machine is slow; is it CPU, memory, or I/O pressure?              | `vmstat 1 10`                | `r`, `b`, `si`, `so`, `us`, `sy`, `id`, `wa`, `st` |
| Is CPU demand spread across all cores or concentrated on one?         | `mpstat -P ALL 1 10`         | Per-CPU `%usr`, `%sys`, `%soft`, `%idle`           |
| Which process is using CPU?                                           | `pidstat -u -p ALL 1 10`     | `PID`, `Command`, `%usr`, `%system`, `%CPU`        |
| Which thread is hot inside a process?                                 | `pidstat -t -u -p 1234 1 10` | `TGID`, `TID`, `%CPU`                              |
| Is a process faulting pages or using substantial resident memory?     | `pidstat -r -p 1234 1 10`    | `minflt/s`, `majflt/s`, `RSS`, `%MEM`              |
| Which process is generating disk I/O?                                 | `pidstat -d -p ALL 1 10`     | `kB_rd/s`, `kB_wr/s`, `iodelay`                    |
| Is a process frequently blocking or being preempted?                  | `pidstat -w -p 1234 1 10`    | `cswch/s`, `nvcswch/s`                             |
| Is interrupt processing or VM host contention consuming CPU capacity? | `mpstat -P ALL 1 10`         | `%irq`, `%soft`, `%steal`                          |

Commands sample once per second for 10 reports. Replace `1234` with the actual PID. `pidstat` and `mpstat` come from **sysstat**; `vmstat` comes from **procps/procps-ng**.

## 1. vmstat: system-wide pressure

```bash
vmstat 1 10
```

### Key fields and interpretation

| Field | Meaning | What to notice / what it indicates |
|---|---|---|
| `r` | Runnable tasks: running or waiting for CPU | Sustained values above available CPU capacity, together with low `id`, suggest CPU contention. A brief spike alone is not enough. |
| `b` | Tasks blocked waiting for I/O | Persistent elevation suggests blocked work; correlate with storage latency and task states. |
| `swpd` | Amount of swap in use | Nonzero does not prove current memory pressure; old, inactive pages can remain swapped out. |
| `free` | Unused memory | Low alone is not a problem: Linux uses memory for caches. Check `MemAvailable` in `/proc/meminfo` or `available` in `free -h`. |
| `buff`, `cache` | Buffer and cache memory | Some is reclaimable; do not treat all of it as unavailable application memory. |
| `si`, `so` | Swap-in and swap-out rates | Sustained activity indicates active swapping; high rates plus poor responsiveness suggest memory pressure. |
| `bi`, `bo` | Block-device input and output rates | Show I/O volume, not latency or device saturation. Use `iostat -xz 1 10` to investigate storage. |
| `in` | Interrupts per second, including clock interrupts | A rise can accompany device activity. Correlate with `mpstat` interrupt CPU time and `/proc/interrupts`. |
| `cs` | Context switches per second | High relative to the workload baseline can reflect concurrency, frequent blocking, or scheduling contention; not inherently a problem. |
| `us` | CPU time executing user-space code | High with low idle time points toward application computation; use `pidstat` to attribute it. |
| `sy` | CPU time executing kernel code | High suggests kernel work such as syscalls, networking, or memory management; does not identify the cause by itself. |
| `id` | CPU idle time | Sustained near-zero values indicate little idle capacity in the aggregate; averages can hide a single busy CPU. |
| `wa` | CPU I/O-wait accounting | Not CPU execution and not a direct measure of disk latency. Correlate with `b` and device statistics. |
| `st` | CPU time stolen by the hypervisor | Sustained elevation indicates VM scheduling contention outside the guest. |

**Sampling caveat:** the first report's CPU and rate fields are averages since boot. Subsequent reports cover the interval. Process and memory fields are current values. Ignore the first report when diagnosing current activity.

Default memory units are KiB; default `bi`/`bo` units are blocks of 1 KiB per second in procps-ng. Memory display units can be changed with `-S`.

## 2. mpstat: aggregate and per-CPU utilization

```bash
mpstat -P ALL 1 10
```

The `all` row is aggregate utilization; numbered rows represent individual **logical CPUs**, not necessarily separate physical cores.

| Field | Meaning | What it indicates |
|---|---|---|
| `CPU` | Logical CPU number, or `all` | Compare rows: one busy CPU with others idle suggests a single-thread bottleneck, affinity restriction, or uneven interrupt distribution. |
| `%usr` | Ordinary user-space execution | High means application computation is consuming CPU. |
| `%nice` | User-space execution of processes with adjusted nice priority | CPU spent on niced work; priority adjustment does not impose a CPU usage limit. |
| `%sys` | Kernel execution, excluding hardware/software interrupt handling | High suggests syscall or other kernel work. |
| `%irq` | Hardware interrupt handling | High suggests substantial device interrupt processing. |
| `%soft` | Software interrupt handling | High often accompanies network processing, but other softirq work is possible. |
| `%iowait` | I/O-wait accounting | Not CPU execution. Investigate blocked tasks and device latency rather than assuming CPU saturation. |
| `%steal` | Hypervisor-stolen CPU time | Guest demand is competing for host CPU capacity. |
| `%idle` | Idle time without outstanding I/O attributed to that CPU | Low on every CPU suggests broad demand; low on only one suggests a localized bottleneck. |

`%guest` and `%gnice`, when present, account for time running virtual CPUs; relevant when the Linux system itself hosts virtual machines.

**Example:** one logical CPU at nearly 100% execution can bottleneck a single-threaded application while aggregate utilization remains low. `vmstat` alone can hide this imbalance.

## 3. pidstat: attribute activity to processes and threads

```bash
pidstat -u -p ALL 1 10        # CPU by process
pidstat -t -u -p 1234 1 10    # CPU by thread
pidstat -r -d -w -p 1234 1 10 # Memory, I/O, and context switches
```

### CPU: `-u`, optionally `-t`

| Field | Meaning | What it indicates |
|---|---|---|
| `PID`, `Command` | Process identity | Identify the owner of the activity, not just the busiest executable name. |
| `TGID`, `TID` | Process/thread-group ID and individual thread ID in thread mode | Locate hot threads within the process; do not add the process summary to its thread rows. |
| `%usr` | User-space CPU usage | Application computation; a profile is needed to identify hot functions. |
| `%system` | Kernel CPU usage charged to the task | Syscall/kernel work associated with the process. |
| `%CPU` | Total task CPU usage | Sustained high values identify CPU consumers; compare against throughput and expected workload. |
| `%wait` | Time the task waited to run on CPU, when supported | Elevated values suggest scheduling contention, not disk I/O wait. |
| `CPU` | CPU number associated with the task sample | A location indicator, not utilization or proof of CPU affinity; tasks can migrate. |

**CPU scale:** by default, `pidstat` reports 100% as one logical CPU's worth of time. A multithreaded process can exceed 100%. `pidstat -I` divides usage by the number of processors. In contrast, `mpstat`'s `all` row and `vmstat` CPU percentages use a whole-system scale.

### Memory and faults: `-r`

| Field | Meaning | What it indicates |
|---|---|---|
| `minflt/s` | Minor page faults per second; no disk load required | Often normal allocation or mapping activity; high values alone do not prove memory pressure. |
| `majflt/s` | Major page faults per second; a page must be loaded from disk | Can add latency; may involve file-backed pages or swap. Not proof of swapping by itself. |
| `VSZ` | Virtual address-space size, in KiB | Not actual RAM consumption; includes mappings and reserved address space. |
| `RSS` | Resident memory, in KiB | Tracks resident pages, including shared pages. Do not sum RSS across processes as unique physical memory usage. |
| `%MEM` | RSS as a percentage of reported physical memory | Useful for identifying large residents; may not reflect a container's memory limit. |

### I/O: `-d`

| Field | Meaning | What it indicates |
|---|---|---|
| `kB_rd/s`, `kB_wr/s` | Process-accounted disk read/write rates | Attribute I/O volume; not application-level read/write volume, device latency, or saturation. Caching and delayed writeback affect interpretation. |
| `kB_ccwr/s` | Cancelled write rate | Writes accounted for but cancelled before disk writeback, for example by truncation. |
| `iodelay` | Accumulated block-I/O and swap-in delay during the reporting interval, in clock ticks | Elevated delay suggests waiting on these operations. Requires kernel task delay accounting; unavailable accounting can produce zero values. |

### Context switches: `-w`

| Field | Meaning | What it indicates |
|---|---|---|
| `cswch/s` | Voluntary switches per second: the task gives up CPU | Often waiting for I/O, locks, timers, or work; not necessarily unhealthy. |
| `nvcswch/s` | Involuntary switches per second: the task is preempted | Can reflect CPU competition or timeslice expiration; interpret with `%wait`, CPU demand, and a workload baseline. |

## Practical triage sequence

1. **Classify pressure with `vmstat 1 10`.** Low `id` plus sustained high `r` suggests CPU contention; `si`/`so` suggests swapping; `b` plus `wa` suggests investigating I/O.
2. **Check CPU distribution with `mpstat -P ALL 1 10`.** Separate one-CPU bottlenecks from broad saturation; inspect interrupt and steal time.
3. **Attribute demand with `pidstat -u -p ALL 1 10`.** For the suspect PID, inspect threads with `-t`, or add `-r`, `-d`, and `-w` according to the pressure observed.
4. **Find the cause, not just the counter.** Use a sampling profiler for hot code, `iostat` for device behavior, and workload/deployment metrics for correlation.
5. **Validate a mitigation under comparable load.** Check latency and throughput as well as CPU; lower CPU caused by completing less work is not a successful fix.

## Interpretation rules

- No universal threshold for context switches, faults, or I/O rates: compare with the application's normal baseline and service performance.
- Load average includes runnable tasks and tasks in uninterruptible sleep; high load alone does not prove high CPU execution.
- In containers, host CPU averages can look healthy while the workload is throttled. Check the actual cgroup's CPU quota and `cpu.stat` throttling counters; account for CPU affinity and cpusets.
- Fields and availability vary with sysstat/procps and kernel versions. Access restrictions can hide process details; check the local manual when output differs.
