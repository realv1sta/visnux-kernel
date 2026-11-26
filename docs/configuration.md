# Kernel configuration

The file [`.config`](../.config) is the product of this repository. This document
explains the policy behind it, lists the important symbols by subsystem, and
describes how to change it without breaking the build or silently losing options.

All values below are taken from the committed `.config` of `visnux-kernel`
(Linux 7.2.15). Counts and defaults can change when the config changes; verify
with `grep`.

## 1. Configuration policy

The config follows five rules, in priority order.

### Rule 1 — Boot-critical code is built in

Anything needed to reach a root filesystem or a usable console is `=y` where the
kernel allows it:

- storage: `SATA_AHCI`, `BLK_DEV_SD`, `VIRTIO_BLK`, `SCSI`
- filesystems: `EXT4_FS`, `BTRFS_FS`, `TMPFS`
- console: `FB`, `FB_EFI`, `FRAMEBUFFER_CONSOLE`, `DRM` core
- platform: `PCI`, `PCI_MSI`, `ACPI`, `EFI`, `EFI_STUB`
- input: `INPUT`, `HID`, `USB` core

This avoids “initramfs pulled the wrong module” boot failures.

### Rule 2 — Desktop responsiveness is not optional

Full preemption, `HZ=1000`, high-resolution timers, `schedutil`, PSI,
autogroup, and `sched_ext` are all enabled. Latency-sensitive desktop workloads
are a first-class target.

### Rule 3 — Memory pressure must be cheap

zswap (zstd) on by default, zram available, MGLRU enabled, DAMON available, KSM
available. The kernel should degrade gracefully when RAM is tight.

### Rule 4 — Development and observability stay usable

DWARF5 debug info, BTF, `KALLSYMS_ALL`, ftrace, kprobes, uprobes, perf, BPF,
`DEBUG_FS`, and `/proc/config.gz` are on. This is a kernel you can profile.

### Rule 5 — Unneeded hardware is disabled, not modularised

A desktop does not need mainframe, embedded, or obsolete bus support. Those
symbols are off. This keeps the build and the attack surface smaller than a
“compile everything” distribution config.

### The unavoidable trade-offs

| Choice | Benefit | Cost |
|---|---|---|
| 6,314 modules | Broad hardware coverage | Large module tree, longer install |
| Debug info + BTF + `KALLSYMS_ALL` | Profiling, tracing, `drgn`, BPF CO-RE | Large `vmlinux`, longer build |
| `KFENCE=y` | Low-overhead memory-error detection | Small runtime overhead |
| `PAGE_POISONING=y` | Catch use-after-free | Memory and speed overhead |
| Full preemption | Low latency | Slightly lower throughput |
| Module signing without enforcement | DKMS keeps working | Unsigned modules can load (taint) |

These are deliberate. Do not “fix” one without understanding the others.

## 2. Identity and platform

| Symbol | Value | Meaning |
|---|---|---|
| `CONFIG_LOCALVERSION` | `"visnux"` | Release suffix |
| `CONFIG_LOCALVERSION_AUTO` | `y` | Append `-g<sha>` inside git |
| `CONFIG_DEFAULT_HOSTNAME` | `"artixlinux"` | Fallback hostname |
| `CONFIG_X86_64` | `y` | 64-bit x86 |
| `CONFIG_SMP` | `y` | Symmetric multiprocessing |
| `CONFIG_NR_CPUS` | `8192` | Maximum CPUs |
| `CONFIG_IA32_EMULATION` | `y` | Run 32-bit binaries |
| `CONFIG_X86_X2APIC` | `y` | Extended APIC |
| `CONFIG_X86_MCE` | `y` | Machine check handling |
| `CONFIG_X86_UMIP` | `y` | User-mode instruction prevention |
| `CONFIG_X86_KERNEL_IBT` | `y` | CET indirect branch tracking |
| `CONFIG_X86_USER_SHADOW_STACK` | `y` | CET shadow stack for userspace |
| `CONFIG_X86_INTEL_MEMORY_PROTECTION_KEYS` | `y` | PKU/pkeys |
| `CONFIG_X86_INTEL_TSX_MODE_AUTO` | `y` | TSX left to firmware |
| `CONFIG_EFI` / `CONFIG_EFI_STUB` | `y` | EFI boot support |
| `CONFIG_ACPI` | `y` | ACPI |
| `CONFIG_PCI` / `CONFIG_PCI_MSI` | `y` | PCI and MSI |
| `CONFIG_PM` / `CONFIG_SUSPEND` / `CONFIG_HIBERNATION` | `y` | Sleep and hibernate |
| `CONFIG_CPU_IDLE` / `CONFIG_INTEL_IDLE` | `y` | Idle states |

