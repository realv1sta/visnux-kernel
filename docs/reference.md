# Reference

A lookup sheet for paths, commands, configuration symbols, and runtime
interfaces. Values are from the committed `.config` of this tree.

## 1. Identity

| Item | Value |
|---|---|
| Kernel version | `7.2.15` |
| Codename | `Baby Opossum Posse` |
| Local version | `"visnux"` |
| Local version auto | `y` |
| Release (tarball) | `7.2.15visnux` |
| Release (git) | `7.2.15visnux-g3cd079313` |
| Packaged release | `7.2.15-visnux` |
| Architecture | `x86_64` |
| Compiler recorded | GCC 16.2.1 |
| Binutils recorded | 2.47 |
| Kernel compression | zstd |
| Module compression | zstd |
| Active symbols | 9,752 |
| Built-in / modules | 3,224 / 6,314 |
| Disabled | 805 |
| Source files | 94,875 |
| Documentation files | 11,299 |

Commands:

```sh
make kernelrelease
make kernelversion
make localversion
uname -r
zcat /proc/config.gz | head
```

## 2. Important paths

### Source tree

| Path | Contents |
|---|---|
| `.config` | Committed configuration |
| `Makefile` | Version and build entry |
| `arch/x86/boot/bzImage` | Bootable image (after build) |
| `vmlinux` | Uncompressed kernel (after build) |
| `System.map` | Symbol map (after build) |
| `certs/signing_key.pem` | Module signing key (generated) |
| `Documentation/` | Upstream docs |
| `docs/` | This documentation set |
| `scripts/config` | Config editing helper |
| `scripts/setlocalversion` | Release suffix logic |

### Installed system

| Path | Contents |
|---|---|
| `/boot/vmlinuz-visnux` | Kernel image |
| `/boot/initramfs-visnux.img` | Default initramfs |
| `/boot/initramfs-visnux-fallback.img` | Fallback initramfs |
| `/boot/System.map-visnux` | Symbol map |
| `/boot/config-visnux` | Config copy |
| `/usr/lib/modules/7.2.15-visnux/` | Modules and indexes |
| `/usr/lib/modules/7.2.15-visnux/build` | DKMS build symlink |
| `/usr/src/linux-7.2.15-visnux/` | Header tree |
| `/etc/mkinitcpio.d/visnux.preset` | initramfs preset |
| `/proc/config.gz` | Running kernel config |
| `/sys/kernel/debug/` | debugfs (if mounted) |

## 3. Build commands

```sh
# Configure
make olddefconfig
make menuconfig
./scripts/config --enable CONFIG_FOO
./scripts/config --module CONFIG_BAR
./scripts/config --set-str CONFIG_LOCALVERSION "-visnux"

# Build
make -j"$(nproc)" bzImage
make -j"$(nproc)" modules
make -j"$(nproc)" bzImage modules
make LLVM=1 -j"$(nproc)" bzImage modules

# Install
sudo make INSTALL_MOD_PATH=/usr modules_install
sudo make INSTALL_HDR_PATH=/usr headers_install
sudo install -m644 arch/x86/boot/bzImage /boot/vmlinuz-visnux
sudo mkinitcpio -P

# Inspect
make kernelrelease
make kernelversion
make localversion
make -s kernelrelease

# Clean
make clean
make mrproper     # deletes .config — avoid here

# Docs
make htmldocs
make htmldocs SPHINXOPTS="-W"

# Checks
make coccicheck
make checkstack
make W=1
make C=1
```

## 4. initramfs commands

```sh
sudo mkinitcpio -P
sudo mkinitcpio -p visnux
lsinitcpio /boot/initramfs-visnux.img
lsinitcpio -a /boot/initramfs-visnux.img
lsinitcpio /boot/initramfs-visnux.img | grep -E 'nvme|ahci|virtio'
```

## 5. Module commands

