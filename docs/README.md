# Visnux kernel documentation

This directory is the long-form documentation for the **`visnux-kernel`**
repository: the Linux 7.2.15 source tree plus the `.config` used to build Visnux
kernels.

Nothing here modifies the kernel. Every document describes the tree, the
configuration, or the workflow around them.

## Start here

| If you want to… | Read |
|---|---|
| Understand what this repository is | [`../README.md`](../README.md) |
| Know your way around the source tree | [architecture.md](architecture.md) |
| Understand or change the kernel config | [configuration.md](configuration.md) |
| Compile the kernel | [building.md](building.md) |
| Install and boot it | [installation.md](installation.md) |
| Learn what each feature does | [features.md](features.md) |
| Tune a running system | [tuning.md](tuning.md) |
| Harden or audit it | [security.md](security.md) |
| Fix a broken boot or module | [troubleshooting.md](troubleshooting.md) |
| Get a quick answer | [faq.md](faq.md) |
| Look up a path, command, or symbol | [reference.md](reference.md) |

## Reading paths

### “I just want to build and boot it”

1. [building.md](building.md) — dependencies and `make`
2. [installation.md](installation.md) — modules, image, initramfs, bootloader
3. [troubleshooting.md](troubleshooting.md) — if it does not come up

### “I want to understand the kernel”

1. [architecture.md](architecture.md) — tree and boot flow
2. [features.md](features.md) — scheduler, memory, I/O, filesystems, security
3. [configuration.md](configuration.md) — why the config looks the way it does
4. [reference.md](reference.md) — symbols and runtime paths

### “I want to tune performance”

1. [tuning.md](tuning.md) — zswap, zram, MGLRU, DAMON, BFQ, PSI, sched_ext
2. [features.md](features.md) — what each knob actually does
3. [security.md](security.md) — the cost of disabling mitigations

### “I want to harden it”

1. [security.md](security.md) — defaults, LSM selection, signing, lockdown
2. [configuration.md](configuration.md) — hardening symbols
3. [troubleshooting.md](troubleshooting.md) — module signing and taint

## Conventions

- Commands assume an Arch-based host and a checkout at `~/visnux-kernel`.
- `$` means an unprivileged shell; `#` means root.
- Config values are quoted exactly as they appear in `.config`.
- “Release string” means the value of `uname -r` / `make kernelrelease`.
- Paths like `/usr/lib/modules/7.2.15-visnux/` refer to the **packaged** lean
  profile; a build from this tree’s `.config` reports `7.2.15visnux`
  (plus `-g<sha>` inside git). See the note in the root README.

## Keeping this documentation honest

Every factual claim should be traceable to one of:

```sh
grep CONFIG_FOO .config          # configuration
grep -E '^(VERSION|PATCHLEVEL|SUBLEVEL|EXTRAVERSION|NAME)' Makefile
git log -1 --format='%H %s'
tar --zstd -xOf <package> .PKGINFO
```

If the configuration changes, update the affected tables in the root README and
in these documents in the same commit.