### Release string

```sh
make kernelrelease
# tarball build:        7.2.15visnux
# inside this git tree: 7.2.15visnux-g3cd079313
```

`CONFIG_LOCALVERSION_AUTO=y` appends `-g<short-sha>` when `scripts/setlocalversion`
detects a git checkout without a `.scmversion` file. The packaged lean profile
uses `CONFIG_LOCALVERSION="-visnux"` and no auto suffix, producing
`7.2.15-visnux`.

> If you need a stable release string for packaging, disable
> `CONFIG_LOCALVERSION_AUTO` or set `LOCALVERSION` explicitly on the `make`
> command line.

## 3. Scheduler and latency

| Symbol | Value | Notes |
|---|---|---|
| `CONFIG_PREEMPT` | `y` | Full preemption |
| `CONFIG_PREEMPT_DYNAMIC` | `y` | Selectable at boot |
| `CONFIG_PREEMPTION` | `y` | Preemptible kernel |
| `CONFIG_PREEMPT_RCU` | `y` | Preemptible RCU |
| `CONFIG_HZ` | `1000` | 1 ms tick |
| `CONFIG_NO_HZ` | `y` | Tickless |
| `CONFIG_NO_HZ_FULL` | `y` | Full dynticks, opt-in via `nohz_full=` |
| `CONFIG_HIGH_RES_TIMERS` | `y` | Nanosecond timers |
| `CONFIG_SCHED_CORE` | `y` | SMT core scheduling |
| `CONFIG_SCHED_CLASS_EXT` | `y` | `sched_ext` (BPF schedulers) |
| `CONFIG_SCHED_AUTOGROUP` | `y` | Per-session fairness |
| `CONFIG_UCLAMP_TASK` | `y` | Utilisation clamping |
| `CONFIG_UCLAMP_BUCKETS_COUNT` | `5` | Clamp buckets |
| `CONFIG_PSI` | `y` | Pressure Stall Information |
| `CONFIG_NUMA_BALANCING` | `y` | Automatic NUMA balancing |
| `CONFIG_SCHED_SMT` | `y` | SMT topology |
| `CONFIG_SCHED_MC` | `y` | Multi-core topology |
| `CONFIG_SCHED_CLUSTER` | `y` | Cluster topology |
| `CONFIG_SCHED_INFO` | `y` | Scheduler run-time info |
| `CONFIG_SCHEDSTATS` | `y` | Scheduler statistics |

### Dynamic preemption

With `CONFIG_PREEMPT_DYNAMIC=y`, the boot parameter selects the model:

```text
preempt=full         # default here
preempt=voluntary
preempt=none
```

Check the active model:

```sh
cat /sys/kernel/debug/sched/preempt
```

### Full dynticks is opt-in

`CONFIG_NO_HZ_FULL=y` only takes effect when CPUs are listed:

```text
nohz_full=2-7 rcu_nocbs=2-7
```

CPU 0 must not be in `nohz_full`. Full dynticks is intended for dedicated,
isolated workloads; enabling it on a normal desktop can reduce throughput.

### sched_ext

`CONFIG_SCHED_CLASS_EXT=y` allows BPF schedulers to be loaded at runtime through
`/sys/kernel/sched_ext`. The default scheduler remains EEVDF until a BPF
scheduler is attached.

## 4. Memory management