```sh
lsmod
modinfo <module>
modprobe <module>
modprobe -r <module>
rmmod <module>
depmod -a
depmod -a "$(uname -r)"
dkms status
sudo dkms autoinstall
cat /proc/sys/kernel/tainted
modinfo -F sig_id <module>.ko
```

## 6. Runtime interfaces

### CPU and scheduler

| Interface | Purpose |
|---|---|
| `/sys/devices/system/cpu/cpu*/cpufreq/` | Frequency scaling |
| `/sys/devices/system/cpu/cpu*/cpuidle/` | Idle states |
| `/sys/kernel/debug/sched/preempt` | Active preemption model |
| `/sys/kernel/sched_ext/` | sched_ext control |
| `/proc/sys/kernel/sched_autogroup_enabled` | Autogroup |
| `/proc/schedstat` | Scheduler stats |
| `/proc/<pid>/sched` | Per-task scheduler info |

### Memory

| Interface | Purpose |
|---|---|
| `/proc/meminfo` | Memory counters |
| `/proc/pressure/{cpu,memory,io}` | PSI |
| `/sys/kernel/mm/lru_gen/` | MGLRU |
| `/sys/kernel/mm/transparent_hugepage/` | THP |
| `/sys/kernel/mm/ksm/` | KSM |
| `/sys/kernel/mm/damon/` | DAMON |
| `/sys/module/zswap/parameters/` | zswap |
| `/sys/block/zram0/` | zram |
| `/sys/fs/cgroup/` | cgroups |

### Block and filesystems

| Interface | Purpose |
|---|---|
| `/sys/block/<dev>/queue/scheduler` | I/O scheduler |
| `/sys/block/<dev>/queue/nr_requests` | Queue depth |
| `/sys/block/<dev>/queue/read_ahead_kb` | Read-ahead |
| `/proc/filesystems` | Registered filesystems |
| `/proc/mounts` | Mounts |
| `/sys/fs/ext4/`, `/sys/fs/btrfs/`, `/sys/fs/xfs/` | FS tunables |

### Networking

| Interface | Purpose |
|---|---|
| `/proc/net/protocols` | Protocols |
| `/proc/net/tcp`, `/proc/net/udp` | Sockets |
| `/proc/sys/net/` | Network sysctls |
| `/sys/class/net/` | Interfaces |
| `/sys/kernel/debug/tracing/` | Tracepoints (incl. net) |

### Security

| Interface | Purpose |
|---|---|
| `/sys/kernel/security/lsm` | Active LSMs |
| `/sys/kernel/security/lockdown` | Lockdown mode |
| `/proc/sys/kernel/yama/ptrace_scope` | Yama |
| `/proc/sys/kernel/dmesg_restrict` | dmesg restriction |
| `/proc/sys/kernel/kptr_restrict` | Kernel pointer restriction |
| `/proc/sys/kernel/unprivileged_bpf_disabled` | Unprivileged BPF |
| `/sys/devices/system/cpu/vulnerabilities/` | Mitigation status |

### Debug and tracing

| Interface | Purpose |
|---|---|
| `/sys/kernel/debug/tracing/` | ftrace |
| `/sys/kernel/debug/kprobes/` | kprobes |
| `/sys/kernel/debug/dynamic_debug/` | dynamic debug |
| `/proc/kallsyms` | Symbols |
| `/proc/kcore` | Kernel core (root) |
| `/proc/config.gz` | Config |
| `/sys/kernel/btf/vmlinux` | BTF |

## 7. Boot parameters

