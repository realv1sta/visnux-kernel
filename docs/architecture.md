# Architecture and source tree

This document explains how the Linux 7.2.15 tree in this repository is organised,
how a build turns it into a kernel, and how that kernel boots. It is a map, not a
replacement for the in-tree documentation under [`../Documentation/`](../Documentation/).

## 1. What “the kernel” is here

The repository contains:

- **the upstream source tree** — C, assembly, Rust scaffolding, Kconfig, Kbuild;
- **the configuration** — `.config`, committed;
- **the build system** — `Makefile`, `scripts/`, `Kbuild` files throughout;
- **documentation** — upstream `Documentation/` plus this `docs/` set.

There are no Visnux source patches. Behaviour comes from `.config` and from how
the result is packaged.

## 2. Top-level directory tour

| Directory | Responsibility | Notable contents |
|---|---|---|
| `arch/` | Architecture-specific code | `arch/x86/boot/` (boot stub, `bzImage`), `arch/x86/kernel/`, `arch/x86/mm/` |
| `block/` | Block layer | I/O schedulers (`bfq-iosched.c`, `mq-deadline.c`, `kyber-iosched.c`), `blk-cgroup.c` |
| `certs/` | Build-time keys | `signing_key.pem` generated during build |
| `crypto/` | Cryptographic algorithms | AES, SHA, BLAKE2, zstd, LZ4, CRC |
| `Documentation/` | Upstream docs | `admin-guide/`, `process/`, `filesystems/`, `mm/`, `scheduler/` |
| `drivers/` | Device drivers | `gpu/`, `net/`, `nvme/`, `ata/`, `virtio/`, `sound/`, `usb/` |
| `fs/` | Filesystems | `ext4/`, `btrfs/`, `xfs/`, `f2fs/`, `fuse/`, `overlayfs/`, `squashfs/` |
| `include/` | Headers | `linux/`, `uapi/`, `asm-generic/`, generated headers |
| `init/` | Early init | `main.c` (`start_kernel()`), `initramfs.c`, `do_mounts.c` |
| `io_uring/` | Async I/O | io_uring implementation |
| `ipc/` | Inter-process communication | SysV IPC, POSIX mqueues |
| `kernel/` | Core kernel | `sched/`, `time/`, `trace/`, `bpf/`, `cgroup/`, `locking/` |
| `lib/` | Generic libraries | `string.c`, `rbtree.c`, `zstd/`, `lz4/`, `crc/` |
| `LICENSES/` | Licence texts | SPDX licence identifiers |
| `mm/` | Memory management | `page_alloc.c`, `vmscan.c` (MGLRU), `zswap.c`, `damon/`, `ksm.c` |
| `net/` | Networking | `ipv4/`, `ipv6/`, `netfilter/`, `sched/`, `wireguard/`, `core/` |
| `rust/` | Rust-for-Linux | `kernel/`, `macros/`, `alloc/` (Rust not enabled here) |
| `samples/` | Examples | `bpf/`, `kprobes/`, `rust/` |
| `scripts/` | Build and analysis | `config`, `kconfig/`, `setlocalversion`, `faddr2line` |
| `security/` | Security | `selinux/`, `apparmor/`, `smack/`, `tomoyo/`, `landlock/`, `integrity/` |
| `sound/` | Audio | `core/`, `pci/hda/`, `usb/`, `soc/` |
| `tools/` | Userspace tools | `perf/`, `bpf/`, `testing/selftests/`, `objtool/` |
| `usr/` | initramfs generation | `gen_init_cpio`, `initramfs_data.cpio` |
| `virt/` | Virtualisation | `kvm/` |

## 3. Build architecture

### 3.1 Kconfig → `.config`

Kconfig files describe options and dependencies. A configuration front-end
(`menuconfig`, `olddefconfig`, `scripts/config`) writes `.config`. The build
then generates:

- `include/config/auto.conf` — resolved symbol values for make;
- `include/generated/autoconf.h` — `#define`s for C;
- `include/config/kernel.release` — the release string.

Kconfig **silently drops** a requested symbol whose dependencies are unmet. That
is why every configuration change must be followed by `make olddefconfig` and a
check of the resulting `.config`.

### 3.2 Kbuild

`Makefile` drives recursive `make` over subdirectories. Each directory has a
`Kbuild` or `Makefile` listing objects:

```make
obj-y += foo.o        # built into vmlinux
obj-m += bar.o        # built as a module
obj-$(CONFIG_X) += x.o
```

`vmlinux` is linked from `built-in.a` archives, then:

```text
vmlinux
  └── arch/x86/boot/compressed/vmlinux.bin.gz (zstd here)
        └── arch/x86/boot/bzImage
```

With `CONFIG_KERNEL_ZSTD=y`, the compressed payload is Zstandard-compressed.
The boot stub decompresses it into memory.

### 3.3 Modules

Modules are linked as `*.ko`, optionally stripped and compressed
(`CONFIG_MODULE_COMPRESS_ZSTD=y`), signed (`CONFIG_MODULE_SIG_ALL=y`), and
installed under `/usr/lib/modules/<release>/kernel/`. `depmod` generates
`modules.dep`, `modules.alias`, `modules.builtin`, and related indexes.

