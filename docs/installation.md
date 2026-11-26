# Installing and booting

This document covers installing a Visnux kernel built from this tree, creating an
initramfs, configuring bootloaders, DKMS, hibernation, Secure Boot, and rollback.

It assumes Arch-based userspace tools (`mkinitcpio`, `pacman`, `grub`,
`systemd-boot`). The kernel itself is distribution-agnostic.

## 1. Install paths

The published split packages install:

| Path | Package | Contents |
|---|---|---|
| `/boot/vmlinuz-visnux` | `visnux` | Kernel image |
| `/boot/System.map-visnux` | `visnux` | Symbol map (optional) |
| `/boot/config-visnux` | `visnux` | Config copy (optional) |
| `/usr/lib/modules/7.2.15-visnux/` | `visnux` | Modules and indexes |
| `/etc/mkinitcpio.d/visnux.preset` | `visnux` | initramfs preset |
| `/usr/src/linux-7.2.15-visnux/` | `visnux-headers` | Configured source tree |
| `/usr/lib/modules/7.2.15-visnux/build` | `visnux-headers` | Symlink for DKMS |

A build from **this** tree’s `.config` reports `7.2.15visnux` (plus `-g<sha>` in
git). If you install it by hand, use a consistent directory name and update the
preset and bootloader entries accordingly. The examples below use the packaged
`7.2.15-visnux` naming.

## 2. Manual installation

```sh
cd ~/visnux-kernel

# Modules
sudo make INSTALL_MOD_PATH=/usr modules_install

# Image and metadata
sudo install -m644 arch/x86/boot/bzImage /boot/vmlinuz-visnux
sudo install -m644 System.map /boot/System.map-visnux
sudo install -m644 .config /boot/config-visnux

# initramfs
sudo mkinitcpio -P
```

Verify:

```sh
ls -l /boot/vmlinuz-visnux /boot/initramfs-visnux.img
ls /usr/lib/modules/
depmod -a "$(make kernelrelease)"
```

## 3. initramfs

### 3.1 The packaged preset

`/etc/mkinitcpio.d/visnux.preset`:

```bash
ALL_config="/etc/mkinitcpio.conf"
ALL_kver="/boot/vmlinuz-visnux"

PRESETS=('default' 'fallback')

default_image="/boot/initramfs-visnux.img"
default_options="-S autodetect,kms"

fallback_image="/boot/initramfs-visnux-fallback.img"
fallback_options="-S autodetect"
```

Why `-S autodetect`:

- The installer builds images inside a chroot that only knows the machine it was
  installed from.
- Disabling `autodetect` includes **all** modules, so the image boots on hardware
  the build host never saw.
- The cost is a larger image and slightly slower early boot.

`default_options` also disables `kms`, so the console stays on the firmware
framebuffer until the real driver loads; the fallback keeps `kms`.

### 3.2 Building images

```sh
sudo mkinitcpio -P                 # all presets
sudo mkinitcpio -p linux-visnux    # one preset, if named that way
```

### 3.3 Inspecting an image

```sh
lsinitcpio /boot/initramfs-visnux.img | less
lsinitcpio -a /boot/initramfs-visnux.img
```

Confirm the drivers you need are present:

```sh
lsinitcpio /boot/initramfs-visnux.img | grep -E 'nvme|ahci|virtio|ext4|btrfs'
```

### 3.4 Minimal initramfs

If you want a small image for known hardware, use `autodetect` and list only the
needed modules in `/etc/mkinitcpio.conf`:

```bash
MODULES=(nvme ahci virtio_blk virtio_pci ext4)
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block filesystems fsck)
```

Then rebuild. Keep a full fallback image for recovery.

## 4. Bootloaders

### 4.1 GRUB

Add `/etc/grub.d/40_custom`:

```bash
menuentry "Visnux Linux" {
    linux  /boot/vmlinuz-visnux root=UUID=<uuid> rw quiet
    initrd /boot/initramfs-visnux.img
}
```

Then:

```sh
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

Use `blkid` to get the root UUID. Keep the distribution’s generated entries as a
fallback.

### 4.2 systemd-boot

Create `/boot/loader/entries/visnux.conf`:

```ini
title   Visnux Linux
linux   /vmlinuz-visnux
initrd  /initramfs-visnux.img
options root=UUID=<uuid> rw quiet
```

If `/boot` is a separate FAT partition, paths are relative to it (`/vmlinuz-visnux`).
If `/boot` is on the root filesystem, systemd-boot needs the kernel and initramfs
on the ESP; adjust accordingly.

### 4.3 EFI stub

`CONFIG_EFI_STUB=y` lets the kernel be booted directly by firmware or by a
bootloader that supports EFI applications:

```sh
sudo cp arch/x86/boot/bzImage /boot/EFI/visnux/vmlinuz.efi
```

The kernel command line can be embedded at build time with
`CONFIG_CMDLINE`/`CONFIG_CMDLINE_OVERRIDE`, but this config does not set them.
Prefer passing parameters from the bootloader.

### 4.4 rEFInd

Add a manual stanza in `refind.conf`:

```ini
menuentry "Visnux Linux" {
    loader /boot/vmlinuz-visnux
    initrd /boot/initramfs-visnux.img
    options "root=UUID=<uuid> rw quiet"
}
```

## 5. Kernel command line

A conservative desktop baseline:

```text
root=UUID=<uuid> rw quiet
```

Useful additions:

| Parameter | Effect |
|---|---|
| `loglevel=3` | Quieter console |
| `nowatchdog` | Disable watchdog (if not needed) |
| `preempt=full` | Select preemption model |
| `zswap.enabled=1 zswap.compressor=zstd` | zswap settings |
| `mitigations=auto` | Default mitigation selection |
| `lsm=...` | Select active LSMs |
| `resume=UUID=<swap>` | Hibernation resume device |
| `nohz_full=2-7 rcu_nocbs=2-7` | Full dynticks (specialist) |
| `module.sig_enforce=1` | Enforce module signatures |
| `panic=10` | Reboot 10 s after panic |
| `nomodeset` | Disable KMS (recovery only) |
| `init=/bin/bash` | Emergency shell (recovery only) |

Do not copy tuning parameters blindly. `nohz_full`, `mitigations=off`, and
`module.sig_enforce=1` have real costs.

## 6. DKMS

### 6.1 How it works

DKMS builds out-of-tree modules against `/lib/modules/$(uname -r)/build`, which
`visnux-headers` points at `/usr/src/linux-7.2.15-visnux`.

```sh
dkms status
sudo dkms autoinstall
```

### 6.2 Common modules

| Module | Package | Notes |
|---|---|---|
| NVIDIA proprietary | `nvidia-dkms` | Needs matching headers |
| VirtualBox Guest Additions | `virtualbox-guest-utils` | `vboxguest`, `vboxsf` |
| ZFS | `zfs-dkms` | Check kernel compatibility |
| VMware | `open-vm-tools-dkms` | Guest modules |
| WireGuard | `wireguard-dkms` | Only if not using in-tree `wireguard` |

### 6.3 When a DKMS build fails

1. Confirm the running kernel and headers match:

   ```sh
   uname -r
   ls -l /lib/modules/$(uname -r)/build
   ```

2. Check the build log:

   ```sh
   sudo dkms autoinstall
   cat /var/lib/dkms/<module>/<version>/build/make.log
   ```

3. Confirm the module supports Linux 7.2.15. Vendor code often lags.
4. Rebuild after every kernel update:

   ```sh
   sudo dkms autoinstall
   ```

A pacman hook normally handles this automatically for packaged DKMS modules.

## 7. Hibernation

Requirements:

- `CONFIG_HIBERNATION=y` (yes);
- a swap partition or swap file at least as large as RAM (for reliable resume);
- `resume=` pointing at the swap device;
- initramfs support (`resume` hook or equivalent).

Example:

```text
resume=UUID=<swap-uuid>
```

For a swap file, the file must be on a filesystem the kernel can resume from and
the offset must be passed (`resume_offset=`). Follow your distribution’s
hibernation guide; the kernel side is already enabled.

Test:

```sh
sudo systemctl hibernate
```

## 8. Secure Boot

The Visnux kernel image is **not signed with a key your firmware trusts**. Three
options:

1. **Disable Secure Boot** in firmware (simplest, weaker).
2. **Enrol a MOK** and sign the kernel and modules yourself:

   ```sh
   sudo mokutil --import certs/signing_key.x509
   # reboot, enrol in MOK manager
   sbsign --key certs/signing_key.pem --cert certs/signing_key.x509 \
     --output /boot/vmlinuz-visnux.signed /boot/vmlinuz-visnux
   ```

3. **Use a signed shim/bootloader** that chainloads an unsigned kernel — this
   usually defeats the point and is not recommended.

If you sign the kernel, also sign every module you need and enable enforcement
only after verifying the system boots.

See [security.md](security.md#6-secure-boot) for the full discussion.

## 9. Verifying the installation

```sh
# Running kernel
uname -r

# Config of the running kernel
zcat /proc/config.gz | grep -E 'CONFIG_PREEMPT=|CONFIG_LRU_GEN=|CONFIG_ZSWAP_DEFAULT_ON='

# Modules
lsmod | head
modinfo zram

# Boot log
dmesg --level=err,warn

# Failed services
systemctl --failed
```

Expected for the packaged lean profile:

```text
7.2.15-visnux
```

Expected for an unmodified build of this tree:

```text
7.2.15visnux-g3cd079313
```

## 10. Upgrading

1. Build or install the new kernel package.
2. Rebuild the initramfs (`mkinitcpio -P`).
3. Rebuild DKMS modules (`dkms autoinstall`).
4. Update the bootloader entry if the image name changed.
5. Reboot, keeping the previous kernel entry available.

Do not remove the old kernel until the new one has booted successfully and its
modules and DKMS modules work.

## 11. Removing / rollback

```sh
# Remove packages (Arch)
sudo pacman -Rns visnux visnux-headers

# Remove manual install
sudo rm -f /boot/vmlinuz-visnux /boot/initramfs-visnux*.img
sudo rm -rf /usr/lib/modules/7.2.15-visnux /usr/src/linux-7.2.15-visnux
sudo rm -f /etc/mkinitcpio.d/visnux.preset

# Regenerate bootloader config
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

Always keep a working kernel and initramfs installed. If a boot fails, select the
previous entry in the bootloader menu, or boot a live ISO and `chroot` to repair.

## 12. Recovery checklist

If the system does not boot:

1. At the bootloader, edit the entry and add `nomodeset` or `init=/bin/bash`.
2. Boot a live ISO, mount the root filesystem, and `arch-chroot`.
3. Check that `/boot/vmlinuz-visnux` and the initramfs exist and are non-empty.
4. Rebuild the initramfs: `mkinitcpio -P`.
5. Verify the bootloader entry paths and root UUID (`blkid`).
6. Check that the root filesystem driver is built in or present in the initramfs.
7. Review `journalctl -b -1` if the system boots far enough.

More in [troubleshooting.md](troubleshooting.md).

## 13. Next steps

- Tune the installed system: [tuning.md](tuning.md)
- Harden it: [security.md](security.md)
- Understand features: [features.md](features.md)