| Parameter | Effect |
|---|---|
| `root=UUID=…` | Root filesystem |
| `ro` / `rw` | Mount root read-only/read-write |
| `init=…` | Userspace init |
| `rdinit=…` | initramfs init |
| `console=…` | Console device |
| `loglevel=…` | Console log level |
| `quiet` | Suppress most boot messages |
| `debug` | Verbose boot |
| `preempt=full|voluntary|none` | Preemption model |
| `nohz_full=…` | Full dynticks CPUs |
| `rcu_nocbs=…` | Offload RCU callbacks |
| `isolcpus=…` | Isolate CPUs |
| `zswap.enabled=1` | Enable zswap |
| `zswap.compressor=zstd` | zswap compressor |
| `zswap.max_pool_percent=…` | zswap pool size |
| `mitigations=auto|off|…` | CPU mitigations |
| `lsm=…` | LSM order |
| `lockdown=none|integrity|confidentiality` | Lockdown mode |
| `module.sig_enforce=1` | Enforce module signatures |
| `resume=UUID=…` | Hibernation resume device |
| `panic=…` | Reboot after panic |
| `nomodeset` | Disable KMS (recovery) |
| `init=/bin/bash` | Emergency shell |
| `sysrq_always_enabled` | Enable SysRq key |

Full list:
[`../Documentation/admin-guide/kernel-parameters.rst`](../Documentation/admin-guide/kernel-parameters.rst).

## 8. sysctl snippets

### Desktop VM

```ini
# /etc/sysctl.d/99-desktop.conf
vm.swappiness = 10
vm.vfs_cache_pressure = 50
vm.dirty_ratio = 10
vm.dirty_background_ratio = 5
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 1024
```

### Hardening

```ini
# /etc/sysctl.d/99-hardening.conf
kernel.dmesg_restrict = 1
kernel.kptr_restrict = 2
kernel.yama.ptrace_scope = 1
kernel.unprivileged_bpf_disabled = 1
kernel.perf_event_paranoid = 3
kernel.randomize_va_space = 2
```

### Network

```ini
# /etc/sysctl.d/99-net.conf
net.ipv4.tcp_congestion_control = bbr
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
```

Apply:

```sh
sudo sysctl --system
```

## 9. Configuration symbols

### Identity and platform

```ini
CONFIG_LOCALVERSION="visnux"
CONFIG_LOCALVERSION_AUTO=y
CONFIG_DEFAULT_HOSTNAME="artixlinux"
CONFIG_X86_64=y
CONFIG_SMP=y
CONFIG_NR_CPUS=8192
CONFIG_IA32_EMULATION=y
CONFIG_X86_KERNEL_IBT=y
CONFIG_X86_USER_SHADOW_STACK=y
CONFIG_X86_UMIP=y
CONFIG_X86_INTEL_MEMORY_PROTECTION_KEYS=y
CONFIG_EFI=y
CONFIG_EFI_STUB=y
CONFIG_ACPI=y
CONFIG_PCI=y
CONFIG_PCI_MSI=y
CONFIG_PM=y
CONFIG_SUSPEND=y
CONFIG_HIBERNATION=y
```

### Scheduler and latency

```ini
CONFIG_PREEMPT=y
CONFIG_PREEMPT_DYNAMIC=y
CONFIG_PREEMPT_RCU=y
CONFIG_HZ=1000
CONFIG_NO_HZ=y
CONFIG_NO_HZ_FULL=y
CONFIG_HIGH_RES_TIMERS=y
CONFIG_SCHED_CORE=y
CONFIG_SCHED_CLASS_EXT=y
CONFIG_SCHED_AUTOGROUP=y
CONFIG_UCLAMP_TASK=y
CONFIG_PSI=y
CONFIG_NUMA_BALANCING=y
CONFIG_SCHED_SMT=y
CONFIG_SCHED_MC=y
CONFIG_SCHED_CLUSTER=y
CONFIG_SCHEDSTATS=y
```

### Memory