### 3.4 Build artifacts

| Artifact | Meaning |
|---|---|
| `vmlinux` | Uncompressed ELF with symbols |
| `arch/x86/boot/bzImage` | Bootable compressed image |
| `System.map` | Symbol address map |
| `Module.symvers` | Exported symbol versions (needed by out-of-tree modules) |
| `modules.order` | Link/installation order |
| `include/generated/` | Generated headers |
| `.config` | Input configuration (never generated in this repo — committed) |

## 4. Boot flow (x86_64)

This is the path from firmware to userspace for a Visnux kernel.

```text
UEFI/BIOS
  │
  ├─ UEFI: EFI stub in bzImage (CONFIG_EFI_STUB=y)
  │    or bootloader (GRUB/systemd-boot) loads bzImage
  │
  ▼
Boot stub: arch/x86/boot/header.S, main.c, compressed/head_64.S
  │  • set up protected/long mode
  │  • decompress payload (zstd)
  │
  ▼
arch/x86/kernel/head_64.S
  │  • build initial page tables
  │  • enter start_kernel()
  │
  ▼
init/main.c: start_kernel()
  │  • trap/IRQ setup, timekeeping, scheduler init
  │  • mm_init(), sched_init(), rcu_init()
  │  • parse early params, ACPI, SMP bring-up
  │  • rest_init() → kernel_init() (PID 1)
  │
  ▼
init/main.c: kernel_init()
  │  • do_basic_setup(): driver initcalls, subsystems
  │  • prepare_namespace(): mount root (initramfs first)
  │
  ▼
initramfs (/init) or root filesystem
  │  • mount real root
  │  • switch_root
  │
  ▼
/sbin/init (systemd, OpenRC, runit, s6, …)
```

### 4.1 Why built-in drivers matter

`CONFIG_VIRTIO_BLK=y`, `CONFIG_SATA_AHCI=y`, `CONFIG_BLK_DEV_SD=y`,
`CONFIG_EXT4_FS=y`, and `CONFIG_BTRFS_FS=y` mean the kernel can reach a root
filesystem without loading a module. That reduces dependence on initramfs module
ordering. `NVMe` is `=m` in this profile, so an initramfs must include it.

### 4.2 initramfs

The packaged preset disables `autodetect`, so the initramfs contains the full
module set and boots on hardware the build machine did not have. The trade-off
is a larger image and slightly slower early boot.

### 4.3 Kernel command line

Parameters are parsed early in `start_kernel()` and by subsystem `early_param`
and `__setup` handlers. Useful groups:

| Group | Examples |
|---|---|
| Root/init | `root=`, `ro`, `rw`, `init=`, `rdinit=` |
| Console | `console=tty0`, `console=ttyS0,115200` |
| Preemption | `preempt=full|voluntary|none` |
| Tickless | `nohz_full=`, `rcu_nocbs=` |
| Memory | `zswap.enabled=`, `zswap.compressor=`, `mem=`, `hugepages=` |
| Security | `lsm=`, `lockdown=`, `mitigations=`, `module.sig_enforce=` |
| Debug | `loglevel=`, `earlyprintk=`, `ignore_loglevel`, `sysrq_always_enabled` |

The authoritative list is
[`../Documentation/admin-guide/kernel-parameters.rst`](../Documentation/admin-guide/kernel-parameters.rst).

## 5. Runtime architecture

```text
userspace
  │  syscalls (arch/x86/entry/)
  ▼
VFS (fs/) ── filesystems (ext4, btrfs, xfs, …)
  │
block layer (block/) ── I/O schedulers (BFQ, mq-deadline, kyber)
  │
drivers (drivers/) ── NVMe, AHCI, virtio, USB, …
  │
hardware
```

Cross-cutting subsystems:

- **Scheduler** — `kernel/sched/` (EEVDF, `sched_ext`, core scheduling, uclamp)
- **Memory** — `mm/` (page allocator, MGLRU, zswap, zram via `drivers/block/zram/`, DAMON, KSM)
- **Networking** — `net/` (socket layer, TCP/IP, netfilter, qdiscs)
- **Security** — `security/` (LSMs), `crypto/` (algorithms)
- **Tracing** — `kernel/trace/`, `kernel/bpf/`, `tools/perf/`

## 6. Configuration profile in this tree

| Metric | Value |
|---|---|
| Active symbols | 9,752 |
| Built-in (`=y`) | 3,224 |
| Modules (`=m`) | 6,314 |
| Disabled | 805 |
| Source files | 94,875 |
| Directories | 6,204 |
| Documentation files | 11,299 |

This is a **full-featured** profile. It prioritises hardware coverage,
observability, and development convenience over minimal image size. See
[configuration.md](configuration.md) for the policy and the symbol-level detail.

## 7. Where to go next

- Build it: [building.md](building.md)
- Install it: [installation.md](installation.md)
- Understand features: [features.md](features.md)
- Tune it: [tuning.md](tuning.md)