| Symbol | Value | Notes |
|---|---|---|
| `CONFIG_LRU_GEN` | `y` | Multi-generational LRU |
| `CONFIG_LRU_GEN_ENABLED` | `y` | Enabled by default |
| `CONFIG_SWAP` | `y` | Swap support |
| `CONFIG_ZSWAP` | `y` | Compressed swap cache |
| `CONFIG_ZSWAP_DEFAULT_ON` | `y` | On without a boot parameter |
| `CONFIG_ZSWAP_COMPRESSOR_DEFAULT` | `"zstd"` | Default compressor |
| `CONFIG_ZSWAP_SHRINKER_DEFAULT_ON` | `y` | Shrinker active |
| `CONFIG_ZRAM` | `m` | Compressed block device |
| `CONFIG_ZRAM_DEF_COMP` | `"zstd"` | Default zram compressor |
| `CONFIG_ZRAM_WRITEBACK` | `y` | zram writeback support |
| `CONFIG_ZRAM_MEMORY_TRACKING` | `y` | Per-entry statistics |
| `CONFIG_DAMON` | `y` | Data Access Monitor |
| `CONFIG_DAMON_VADDR` / `PADDR` | `y` | Virtual/physical address monitoring |
| `CONFIG_DAMON_SYSFS` | `y` | Configure via sysfs |
| `CONFIG_DAMON_RECLAIM` | `y` | Proactive reclaim |
| `CONFIG_DAMON_LRU_SORT` | `y` | LRU sorting |
| `CONFIG_DAMON_STAT` | `y` | Statistics |
| `CONFIG_KSM` | `y` | Same-page merging |
| `CONFIG_TRANSPARENT_HUGEPAGE` | `y` | THP |
| `CONFIG_TRANSPARENT_HUGEPAGE_ALWAYS` | `y` | THP `always` policy |
| `CONFIG_MEMCG` | `y` | Memory cgroup |
| `CONFIG_CGROUP_BPF` | `y` | BPF on cgroups |
| `CONFIG_INIT_ON_ALLOC_DEFAULT_ON` | `y` | Zero allocations |
| `CONFIG_DEFAULT_MMAP_MIN_ADDR` | `65536` | Minimum mmap address |
| `CONFIG_PAGE_POISONING` | `y` | Poison freed pages |
| `CONFIG_KFENCE` | `y` | Sampling memory-error detector |

### Notes

- **zswap vs zram.** zswap is a cache in front of a real swap device; zram is a
  compressed block device used *as* swap. This config enables zswap by default
  and makes zram available as a module. Running both is possible but usually
  redundant; pick one primary strategy.
- **MGLRU** is enabled. It changes reclaim behaviour and is generally better on
  desktop workloads, but it is not a free lunch under every workload.
- **THP is `always`.** This is the aggressive policy. For some databases and
  latency-sensitive services, `madvise` is preferable.
- **KFENCE** is a sampling detector with low overhead; it is not KASAN. It is
  safe to leave on, but a production build can turn it off for the last few
  percent.

## 5. Block layer and I/O

| Symbol | Value | Notes |
|---|---|---|
| `CONFIG_BLK_CGROUP` | `y` | Block cgroup support |
| `CONFIG_BLK_CGROUP_IOPRIO` | `y` | I/O priority |
| `CONFIG_BLK_CGROUP_IOLATENCY` | `y` | I/O latency targets |
| `CONFIG_BLK_CGROUP_IOCOST` | `y` | IOCOST model |
| `CONFIG_BLK_DEV_THROTTLING` | `y` | Throttling |
| `CONFIG_BLK_WBT` | `y` | Writeback throttling |
| `CONFIG_MQ_IOSCHED_DEADLINE` | `y` | mq-deadline |
| `CONFIG_MQ_IOSCHED_KYBER` | `y` | Kyber |
| `CONFIG_IOSCHED_BFQ` | `y` | BFQ |
| `CONFIG_BFQ_GROUP_IOSCHED` | `y` | BFQ cgroup groups |
| `CONFIG_BLK_DEV_LOOP` | `m` | Loop devices |
| `CONFIG_BLK_DEV_LOOP_MIN_COUNT` | `0` | Create on demand |
| `CONFIG_BLK_DEV_DM` | `m` | Device mapper |
| `CONFIG_DM_CRYPT` | `m` | dm-crypt / LUKS |
| `CONFIG_MD` | `y` | MD core |
| `CONFIG_MD_RAID0/1/10/456` | `m` | RAID personalities |

The default scheduler is chosen by the block layer per device. Set it
explicitly if you care:

```sh
echo bfq | sudo tee /sys/block/nvme0n1/queue/scheduler
```

BFQ is a good default for interactive desktop I/O; `none` is often best for
fast NVMe with a modern filesystem. Measure before committing to a choice.