```ini
CONFIG_LRU_GEN=y
CONFIG_LRU_GEN_ENABLED=y
CONFIG_SWAP=y
CONFIG_ZSWAP=y
CONFIG_ZSWAP_DEFAULT_ON=y
CONFIG_ZSWAP_COMPRESSOR_DEFAULT="zstd"
CONFIG_ZSWAP_SHRINKER_DEFAULT_ON=y
CONFIG_ZRAM=m
CONFIG_ZRAM_DEF_COMP="zstd"
CONFIG_ZRAM_WRITEBACK=y
CONFIG_ZRAM_MEMORY_TRACKING=y
CONFIG_DAMON=y
CONFIG_DAMON_VADDR=y
CONFIG_DAMON_PADDR=y
CONFIG_DAMON_SYSFS=y
CONFIG_DAMON_RECLAIM=y
CONFIG_DAMON_LRU_SORT=y
CONFIG_DAMON_STAT=y
CONFIG_KSM=y
CONFIG_TRANSPARENT_HUGEPAGE=y
CONFIG_TRANSPARENT_HUGEPAGE_ALWAYS=y
CONFIG_MEMCG=y
CONFIG_CGROUP_BPF=y
CONFIG_INIT_ON_ALLOC_DEFAULT_ON=y
CONFIG_DEFAULT_MMAP_MIN_ADDR=65536
CONFIG_PAGE_POISONING=y
CONFIG_KFENCE=y
```

### Block and I/O

```ini
CONFIG_BLK_CGROUP=y
CONFIG_BLK_CGROUP_IOPRIO=y
CONFIG_BLK_CGROUP_IOLATENCY=y
CONFIG_BLK_CGROUP_IOCOST=y
CONFIG_BLK_DEV_THROTTLING=y
CONFIG_BLK_WBT=y
CONFIG_MQ_IOSCHED_DEADLINE=y
CONFIG_MQ_IOSCHED_KYBER=y
CONFIG_IOSCHED_BFQ=y
CONFIG_BFQ_GROUP_IOSCHED=y
CONFIG_BLK_DEV_LOOP=m
CONFIG_BLK_DEV_DM=m
CONFIG_DM_CRYPT=m
CONFIG_MD=y
```

### Filesystems

```ini
CONFIG_EXT4_FS=y
CONFIG_BTRFS_FS=y
CONFIG_XFS_FS=m
CONFIG_F2FS_FS=m
CONFIG_TMPFS=y
CONFIG_FUSE_FS=y
CONFIG_OVERLAY_FS=m
CONFIG_SQUASHFS=m
CONFIG_EROFS_FS=m
CONFIG_ISO9660_FS=m
CONFIG_FAT_FS=m
CONFIG_VFAT_FS=m
CONFIG_EXFAT_FS=m
CONFIG_NTFS3_FS=m
CONFIG_NFS_FS=m
CONFIG_NFSD=m
CONFIG_CIFS=m
CONFIG_9P_FS=m
CONFIG_ZONEFS_FS=m
CONFIG_FS_ENCRYPTION=y
CONFIG_UNICODE=y
```

### Storage drivers

```ini
CONFIG_SATA_AHCI=y
CONFIG_ATA=y
CONFIG_SCSI=y
CONFIG_BLK_DEV_SD=y
CONFIG_BLK_DEV_NVME=m
CONFIG_NVME_CORE=m
CONFIG_VIRTIO_BLK=y
CONFIG_VIRTIO_PCI=y
CONFIG_VIRTIO_FS=y
CONFIG_VIRTIO_NET=m
CONFIG_VMWARE_PVSCSI=m
```

### Networking

```ini
CONFIG_NET=y
CONFIG_INET=y
CONFIG_IPV6=y
CONFIG_NETFILTER=y
CONFIG_NF_TABLES=m
CONFIG_BRIDGE=m
CONFIG_VLAN_8021Q=m
CONFIG_BONDING=m
CONFIG_MACVLAN=m
CONFIG_VETH=m
CONFIG_TUN=m
CONFIG_TAP=m
CONFIG_WIREGUARD=m
CONFIG_NET_SCHED=y
CONFIG_NET_CLS_BPF=m
CONFIG_NET_ACT_BPF=m
CONFIG_NET_SCH_FQ_CODEL=y
CONFIG_NET_SCH_FQ=m
CONFIG_DEFAULT_TCP_CONG="cubic"
CONFIG_TCP_CONG_BBR=m
CONFIG_BPF_SYSCALL=y
CONFIG_BPF_JIT=y
CONFIG_BPF_JIT_ALWAYS_ON=y
CONFIG_BPF_UNPRIV_DEFAULT_OFF=y
CONFIG_XDP_SOCKETS=y
```

