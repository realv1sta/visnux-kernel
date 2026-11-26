# FAQ

Short answers to the questions that come up most often. Each answer links to the
detailed document where relevant.

## General

### What is Visnux?

A Linux-based desktop operating system aimed at gaming, development, and everyday
use. This repository is its kernel.

### What is this repository, exactly?

The complete vanilla Linux 7.2.15 source tree plus the committed `.config` used to
build a Visnux kernel. No Visnux source patches. See
[architecture.md](architecture.md).

### Is Visnux a distribution?

Visnux is the operating system/distribution; `visnux-kernel` is the kernel source
and configuration. The project also ships an installer, ISO, and package
repository.

### Is this Zen, CachyOS, XanMod, or Liquorix?

No. Those carry out-of-tree patches. Visnux uses vanilla source and changes
behaviour through configuration and packaging only.

### Which architectures are supported?

This configuration targets `x86_64`. The source tree contains other
architectures, but the committed `.config` is x86-specific.

### What licence is it under?

GPL-2.0 only. See [`../COPYING`](../COPYING) and [`../LICENSES/`](../LICENSES/).

## Versions and naming

### What version is this?

Linux `7.2.15`, codename `Baby Opossum Posse`, with `CONFIG_LOCALVERSION="visnux"`.

### What will `uname -r` say?

- Building this tree from a tarball: `7.2.15visnux`
- Building inside this git checkout: `7.2.15visnux-g3cd079313` (auto `-g<sha>`)
- The packaged lean profile: `7.2.15-visnux`