## 6. Filesystems

| Symbol | Value | Use |
|---|---|---|
| `CONFIG_EXT4_FS` | `y` | Root/general |
| `CONFIG_BTRFS_FS` | `y` | Snapshots, compression |
| `CONFIG_XFS_FS` | `m` | Large storage |
| `CONFIG_F2FS_FS` | `m` | Flash |
| `CONFIG_TMPFS` | `y` | RAM filesystems |
| `CONFIG_FUSE_FS` | `y` | Userspace filesystems |
| `CONFIG_OVERLAY_FS` | `m` | Containers |
| `CONFIG_SQUASHFS` | `m` | Live images |
| `CONFIG_EROFS_FS` | `m` | Read-only images |
| `CONFIG_ISO9660_FS` | `m` | Optical |
| `CONFIG_FAT_FS` / `VFAT_FS` | `m` | EFI ESP |
| `CONFIG_EXFAT_FS` | `m` | exFAT media |
| `CONFIG_NTFS3_FS` | `m` | Windows volumes |
| `CONFIG_NFS_FS` / `NFSD` | `m` | NFS client/server |
| `CONFIG_CIFS` | `m` | SMB/CIFS |
| `CONFIG_9P_FS` | `m` | 9p (virtio) |
| `CONFIG_ZONEFS_FS` | `m` | Zoned devices |
| `CONFIG_FS_ENCRYPTION` | `y` | fscrypt |
| `CONFIG_UNICODE` | `y` | Casefolding |
| `CONFIG_QUOTA` | `y` | Disk quotas |

`bcachefs` is not enabled in this configuration.

## 7. Storage drivers

| Symbol | Value |
|---|---|
| `CONFIG_SATA_AHCI` | `y` |
| `CONFIG_ATA` | `y` |
| `CONFIG_SCSI` | `y` |
| `CONFIG_BLK_DEV_SD` | `y` |
| `CONFIG_BLK_DEV_NVME` | `m` |
| `CONFIG_NVME_CORE` | `m` |
| `CONFIG_VIRTIO_BLK` | `y` |
| `CONFIG_VIRTIO_PCI` | `y` |
| `CONFIG_VIRTIO_FS` | `y` |
| `CONFIG_VIRTIO_NET` | `m` |
| `CONFIG_VMWARE_PVSCSI` | `m` |

NVMe is a module. The packaged initramfs preset disables `autodetect`, so it is
included; a hand-built minimal initramfs must add it explicitly.

## 8. Networking

| Symbol | Value |
|---|---|
| `CONFIG_NET` | `y` |
| `CONFIG_INET` | `y` |
| `CONFIG_IPV6` | `y` |
| `CONFIG_NETFILTER` | `y` |
| `CONFIG_NF_TABLES` | `m` |
| `CONFIG_BRIDGE` | `m` |
| `CONFIG_VLAN_8021Q` | `m` |
| `CONFIG_BONDING` | `m` |
| `CONFIG_MACVLAN` | `m` |
| `CONFIG_VETH` | `m` |
| `CONFIG_TUN` / `TAP` | `m` |
| `CONFIG_WIREGUARD` | `m` |
| `CONFIG_NET_SCHED` | `y` |
| `CONFIG_NET_CLS_BPF` | `m` |
| `CONFIG_NET_ACT_BPF` | `m` |
| `CONFIG_NET_SCH_FQ_CODEL` | `y` |
| `CONFIG_NET_SCH_FQ` | `m` |
| `CONFIG_DEFAULT_TCP_CONG` | `"cubic"` |
| `CONFIG_TCP_CONG_BBR` | `m` |
| `CONFIG_BPF_SYSCALL` | `y` |
| `CONFIG_BPF_JIT` | `y` |
| `CONFIG_BPF_JIT_ALWAYS_ON` | `y` |
| `CONFIG_BPF_UNPRIV_DEFAULT_OFF` | `y` |
| `CONFIG_XDP_SOCKETS` | `y` |

## 9. Virtualisation

