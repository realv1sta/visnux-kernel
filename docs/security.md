# Security and hardening

This document describes the security posture of the Visnux kernel configuration,
how to audit it, and how to change it deliberately. It is not a certification and
does not replace a threat model for your specific system.

## 1. Posture summary

The configuration is **hardened by default, but not locked down**:

| Area | Default |
|---|---|
| Kernel address randomisation | on (KASLR + memory layout) |
| W^X for kernel text/modules | enforced |
| Stack protection | strong canary + VMAP stacks + random offset |
| Bounds checking | `FORTIFY_SOURCE`, hardened usercopy |
| Memory initialisation | zero-on-alloc |
| Allocator hardening | freelist randomisation + hardening, page shuffle |
| `dmesg` | restricted to privileged users |
| `/dev/mem` | strict |
| Unprivileged BPF | disabled |
| CPU mitigations | on |
| CET | IBT + user shadow stack |
| LSMs | Landlock, Yama, Lockdown, Integrity, BPF active |
| SELinux/AppArmor/Smack/TOMOYO | compiled, **not active by default** |
| Module signing | all in-tree modules signed, **not enforced** |
| Lockdown | compiled, default `none` |
| Secure Boot | **not** signed for your firmware |

The two deliberate weakenings are module-signature enforcement (for DKMS) and
lockdown default (for usability). Both are documented and reversible.

## 2. Threat model

Visnux targets a **single-user desktop** used for gaming, development, and general
work. The relevant adversaries are:

- malicious local processes trying to escalate privileges;
- malicious or compromised third-party kernel modules;
- untrusted removable media and downloaded files;
- network attackers against exposed services;
- physical attackers (partially mitigated by disk encryption, not by this
  kernel config).

It does **not** claim to defend against:

- a compromised firmware or bootloader;
- physical DMA attacks without an IOMMU policy;
- a malicious kernel module with full privileges;
- side channels on shared hardware beyond what upstream mitigations provide.

## 3. Kernel self-protection

### 3.1 Address space layout randomisation

```ini
CONFIG_RANDOMIZE_BASE=y
CONFIG_RANDOMIZE_MEMORY=y
CONFIG_RANDOMIZE_KSTACK_OFFSET_DEFAULT=y
```

KASLR randomises the kernel image base; memory randomisation randomises the
physical mapping regions; stack offset randomisation makes stack-based attacks
less deterministic.

Check:

```sh
dmesg | grep -i 'Kernel Offset'
cat /proc/kallsyms | head
```

### 3.2 W^X and read-only data

```ini
CONFIG_STRICT_KERNEL_RWX=y
CONFIG_STRICT_MODULE_RWX=y
CONFIG_DEBUG_WX=y
```

Kernel text is not writable and data is not executable. `DEBUG_WX` reports
unexpected W+X mappings at boot.

### 3.3 Stack protection

```ini
CONFIG_STACKPROTECTOR=y
CONFIG_STACKPROTECTOR_STRONG=y
CONFIG_VMAP_STACK=y
CONFIG_RANDOMIZE_KSTACK_OFFSET_DEFAULT=y
```

Strong stack canaries, virtually mapped stacks with guard pages, and random stack
offsets raise the cost of stack corruption exploits.

### 3.4 Usercopy and fortification

```ini
CONFIG_FORTIFY_SOURCE=y
CONFIG_HARDENED_USERCOPY=y
CONFIG_INIT_ON_ALLOC_DEFAULT_ON=y
```

`FORTIFY_SOURCE` adds compile-time and run-time bounds checks to common string
and memory functions. Hardened usercopy validates copies to and from userspace.
Zero-on-alloc removes uninitialised-memory leaks.

### 3.5 Allocator hardening

```ini
CONFIG_SLAB_FREELIST_RANDOM=y
CONFIG_SLAB_FREELIST_HARDENED=y
CONFIG_SHUFFLE_PAGE_ALLOCATOR=y
```

Freelist randomisation makes heap grooming harder; freelist hardening detects
corruption; page-allocator shuffling randomises page assignment.

### 3.6 CPU mitigations

```ini
CONFIG_CPU_MITIGATIONS=y
CONFIG_MITIGATION_RETPOLINE=y
CONFIG_MITIGATION_SPECTRE_V2=y
CONFIG_MITIGATION_PAGE_TABLE_ISOLATION=y
```

Mitigations are on by default. `mitigations=off` is a real performance/security
trade-off; do not use it on a system exposed to untrusted code.

```sh
cat /sys/devices/system/cpu/vulnerabilities/*
```

### 3.7 CET and control-flow integrity

```ini
CONFIG_X86_KERNEL_IBT=y
CONFIG_X86_USER_SHADOW_STACK=y
CONFIG_X86_UMIP=y
CONFIG_X86_INTEL_MEMORY_PROTECTION_KEYS=y
```