See [configuration.md](configuration.md#release-string).

### Why does the package use `7.2.15-visnux` but this tree produces `7.2.15visnux`?

They are two configuration profiles. This tree’s `.config` uses
`CONFIG_LOCALVERSION="visnux"` with `LOCALVERSION_AUTO=y`; the shipped lean
profile uses `"-visnux"` with auto off. See the root README section
“Relationship to other Visnux repositories”.

### How do I get a stable release string for packaging?

Disable `CONFIG_LOCALVERSION_AUTO` or pass `LOCALVERSION=` explicitly to `make`.

## Building

### What do I need to build it?

A 64-bit host, a recent GCC/binutils, `bc`, `cpio`, `gettext`, `libelf`,
`pahole`, `perl`, `tar`, `xz`, and `zstd`. See
[building.md](building.md#2-host-requirements).

### How long does a build take?

Depends on the machine. A modern 8–16 core desktop typically takes tens of
minutes for a full build with DWARF5 + BTF. Incremental builds are much faster,
especially with `ccache`.

### Why is the build so large?

`CONFIG_DEBUG_INFO=y` (DWARF5), `CONFIG_DEBUG_INFO_BTF=y`, and
`CONFIG_KALLSYMS_ALL=y` add a lot of data. This is deliberate for debugging and
profiling.

### Can I build with Clang?

Yes: `make LLVM=1 olddefconfig && make LLVM=1 -j"$(nproc)" bzImage modules`.
Review config changes made by `olddefconfig`.

### Can I cross-compile?

Yes, for another architecture with the matching config and toolchain. The
committed `.config` is not portable across architectures. See
[building.md](building.md#5-cross-compilation).

### `make mrproper` deleted my `.config`. What now?

Restore it:

```sh
git checkout -- .config
```

Never run `mrproper` in this repository unless you mean it.

### The build fails with a signing/OpenSSL error. Why?

`CONFIG_MODULE_SIG_ALL=y` needs OpenSSL to generate/use the signing key. Install
`openssl`, or provide a key pair in `certs/`. See
[building.md](building.md#9-module-signing).

### Why does `pahole` matter?

`CONFIG_DEBUG_INFO_BTF=y` uses `pahole` to generate BTF. Without it, the build
fails at the BTF step.

## Installing and booting

### How do I install it?

```sh
sudo make INSTALL_MOD_PATH=/usr modules_install
sudo install -m644 arch/x86/boot/bzImage /boot/vmlinuz-visnux
sudo mkinitcpio -P
```

Then add a bootloader entry. See [installation.md](installation.md).

### Do I need an initramfs?

Usually yes, because NVMe, LUKS, LVM, and many other pieces are modular. The
packaged preset builds a full image with `-S autodetect`, so it boots on hardware
the build host did not have.

### Why does the preset disable `autodetect`?

Because the installer builds images in a chroot that only knows the install
machine. Including all modules avoids “root device not found” on different
hardware. The cost is a larger image.

### The system boots to an emergency shell. What is wrong?

Usually the root device cannot be found: missing storage module in the initramfs,
wrong `root=` UUID, or an unactivated LVM/MD/LUKS layer. See
[troubleshooting.md](troubleshooting.md#23-root-device-not-found--emergency-shell).

### Can I boot it with Secure Boot enabled?

Not out of the box. Sign the kernel and modules with your own MOK, or disable
Secure Boot. See [security.md](security.md#6-secure-boot).

### Does it support hibernation?

Yes (`CONFIG_HIBERNATION=y`), with a sufficiently large swap and `resume=`.

## Features

### Is zswap on by default?

Yes: `CONFIG_ZSWAP_DEFAULT_ON=y`, compressor `zstd`, shrinker on.

### Should I use zswap or zram?

zswap if you have a real swap device; zram if you want compressed swap in RAM.
Running both is usually redundant. See [features.md](features.md#23-zram-config_zramm).

### Is MGLRU enabled?

Yes: `CONFIG_LRU_GEN=y` and `CONFIG_LRU_GEN_ENABLED=y`.

### Is BFQ the default I/O scheduler?

BFQ is built in, but the default is chosen by the block layer per device. Set it
explicitly if you want it. See [tuning.md](tuning.md#9-io-scheduler).

### Is `sched_ext` active?

It is compiled in, but no BPF scheduler is attached by default. EEVDF remains the
default until you load one.

### Is full dynticks active?

`CONFIG_NO_HZ_FULL=y` is compiled in but inactive until you pass `nohz_full=`.
Do not include CPU 0.

### Is Rust support enabled?

No. `CONFIG_RUST` is not set, even though a Rust toolchain version is recorded.

### Is KASAN enabled?

No. The config has `DEBUG_KERNEL`, `KFENCE`, and `PAGE_POISONING`, but not the
heavy sanitizers (KASAN/UBSAN/KCSAN) or lockdep.

### Is bcachefs enabled?

No.

### Does it include proprietary drivers?

No. The kernel is GPL-2.0. Proprietary drivers such as NVIDIA are built
out-of-tree via DKMS.

## Security

### Is module signing enforced?

No. All in-tree modules are signed, but `CONFIG_MODULE_SIG_FORCE` is unset, so
unsigned third-party modules still load (and taint the kernel).

### Why not enforce module signing?

Because DKMS modules (NVIDIA, VirtualBox, ZFS, …) would stop loading unless signed
with a key in the kernel keyring. Enforcement is a deliberate opt-in
(`module.sig_enforce=1`).

### Are SELinux or AppArmor active?

Compiled, but not in the default LSM order
(`landlock,lockdown,yama,integrity,bpf`). Activate one with `lsm=` and configure
userspace first.

### Are CPU mitigations on?

Yes. `mitigations=off` is available but is a real security trade-off.

### Is KASLR on?

Yes (`CONFIG_RANDOMIZE_BASE=y`, `CONFIG_RANDOMIZE_MEMORY=y`).

### Why is `dmesg` restricted?

`CONFIG_SECURITY_DMESG_RESTRICT=y` prevents unprivileged users from reading kernel
addresses and other sensitive log data. Root can still read it.

## Modules and DKMS

### Where are the headers?

`/usr/src/linux-7.2.15-visnux/`, with `/lib/modules/7.2.15-visnux/build` pointing
at it (from `visnux-headers`).

### Why does DKMS fail after a kernel update?

The module was built for the old release string, or headers are missing. Check
`uname -r` against `/lib/modules/*/build`, then `dkms autoinstall`.

### “module verification failed” — is that fatal?

No. The module loads and the kernel is tainted. It becomes fatal only if you
enforce signatures.

### Why are modules compressed?

`CONFIG_MODULE_COMPRESS_ZSTD=y` saves disk space. The kernel and `modprobe`
handle it transparently.

### Where is the signing key?

`certs/signing_key.pem` (private) and `certs/signing_key.x509` (certificate).
Keep the private key secret.

## Performance

### Why does `preempt=full` cost throughput?

Full preemption allows more context switches and reduces batching. It is the right
trade-off for a desktop; servers may prefer `voluntary` or `none`.

### Should I disable THP?

Only if your workload prefers `madvise`. The config uses `always`. Test before
changing.

### Why is my NVMe slower with BFQ?

BFQ adds scheduling overhead. On fast NVMe, `none` is often faster. Measure with
`fio`.

### How do I reduce memory usage?

Check caches, zswap/zram, and per-cgroup limits. Do not disable MGLRU or THP
without measuring. `smem`, `slabtop`, and PSI help.

## Development

### Can I add a patch?

This tree is vanilla by design. Prefer configuration changes. If a patch is
genuinely needed, document it explicitly and consider a separate branch.

### How do I change the config?

Use `./scripts/config`, then `make olddefconfig`, verify the symbol stuck, rebuild,
boot, and update the docs. See
[configuration.md](configuration.md#15-changing-the-configuration-safely).

### How do I report a bug?

Collect `uname -a`, `/proc/config.gz`, `dmesg`, `lspci -k`, and reproduction
steps. Upstream kernel bugs go to the kernel bugzilla/mailing lists; Visnux
configuration issues go to the project repository.

### Can I use AI assistance?

Yes, but read
[`../Documentation/process/coding-assistants.rst`](../Documentation/process/coding-assistants.rst)
first. It covers licensing, attribution, and the DCO.

## Documentation

### Where is the upstream documentation?

[`../Documentation/`](../Documentation/), or online at
<https://docs.kernel.org/>.

### How do I build the HTML docs?

```sh
make htmldocs
# Documentation/output/
```

Requires Sphinx and related Python packages.

### This document is wrong / out of date. What do I do?

Check the claim against `.config` and the source, then update the document in the
same commit as the change. Documentation is expected to be verifiable, not
aspirational.
