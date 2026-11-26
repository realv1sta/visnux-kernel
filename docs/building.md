# Building the Visnux kernel

This guide covers host preparation, native and cross builds, packaging, module
signing, and reproducibility. It assumes a checkout of this repository at
`~/visnux-kernel`.

## 1. What you are building

From this tree and its `.config` you get:

| Output | Path |
|---|---|
| Bootable image | `arch/x86/boot/bzImage` |
| Uncompressed kernel | `vmlinux` |
| Symbol map | `System.map` |
| Modules | `*.ko` throughout the tree |
| Release string | `include/config/kernel.release` |

With the committed `.config`, the release is `7.2.15visnux` (plus `-g<sha>`
inside git). The image is Zstandard-compressed (`CONFIG_KERNEL_ZSTD=y`).

## 2. Host requirements

### 2.1 Hardware

| Resource | Recommendation |
|---|---|
| CPU | 4+ cores (more is better with `-j`) |
| RAM | 8 GB minimum; 16 GB+ for `-j$(nproc)` with debug info |
| Disk | 25–40 GB free for a full build with DWARF5 + BTF |
| Architecture | `x86_64` host for a native build |

Debug info (`CONFIG_DEBUG_INFO=y`, DWARF5, BTF) makes the build significantly
larger than a stripped kernel. Plan for it.

### 2.2 Software (Arch-based)

```sh
sudo pacman -S --needed \
  base-devel bc cpio gettext libelf pahole perl tar xz zstd
```

Optional:

| Package | Why |
|---|---|
| `openssl` | Generate/use module signing keys |
| `ccache` | Cache compilation across rebuilds |
| `clang` `lld` `llvm` | Build with LLVM instead of GCC |
| `rust` `rust-src` `bindgen` | Only if you enable `CONFIG_RUST` |
| `python-sphinx` `graphviz` `imagemagick` | `make htmldocs` |
| `qemu-full` `virt-manager` | Test the kernel in a VM |
| `ncurses` | `make menuconfig` |

The authoritative list for the kernel version is
[`../Documentation/process/changes.rst`](../Documentation/process/changes.rst).

### 2.3 Verify the toolchain

```sh
gcc --version
ld --version
make --version
pahole --version
```

The committed config records GCC 16.2.1 and binutils 2.47. Other recent versions
usually work; the recorded values matter only for reproducing a specific build.

## 3. Native build

### 3.1 Configure

```sh
cd ~/visnux-kernel
make olddefconfig
```

`olddefconfig` resolves the committed `.config` against the tree’s Kconfig
without prompting. It preserves explicit values where dependencies allow and
fills new symbols with defaults.

Check the result:

```sh
grep -E '^CONFIG_LOCALVERSION|^CONFIG_PREEMPT=|^CONFIG_HZ=' .config
make kernelrelease
```

### 3.2 Build

```sh
make -j"$(nproc)" bzImage modules
```

Or split it:

```sh
make -j"$(nproc)" bzImage    # image
make -j"$(nproc)" modules    # modules
```

### 3.3 Build targets

| Target | Result |
|---|---|
| `bzImage` | `arch/x86/boot/bzImage` |
| `vmlinux` | Uncompressed ELF |
| `modules` | All `*.ko` |
| `modules_install` | Install modules under `INSTALL_MOD_PATH` |
| `headers_install` | UAPI headers under `INSTALL_HDR_PATH` |
| `kernelrelease` | Release string |
| `kernelversion` | Version without local suffix |
| `localversion` | Local suffix only |
| `olddefconfig` | Resolve config non-interactively |
| `menuconfig` | ncurses configuration UI |
| `nconfig` | alternative UI |
| `xconfig` | Qt configuration UI |
| `defconfig` | Architecture default (discards this config) |
| `clean` | Remove most build output, keep `.config` |
| `mrproper` | Remove build output **and** `.config` |
| `distclean` | `mrproper` plus editor/backup files |
| `htmldocs` | Build `Documentation/output/` |
| `coccicheck` | Run Coccinelle checks |
| `checkstack` | Stack usage report |
| `modules_prepare` | Prepare tree for external modules |

> `make mrproper` deletes `.config`. In this repository that is destructive to
> the product. Recover with `git checkout -- .config`.

### 3.4 Parallelism and memory

