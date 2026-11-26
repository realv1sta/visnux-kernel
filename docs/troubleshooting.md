# Troubleshooting

Symptom → likely cause → fix. Start with the general diagnostics, then find your
symptom.

## 1. First responders

```sh
uname -a
cat /proc/version
zcat /proc/config.gz | head -50
dmesg --level=err,warn
journalctl -b -p err
systemctl --failed
lsmod
lspci -k
```

If the system does not boot, use a live ISO and `arch-chroot`, or boot a previous
kernel entry.

## 2. Boot problems

### 2.1 Bootloader menu does not show Visnux

| Cause | Fix |
|---|---|
| Bootloader config not regenerated | `grub-mkconfig -o /boot/grub/grub.cfg` or update systemd-boot |
| Wrong paths | Check `/boot/vmlinuz-visnux` and initramfs exist |
| ESP not mounted | Mount `/boot` (or `/efi`) before regenerating |

### 2.2 Kernel panics immediately

| Cause | Fix |
|---|---|
| Missing initramfs | Rebuild `mkinitcpio -P` |
| Corrupt image | Reinstall the kernel package |
| Wrong architecture | Use the x86_64 build |
| Bad command line | Remove recent additions (`nohz_full`, `mitigations`, `lsm`) |

### 2.3 “Root device not found” / emergency shell

The kernel cannot find or mount the root filesystem.

| Cause | Fix |
|---|---|
| Storage driver missing from initramfs | Use the packaged preset (`-S autodetect`) or add `nvme`/`ahci`/`virtio_blk` |
| Wrong `root=` UUID | `blkid`, correct the bootloader entry |
| Filesystem driver missing | Ensure ext4/btrfs is built in (it is) |
| LUKS not unlocked | Add the encryption hook and `cryptdevice=` |
| LVM/MD not activated | Add `lvm2`/`mdadm` hooks |

Check from the emergency shell:

```sh
cat /proc/cmdline
ls /dev/disk/by-uuid
lsinitcpio /boot/initramfs-visnux.img | grep -E 'nvme|ahci|virtio'
```

### 2.4 Black screen after bootloader

| Cause | Fix |
|---|---|
| Missing/incorrect KMS driver | Boot with `nomodeset`, install the right driver |
| NVIDIA proprietary not built | Install `nvidia-dkms` against `visnux-headers` |
| Wrong output | Try another port/monitor, `video=` parameter |
| Framebuffer conflict | `nomodeset` temporarily |

### 2.5 System freezes during boot

| Cause | Fix |
|---|---|
| `nohz_full` misconfiguration | Remove it; never include CPU 0 |
| Aggressive `mitigations=off` | Remove it |
| Broken DKMS module | Boot previous kernel, rebuild module |
| Faulty hardware | Check `dmesg`, run memory test |

## 3. Module problems

### 3.1 “Unknown symbol in module”

The module was built against a different kernel or lacks a dependency.

```sh
modinfo <module>
modprobe -v <module>
dmesg | tail -50
```

Fix: rebuild against the running kernel’s headers, or install the matching
module package.

### 3.2 “module verification failed: signature and/or required key missing”

Expected for unsigned third-party modules when signing is enabled but not
enforced. The module loads and the kernel taints.

```sh
cat /proc/sys/kernel/tainted
```

Options:

- ignore it (functionality unaffected);
- sign the module with your key;
- enforce signatures only after everything is signed (`module.sig_enforce=1`).

### 3.3 DKMS build fails

```sh
uname -r
ls -l /lib/modules/$(uname -r)/build
sudo dkms autoinstall
cat /var/lib/dkms/<module>/<version>/build/make.log
```

| Cause | Fix |
|---|---|
| Headers missing | Install `visnux-headers` |
| Header/kernel mismatch | Reinstall matching versions |
| Module does not support 7.2.15 | Update the module or patch it |
| Missing build tools | Install `base-devel`, `bc`, `libelf`, `pahole` |

### 3.4 Module not found after kernel update

The module directory name changed (for example from `7.2.15visnux` to
`7.2.15-visnux`). Check:

