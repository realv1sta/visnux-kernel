# Runtime tuning

This guide collects practical tuning for a running Visnux kernel. Nothing here
requires a rebuild unless noted. Persistent settings belong in `/etc/sysctl.d/`,
`/etc/modprobe.d/`, `/etc/udev/rules.d/`, or your init system’s configuration.

> Tune one thing at a time, measure, and keep notes. Most “performance” settings
> are workload-dependent.

## 1. CPU frequency

### Check the governor

```sh
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_driver
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_governors
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
```

The compiled default is `schedutil` (`CONFIG_CPU_FREQ_DEFAULT_GOV_SCHEDUTIL=y`).
`intel_pstate` and `amd_pstate` are built in.

### Set it

```sh
echo schedutil | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

Persist with `cpupower` or a systemd unit:

```sh
sudo cpupower frequency-set -g schedutil
```

### P-state drivers

| CPU | Driver | Notes |
|---|---|---|
| Intel | `intel_pstate` | Built in; `active` by default |
| AMD | `amd_pstate` | Built in; active mode (`CONFIG_X86_AMD_PSTATE_DEFAULT_MODE=3`) |
| Generic | `acpi-cpufreq` | Module |

```sh
cat /sys/devices/system/cpu/amd_pstate/status   # AMD
```

## 2. Preemption model

`CONFIG_PREEMPT_DYNAMIC=y` allows a boot-time choice:

```text
preempt=full
preempt=voluntary
preempt=none
```

Check:

```sh
cat /sys/kernel/debug/sched/preempt
```

`full` is the default and the right choice for a desktop. `none` favours
throughput on servers.

## 3. Full dynticks (specialist)

`CONFIG_NO_HZ_FULL=y` is compiled in. Activate it only for isolated CPUs:

```text
nohz_full=2-7 rcu_nocbs=2-7
```

Rules:

- never include CPU 0;
- pair with `rcu_nocbs` for the same CPUs;
- isolate the CPUs (`isolcpus=`) and pin workloads if you want the benefit;
- expect lower throughput on those CPUs.

For a normal desktop, leave it off.

## 4. Memory: zswap

zswap is on by default with zstd.

```sh
cat /sys/module/zswap/parameters/enabled
cat /sys/module/zswap/parameters/compressor
cat /sys/module/zswap/parameters/max_pool_percent
cat /sys/module/zswap/parameters/shrinker_enabled
```

### Change the pool size at runtime

```sh
echo 30 | sudo tee /sys/module/zswap/parameters/max_pool_percent
```

### Change the compressor

Available compressors depend on the build; `zstd` is the default.

```sh
echo lz4 | sudo tee /sys/module/zswap/parameters/compressor
```

Changing the compressor may require disabling and re-enabling zswap, and only
works when the pool is empty.

### Boot parameters

```text
zswap.enabled=1 zswap.compressor=zstd zswap.max_pool_percent=25
```

### Is it working?

```sh
grep -E 'zswap|swap' /proc/meminfo
```

Look at `Zswap`, `Zswapped`, `SwapTotal`, `SwapFree`.

## 5. Memory: zram

### One-off setup

```sh
sudo modprobe zram
echo zstd | sudo tee /sys/block/zram0/comp_algorithm
echo 4G    | sudo tee /sys/block/zram0/disksize
sudo mkswap /dev/zram0
sudo swapon -p 100 /dev/zram0
```

### Check

```sh
zramctl
swapon --show
cat /sys/block/zram0/mm_stat
```

### Persist with systemd

`/etc/systemd/zram-generator.conf` (if `zram-generator` is installed):

```ini
[zram0]
zram-size = min(ram / 2, 8192)
compression-algorithm = zstd
swap-priority = 100
```

### zram or zswap?

Use **zram** when there is no disk swap or the disk is slow/small. Use **zswap**
when there is a real swap device and you want a compressed cache in front of it.
Running both is usually redundant.

## 6. Memory: MGLRU

```sh
cat /sys/kernel/mm/lru_gen/enabled    # 0x0007
cat /sys/kernel/mm/lru_gen/min_ttl_ms
```

MGLRU is enabled by default. `min_ttl_ms` controls the minimum time a page stays
in the inactive generation. The default is suitable for most systems.

Debugfs (if mounted):

```sh
sudo cat /sys/kernel/debug/lru_gen
```

## 7. Memory: DAMON

DAMON provides access monitoring and proactive reclaim.

```sh
ls /sys/kernel/mm/damon/admin
```

The easiest path is the `damo` userspace tool:

```sh
sudo damo start --damos_action pageout \
  --monitoring_intervals 5s 100ms 1s \
  --damos_access_rate 0% 0% \
  --damos_age 10s 60s