`-j$(nproc)` is a good default. With very high core counts and debug info, the
link step (`vmlinux`) can be memory-hungry; reduce `-j` if the OOM killer fires.

### 3.5 ccache

```sh
export CC="ccache gcc"
make -j"$(nproc)" bzImage modules
```

or configure `ccache` globally. Incremental rebuilds after a config tweak are
much faster.

## 4. Building with LLVM/Clang

```sh
make LLVM=1 olddefconfig
make LLVM=1 -j"$(nproc)" bzImage modules
```

Requirements: `clang`, `lld`, `llvm`, `llvm-lib`, and for BTF a recent `pahole`.
If the committed `.config` was produced with GCC, `olddefconfig` may adjust
compiler-capability symbols; review `git diff -- .config` afterwards.

## 5. Cross-compilation

The committed config is for `x86_64`. To build for another architecture you need
a matching config and cross toolchain.

Example for `aarch64`:

```sh
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-
make defconfig                 # start from the arch default
# apply the options you need, then:
make -j"$(nproc)" Image modules
```

Notes:

- The `.config` in this repository is **not** portable across architectures;
  many symbols are x86-specific.
- Use a cross `pahole`/`libelf` if you need BTF.
- `CROSS_COMPILE` must end with a dash and name a toolchain prefix present in
  `PATH`.

## 6. Installing modules

```sh
sudo make INSTALL_MOD_PATH=/usr modules_install
```

This installs to `/usr/lib/modules/<release>/`. `INSTALL_MOD_PATH` is a prefix:
`/usr` means modules land in `/usr/lib/modules`, not `/usr/usr/lib/modules`.

Useful variables:

| Variable | Effect |
|---|---|
| `INSTALL_MOD_PATH` | Root prefix for module installation |
| `INSTALL_MOD_STRIP` | Strip modules (`=1` uses `strip`) |
| `INSTALL_MOD_DIR` | Subdirectory under `kernel/` |
| `DEPMOD` | `depmod` binary to use |
| `STRIP` | Strip binary |

Then generate dependency indexes if your packaging does not:

```sh
sudo depmod -a "$(make kernelrelease)"
```

## 7. Installing the image

```sh
sudo install -m644 arch/x86/boot/bzImage /boot/vmlinuz-visnux
sudo install -m644 System.map /boot/System.map-visnux
sudo install -m644 .config /boot/config-visnux
sudo mkinitcpio -P
```