| Symbol | Value |
|---|---|
| `CONFIG_VIRTUALIZATION` | `y` |
| `CONFIG_KVM` | `m` |
| `CONFIG_KVM_INTEL` | `m` |
| `CONFIG_KVM_AMD` | `m` |
| `CONFIG_XEN` | `y` |
| `CONFIG_XEN_PV` | `y` |
| `CONFIG_HYPERV` | `y` |
| `CONFIG_HYPERVISOR_GUEST` | `y` |
| `CONFIG_PARAVIRT` | `y` |
| `CONFIG_VBOXGUEST` | `m` |
| `CONFIG_VBOXSF_FS` | `m` |
| `CONFIG_VMWARE_PVSCSI` | `m` |

KVM is modular: load `kvm_intel` or `kvm_amd` (which pulls in `kvm`) before
starting a VM. Xen PV and Hyper-V guest support are built in for cloud and VM
images.

## 10. Graphics, audio, input

| Symbol | Value |
|---|---|
| `CONFIG_DRM` | `y` |
| `CONFIG_DRM_FBDEV_EMULATION` | `y` |
| `CONFIG_DRM_AMDGPU` | `m` |
| `CONFIG_DRM_I915` | `m` |
| `CONFIG_DRM_XE` | `m` |
| `CONFIG_DRM_NOUVEAU` | `m` |
| `CONFIG_DRM_VIRTIO_GPU` | `m` |
| `CONFIG_DRM_QXL` | `m` |
| `CONFIG_DRM_BOCHS` | `m` |
| `CONFIG_DRM_VMWGFX` | `m` |
| `CONFIG_FB` / `FB_EFI` | `y` |
| `CONFIG_FRAMEBUFFER_CONSOLE` | `y` |
| `CONFIG_SOUND` / `SND` | `m` |
| `CONFIG_SND_HDA_INTEL` | `m` |
| `CONFIG_SND_USB_AUDIO` | `m` |
| `CONFIG_SND_SOC_SOF_TOPLEVEL` | `y` |
| `CONFIG_INPUT` | `y` |
| `CONFIG_HID` | `y` |
| `CONFIG_USB` | `y` |

The framebuffer console and EFI framebuffer are built in, so a blank screen is
usually a KMS/driver issue rather than a missing console driver.

## 11. Security and hardening

| Symbol | Value | Effect |
|---|---|---|
| `CONFIG_SECURITY` | `y` | LSM framework |
| `CONFIG_LSM` | `"landlock,lockdown,yama,integrity,bpf"` | Default active order |
| `CONFIG_SECURITY_SELINUX` | `y` | SELinux compiled |
| `CONFIG_SECURITY_APPARMOR` | `y` | AppArmor compiled |
| `CONFIG_SECURITY_SMACK` | `y` | Smack compiled |
| `CONFIG_SECURITY_TOMOYO` | `y` | TOMOYO compiled |
| `CONFIG_SECURITY_LANDLOCK` | `y` | Landlock active |
| `CONFIG_SECURITY_YAMA` | `y` | ptrace scope |
| `CONFIG_SECURITY_LOCKDOWN_LSM` | `y` | Lockdown active |
| `CONFIG_LOCK_DOWN_KERNEL_FORCE_NONE` | `y` | Default `none` |
| `CONFIG_SECURITY_LOADPIN` | `y` | LoadPin |
| `CONFIG_SECURITY_SAFESETID` | `y` | SafeSetID |
| `CONFIG_SECURITY_IPE` | `y` | Integrity Policy Enforcement |
| `CONFIG_INTEGRITY` | `y` | Integrity subsystem |
| `CONFIG_INTEGRITY_SIGNATURE` | `y` | Signature verification |
| `CONFIG_RANDOMIZE_BASE` | `y` | KASLR |
| `CONFIG_RANDOMIZE_MEMORY` | `y` | Memory layout randomisation |
| `CONFIG_RANDOMIZE_KSTACK_OFFSET_DEFAULT` | `y` | Kernel stack offset randomisation |
| `CONFIG_STRICT_KERNEL_RWX` | `y` | W^X for kernel text |
| `CONFIG_STRICT_MODULE_RWX` | `y` | W^X for modules |
| `CONFIG_STACKPROTECTOR_STRONG` | `y` | Strong stack canary |
| `CONFIG_VMAP_STACK` | `y` | Guarded kernel stacks |
| `CONFIG_FORTIFY_SOURCE` | `y` | Compile-time bounds checks |
| `CONFIG_HARDENED_USERCOPY` | `y` | Copy bounds checks |
| `CONFIG_INIT_ON_ALLOC_DEFAULT_ON` | `y` | Zero allocations |
| `CONFIG_SLAB_FREELIST_RANDOM` | `y` | Random freelist |
| `CONFIG_SLAB_FREELIST_HARDENED` | `y` | Freelist integrity |
| `CONFIG_SHUFFLE_PAGE_ALLOCATOR` | `y` | Random page allocator |
| `CONFIG_CPU_MITIGATIONS` | `y` | CPU mitigations |
| `CONFIG_MITIGATION_RETPOLINE` | `y` | Retpoline |
| `CONFIG_MITIGATION_SPECTRE_V2` | `y` | Spectre v2 |
| `CONFIG_SECURITY_DMESG_RESTRICT` | `y` | Restrict `dmesg` |
| `CONFIG_IO_STRICT_DEVMEM` | `y` | Restrict `/dev/mem` |
| `CONFIG_BPF_UNPRIV_DEFAULT_OFF` | `y` | No unprivileged BPF |
| `CONFIG_MODULE_SIG` | `y` | Module signatures |
| `CONFIG_MODULE_SIG_ALL` | `y` | Sign all modules |
| `CONFIG_MODULE_SIG_HASH` | `"sha512"` | Signature hash |
| `CONFIG_MODULE_SIG_FORCE` | not set | Enforcement off |

