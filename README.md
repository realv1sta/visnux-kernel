# Visnux Linux kernel

**Visnux** is a Linux-based desktop operating system built for gaming, development,
and everyday use. This repository, **`visnux-kernel`**, contains the complete Linux
kernel source tree and the committed `.config` that defines a Visnux kernel build.

- **Kernel line:** Linux `7.2.15` (codename *Baby Opossum Posse*)
- **Architecture:** `x86_64` (with 32-bit `IA32` emulation)
- **Configuration:** committed in [`.config`](.config) — the single source of truth
- **Upstream:** vanilla `linux-7.2.15` from <https://kernel.org>, no out-of-tree patches
- **Licence:** GPL-2.0 only — see [`COPYING`](COPYING)
- **Project:** [`visnux.org`](https://github.com/realv1sta/visnux)
- **Packages:** [`realv1sta/ELOV`](https://github.com/realv1sta/ELOV)
- **Installer / ISO:** [`realv1sta/visnux-testing`](https://github.com/realv1sta/visnux-testing)

> **Documentation scope**
> This README and the files under [`docs/`](docs/) describe the kernel, its
> configuration, and how to build, install, tune, and troubleshoot it. They do
> not change the kernel source or `.config`.

---

## Table of contents

1. [What this repository is](#what-this-repository-is)
2. [Quick start](#quick-start)
3. [Kernel identity](#kernel-identity)
4. [Design goals](#design-goals)
5. [Feature highlights](#feature-highlights)
6. [Configuration at a glance](#configuration-at-a-glance)
7. [Building from source](#building-from-source)
8. [Installing and booting](#installing-and-booting)
9. [Headers and DKMS](#headers-and-dkms)
10. [Modules, signing, and compression](#modules-signing-and-compression)
11. [Runtime tuning](#runtime-tuning)
12. [Security posture](#security-posture)
13. [Virtualization and guests](#virtualization-and-guests)
14. [Filesystems and storage](#filesystems-and-storage)
15. [Networking](#networking)
16. [Graphics, audio, and input](#graphics-audio-and-input)
17. [Debugging and tracing](#debugging-and-tracing)
18. [Troubleshooting](#troubleshooting)
19. [Repository layout](#repository-layout)
20. [Documentation map](#documentation-map)
21. [Versioning and release process](#versioning-and-release-process)
22. [Relationship to other Visnux repositories](#relationship-to-other-visnux-repositories)
23. [Contributing](#contributing)
24. [FAQ](#faq)
25. [Licence and credits](#licence-and-credits)

---

## What this repository is

`visnux-kernel` is a **full Linux kernel source tree** plus the configuration
used to produce Visnux kernels. It is not a patch set, a module collection, or a
wrapper script: it is the upstream kernel, byte-for-byte, with a `.config`
committed alongside it.

| Component | Where it lives | Purpose |
|---|---|---|
| Kernel sources | `arch/`, `block/`, `crypto/`, `drivers/`, `fs/`, `include/`, `kernel/`, `lib/`, `mm/`, `net/`, `security/`, `sound/`, `virt/`, … | Vanilla Linux 7.2.15 |
| Kernel configuration | [`.config`](.config) | Every build option, committed on purpose |
| Build system | `Makefile`, `Kbuild`, `Kconfig`, `scripts/` | Kbuild |
| In-tree documentation | [`Documentation/`](Documentation/) | 11,299 upstream documentation files |
| Rust support | `rust/` | Upstream Rust-for-Linux sources (not enabled in this config) |
| Tools and samples | `tools/`, `samples/` | Perf, BPF, selftests, examples |
| Visnux documentation | [`README.md`](README.md), [`docs/`](docs/) | This documentation set |

### What “vanilla source” means

There are **no Visnux-specific patches** in the C source. Every behavioural
difference between a Visnux kernel and a stock `linux-7.2.15` build comes from:

1. the committed `.config`;
2. the release string (`CONFIG_LOCALVERSION`);
3. packaging choices (initramfs preset, install paths, module signing).

That makes the tree easy to audit, rebase, and compare against upstream.

---

## Quick start

```sh
# 1. Get the source
git clone https://github.com/realv1sta/visnux-kernel.git
cd visnux-kernel

# 2. Honour the committed configuration
make olddefconfig

# 3. Build the compressed kernel image and all modules
make -j"$(nproc)" bzImage modules

# 4. Install modules, image, and initramfs
sudo make INSTALL_MOD_PATH=/usr modules_install
sudo install -m644 arch/x86/boot/bzImage /boot/vmlinuz-visnux
sudo mkinitcpio -P

# 5. Point your bootloader at /boot/vmlinuz-visnux and reboot
```

> The release string produced by this tree’s `.config` is `7.2.15visnux`
> (plus `-g<sha>` when built inside this git checkout). The packaged Visnux
> kernel shipped through ELOV uses the tuned `-visnux` configuration and installs
> as `7.2.15-visnux`. See [Kernel identity](#kernel-identity) and
> [Relationship to other Visnux repositories](#relationship-to-other-visnux-repositories).

For the long-form version — host dependencies, cross-compilation, packaging,
bootloader entries, Secure Boot, DKMS, and rollback — read
[`docs/building.md`](docs/building.md) and [`docs/installation.md`](docs/installation.md).

---

## Kernel identity

| Property | Value |
|---|---|
| Version | `7.2.15` |
| Codename | `Baby Opossum Posse` |
| `CONFIG_LOCALVERSION` | `"visnux"` |
| `CONFIG_LOCALVERSION_AUTO` | `y` |
| Release (tarball build) | `7.2.15visnux` |
| Release (this git checkout) | `7.2.15visnux-g3cd079313` (auto `-g<sha>`) |
| Machine | `x86_64` |
| SMP | yes, `CONFIG_NR_CPUS=8192` |
| 32-bit compat | yes, `CONFIG_IA32_EMULATION=y` |
| Default hostname | `artixlinux` |
| Kernel image compression | Zstandard (`CONFIG_KERNEL_ZSTD=y`) |
| Recorded compiler | GCC 16.2.1 (`CONFIG_CC_VERSION_TEXT`) |
| Recorded binutils | 2.47 |
| Recorded Rust toolchain | 1.100.0 / LLVM 23 (Rust not enabled) |
| Active config symbols | 9,752 |
| Built-in (`=y`) | 3,224 |
| Modules (`=m`) | 6,314 |
| Explicitly disabled | 805 |
| Source files | 94,875 |
| Documentation files | 11,299 |
| Working-tree size | ≈ 2.1 GB |

Check the exact release string of any build with:

```sh
make kernelrelease
# or, after building:
cat include/config/kernel.release
```

`uname -r` on a running kernel reports the same string.

---

## Design goals

The configuration is opinionated. Its goals, in order:

1. **Boot reliably, everywhere.** Storage and virtualisation drivers needed to
   reach a root filesystem or a display are built in (`=y`) where practical, so
   boot does not depend on an initramfs pulling the right module in the right
   order.
2. **Feel responsive on a desktop.** Full preemption, `HZ=1000`, high-resolution
   timers, `schedutil`, `sched_ext`, PSI, and autogroup are all enabled.
3. **Keep memory pressure cheap.** zswap (zstd) is on by default, zram is
   available, MGLRU is enabled, and DAMON provides proactive reclaim and LRU
   sorting.
4. **Stay useful for development.** Debug info (DWARF5 + BTF), ftrace, kprobes,
   uprobes, perf events, BPF, and `DEBUG_FS` are present.
5. **Keep the attack surface comprehensible.** Hardware that a desktop does not
   need is disabled rather than compiled as a module, and hardening defaults are
   on.
6. **Remain a normal kernel.** No out-of-tree patches, no vendor forks, no
   surprise ABI. Anything that works on upstream Linux 7.2.15 works here.

These goals sometimes conflict. Where they do, the choice is documented in
[`docs/configuration.md`](docs/configuration.md).

---

## Feature highlights

### Scheduling and latency

| Feature | State | Why it matters |
|---|---|---|
| Full preemption (`CONFIG_PREEMPT=y`) | on | Desktop workloads preempt in-kernel work |
| Dynamic preemption (`CONFIG_PREEMPT_DYNAMIC=y`) | on | Switch model at boot with `preempt=` |
| `HZ=1000` | on | 1 ms timer granularity |
| High-resolution timers | on | Accurate sleeps and timeouts |
| Tickless idle (`NO_HZ_IDLE` behaviour) | on | Less idle wakeup overhead |
| Full dynticks (`CONFIG_NO_HZ_FULL=y`) | compiled in | Opt-in with `nohz_full=` |
| `sched_ext` (`CONFIG_SCHED_CLASS_EXT=y`) | on | Loadable BPF schedulers |
| Core scheduling (`CONFIG_SCHED_CORE=y`) | on | SMT-aware scheduling |
| `uclamp` (`CONFIG_UCLAMP_TASK=y`) | on | Utilisation clamping for power/perf |
| PSI (`CONFIG_PSI=y`) | on | Pressure Stall Information for monitoring |
| Autogroup (`CONFIG_SCHED_AUTOGROUP=y`) | on | Fairness between desktop sessions |
| `schedutil` default governor | on | Frequency scaling follows the scheduler |

### Memory management

| Feature | State | Notes |
|---|---|---|
| MGLRU (`CONFIG_LRU_GEN=y`) | on and enabled by default | Multi-generational LRU |
| zswap | on by default, `zstd` | Compressed writeback cache in front of swap |
| zram | `=m`, `zstd` default | Compressed block device for swap/tmp |
| DAMON | on (`VADDR`, `PADDR`, `SYSFS`, `RECLAIM`, `LRU_SORT`, `STAT`) | Data Access Monitor |
| KSM | on | Kernel Same-page Merging |
| Swap | on | zswap/zram/hibernation support |
| Memory cgroup | on | `memory.max`, `memory.high`, PSI, etc. |
| Cgroup BPF | on | BPF programs attached to cgroups |
| `INIT_ON_ALLOC_DEFAULT_ON` | on | Zero pages on allocation |

### Storage and I/O

| Feature | State |
|---|---|
| BFQ I/O scheduler | built in, cgroup-aware (`BFQ_GROUP_IOSCHED=y`) |
| mq-deadline | built in |
| Kyber | built in |
| blk-cgroup iocost / iolatency / ioprio | on |
| Writeback throttling (`BLK_WBT`) | on |
| NVMe | `=m` |
| AHCI / SATA | `=y` |
| SCSI disk | `=y` |
| virtio-blk | `=y` |
| Device mapper / dm-crypt / LUKS | `=m` |
| MD RAID 0/1/10/5/6 | `=m` |
| Loop devices | `=m` |

### Filesystems

| Filesystem | State | Typical use |
|---|---|---|
| ext4 | `=y` | Root, `/home`, general purpose |
| Btrfs | `=y` | Snapshots, subvolumes, compression |
| XFS | `=m` | Large files, high-throughput storage |
| F2FS | `=m` | Flash-friendly storage |
| VFAT / FAT / exFAT | `=m` | EFI ESP, removable media |
| NTFS3 | `=m` | Windows volumes |
| FUSE | `=y` | `sshfs`, `ntfs-3g`, AppImages, … |
| OverlayFS | `=m` | Containers, live systems |
| SquashFS | `=m` | Live ISO, snap packages |
| EROFS | `=m` | Read-only compressed images |
| ISO 9660 | `=m` | Optical media |
| NFS / NFSD / CIFS / 9p | `=m` | Network and virtual filesystems |
| tmpfs | `=y` | `/tmp`, `/run`, `/dev/shm` |
| fscrypt / casefolding | `=y` | Per-directory encryption, Unicode names |
| zonefs | `=m` | Zoned block devices |

### Networking

| Feature | State |
|---|---|
| IPv4 / IPv6 | `=y` |
| Netfilter / nf_tables | `=y` / `=m` |
| WireGuard | `=m` |
| Bridge, VLAN, bonding, MACVLAN, veth, TUN/TAP | `=m` |
| BPF syscall + JIT | `=y` |
| XDP sockets | `=y` |
| TCP default congestion control | `cubic` |
| BBR | `=m` |
| `fq_codel` / `fq` | `=y` / `=m` |

### Virtualisation

| Feature | State |
|---|---|
| KVM (Intel / AMD) | `=m` |
| Xen PV guest | `=y` |
| Hyper-V guest | `=y` |
| VirtualBox Guest Additions | `=m` |
| VMware paravirtual SCSI | `=m` |
| virtio-blk | `=y` |
| virtio-net | `=m` |
| virtio-gpu | `=m` |
| virtio-fs | `=y` |
| virtio-pci | `=y` |
| 9p filesystem | `=m` |

### Security and hardening

| Feature | State |
|---|---|
| LSM framework | on |
| Default LSM order | `landlock,lockdown,yama,integrity,bpf` |
| SELinux / AppArmor / Smack / TOMOYO | compiled in (not in default order) |
| Landlock | on |
| Yama | on |
| Lockdown LSM | on (`none` by default) |
| Module signing | on, all modules signed, **not** enforced |
| Module compression | Zstandard |
| KASLR (`RANDOMIZE_BASE`) | on |
| Memory layout randomisation | on |
| `STRICT_KERNEL_RWX` / `STRICT_MODULE_RWX` | on |
| Stack protector | strong |
| `FORTIFY_SOURCE` | on |
| Hardened usercopy | on |
| `INIT_ON_ALLOC_DEFAULT_ON` | on |
| SLAB freelist randomisation + hardening | on |
| Page allocator shuffle | on |
| `dmesg_restrict` | on |
| `IO_STRICT_DEVMEM` | on |
| Unprivileged BPF | off by default |
| CPU mitigations | on |
| CET: IBT + user shadow stack | on |
| UMIP / MPK | on |

### Graphics, audio, input

| Feature | State |
|---|---|
| DRM | `=y` |
| AMDGPU | `=m` |
| i915 (Intel) | `=m` |
| Xe (Intel, new) | `=m` |
| Nouveau | `=m` |
| virtio-gpu / QXL / Bochs / vmwgfx | `=m` |
| EFI framebuffer + framebuffer console | `=y` |
| DRM fbdev emulation | `=y` |
| ALSA | `=m` |
| HDA Intel | `=m` |
| USB audio | `=m` |
| Sound Open Firmware (SOF) | `=y` top level |
| HID / USB / input core | `=y` |

---

## Configuration at a glance

The complete configuration is [`.config`](.config). These are the settings most
people ask about.

```ini
# Identity
CONFIG_LOCALVERSION="visnux"
CONFIG_LOCALVERSION_AUTO=y
CONFIG_DEFAULT_HOSTNAME="artixlinux"

# Platform
CONFIG_X86_64=y
CONFIG_SMP=y
CONFIG_NR_CPUS=8192
CONFIG_IA32_EMULATION=y
CONFIG_X86_KERNEL_IBT=y
CONFIG_X86_USER_SHADOW_STACK=y

# Latency
CONFIG_PREEMPT=y
CONFIG_PREEMPT_DYNAMIC=y
CONFIG_HZ=1000
CONFIG_NO_HZ_FULL=y
CONFIG_HIGH_RES_TIMERS=y

# Scheduler
CONFIG_SCHED_CORE=y
CONFIG_SCHED_CLASS_EXT=y
CONFIG_SCHED_AUTOGROUP=y
CONFIG_UCLAMP_TASK=y
CONFIG_PSI=y
CONFIG_NUMA_BALANCING=y

# Memory
CONFIG_LRU_GEN=y
CONFIG_LRU_GEN_ENABLED=y
CONFIG_ZSWAP=y
CONFIG_ZSWAP_DEFAULT_ON=y
CONFIG_ZSWAP_COMPRESSOR_DEFAULT="zstd"
CONFIG_ZRAM=m
CONFIG_DAMON=y
CONFIG_KSM=y
CONFIG_SWAP=y

# I/O
CONFIG_IOSCHED_BFQ=y
CONFIG_BFQ_GROUP_IOSCHED=y
CONFIG_MQ_IOSCHED_DEADLINE=y
CONFIG_MQ_IOSCHED_KYBER=y
CONFIG_BLK_WBT=y

# CPU frequency
CONFIG_CPU_FREQ_DEFAULT_GOV_SCHEDUTIL=y
CONFIG_X86_INTEL_PSTATE=y
CONFIG_X86_AMD_PSTATE=y

# Hardening
CONFIG_RANDOMIZE_BASE=y
CONFIG_RANDOMIZE_MEMORY=y
CONFIG_STRICT_KERNEL_RWX=y
CONFIG_STRICT_MODULE_RWX=y
CONFIG_STACKPROTECTOR_STRONG=y
CONFIG_FORTIFY_SOURCE=y
CONFIG_HARDENED_USERCOPY=y
CONFIG_INIT_ON_ALLOC_DEFAULT_ON=y
CONFIG_SLAB_FREELIST_RANDOM=y
CONFIG_SLAB_FREELIST_HARDENED=y
CONFIG_SHUFFLE_PAGE_ALLOCATOR=y
CONFIG_SECURITY_DMESG_RESTRICT=y
CONFIG_BPF_UNPRIV_DEFAULT_OFF=y
CONFIG_CPU_MITIGATIONS=y
CONFIG_LSM="landlock,lockdown,yama,integrity,bpf"

# Modules
CONFIG_MODULES=y
CONFIG_MODULE_UNLOAD=y
CONFIG_MODULE_SIG=y
CONFIG_MODULE_SIG_ALL=y
CONFIG_MODULE_COMPRESS=y
CONFIG_MODULE_COMPRESS_ZSTD=y

# Debug / observability
CONFIG_DEBUG_INFO=y
CONFIG_DEBUG_INFO_DWARF5=y
CONFIG_DEBUG_INFO_BTF=y
CONFIG_KALLSYMS_ALL=y
CONFIG_DEBUG_FS=y
CONFIG_FTRACE=y
CONFIG_KPROBES=y
CONFIG_UPROBES=y
CONFIG_PERF_EVENTS=y
```

For every important symbol, its dependencies, and the reasoning behind it, see
[`docs/configuration.md`](docs/configuration.md) and
[`docs/reference.md`](docs/reference.md).

---

## Building from source

### Host requirements

A 64-bit x86 host with a working toolchain. On Arch-based systems:

```sh
sudo pacman -S --needed base-devel bc cpio gettext libelf pahole perl tar xz zstd
```

Optional but recommended:

| Package | Needed for |
|---|---|
| `openssl` | module signing key generation |
| `rust` / `rust-src` / `bindgen` | Rust support (not enabled in this config) |
| `clang` / `lld` | building with LLVM instead of GCC |
| `ccache` | faster rebuilds |
| `python-sphinx`, `graphviz`, `imagemagick` | building `make htmldocs` |
| `qemu-full` | booting the result in a VM |

The authoritative dependency list is
[`Documentation/process/changes.rst`](Documentation/process/changes.rst).

### Build

```sh
make olddefconfig                 # resolve the committed .config against Kconfig
make -j"$(nproc)" bzImage modules # kernel image + modules
```

Useful variants:

```sh
make -j"$(nproc)" all             # bzImage + modules + dtbs (x86: same as above)
make -j"$(nproc)" bzImage         # image only
make -j"$(nproc)" modules         # modules only
make -j"$(nproc)" M=drivers/foo   # one external/in-tree module directory
make kernelrelease                # exact release string
make clean                        # remove most build output
make mrproper                     # remove build output AND .config (careful!)
```

> **Do not run `make mrproper`** unless you intend to delete the committed
> `.config`. If it happens, restore it with `git checkout -- .config`.

Outputs:

| Path | What it is |
|---|---|
| `arch/x86/boot/bzImage` | Compressed bootable kernel image |
| `vmlinux` | Uncompressed ELF with symbols (debug) |
| `System.map` | Symbol table |
| `modules.order`, `Module.symvers` | Module build metadata |
| `include/config/kernel.release` | Release string |
| `*.ko` (throughout the tree) | Loadable modules |

### Reproducibility notes

The build records the compiler version and embeds build metadata. To make two
builds comparable:

- use the same compiler and binutils versions;
- build from the same git commit (or set `KBUILD_BUILD_VERSION`,
  `KBUILD_BUILD_TIMESTAMP`, and `KBUILD_BUILD_USER`/`HOSTNAME`);
- do not rely on `CONFIG_LOCALVERSION_AUTO` if you need a stable release string —
  it appends the git SHA.

Full details, including a cross-compilation walkthrough and a reference
`PKGBUILD`, are in [`docs/building.md`](docs/building.md).

---

## Installing and booting

The packaged kernel installs these paths:

| Path | Contents |
|---|---|
| `/boot/vmlinuz-visnux` | Kernel image |
| `/usr/lib/modules/7.2.15-visnux/` | Modules, `modules.dep`, `modules.builtin`, … |
| `/etc/mkinitcpio.d/visnux.preset` | initramfs preset |
| `/usr/src/linux-7.2.15-visnux/` | Headers and build scripts (`visnux-headers`) |
| `/usr/lib/modules/7.2.15-visnux/build` | Symlink used by DKMS |

The preset creates two images:

| Image | Options | Purpose |
|---|---|---|
| `/boot/initramfs-visnux.img` | `-S autodetect,kms` | Generic image with every driver; boots on hardware the installer did not see |
| `/boot/initramfs-visnux-fallback.img` | `-S autodetect` | Same, but keeps the `kms` hook for a clean console handover |

Build and install by hand:

```sh
sudo make INSTALL_MOD_PATH=/usr modules_install
sudo install -m644 arch/x86/boot/bzImage /boot/vmlinuz-visnux
sudo install -Dm644 System.map /boot/System.map-visnux
sudo mkinitcpio -P
```

A minimal GRUB entry:

```ini
menuentry "Visnux Linux" {
    linux  /boot/vmlinuz-visnux root=UUID=<uuid> rw quiet
    initrd /boot/initramfs-visnux.img
}
```

A systemd-boot entry (`/boot/loader/entries/visnux.conf`):

```ini
title   Visnux Linux
linux   /vmlinuz-visnux
initrd  /initramfs-visnux.img
options root=UUID=<uuid> rw quiet
```

Keep at least one known-good kernel entry installed. Bootloader configuration,
Secure Boot, hibernation, and rollback are covered in
[`docs/installation.md`](docs/installation.md).

---

## Headers and DKMS

The split `visnux-headers` package installs the configured source tree under
`/usr/src/linux-7.2.15-visnux/` together with generated headers and the
`build` symlink, so out-of-tree modules can compile:

```sh
dkms status
sudo dkms autoinstall
```

Typical DKMS consumers:

- `nvidia-dkms` (proprietary NVIDIA driver)
- VirtualBox Guest Additions (`vboxguest`, `vboxsf`)
- WireGuard (if not using the in-tree module)
- ZFS (`zfs-dkms`)
- VMware modules
- third-party Wi-Fi and capture drivers

If a module fails to build, check that:

1. the running kernel is the one the headers were built for (`uname -r`);
2. `/lib/modules/$(uname -r)/build` points at the matching source tree;
3. the module supports Linux 7.2.15.

See [`docs/installation.md`](docs/installation.md#6-dkms) for details.

---

## Modules, signing, and compression

| Setting | Value | Consequence |
|---|---|---|
| `CONFIG_MODULES` | `y` | Loadable modules supported |
| `CONFIG_MODULE_UNLOAD` | `y` | Modules can be removed |
| `CONFIG_MODULE_SIG` | `y` | Signature verification available |
| `CONFIG_MODULE_SIG_ALL` | `y` | Every build-time module is signed |
| `CONFIG_MODULE_SIG_FORCE` | not set | Unsigned modules still load (kernel taints) |
| `CONFIG_MODULE_COMPRESS` | `y` | Modules compressed on install |
| `CONFIG_MODULE_COMPRESS_ZSTD` | `y` | Zstandard compression |
| `CONFIG_MODULE_SIG_KEY` | `certs/signing_key.pem` | Build-time signing key |

Implications:

- Out-of-tree modules built against the headers are signed with the key present
  in the header tree, if one was generated.
- Because signature enforcement is **off**, third-party unsigned modules load,
  but the kernel is marked tainted (`TAINT_UNSIGNED_MODULE`, flag `E`).
- Compressed modules are decompressed by the kernel loader; tools such as
  `modprobe` and `depmod` handle this transparently.

Check taint state:

```sh
cat /proc/sys/kernel/tainted
dmesg | grep -i taint
```

---

## Runtime tuning

Everything below can be changed at runtime without rebuilding. Persistent
settings belong in `/etc/sysctl.d/`, `/etc/modprobe.d/`, `/etc/systemd/`, or your
init system’s equivalent.

### CPU frequency

```sh
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_governors
echo schedutil | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

`schedutil` is the compiled default. `intel_pstate` (Intel) and `amd_pstate`
(AMD, active mode) are built in.

### Preemption model at boot

Because `CONFIG_PREEMPT_DYNAMIC=y`:

```text
preempt=full        # default
preempt=voluntary
preempt=none
```

### Full dynticks

`CONFIG_NO_HZ_FULL=y` is compiled in but inactive until CPUs are listed:

```text
nohz_full=2-7 rcu_nocbs=2-7
```

Do not include CPU 0. Full dynticks is a specialist setting; it can hurt
throughput and is not needed for normal desktop use.

### zswap

```sh
cat /sys/module/zswap/parameters/enabled          # Y
cat /sys/module/zswap/parameters/compressor       # zstd
cat /sys/module/zswap/parameters/max_pool_percent # 20
```

Persist via kernel command line:

```text
zswap.enabled=1 zswap.compressor=zstd zswap.max_pool_percent=25
```

### zram

```sh
sudo modprobe zram
echo zstd | sudo tee /sys/block/zram0/comp_algorithm
echo 4G    | sudo tee /sys/block/zram0/disksize
sudo mkswap /dev/zram0
sudo swapon -p 100 /dev/zram0
```

### MGLRU

```sh
cat /sys/kernel/mm/lru_gen/enabled    # 0x0007 when fully enabled
cat /sys/kernel/mm/lru_gen/min_ttl_ms
```

MGLRU is enabled by default in this configuration.

### DAMON

```sh
ls /sys/kernel/mm/damon/admin
# or use the userspace tool:
damo --help
```

DAMON is configured with `VADDR`, `PADDR`, `SYSFS`, `RECLAIM`, `LRU_SORT`, and
`STAT`.

### I/O scheduler

```sh
cat /sys/block/nvme0n1/queue/scheduler
echo bfq | sudo tee /sys/block/nvme0n1/queue/scheduler
```

BFQ, mq-deadline, and Kyber are built in. The default is chosen by the block
layer; set it per device or with a udev rule.

### PSI

```sh
cat /proc/pressure/cpu
cat /proc/pressure/memory
cat /proc/pressure/io
```

### sched_ext

```sh
ls /sys/kernel/sched_ext
# load a BPF scheduler with the scx toolset, e.g.:
# scx_lavd
```

### Memory tunables

```sh
# Transparent huge pages
cat /sys/kernel/mm/transparent_hugepage/enabled

# KSM
echo 1 | sudo tee /sys/kernel/mm/ksm/run

# Swappiness
sysctl vm.swappiness
```

More recipes, including sysctl snippets and systemd units, are in
[`docs/tuning.md`](docs/tuning.md).

---

## Security posture

This configuration is **hardened by default but not locked down**:

- KASLR and memory-layout randomisation are on.
- W^X is enforced for kernel text and modules.
- Strong stack protector, `FORTIFY_SOURCE`, hardened usercopy, zero-on-alloc,
  freelist randomisation, and page-allocator shuffling are on.
- `dmesg_restrict=1` and `IO_STRICT_DEVMEM=y` are on.
- Unprivileged BPF is disabled by default.
- CPU mitigations are on. Disabling them (`mitigations=off`) is a deliberate
  performance/security trade-off.
- Module signing is available and all in-tree modules are signed, but signature
  enforcement is **off** so DKMS and third-party modules keep working.
- Lockdown is compiled in but defaults to `none`.

### LSMs

Compiled: Landlock, Lockdown, Yama, Integrity, BPF, SELinux, AppArmor, Smack,
TOMOYO.

Default order:

```ini
CONFIG_LSM="landlock,lockdown,yama,integrity,bpf"
```

SELinux, AppArmor, Smack, and TOMOYO are **not active by default**. To use one,
select it explicitly on the kernel command line, for example:

```text
lsm=landlock,lockdown,yama,integrity,bpf,apparmor
```

or

```text
lsm=landlock,lockdown,yama,integrity,bpf,selinux
```

Make sure the userspace policy/tools are installed and configured before doing
so.

The full threat model, Secure Boot notes, and hardening checklist are in
[`docs/security.md`](docs/security.md).

---

## Virtualization and guests

Visnux boots both as a host and as a guest:

| Role | Support |
|---|---|
| KVM host | `kvm`, `kvm_intel`, `kvm_amd` as modules |
| Xen guest | built in (`XEN_PV=y`) |
| Hyper-V guest | built in (`HYPERV=y`) |
| VirtualBox guest | `vboxguest` module |
| VMware guest | `vmw_pvscsi`, `vmwgfx` modules |
| QEMU/KVM guest | `virtio-blk` built in; `virtio-net`, `virtio-gpu` modules |
| Shared folders | `virtio-fs` built in; `9p` module |

For a VM, the built-in `virtio-blk` and `virtio-pci` support means the root disk
is reachable even with a minimal initramfs. `virtio-net` and `virtio-gpu` are
modules, so an initramfs built without `autodetect` (the packaged preset does
this) will include them.

---

## Filesystems and storage

Recommended layouts:

| Use case | Filesystem | Notes |
|---|---|---|
| General desktop | ext4 or Btrfs | Both built in |
| Snapshots / rollback | Btrfs | Subvolumes + `snapper`/`timeshift` |
| Large media / scratch | XFS | Module, load before use |
| Flash storage | F2FS | Module |
| EFI system partition | FAT32 / VFAT | Module; present in initramfs |
| Windows dual-boot | NTFS3 | Module |
| Containers | OverlayFS | Module |
| Live images | SquashFS / EROFS | Modules |

Encryption:

- dm-crypt/LUKS: `CONFIG_DM_CRYPT=m`, plus AES, XTS, SHA-256/512 built in.
- fscrypt (per-directory): `CONFIG_FS_ENCRYPTION=y`, `CONFIG_UNICODE=y`.
- Swap encryption: use `dm-crypt` or zram with encryption, not plain swap.

`CONFIG_BLK_DEV_LOOP_MIN_COUNT=0` means loop devices are created on demand
rather than preallocated.

---

## Networking

- IPv4 and IPv6 are built in.
- Netfilter and nf_tables are available; nf_tables is a module.
- WireGuard is a module.
- Bridge, VLAN, bonding, MACVLAN, veth, TUN, and TAP are modules.
- BPF and XDP are enabled for high-performance networking and observability.
- Default TCP congestion control is `cubic`; BBR is available as a module
  (`sysctl net.ipv4.tcp_congestion_control=bbr` after loading `tcp_bbr`).

Firewall tooling (nftables, iptables-nft) is a userspace concern; the kernel
provides the hooks.

---

## Graphics, audio, and input

- DRM core is built in; framebuffer console and EFI framebuffer work without
  KMS drivers, which makes early boot and recovery easier.
- AMDGPU, i915, Xe, Nouveau, virtio-gpu, QXL, Bochs, and vmwgfx are modules.
- For NVIDIA proprietary drivers, use `nvidia-dkms` against `visnux-headers`.
- ALSA is modular. HDA Intel, USB audio, and SOF are available.
- HID, USB, and input core are built in, so basic keyboards and mice work
  before modules load.

If the console is blank but the system boots, try a different KMS driver or add
`nomodeset` temporarily, then install the correct driver.

---

## Debugging and tracing

This kernel is deliberately observability-friendly:

| Facility | State |
|---|---|
| `DEBUG_INFO` (DWARF5) | on |
| BTF | on |
| `KALLSYMS_ALL` | on |
| `DEBUG_FS` | on |
| ftrace / function tracer / dynamic ftrace | on |
| kprobes / uprobes | on |
| perf events / BPF events | on |
| `IKCONFIG` + `/proc/config.gz` | on |
| kexec / kexec_file / crash dump | on |
| Magic SysRq | on |
| `/proc/kcore` | on |

Examples:

```sh
zcat /proc/config.gz | grep CONFIG_PREEMPT
sudo perf top
sudo bpftool prog list
sudo cat /sys/kernel/debug/tracing/available_tracers
```

Because `CONFIG_DEBUG_INFO=y`, `vmlinux` is large. Strip or package debug info
separately if disk space matters. `scripts/faddr2line`, `gdb`, `pahole`, and
`drgn` all work with this build.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Boot drops to emergency shell, root not found | Missing storage driver in initramfs | Rebuild with the packaged preset (`-S autodetect`) or add the module |
| Black screen after bootloader | Missing/incorrect KMS driver | Boot with `nomodeset`, install the right driver |
| `Unknown symbol` when loading a module | Module built for a different kernel | Rebuild against `visnux-headers`; check `uname -r` |
| DKMS build fails | Headers missing or mismatched | Install `visnux-headers`, check `/lib/modules/$(uname -r)/build` |
| `module verification failed` | Unsigned third-party module | Expected; kernel taints but loads. Sign it or ignore |
| zswap not used | Disabled at runtime or no swap | Check `/sys/module/zswap/parameters/enabled`, enable swap |
| High memory pressure | zram/zswap misconfigured | Tune `vm.swappiness`, zram size, zswap pool |
| Hibernation fails | No resume device / swap too small | Add `resume=UUID=…`, ensure swap ≥ RAM |
| Wi-Fi missing | Firmware not installed | Install `linux-firmware`; driver is likely modular |
| NVIDIA driver fails after update | DKMS not rebuilt | `sudo dkms autoinstall`, reboot |
| Secure Boot refuses to boot | Kernel not signed for MOK | Enrol/sign the kernel or disable Secure Boot |

Detailed diagnostics are in [`docs/troubleshooting.md`](docs/troubleshooting.md).

---

## Repository layout

```text
visnux-kernel/
├── .config                  # committed kernel configuration (the product)
├── Makefile                 # top-level Kbuild entry point
├── Kconfig, Kbuild          # configuration and build glue
├── README                   # upstream Linux README
├── README.md                # this document
├── docs/                    # Visnux documentation set
│   ├── README.md            # documentation index
│   ├── architecture.md      # source tree and boot flow
│   ├── configuration.md     # configuration policy and symbol reference
│   ├── building.md          # build, cross-compile, package
│   ├── installation.md      # install, boot, initramfs, DKMS
│   ├── features.md          # feature deep dive
│   ├── tuning.md            # runtime tuning recipes
│   ├── security.md          # hardening and threat model
│   ├── troubleshooting.md   # diagnosis and recovery
│   ├── faq.md               # frequently asked questions
│   └── reference.md         # command and path cheat sheet
├── arch/x86/                # x86 architecture code and boot stub
├── block/                   # block layer and I/O schedulers
├── certs/                   # build-time signing keys
├── crypto/                  # cryptographic algorithms
├── Documentation/           # upstream documentation (11,299 files)
├── drivers/                 # device drivers
├── fs/                      # filesystems
├── include/                 # headers
├── init/                    # `start_kernel()` and early init
├── io_uring/                # io_uring
├── ipc/                     # SysV IPC, POSIX message queues
├── kernel/                  # scheduler, time, tracing, cgroups, BPF
├── lib/                     # generic library code
├── LICENSES/                # licence texts
├── mm/                      # memory management
├── net/                     # networking stack
├── rust/                    # Rust-for-Linux support (not enabled)
├── samples/                 # example code
├── scripts/                 # build, config, and analysis scripts
├── security/                # LSMs and integrity
├── sound/                   # ALSA
├── tools/                   # perf, BPF, selftests, etc.
├── usr/                     # initramfs generation
└── virt/                    # KVM
```

### Where to look for what

| Question | Start here |
|---|---|
| How does the kernel boot? | [`docs/architecture.md`](docs/architecture.md), `init/main.c` |
| How is the config chosen? | [`docs/configuration.md`](docs/configuration.md), `.config` |
| How do I build/package it? | [`docs/building.md`](docs/building.md) |
| How do I install/boot it? | [`docs/installation.md`](docs/installation.md) |
| What does feature X do? | [`docs/features.md`](docs/features.md) |
| How do I tune it? | [`docs/tuning.md`](docs/tuning.md) |
| Is it secure? | [`docs/security.md`](docs/security.md) |
| It broke — help | [`docs/troubleshooting.md`](docs/troubleshooting.md) |
| Command cheat sheet | [`docs/reference.md`](docs/reference.md) |

---

## Documentation map

| Document | What it covers |
|---|---|
| [`docs/README.md`](docs/README.md) | Index and reading paths |
| [`docs/architecture.md`](docs/architecture.md) | Tree tour, boot flow, build flow, module flow |
| [`docs/configuration.md`](docs/configuration.md) | Configuration philosophy, symbol tables, how to change safely |
| [`docs/building.md`](docs/building.md) | Dependencies, native/cross builds, packaging, reproducibility |
| [`docs/installation.md`](docs/installation.md) | Install paths, initramfs, bootloaders, Secure Boot, DKMS, rollback |
| [`docs/features.md`](docs/features.md) | Scheduling, memory, I/O, filesystems, networking, security, debugging |
| [`docs/tuning.md`](docs/tuning.md) | zswap, zram, MGLRU, DAMON, BFQ, PSI, sched_ext, governor, sysctl |
| [`docs/security.md`](docs/security.md) | Hardening, LSM selection, module signing, lockdown, mitigations |
| [`docs/troubleshooting.md`](docs/troubleshooting.md) | Symptom → cause → fix, recovery, diagnostics |
| [`docs/faq.md`](docs/faq.md) | Common questions and misconceptions |
| [`docs/reference.md`](docs/reference.md) | Paths, commands, config symbols, sysfs/proc locations |

Upstream documentation remains in [`Documentation/`](Documentation/) and is built
with:

```sh
make htmldocs
# output: Documentation/output/
```

---

## Versioning and release process

The kernel version comes from the top-level `Makefile`:

```make
VERSION = 7
PATCHLEVEL = 2
SUBLEVEL = 15
EXTRAVERSION =
NAME = Baby Opossum Posse
```

The Visnux release string is added through `.config`:

```ini
CONFIG_LOCALVERSION="visnux"
CONFIG_LOCALVERSION_AUTO=y
```

A Visnux kernel release follows these steps:

1. **Import upstream.** Start from the matching vanilla `linux-7.2.15` tree.
2. **Apply configuration.** Commit the intended `.config` (no source patches).
3. **Build.** `make olddefconfig && make -j"$(nproc)" bzImage modules`.
4. **Test.** Boot on real hardware and in a VM; exercise storage, networking,
   graphics, suspend, and DKMS modules.
5. **Package.** Produce the split `visnux` / `visnux-headers` packages with the
   mkinitcpio preset and install paths described above.
6. **Publish.** Push to ELOV and update the installer/ISO.
7. **Document.** Update this README and `docs/` if behaviour changed.

### Updating the kernel version

When moving to a new upstream release:

```sh
# 1. Replace the source tree with the new vanilla release, keeping this repo's
#    README.md, docs/, and .config.
# 2. Resolve the config against the new Kconfig:
make olddefconfig
# 3. Review every symbol that changed:
git diff -- .config
# 4. Build and test before committing.
```

Never hand-edit generated files; edit `.config` with `scripts/config` or a
configuration front-end, then run `make olddefconfig`.

---

## Relationship to other Visnux repositories

| Repository | Role |
|---|---|
| **`visnux-kernel`** (this one) | Full kernel source + committed configuration |
| `linux-visnux` | Lean, tuned kernel configuration used for shipped `-visnux` packages |
| `ELOV` | Pacman repository with built `visnux` and `visnux-headers` packages |
| `visnux-testing` | Installer and ISO build |
| `visbranch` | PKGBUILD-backed pacman shim used by the project |

### Two configuration profiles

The Visnux project maintains two related profiles:

| | **This tree (`visnux-kernel`)** | **Shipped lean profile** |
|---|---|---|
| Purpose | Full-featured reference build | Small, fast, boot-focused build |
| `CONFIG_LOCALVERSION` | `"visnux"` | `"-visnux"` |
| Release string | `7.2.15visnux` (+ `-g<sha>`) | `7.2.15-visnux` |
| Active symbols | 9,752 | ≈ 2,083 |
| Built-in / modules | 3,224 / 6,314 | lean, boot-critical built in |
| Typical use | Development, debugging, hardware coverage | Installed desktop systems |

If you build **this** tree unmodified, you get the full-featured profile. If you
want the exact configuration of the published packages, use the `.config` from
`linux-visnux`. Both are valid Visnux kernels; they optimise for different
things.

> **Note:** The package metadata in ELOV (`visnux 7.2.15-1`) and the mkinitcpio
> preset (`/boot/vmlinuz-visnux`, modules under `7.2.15-visnux`) describe the
> lean profile. Keep the release string and module directory consistent with the
> profile you actually build.

---

## Contributing

### Documentation

Documentation improvements are welcome and low-risk. Keep claims verifiable:
if you change a number, a path, or a symbol, cite the file it comes from
(`.config`, `Makefile`, package metadata, or a command’s output).

### Configuration changes

`.config` **is** the product. Treat changes as code:

1. Edit with `./scripts/config --enable FOO` / `--module BAR`, not by hand.
2. Run `make olddefconfig` and confirm the symbol stuck — Kconfig silently drops
   options whose dependencies are unmet.
3. Record the reason in the commit message.
4. Rebuild and boot before merging.
5. Update the tables in this README and `docs/configuration.md` if a claim
   changed.

### Source changes

This tree is vanilla. If you need a behavioural change, prefer a configuration
change. If a patch is genuinely required, keep it out of this repository’s
“vanilla” promise or document it explicitly.

### Reporting issues

Include:

```sh
uname -a
cat /proc/version
zcat /proc/config.gz | head
dmesg
lspci -k
```

and the exact steps to reproduce. For upstream kernel bugs, see
[`Documentation/admin-guide/reporting-issues.rst`](Documentation/admin-guide/reporting-issues.rst).

### AI-assisted contributions

If you use an AI coding assistant, read
[`Documentation/process/coding-assistants.rst`](Documentation/process/coding-assistants.rst)
first. It covers licensing, attribution, and the Developer Certificate of
Origin.

---

## FAQ

**Is Visnux a Linux distribution?**
Visnux is a Linux-based desktop OS; this repository is its kernel. The distro
also ships an installer, ISO, package repository, and userspace.

**Is this Zen, CachyOS, or XanMod?**
No. The source is vanilla Linux 7.2.15. The differences are configuration and
packaging, not patches.

**Why are there so many modules?**
This tree is the full-featured profile: it aims to boot on a wide range of
hardware and support development and debugging. The shipped lean profile is much
smaller.

**Will it work on my laptop?**
Most likely. Storage, USB, input, and EFI support are built in; Wi-Fi, GPU, and
audio are modules. Install `linux-firmware` and the matching GPU driver.

**Does it support Secure Boot?**
Not out of the box — the image is not signed for your firmware. You can sign it
with your own MOK key. See [`docs/security.md`](docs/security.md).

**Does it support hibernation?**
Yes (`CONFIG_HIBERNATION=y`), provided swap is at least as large as RAM and the
`resume=` parameter is set.

**Can I use it on Arch/Artix?**
Yes. The packaging targets Arch-based systems; the kernel itself is generic
x86_64.

**How do I get the exact release string?**
`make kernelrelease`, or `uname -r` on a running system.

**Where is the configuration?**
`.config` at the repository root, and `/proc/config.gz` on a running kernel
(`CONFIG_IKCONFIG_PROC=y`).

**Can I disable a feature without rebuilding?**
Some, yes: preemption model, `nohz_full`, mitigations, zswap, LSMs, and module
parameters. Others require a rebuild.

More in [`docs/faq.md`](docs/faq.md).

---

## Licence and credits

The Linux kernel is licensed under **GPL-2.0 only**. See [`COPYING`](COPYING)
and [`LICENSES/`](LICENSES/).

- Upstream kernel: <https://kernel.org>, <https://git.kernel.org/>
- Upstream documentation: <https://docs.kernel.org/>
- Visnux: <https://visnux.org>
- Kernel repository: <https://github.com/realv1sta/visnux-kernel>
- Package repository: <https://github.com/realv1sta/ELOV>

All upstream copyrights and attribution remain with their respective authors.
Visnux-specific configuration and documentation are maintained by the Visnux
project.

---

*This documentation describes Linux 7.2.15 with the configuration committed in
this repository. If the configuration changes, the tables and claims here must be
updated with it.*