### Virtualisation

```ini
CONFIG_VIRTUALIZATION=y
CONFIG_KVM=m
CONFIG_KVM_INTEL=m
CONFIG_KVM_AMD=m
CONFIG_XEN=y
CONFIG_XEN_PV=y
CONFIG_HYPERV=y
CONFIG_HYPERVISOR_GUEST=y
CONFIG_PARAVIRT=y
CONFIG_VBOXGUEST=m
CONFIG_VBOXSF_FS=m
```

### Graphics, audio, input

```ini
CONFIG_DRM=y
CONFIG_DRM_FBDEV_EMULATION=y
CONFIG_DRM_AMDGPU=m
CONFIG_DRM_I915=m
CONFIG_DRM_XE=m
CONFIG_DRM_NOUVEAU=m
CONFIG_DRM_VIRTIO_GPU=m
CONFIG_DRM_QXL=m
CONFIG_DRM_BOCHS=m
CONFIG_DRM_VMWGFX=m
CONFIG_FB=y
CONFIG_FB_EFI=y
CONFIG_FRAMEBUFFER_CONSOLE=y
CONFIG_SOUND=m
CONFIG_SND=m
CONFIG_SND_HDA_INTEL=m
CONFIG_SND_USB_AUDIO=m
CONFIG_SND_SOC_SOF_TOPLEVEL=y
CONFIG_INPUT=y
CONFIG_HID=y
CONFIG_USB=y
```

### Security

```ini
CONFIG_SECURITY=y
CONFIG_LSM="landlock,lockdown,yama,integrity,bpf"
CONFIG_SECURITY_SELINUX=y
CONFIG_SECURITY_APPARMOR=y
CONFIG_SECURITY_SMACK=y
CONFIG_SECURITY_TOMOYO=y
CONFIG_SECURITY_LANDLOCK=y
CONFIG_SECURITY_YAMA=y
CONFIG_SECURITY_LOCKDOWN_LSM=y
CONFIG_LOCK_DOWN_KERNEL_FORCE_NONE=y
CONFIG_SECURITY_LOADPIN=y
CONFIG_SECURITY_SAFESETID=y
CONFIG_SECURITY_IPE=y
CONFIG_INTEGRITY=y
CONFIG_INTEGRITY_SIGNATURE=y
CONFIG_RANDOMIZE_BASE=y
CONFIG_RANDOMIZE_MEMORY=y
CONFIG_RANDOMIZE_KSTACK_OFFSET_DEFAULT=y
CONFIG_STRICT_KERNEL_RWX=y
CONFIG_STRICT_MODULE_RWX=y
CONFIG_STACKPROTECTOR_STRONG=y
CONFIG_VMAP_STACK=y
CONFIG_FORTIFY_SOURCE=y
CONFIG_HARDENED_USERCOPY=y
CONFIG_INIT_ON_ALLOC_DEFAULT_ON=y
CONFIG_SLAB_FREELIST_RANDOM=y
CONFIG_SLAB_FREELIST_HARDENED=y
CONFIG_SHUFFLE_PAGE_ALLOCATOR=y
CONFIG_CPU_MITIGATIONS=y
CONFIG_MITIGATION_RETPOLINE=y
CONFIG_MITIGATION_SPECTRE_V2=y
CONFIG_SECURITY_DMESG_RESTRICT=y
CONFIG_IO_STRICT_DEVMEM=y
CONFIG_BPF_UNPRIV_DEFAULT_OFF=y
CONFIG_MODULE_SIG=y
CONFIG_MODULE_SIG_ALL=y
CONFIG_MODULE_SIG_HASH="sha512"
```

### Modules and debug