sudo damo report
sudo damo stop
```

Schemes are policy; a badly chosen scheme can cause thrashing. Start with the
`damo` defaults and observe.

## 8. Memory: THP, KSM, swappiness

### Transparent Huge Pages

```sh
cat /sys/kernel/mm/transparent_hugepage/enabled
cat /sys/kernel/mm/transparent_hugepage/defrag
```

The build default is `always`. For databases:

```sh
echo madvise | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
```

### KSM

```sh
echo 1 | sudo tee /sys/kernel/mm/ksm/run
cat /sys/kernel/mm/ksm/pages_shared
cat /sys/kernel/mm/ksm/pages_sharing
```

KSM saves memory for VMs and containers; it costs CPU.

### Swappiness

```sh
sysctl vm.swappiness
echo 'vm.swappiness=10' | sudo tee /etc/sysctl.d/99-swap.conf
```

With zswap/zram, a low swappiness is often fine because swap is cheap. Measure.

## 9. I/O scheduler

```sh
cat /sys/block/nvme0n1/queue/scheduler
echo bfq | sudo tee /sys/block/nvme0n1/queue/scheduler
```

### udev rule

```udev
# /etc/udev/rules.d/60-iosched.rules
ACTION=="add|change", KERNEL=="nvme0n1", ATTR{queue/scheduler}="bfq"
ACTION=="add|change", KERNEL=="sd[a-z]",  ATTR{queue/scheduler}="bfq"
```

Guidance:

| Device | Suggestion |
|---|---|
| SATA SSD/HDD | `bfq` |
| NVMe (desktop) | `bfq` or `none`; measure |
| NVMe (server) | `none` or `mq-deadline` |
| Virtual disk | `mq-deadline` |

### Queue depth and read-ahead

```sh
cat /sys/block/nvme0n1/queue/nr_requests
cat /sys/block/nvme0n1/queue/read_ahead_kb
```

Do not blindly increase these; the defaults are tuned for the device.

## 10. PSI and pressure-based actions

```sh
cat /proc/pressure/cpu
cat /proc/pressure/memory
cat /proc/pressure/io
```

`systemd-oomd` uses PSI to kill processes before the system thrashes. Configure
it in `/etc/systemd/oomd.conf` and per-slice `ManagedOOMMemoryPressure=`.

## 11. sched_ext

```sh
ls /sys/kernel/sched_ext
```

Load a BPF scheduler with the `scx` toolset (for example `scx_lavd`,
`scx_rusty`). Detach if the system becomes unresponsive:

```sh
# The exact interface depends on the scheduler; generally:
# echo 0 | sudo tee /sys/kernel/sched_ext/root/ops
```

`sched_ext` is experimental. Keep a way to recover (TTY, reboot).

## 12. Autogroup

```sh
cat /proc/sys/kernel/sched_autogroup_enabled
echo 1 | sudo tee /proc/sys/kernel/sched_autogroup_enabled
```

Autogroup improves desktop fairness. Disable it only if you have a specific
reason (some server workloads prefer global fairness).

## 13. Networking

### Congestion control

```sh
sysctl net.ipv4.tcp_congestion_control
sudo modprobe tcp_bbr
sudo sysctl net.ipv4.tcp_congestion_control=bbr
```

Persist:

```ini
# /etc/sysctl.d/99-net.conf
net.ipv4.tcp_congestion_control = bbr
```

### Queue discipline

```sh
tc qdisc show dev eth0
sudo tc qdisc replace dev eth0 root fq_codel
```

`fq_codel` is built in; `fq` is a module.

### Buffers

```ini
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
```

Only change these if you have a measured need.

## 14. Virtual memory sysctls

A conservative desktop set:

```ini
# /etc/sysctl.d/99-vm.conf
vm.swappiness = 10
vm.vfs_cache_pressure = 50
vm.dirty_ratio = 10
vm.dirty_background_ratio = 5
```

Notes:

- `vm.swappiness` interacts with zswap/zram; low values are not always better.
- `vm.dirty_*` affects writeback latency; lower values reduce stalls but can
  reduce throughput.
- `vm.vfs_cache_pressure` above 100 reclaims inode/dentry caches more
  aggressively.

## 15. File descriptor and inotify limits

```ini
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 1024
fs.file-max = 2097152
```

Useful for IDEs, browsers, and development tooling.

## 16. Hibernation

```sh
cat /sys/power/state        # should include "disk"
cat /sys/power/resume
```

Set `resume=UUID=<swap>` on the kernel command line. Ensure swap ≥ RAM for
reliable hibernation.

## 17. Measuring, not guessing

Useful tools:

| Tool | Purpose |
|---|---|
| `perf stat` / `perf top` | CPU and cache behaviour |
| `bpftrace` | Latency tracing |
| `iostat`, `vmstat`, `mpstat` | System-wide counters |
| `pressure stall info` | Contention |
| `powertop` | Power |
| `turbostat` | Frequency and C-states |
| `fio` | Storage |
| `iperf3` | Network |

Example: check whether BFQ helps your workload:

```sh
for s in none mq-deadline bfq; do
  echo "$s" | sudo tee /sys/block/nvme0n1/queue/scheduler
  fio --name=test --rw=randread --bs=4k --iodepth=32 --runtime=30 --time_based ...
done
```

Change one variable at a time.

## 18. Reverting

All runtime tuning is reversible:

- sysctl: remove the file and `sysctl --system`, or set the old value;
- module parameters: reload the module or reboot;
- scheduler: write the old scheduler name;
- zram: `swapoff`, reset the device, unload the module;
- sched_ext: detach the BPF scheduler.

If a change makes the system unstable, boot with the previous kernel entry or a
live ISO and remove the offending configuration file.