### LSM selection gotcha

`CONFIG_LSM` lists only the LSMs that are **active by default**. SELinux,
AppArmor, Smack, and TOMOYO are compiled but not in that list. To activate one,
add it to `lsm=` on the kernel command line **and** configure userspace:

```text
lsm=landlock,lockdown,yama,integrity,bpf,apparmor
```

Order matters: the first LSM that claims a hook wins for that hook. See
[security.md](security.md).

## 12. Modules and signing

| Symbol | Value |
|---|---|
| `CONFIG_MODULES` | `y` |
| `CONFIG_MODULE_UNLOAD` | `y` |
| `CONFIG_MODULE_SIG` | `y` |
| `CONFIG_MODULE_SIG_ALL` | `y` |
| `CONFIG_MODULE_SIG_HASH` | `"sha512"` |
| `CONFIG_MODULE_SIG_KEY` | `"certs/signing_key.pem"` |
| `CONFIG_MODULE_COMPRESS` | `y` |
| `CONFIG_MODULE_COMPRESS_ZSTD` | `y` |
| `CONFIG_MODULE_SIG_FORCE` | not set |

Consequences are covered in [security.md](security.md#5-module-signing) and
[installation.md](installation.md#6-dkms).

## 13. Debugging and observability

| Symbol | Value |
|---|---|
| `CONFIG_DEBUG_KERNEL` | `y` |
| `CONFIG_DEBUG_INFO` | `y` |
| `CONFIG_DEBUG_INFO_DWARF5` | `y` |
| `CONFIG_DEBUG_INFO_BTF` | `y` |
| `CONFIG_DEBUG_INFO_BTF_MODULES` | `y` |
| `CONFIG_KALLSYMS` / `KALLSYMS_ALL` | `y` |
| `CONFIG_GDB_SCRIPTS` | `y` |
| `CONFIG_DEBUG_FS` | `y` |
| `CONFIG_DYNAMIC_DEBUG` | `y` |
| `CONFIG_FTRACE` | `y` |
| `CONFIG_FUNCTION_TRACER` | `y` |
| `CONFIG_FUNCTION_GRAPH_TRACER` | `y` |
| `CONFIG_DYNAMIC_FTRACE` | `y` |
| `CONFIG_KPROBES` | `y` |
| `CONFIG_UPROBES` | `y` |
| `CONFIG_KPROBE_EVENTS` | `y` |
| `CONFIG_UPROBE_EVENTS` | `y` |
| `CONFIG_PERF_EVENTS` | `y` |
| `CONFIG_BPF_EVENTS` | `y` |
| `CONFIG_IKCONFIG` / `IKCONFIG_PROC` | `y` |
| `CONFIG_KEXEC` / `KEXEC_FILE` / `CRASH_DUMP` | `y` |
| `CONFIG_MAGIC_SYSRQ` | `y` |
| `CONFIG_MAGIC_SYSRQ_DEFAULT_ENABLE` | `0x0` |
| `CONFIG_PAGE_POISONING` | `y` |
| `CONFIG_KFENCE` | `y` |
| `CONFIG_LOCKUP_DETECTOR` | `y` |
| `CONFIG_SOFTLOCKUP_DETECTOR` | `y` |
| `CONFIG_HARDLOCKUP_DETECTOR` | `y` |
| `CONFIG_UNWINDER_ORC` | `y` |
| `CONFIG_PANIC_TIMEOUT` | `0` |

`CONFIG_MAGIC_SYSRQ_DEFAULT_ENABLE=0x0` means the SysRq **key** is disabled by
default; `/proc/sysrq-trigger` still works. Enable the key with
`sysctl kernel.sysrq=1`.

`CONFIG_PANIC_TIMEOUT=0` means no automatic reboot on panic. Set
`panic=<seconds>` on the command line if you want one.

## 14. Toolchain and build

| Symbol | Value |
|---|---|
| `CONFIG_CC_VERSION_TEXT` | `"gcc (GCC) 16.2.1 20260810"` |
| `CONFIG_CC_IS_GCC` | `y` |
| `CONFIG_GCC_VERSION` | `160201` |
| `CONFIG_AS_VERSION` | `24700` |
| `CONFIG_LD_VERSION` | `24700` |
| `CONFIG_LD_IS_BFD` | `y` |
| `CONFIG_RUSTC_VERSION` | `110000` |
| `CONFIG_RUSTC_LLVM_VERSION` | `230105` |
| `CONFIG_CC_OPTIMIZE_FOR_PERFORMANCE` | `y` |
| `CONFIG_LTO_NONE` | `y` |
| `CONFIG_UNWINDER_ORC` | `y` |

Rust support (`CONFIG_RUST`) is **not** enabled in this configuration, even
though a Rust toolchain version is recorded.

## 15. Changing the configuration safely

### 15.1 Use the tools, not a text editor

```sh
# Enable a boolean
./scripts/config --enable CONFIG_FOO

# Build as a module
./scripts/config --module CONFIG_BAR

# Disable
./scripts/config --disable CONFIG_BAZ

# Set a value
./scripts/config --set-val CONFIG_NR_CPUS 256
./scripts/config --set-str CONFIG_LOCALVERSION "-visnux"
```

Then resolve dependencies:

```sh
make olddefconfig
```

`olddefconfig` keeps your explicit choices where possible and fills new symbols
with their defaults. It does **not** warn you when a symbol was dropped because
a dependency is unmet. Verify:

```sh
grep CONFIG_FOO .config
```

### 15.2 Understand dependencies

Kconfig expressions like `depends on`, `select`, and `imply` mean:

- If a dependency is off, your symbol disappears silently.
- `select` can force other symbols on, sometimes unexpectedly.
- Architecture and toolchain capabilities (`CC_HAS_*`, `HAVE_*`) gate many
  options.

When a symbol will not stick, inspect its `Kconfig` entry:

```sh
grep -rn "config FOO" --include=Kconfig .
```

and read the `depends on` line.

### 15.3 Keep the diff reviewable

```sh
git diff -- .config
```

A config change should be a small, intentional diff. A diff with thousands of
unrelated lines usually means you ran `olddefconfig` against a different kernel
version or a different front-end.

### 15.4 Rebuild and boot

A configuration change is not done until the kernel builds **and boots** on the
target hardware. At minimum test:

- normal boot to a desktop;
- root filesystem mount;
- networking;
- suspend/resume (if enabled);
- one DKMS module;
- the specific feature you changed.

### 15.5 Update documentation

If you change a value listed in this document or the root README, update it in
the same commit. The tables are meant to be verifiable, not aspirational.

## 16. Full-symbol reference

For a complete list, read `.config` directly. For a searchable summary of the
symbols used in this documentation, see [reference.md](reference.md#9-configuration-symbols).

Useful commands:

```sh
# All enabled booleans
grep '=y$' .config

# All modules
grep '=m$' .config

# All strings
grep '="' .config

# Count active symbols
grep -cE '^CONFIG_[A-Z0-9_]+=' .config

# Running kernel config
zcat /proc/config.gz | less
```
