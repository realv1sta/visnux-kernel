# Feature deep dive

This document explains what the major enabled features do, why they are in the
Visnux configuration, and how to observe them. Values are from the committed
`.config`.

## 1. Scheduling and latency

### 1.1 Full preemption (`CONFIG_PREEMPT=y`)

A fully preemptible kernel can interrupt kernel code at almost any point to run a
higher-priority task. On a desktop this reduces latency spikes caused by long
kernel operations (filesystem work, driver code, memory reclaim).

`CONFIG_PREEMPT_DYNAMIC=y` additionally allows the model to be selected at boot:

```text
preempt=full        # default
preempt=voluntary
preempt=none
```

Observe:

```sh
cat /sys/kernel/debug/sched/preempt
```

Trade-off: full preemption can reduce throughput slightly and increases locking
complexity in the kernel; it is the right default for interactive systems.

### 1.2 Tick rate and timers (`CONFIG_HZ=1000`, high-res timers)

`HZ=1000` gives a 1 ms scheduling tick. Combined with high-resolution timers,
sleeps and timeouts are accurate to nanoseconds where hardware allows. This
benefits audio, gaming, and latency-sensitive applications.

### 1.3 Tickless operation (`CONFIG_NO_HZ`)

The idle CPU stops its periodic tick, reducing wakeups and power use.
`CONFIG_NO_HZ_FULL=y` is also compiled in, but only activates for CPUs listed in
`nohz_full=`. For normal desktop use, the idle behaviour is what matters.

### 1.4 EEVDF scheduler

Linux 7.2.15 uses EEVDF (Earliest Eligible Virtual Deadline First) as the default
fair scheduler. It replaces the older CFS pick logic and provides better latency
for interactive tasks while keeping fairness.

Relevant tunables:

```sh
ls /proc/sys/kernel/sched_*
cat /proc/sys/kernel/sched_base_slice_ns   # if present
```

### 1.5 `sched_ext` (`CONFIG_SCHED_CLASS_EXT=y`)

`sched_ext` allows BPF programs to implement scheduling policies and attach them
at runtime. The default scheduler is unaffected until a policy is loaded.

```sh
ls /sys/kernel/sched_ext
# with the scx toolset:
# scx_lavd
# scx_rusty
```

This is experimental territory: a bad BPF scheduler can degrade the system.
Detach it if performance collapses.

### 1.6 Core scheduling (`CONFIG_SCHED_CORE=y`)

Core scheduling controls which tasks may share an SMT core. It is useful for
security-sensitive workloads (avoiding cross-thread side channels) and for
grouping related tasks. It is off by default unless enabled per cgroup
(`cpu.core_sched` interface).

### 1.7 `uclamp` (`CONFIG_UCLAMP_TASK=y`)

Utilisation clamping lets tasks request a minimum and maximum performance level.
It is used by power-aware schedulers and can be set through cgroup interfaces.
For most desktops it is transparent.

### 1.8 PSI (`CONFIG_PSI=y`)

Pressure Stall Information exposes how much time tasks spend stalled on CPU,
memory, or I/O:

```sh
cat /proc/pressure/cpu
cat /proc/pressure/memory
cat /proc/pressure/io
```

Desktop environments, `systemd-oomd`, and monitoring tools use PSI to make
better decisions than raw utilisation alone.

### 1.9 Autogroup (`CONFIG_SCHED_AUTOGROUP=y`)

Autogroup gives each session/terminal a fair share of CPU, preventing one
CPU-hungry process from starving the desktop. Toggle:

```sh
cat /proc/sys/kernel/sched_autogroup_enabled
```

## 2. Memory management

### 2.1 Multi-generational LRU (`CONFIG_LRU_GEN=y`)

MGLRU tracks page generations and makes better reclaim decisions, especially
under memory pressure and for workloads with large inactive working sets.

```sh
cat /sys/kernel/mm/lru_gen/enabled    # 0x0007 = all features
cat /sys/kernel/mm/lru_gen/min_ttl_ms
```

It is enabled by default (`CONFIG_LRU_GEN_ENABLED=y`).

### 2.2 zswap (`CONFIG_ZSWAP=y`, default on, zstd)

zswap is a compressed cache in front of a swap device. Pages being swapped out
are compressed and stored in RAM; only when the pool is full or memory is tight
are they written to disk. This reduces swap I/O and improves responsiveness
under moderate memory pressure.

```sh
cat /sys/module/zswap/parameters/enabled
cat /sys/module/zswap/parameters/compressor
cat /sys/module/zswap/parameters/max_pool_percent
```

Boot parameters:

```text
zswap.enabled=1 zswap.compressor=zstd zswap.max_pool_percent=25
```

`CONFIG_ZSWAP_SHRINKER_DEFAULT_ON=y` lets the shrinker reclaim zswap entries
under pressure.

### 2.3 zram (`CONFIG_ZRAM=m`)

zram creates a compressed block device in RAM, commonly used as swap or `/tmp`.
It trades CPU for memory and avoids disk I/O entirely.