```sh
ls /usr/lib/modules/
uname -r
```

Keep the release string and module directory consistent.

## 4. Memory and swap

### 4.1 zswap not active

```sh
cat /sys/module/zswap/parameters/enabled
grep -i zswap /proc/meminfo
swapon --show
```

| Cause | Fix |
|---|---|
| No swap device | Create/enable swap |
| Disabled at boot | `zswap.enabled=1` |
| Pool full | Increase `max_pool_percent` |
| zram used instead | Choose one strategy |

### 4.2 zram not working

```sh
zramctl
lsmod | grep zram
cat /sys/block/zram0/mm_stat
```

| Cause | Fix |
|---|---|
| Module not loaded | `modprobe zram` |
| No size set | Write `disksize` |
| Not formatted | `mkswap` |
| Not enabled | `swapon` |

### 4.3 System thrashes under memory pressure

- Check PSI: `cat /proc/pressure/memory`.
- Check zswap/zram configuration.
- Reduce `vm.swappiness` or tune zram size.
- Consider `systemd-oomd` with PSI.
- Check for a memory leak (`smem`, `ps_mem`).

### 4.4 OOM killer kills the wrong process

OOM decisions depend on cgroup limits, `oom_score_adj`, and PSI. Configure
`systemd-oomd` or set `memory.max`/`memory.high` per slice.

## 5. Storage and filesystems

### 5.1 Filesystem mounts read-only after an error

The kernel detected corruption and remounted read-only to protect data.

```sh
dmesg | grep -i -E 'ext4|xfs|btrfs|error'
```

Run the appropriate check from a live environment:

```sh
sudo fsck.ext4 -f /dev/sdXN
sudo btrfs check /dev/sdXN
sudo xfs_repair /dev/sdXN
```

Back up first.

### 5.2 NVMe disk not detected

NVMe is a module in this profile.

```sh
lsmod | grep nvme
lsinitcpio /boot/initramfs-visnux.img | grep nvme
```

Ensure `nvme` is in the initramfs (the packaged preset includes it).

### 5.3 “Structure needs cleaning” (XFS)

XFS found metadata corruption. Boot a live ISO and run `xfs_repair`; mount only
after repair.

### 5.4 Btrfs errors

```sh
sudo btrfs device stats /
sudo btrfs scrub start /
dmesg | grep -i btrfs
```

Do not run `btrfs check --repair` casually; it can make things worse.

## 6. Graphics

### 6.1 NVIDIA

| Symptom | Fix |
|---|---|
| `nvidia` module missing | Install `nvidia-dkms`, rebuild DKMS |
| Black screen | Boot `nomodeset`, check `dmesg` |
| Wayland issues | Check driver/Wayland compatibility |
| Secure Boot | Sign the module or disable Secure Boot |

### 6.2 AMD/Intel

```sh
dmesg | grep -i -E 'amdgpu|i915|xe'
glxinfo -B
```

Install firmware (`linux-firmware`) and Mesa. `xe` is the newer Intel driver;
`i915` remains the safer default for many GPUs.

### 6.3 Console works, GUI does not

The kernel is fine; the issue is userspace (display manager, driver, X/Wayland).
Check `journalctl -b` and the display manager logs.

## 7. Networking

### 7.1 No Wi-Fi

| Cause | Fix |
|---|---|
| Firmware missing | Install `linux-firmware` |
| Driver module not loaded | `lspci -k`, `modprobe <driver>` |
| Regulatory domain | `iw reg set <CC>` |
| rfkill | `rfkill list`, `rfkill unblock all` |

### 7.2 No Ethernet

```sh
ip link
lspci -k | grep -A3 Ethernet
dmesg | grep -i eth
```

Check the driver module and firmware.

### 7.3 nftables/iptables fails

`nf_tables` is a module:

```sh
sudo modprobe nf_tables
```

### 7.4 WireGuard missing

```sh
sudo modprobe wireguard
```

If it fails, the module may not be installed for the running kernel.

## 8. Audio

```sh
aplay -l
pactl info
dmesg | grep -i -E 'snd|hda|sof'
```