If you do not use the packaged preset, create one (see
[installation.md](installation.md#initramfs)).

## 8. Packaging

The shipped packages are split:

| Package | Contents |
|---|---|
| `visnux` | `/boot/vmlinuz-visnux`, `/usr/lib/modules/7.2.15-visnux/`, mkinitcpio preset |
| `visnux-headers` | `/usr/src/linux-7.2.15-visnux/`, build symlink |

Recorded metadata from the published `visnux 7.2.15-1` package:

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

A representative split `PKGBUILD` (illustrative — the real packaging lives with
the distribution, not in this documentation-only tree):

```bash
pkgbase=visnux
pkgname=(visnux visnux-headers)
pkgver=7.2.15
pkgrel=1
arch=(x86_64)
url=https://visnux.org
license=(GPL2)
depends=(coreutils kmod initramfs)
makedepends=(bc cpio gettext libelf pahole perl tar xz)
options=(!strip)

build() {
  cd "$srcdir/linux-$pkgver"
  make olddefconfig
  make -j"$(nproc)" bzImage modules
}

package_visnux() {
  cd "$srcdir/linux-$pkgver"
  make INSTALL_MOD_PATH="$pkgdir/usr" modules_install
  install -Dm644 arch/x86/boot/bzImage "$pkgdir/boot/vmlinuz-visnux"
  install -Dm644 System.map "$pkgdir/boot/System.map-visnux"
  install -Dm644 .config "$pkgdir/boot/config-visnux"
  install -Dm644 /path/to/visnux.preset \
    "$pkgdir/etc/mkinitcpio.d/visnux.preset"
}

package_visnux-headers() {
  cd "$srcdir/linux-$pkgver"
  make INSTALL_HDR_PATH="$pkgdir/usr" headers_install
  install -d "$pkgdir/usr/src/linux-$pkgver-visnux"
  # copy the configured tree and generated headers, then:
  # ln -s /usr/src/linux-$pkgver-visnux \
  #   "$pkgdir/usr/lib/modules/$pkgver-visnux/build"
}
```

Do not copy this verbatim into a real package without matching the project’s
release string and install paths.

## 9. Module signing

`CONFIG_MODULE_SIG=y` and `CONFIG_MODULE_SIG_ALL=y` sign modules at build time
with `certs/signing_key.pem`. If the key does not exist, the build generates one.

Inspect and verify:

```sh
ls -l certs/signing_key.pem certs/signing_key.x509
modinfo -F sig_id <module>.ko
```

Enforcement is off (`CONFIG_MODULE_SIG_FORCE` unset), so unsigned modules still
load. To enforce at runtime:

```text
module.sig_enforce=1
```

Only do this if every module you need — including DKMS modules — is signed with a
key in the kernel keyring.

For reproducible packaging, provide your own key and certificate:

```sh
openssl req -new -x509 -newkey rsa:4096 -nodes -days 3650 \
  -subj "/CN=Visnux module signing/" \
  -keyout certs/signing_key.pem -out certs/signing_key.x509
```

Keep the private key secret and out of the repository.

## 10. Reproducibility

Kernel builds embed metadata. For comparable builds:

```sh
export KBUILD_BUILD_USER=visnux
export KBUILD_BUILD_HOST=visnux
export KBUILD_BUILD_TIMESTAMP="$(date -u -d @0)"
export KBUILD_BUILD_VERSION=1
```

Also:

- use the same compiler/binutils versions;
- build from the same commit;
- avoid `CONFIG_LOCALVERSION_AUTO` if you need a stable release string;
- build with the same `KBUILD_*` variables;
- compare `vmlinux` with `diffoscope` or `sha256sum` after stripping timestamps.

`CONFIG_DEBUG_INFO_BTF` embeds type information derived from the build; it is
deterministic given the same toolchain and source.

## 11. Build troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `bc: command not found` | Missing `bc` | Install `bc` |
| `flex`/`bison` errors | Missing parser generators | Install `flex bison` |
| `libelf` / `pahole` errors | BTF generation | Install `libelf pahole` |
| `openssl` errors during signing | Missing OpenSSL | Install `openssl` |
| `No rule to make target 'debian/certs/...'` | `CONFIG_SYSTEM_TRUSTED_KEYS` points at a missing file | Set it to `""` or provide the file |
| `Cannot use CONFIG_CC_STACKPROTECTOR_STRONG` | Compiler lacks support | Use a newer compiler or disable the option |
| `BTF: .tmp_vmlinux.btf: pahole: not found` | `pahole` missing | Install `pahole` |
| `Kernel configuration is invalid` | Unresolved symbols | `make olddefconfig` |
| OOM during link | Too much parallelism | Reduce `-j` |
| `No space left on device` | Debug info is large | Free space or disable `DEBUG_INFO` |

## 12. Testing the build

### 12.1 In a VM

```sh
qemu-system-x86_64 \
  -kernel arch/x86/boot/bzImage \
  -append "root=/dev/vda1 rw console=ttyS0" \
  -drive file=disk.img,if=virtio \
  -nographic -m 4G -smp 4
```

`virtio-blk` and `virtio-pci` are built in, so this works with a minimal
initramfs. `virtio-net` and `virtio-gpu` are modules.

### 12.2 On hardware

Install alongside your current kernel, keep the old entry, and boot the new one
once. Verify:

```sh
uname -r
zcat /proc/config.gz | grep CONFIG_PREEMPT
dmesg --level=err,warn
systemctl --failed
```

### 12.3 Sanity checks

```sh
# Modules load
sudo modprobe zram && lsmod | grep zram

# zswap active
cat /sys/module/zswap/parameters/enabled

# MGLRU
cat /sys/kernel/mm/lru_gen/enabled

# PSI
cat /proc/pressure/cpu

# Scheduler
cat /sys/kernel/debug/sched/preempt
```

## 13. Cleaning up

```sh
make clean          # keep .config
make mrproper       # deletes .config — avoid in this repo
git status          # confirm only intended changes
```

If build artifacts are accidentally tracked, do not commit them; `.gitignore`
already excludes standard outputs.

## 14. Next steps

- Install and boot: [installation.md](installation.md)
- Understand the options: [configuration.md](configuration.md)
- Tune the result: [tuning.md](tuning.md)