```sh
sudo modprobe zram
echo zstd | sudo tee /sys/block/zram0/comp_algorithm
echo 4G    | sudo tee /sys/block/zram0/disksize
sudo mkswap /dev/zram0
sudo swapon -p 100 /dev/zram0
```

`CONFIG_ZRAM_WRITEBACK=y` allows idle pages to be written back to a backing
device. `CONFIG_ZRAM_MEMORY_TRACKING=y` provides per-entry statistics.

**zswap or zram?** They solve overlapping problems:

| | zswap | zram |
|---|---|---|
| What it is | Cache before a swap device | Compressed block device used as swap |
| Needs a swap device | Yes | No (it *is* the swap) |
| Best for | Systems with disk swap | RAM-only or flash-friendly setups |
| Running both | Possible but usually redundant | Possible but usually redundant |

Pick one primary strategy. Running zswap on top of zram adds a layer without
much benefit.

### 2.4 DAMON (`CONFIG_DAMON=y`)

DAMON (Data Access Monitor) samples access patterns and can drive proactive
reclaim, LRU sorting, and statistics. Enabled components:

- `DAMON_VADDR` — virtual address monitoring
- `DAMON_PADDR` — physical address monitoring
- `DAMON_SYSFS` — configure through sysfs
- `DAMON_RECLAIM` — proactive reclaim
- `DAMON_LRU_SORT` — sort LRU lists by access temperature
- `DAMON_STAT` — statistics

```sh
ls /sys/kernel/mm/damon/admin
damo --help        # userspace tool, if installed
```

DAMON is powerful but not automatic; it needs a policy to be useful.

### 2.5 KSM (`CONFIG_KSM=y`)

Kernel Same-page Merging deduplicates identical pages. It saves memory for
virtual machines, containers, and many similar processes, at the cost of CPU.

```sh
echo 1 | sudo tee /sys/kernel/mm/ksm/run
cat /sys/kernel/mm/ksm/pages_shared
```

### 2.6 Transparent Huge Pages (`CONFIG_TRANSPARENT_HUGEPAGE_ALWAYS=y`)

THP backs suitable memory with 2 MB pages, reducing TLB pressure. The config uses
the aggressive `always` policy.

```sh
cat /sys/kernel/mm/transparent_hugepage/enabled
```

For databases and latency-sensitive services, `madvise` is often a better
policy. Change at runtime:

```sh
echo madvise | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
```

### 2.7 Memory cgroup and cgroup BPF

`CONFIG_MEMCG=y` provides `memory.max`, `memory.high`, `memory.low`, swap
accounting, and PSI integration. `CONFIG_CGROUP_BPF=y` allows BPF programs to
enforce policies on cgroup events.

### 2.8 Zero-on-alloc and KFENCE

`CONFIG_INIT_ON_ALLOC_DEFAULT_ON=y` zeroes pages on allocation, removing a class
of information leaks and use-of-uninitialised-memory bugs. It costs some
performance.

`CONFIG_KFENCE=y` adds a low-overhead sampling detector for out-of-bounds and
use-after-free errors. It is not a full sanitizer; KASAN is not enabled.

## 3. Block layer and I/O

### 3.1 I/O schedulers

| Scheduler | Best for | Notes |
|---|---|---|
| `bfq` | Interactive desktop, slow disks | Fair queuing, low latency |
| `mq-deadline` | General-purpose, predictable latency | Simple, robust |
| `kyber` | Fast devices, low overhead | Tunable latency targets |
| `none` | Fast NVMe with a modern FS | No scheduling overhead |

All three schedulers are built in. Select per device:

```sh
cat /sys/block/nvme0n1/queue/scheduler
echo bfq | sudo tee /sys/block/nvme0n1/queue/scheduler
```

Persist with a udev rule:

```udev
# /etc/udev/rules.d/60-iosched.rules
ACTION=="add|change", KERNEL=="nvme0n1", ATTR{queue/scheduler}="bfq"
```

### 3.2 blk-cgroup

Enabled controllers:

- `io.weight` / `io.latency` / `io.max` / `io.cost`
- `iocost` model
- `ioprio` priorities
- writeback throttling (`CONFIG_BLK_WBT=y`)

These let systemd and container runtimes enforce I/O policies.

### 3.3 Device mapper, dm-crypt, MD RAID

- `dm-crypt` provides LUKS.
- MD provides RAID 0/1/10/5/6.
- Loop devices are created on demand (`LOOP_MIN_COUNT=0`).

## 4. Filesystems

### 4.1 Built in

- **ext4** — the safe default for root and `/home`.
- **Btrfs** — snapshots, subvolumes, checksums, compression.
- **tmpfs** — `/tmp`, `/run`, `/dev/shm`.
- **FUSE** — userspace filesystems (AppImages, `sshfs`, `ntfs-3g`).

### 4.2 Modules