| Cause | Fix |
|---|---|
| Module not loaded | `modprobe snd_hda_intel` |
| Wrong default sink | `pactl set-default-sink` |
| SOF firmware missing | Install `sof-firmware` |
| HDMI not detected | Check `snd_hda_codec_hdmi` |

## 9. Suspend and hibernation

### 9.1 Suspend does not resume

```sh
cat /sys/power/state
dmesg | grep -i -E 'suspend|resume|PM:'
```

Check firmware, GPU driver, and USB wakeup settings. Try
`echo devices > /sys/power/pm_test` for a test cycle.

### 9.2 Hibernation fails

| Cause | Fix |
|---|---|
| No `resume=` | Add `resume=UUID=<swap>` |
| Swap too small | Make swap ≥ RAM |
| Swap file offset | Add `resume_offset=` |
| Initramfs lacks resume hook | Add the hook and rebuild |

## 10. Performance

### 10.1 High latency / stutter

```sh
cat /proc/pressure/cpu
cat /proc/pressure/io
perf top
```

Check:

- CPU governor (`schedutil`);
- I/O scheduler (try `bfq` or `none`);
- zswap/zram thrashing;
- `sched_ext` policy (detach if experimental);
- `nohz_full` misconfiguration.

### 10.2 Low throughput

- `preempt=full` costs some throughput; try `voluntary` for batch workloads.
- THP `always` can hurt some databases; try `madvise`.
- BFQ adds overhead on fast NVMe; try `none`.
- Debug features (`KFENCE`, `PAGE_POISONING`, `DEBUG_INFO`) have costs; build a
  production profile if needed.

### 10.3 High memory usage

```sh
smem -tk
cat /proc/meminfo
slabtop
```

Caches are not leaks. Check `Slab`, `SReclaimable`, zswap/zram, and per-cgroup
usage.

## 11. Boot log analysis

```sh
dmesg --level=err,warn
journalctl -b -p warning
journalctl -b -k
```

Common patterns:

| Message | Meaning |
|---|---|
| `ACPI Error` | Firmware table issue; often harmless |
| `DMAR: [Firmware Bug]` | IOMMU firmware issue |
| `nouveau ... unknown chipset` | GPU too new for Nouveau |
| `amdgpu ... firmware load failed` | Missing firmware |
| `EXT4-fs error` | Filesystem corruption |
| `BUG: unable to handle page fault` | Kernel bug; report with `dmesg` |
| `tainted` | Proprietary/unsigned module or other taint |

## 12. Collecting a good bug report

```sh
uname -a
cat /proc/version
zcat /proc/config.gz > config.txt
dmesg > dmesg.txt
lspci -vv > lspci.txt
lsmod > lsmod.txt
journalctl -b > journal.txt
```

Include:

- what you did;
- what you expected;
- what happened;
- whether it reproduces;
- whether it happens on the previous kernel;
- any recent config or module changes.

## 13. Recovery

### Boot the previous kernel

Select it in the bootloader menu. If the menu is hidden, hold Shift (GRUB) or
press Space (systemd-boot) during boot.

### Live ISO + chroot

```sh
sudo mount /dev/sdXN /mnt
sudo mount /dev/sdX1 /mnt/boot   # if separate
sudo arch-chroot /mnt
mkinitcpio -P
grub-mkconfig -o /boot/grub/grub.cfg
```

### Emergency shell

Add to the kernel command line:

```text
init=/bin/bash
```

Then remount root read-write:

```sh
mount -o remount,rw /
```

### Reset a broken config

If a runtime setting broke the system, boot with the default command line and
remove the file under `/etc/sysctl.d/`, `/etc/modprobe.d/`, or
`/etc/udev/rules.d/`.

## 14. When to report upstream

Report to the Linux kernel if:

- you see a `BUG:`, `WARNING:`, or `Oops` in `dmesg`;
- a hardware driver fails on vanilla 7.2.15;
- a filesystem corrupts data.

See
[`../Documentation/admin-guide/reporting-issues.rst`](../Documentation/admin-guide/reporting-issues.rst).
For Visnux configuration issues, use the project repository.