```ini
CONFIG_MODULES=y
CONFIG_MODULE_UNLOAD=y
CONFIG_MODULE_COMPRESS=y
CONFIG_MODULE_COMPRESS_ZSTD=y
CONFIG_DEBUG_KERNEL=y
CONFIG_DEBUG_INFO=y
CONFIG_DEBUG_INFO_DWARF5=y
CONFIG_DEBUG_INFO_BTF=y
CONFIG_DEBUG_INFO_BTF_MODULES=y
CONFIG_KALLSYMS=y
CONFIG_KALLSYMS_ALL=y
CONFIG_GDB_SCRIPTS=y
CONFIG_DEBUG_FS=y
CONFIG_DYNAMIC_DEBUG=y
CONFIG_FTRACE=y
CONFIG_FUNCTION_TRACER=y
CONFIG_FUNCTION_GRAPH_TRACER=y
CONFIG_DYNAMIC_FTRACE=y
CONFIG_KPROBES=y
CONFIG_UPROBES=y
CONFIG_PERF_EVENTS=y
CONFIG_BPF_EVENTS=y
CONFIG_IKCONFIG=y
CONFIG_IKCONFIG_PROC=y
CONFIG_KEXEC=y
CONFIG_KEXEC_FILE=y
CONFIG_CRASH_DUMP=y
CONFIG_MAGIC_SYSRQ=y
CONFIG_MAGIC_SYSRQ_DEFAULT_ENABLE=0x0
CONFIG_UNWINDER_ORC=y
CONFIG_PANIC_TIMEOUT=0
```

## 10. Package metadata

From the published split package (`visnux 7.2.15-1`):

```ini
pkgname = visnux
pkgbase = visnux
pkgver = 7.2.15-1
pkgdesc = Visnux Linux kernel - lean, init-agnostic, tuned for gaming and development
url = https://visnux.org
arch = x86_64
license = GPL2
provides = kernel=7.2.15
depend = coreutils
depend = kmod
depend = initramfs
makedepend = bc cpio gettext libelf pahole perl tar xz
```

| Package | Installed size |
|---|---|
| `visnux` | ≈ 100 MB |
| `visnux-headers` | ≈ 280 MB |

## 11. Glossary

| Term | Meaning |
|---|---|
| **BFQ** | Budget Fair Queueing I/O scheduler |
| **BTF** | BPF Type Format; enables CO-RE |
| **CET** | Control-flow Enforcement Technology (IBT + shadow stack) |
| **DKMS** | Dynamic Kernel Module Support |
| **DAMON** | Data Access Monitor |
| **EEVDF** | Earliest Eligible Virtual Deadline First scheduler |
| **KASLR** | Kernel Address Space Layout Randomisation |
| **KFENCE** | Kernel Electric-Fence, sampling memory-error detector |
| **KSM** | Kernel Same-page Merging |
| **LSM** | Linux Security Module |
| **MGLRU** | Multi-Generational LRU |
| **MOK** | Machine Owner Key (Secure Boot) |
| **PSI** | Pressure Stall Information |
| **sched_ext** | Extensible scheduler via BPF |
| **THP** | Transparent Huge Pages |
| **W^X** | Write xor Execute memory protection |
| **zram** | Compressed RAM block device |
| **zswap** | Compressed swap cache |

## 12. Quick verification script

```sh
#!/bin/sh
set -eu
echo "release: $(uname -r)"
echo "config:"
zcat /proc/config.gz | grep -E \
 'CONFIG_(PREEMPT|HZ|LRU_GEN_ENABLED|ZSWAP_DEFAULT_ON|IOSCHED_BFQ|BFQ_GROUP_IOSCHED|SCHED_AUTOGROUP|PSI|DAMON|KSM|MODULE_SIG|LSM)='
echo "psi:"; cat /proc/pressure/cpu
echo "zswap:"; cat /sys/module/zswap/parameters/enabled
echo "mglru:"; cat /sys/kernel/mm/lru_gen/enabled
echo "lsm:"; cat /sys/kernel/security/lsm 2>/dev/null || true
echo "taint:"; cat /proc/sys/kernel/tainted
```