- **XFS** — large files, high-throughput storage.
- **F2FS** — flash-friendly.
- **VFAT/FAT/exFAT** — EFI ESP and removable media.
- **NTFS3** — Windows volumes.
- **OverlayFS** — containers and live systems.
- **SquashFS/EROFS** — read-only compressed images.
- **NFS/CIFS/9p** — network and virtual filesystems.
- **zonefs** — zoned block devices.

### 4.3 Encryption

- **dm-crypt/LUKS** — full-disk encryption (`DM_CRYPT=m`).
- **fscrypt** — per-directory encryption (`FS_ENCRYPTION=y`).
- **casefolding** — Unicode-aware case-insensitive names (`UNICODE=y`).

## 5. Networking

### 5.1 Core

IPv4/IPv6, netfilter, nf_tables, sockets, and traffic control are available.
`nf_tables` is a module; load it before using `nft`.

### 5.2 VPN and tunnelling

- WireGuard (`wireguard.ko`)
- TUN/TAP
- veth, bridge, VLAN, MACVLAN, bonding

### 5.3 Congestion control

Default is `cubic`. BBR is available as a module:

```sh
sudo modprobe tcp_bbr
sudo sysctl net.ipv4.tcp_congestion_control=bbr
```

### 5.4 BPF and XDP

BPF syscall, JIT, and XDP sockets are enabled. Unprivileged BPF is off by default
(`CONFIG_BPF_UNPRIV_DEFAULT_OFF=y`), which is a security hardening choice.

## 6. Virtualisation

### 6.1 Host

KVM is modular:

```sh
sudo modprobe kvm_intel   # or kvm_amd
lsmod | grep kvm
```

`CONFIG_KVM=m` keeps the core kernel smaller; the module loads on demand.

### 6.2 Guest

Built in: Xen PV, Hyper-V, virtio-blk, virtio-pci, virtio-fs.
Modules: virtio-net, virtio-gpu, vboxguest, vmw_pvscsi, vmwgfx.

This makes the kernel usable as a cloud image and as a VirtualBox/VMware guest.

## 7. Graphics, audio, input

### 7.1 Graphics

- DRM core built in; framebuffer console and EFI framebuffer work without KMS.
- AMDGPU, i915, Xe, Nouveau, virtio-gpu, QXL, Bochs, vmwgfx as modules.
- `DRM_FBDEV_EMULATION=y` provides a framebuffer console on top of KMS.

For NVIDIA proprietary drivers, use `nvidia-dkms` against `visnux-headers`.

### 7.2 Audio

ALSA is modular. HDA Intel, USB audio, and SOF are available. If audio is
missing, check that the codec module and firmware are installed.

### 7.3 Input

HID, USB, and input core are built in, so basic keyboards and mice work before
modules load. Specialised devices may need additional modules.

## 8. Security

See [security.md](security.md) for the full treatment. Summary:

- KASLR, W^X, stack protector, `FORTIFY_SOURCE`, hardened usercopy.
- Zero-on-alloc, freelist randomisation, page-allocator shuffling.
- `dmesg_restrict=1`, `IO_STRICT_DEVMEM=y`.
- CPU mitigations on; CET IBT and shadow stacks on.
- Landlock, Yama, Lockdown active by default.
- SELinux/AppArmor/Smack/TOMOYO compiled but not in the default LSM order.
- Module signing available, all in-tree modules signed, enforcement off.

## 9. Debugging and tracing

| Tool | Use |
|---|---|
| `perf` | Sampling, counters, tracepoints |
| `bpftrace` / `bpftool` | BPF observability |
| `ftrace` | Function/graph tracing |
| `kprobes` / `uprobes` | Dynamic probes |
| `drgn` | Scriptable kernel debugging |
| `crash` / `kdump` | Post-crash analysis |
| `pahole` | BTF/type inspection |
| `faddr2line` | Symbolise addresses |

```sh
sudo perf top
sudo bpftool prog list
sudo cat /sys/kernel/debug/tracing/available_tracers
```

`CONFIG_DEBUG_INFO_BTF=y` enables CO-RE BPF programs, which is why tools like
`bpftrace` work without kernel headers at runtime.

## 10. Power management

- `CONFIG_SUSPEND=y`, `CONFIG_HIBERNATION=y`
- `CONFIG_CPU_IDLE=y`, `CONFIG_INTEL_IDLE=y`
- `intel_pstate` and `amd_pstate` (active mode) built in
- `schedutil` default governor
- Thermal framework and Intel powerclamp/RAPL modules

```sh
cat /sys/power/state
sudo systemctl suspend
sudo systemctl hibernate
```

## 11. What is *not* enabled

To keep the profile comprehensible, these are absent or off:

- Rust support (`CONFIG_RUST` unset)
- KASAN, UBSAN, KCSAN, lockdep (debug-only; `DEBUG_KERNEL` is on, but heavy
  sanitizers are not)
- bcachefs
- Mainframe, embedded, and most legacy buses
- Many exotic filesystems and network protocols

If you need one, enable it explicitly and rebuild. See
[configuration.md](configuration.md#15-changing-the-configuration-safely).