Indirect Branch Tracking and shadow stacks raise the cost of control-flow
hijacking on supporting CPUs. UMIP blocks user-mode access to privileged
instructions; MPK provides memory protection keys.

Kernel CFI (`CONFIG_CFI_CLANG`) is **not** enabled (the build uses GCC, and
`LTO_NONE=y`).

## 4. LSMs

### 4.1 What is active

```ini
CONFIG_LSM="landlock,lockdown,yama,integrity,bpf"
```

| LSM | Role |
|---|---|
| Landlock | Unprivileged sandboxing by processes |
| Lockdown | Restrict kernel features in secure-boot/lockdown modes |
| Yama | ptrace restrictions (`ptrace_scope`) |
| Integrity | IMA/EVM hooks |
| BPF | BPF LSM programs |

Compiled but **not in the default order**:

- SELinux
- AppArmor
- Smack
- TOMOYO

### 4.2 Activating another LSM

Order matters: for each hook, the first LSM that implements it wins. Put the
policy LSM early, but keep `landlock`, `lockdown`, and `yama` in a sensible order.

AppArmor:

```text
lsm=landlock,lockdown,yama,integrity,bpf,apparmor
```

SELinux:

```text
lsm=landlock,lockdown,yama,integrity,bpf,selinux
```

Before rebooting:

1. install the userspace policy and tools;
2. label files (SELinux) or load profiles (AppArmor);
3. set the enforcing mode only after a permissive test;
4. verify with `aa-status`, `sestatus`, or `smackctl`.

A misconfigured policy LSM can make the system unusable. Keep a recovery entry
with `lsm=landlock,lockdown,yama,integrity,bpf` (the default).

### 4.3 Landlock

Landlock lets unprivileged processes restrict their own filesystem, network, and
other access. It is used by sandboxes and browsers. It requires no system-wide
policy and is safe to leave on.

### 4.4 Yama

```sh
cat /proc/sys/kernel/yama/ptrace_scope
```

`1` restricts `ptrace` to descendants; `2` restricts it to CAP_SYS_PTRACE; `3`
disables it entirely. Many distributions default to `1`.

### 4.5 Lockdown

```ini
CONFIG_SECURITY_LOCKDOWN_LSM=y
CONFIG_LOCK_DOWN_KERNEL_FORCE_NONE=y
```

Modes:

| Mode | Effect |
|---|---|
| `none` | No restrictions (default) |
| `integrity` | Restrict features that could modify the running kernel |
| `confidentiality` | Also restrict reading kernel memory |

Set at boot:

```text
lockdown=integrity
```

or via `/sys/kernel/security/lockdown`. Lockdown is most useful with Secure Boot,
where firmware can attest the boot chain.

## 5. Module signing

### 5.1 What is configured

```ini
CONFIG_MODULE_SIG=y
CONFIG_MODULE_SIG_ALL=y
CONFIG_MODULE_SIG_HASH="sha512"
CONFIG_MODULE_SIG_KEY="certs/signing_key.pem"
```

Every module built in-tree is signed with the build key. If the key is absent,
the build generates one.

### 5.2 Enforcement is off

`CONFIG_MODULE_SIG_FORCE` is **not set**, and the default is not to enforce.
Consequences:

- unsigned third-party modules load;
- the kernel is tainted (`TAINT_UNSIGNED_MODULE`, flag `E`);
- Secure Boot does not automatically enforce module signatures unless you pass
  `module.sig_enforce=1`.

Check:

```sh
cat /proc/sys/kernel/tainted
modinfo -F sig_id <module>.ko
```

### 5.3 Enforcing

Only enforce after signing everything you need:

```text
module.sig_enforce=1
```

Then verify that DKMS modules, VirtualBox, NVIDIA, ZFS, and any other
out-of-tree module are signed with a key in the kernel keyring. If not, the
system may fail to bring up networking or graphics.

### 5.4 Key management

- The private key is `certs/signing_key.pem`; keep it secret.
- The certificate is `certs/signing_key.x509`.
- For reproducible packaging, provide your own key pair.
- Never commit a private key to a public repository.

## 6. Secure Boot

The kernel image is not signed with a key your firmware trusts. Options:

1. **Disable Secure Boot** — simplest, weakest.
2. **Enrol a MOK and sign the kernel + modules** — recommended if you need
   Secure Boot.
3. **Use a signed shim** — only if you control the chain and understand it.

Signing outline:

```sh
sudo mokutil --import certs/signing_key.x509
# reboot and complete enrolment in MOK Manager

sbsign --key certs/signing_key.pem --cert certs/signing_key.x509 \
  --output /boot/vmlinuz-visnux.signed /boot/vmlinuz-visnux
```

Then update the bootloader to use the signed image, sign modules, and consider
`module.sig_enforce=1` and `lockdown=integrity`.

Caveats:

- Signing the kernel does not sign the initramfs or the bootloader.
- Firmware updates can reset keys.
- A wrong key bricks boot until you disable Secure Boot or re-enrol.

## 7. Attack surface

The full-featured profile enables 3,224 built-in options and 6,314 modules. That
is broad. To reduce surface:

- disable hardware you do not have;
- disable network protocols and filesystems you do not use;
- avoid loading unnecessary modules;
- consider the lean profile for production systems.

Useful audits:

```sh
# Loaded modules
lsmod

# Built-in filesystems
cat /proc/filesystems

# Network protocols
cat /proc/net/protocols

# Kernel config
zcat /proc/config.gz | grep '=y$'
```

Debug features (`DEBUG_FS`, `KALLSYMS_ALL`, `DEBUG_INFO`) are useful but expose
information to privileged users. They do not grant unprivileged access, but they
help an attacker who already has root or a kernel read primitive.

## 8. Runtime hardening checklist

```sh
# dmesg restricted
sysctl kernel.dmesg_restrict

# kptr restrictions
sysctl kernel.kptr_restrict

# ptrace scope
sysctl kernel.yama.ptrace_scope

# unprivileged BPF
sysctl kernel.unprivileged_bpf_disabled

# perf event paranoia
sysctl kernel.perf_event_paranoid

# unprivileged user namespaces (if you do not need them)
sysctl kernel.unprivileged_userns_clone 2>/dev/null

# ASLR
sysctl kernel.randomize_va_space
```

Suggested `/etc/sysctl.d/99-hardening.conf`:

```ini
kernel.dmesg_restrict = 1
kernel.kptr_restrict = 2
kernel.yama.ptrace_scope = 1
kernel.unprivileged_bpf_disabled = 1
kernel.perf_event_paranoid = 3
kernel.randomize_va_space = 2
```

Test each change; `perf_event_paranoid=3` breaks unprivileged profiling, and
`unprivileged_bpf_disabled=1` breaks tools that rely on unprivileged BPF.

## 9. Verifying the configuration

```sh
zcat /proc/config.gz | grep -E \
 'CONFIG_(RANDOMIZE_BASE|STRICT_KERNEL_RWX|STRICT_MODULE_RWX|STACKPROTECTOR_STRONG|FORTIFY_SOURCE|HARDENED_USERCOPY|INIT_ON_ALLOC_DEFAULT_ON|SLAB_FREELIST_RANDOM|SLAB_FREELIST_HARDENED|SHUFFLE_PAGE_ALLOCATOR|CPU_MITIGATIONS|SECURITY_DMESG_RESTRICT|IO_STRICT_DEVMEM|BPF_UNPRIV_DEFAULT_OFF|MODULE_SIG|MODULE_SIG_FORCE|LSM)='
```

Expected highlights:

```text
CONFIG_RANDOMIZE_BASE=y
CONFIG_STRICT_KERNEL_RWX=y
CONFIG_STRICT_MODULE_RWX=y
CONFIG_STACKPROTECTOR_STRONG=y
CONFIG_FORTIFY_SOURCE=y
CONFIG_HARDENED_USERCOPY=y
CONFIG_INIT_ON_ALLOC_DEFAULT_ON=y
CONFIG_SLAB_FREELIST_RANDOM=y
CONFIG_SLAB_FREELIST_HARDENED=y
CONFIG_SHUFFLE_PAGE_ALLOCATOR=y
CONFIG_CPU_MITIGATIONS=y
CONFIG_SECURITY_DMESG_RESTRICT=y
CONFIG_IO_STRICT_DEVMEM=y
CONFIG_BPF_UNPRIV_DEFAULT_OFF=y
CONFIG_MODULE_SIG=y
CONFIG_LSM="landlock,lockdown,yama,integrity,bpf"
```

## 10. Common mistakes

| Mistake | Risk |
|---|---|
| `mitigations=off` on an untrusted-code system | Re-opens CPU side channels |
| `module.sig_enforce=1` with unsigned DKMS modules | Boots without graphics/network |
| Enabling SELinux/AppArmor without policy | Boot failure or broken services |
| `kptr_restrict=0` and `dmesg_restrict=0` | Leaks kernel addresses |
| Loading modules from untrusted sources | Arbitrary kernel code |
| Disabling `lockdown` while claiming Secure Boot | False sense of security |
| Assuming module signing = Secure Boot | Different mechanisms |

## 11. Reporting vulnerabilities

For upstream Linux security issues, follow
[`../Documentation/process/security-bugs.rst`](../Documentation/process/security-bugs.rst)
and [`../Documentation/process/cve.rst`](../Documentation/process/cve.rst).

For Visnux-specific configuration issues, open an issue in the project
repository with:

```sh
uname -a
zcat /proc/config.gz | grep CONFIG_
dmesg
```

and a clear description of the impact.
