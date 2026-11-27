<div align="center">

# OnePlus 13 BakaSU + SUSFS Kernel

**My personal kernel build repository for the OnePlus 13**

English · [简体中文](README_CN.md)

</div>

## About this repository

This is my own streamlined build repository for the **OnePlus 13**. It is not a general-purpose multi-device kernel project: the extra device configurations and manifests from upstream have intentionally been removed so that the repository has one clear target and one reproducible build profile.

The build keeps the parts I use—BakaSU, SUSFS, networking features, and selected optimizations—while disabling optional components that are unnecessary for my device or usage. The repository remains public for transparency and reproducibility, but every build should be treated as a personal/custom kernel rather than an official OnePlus, BakaSU, or WildKernels release.

## Supported target

| Item | Value |
| --- | --- |
| Device | OnePlus 13 |
| SoC | Snapdragon 8 Elite (`sun` / SM8750) |
| Firmware target | OxygenOS 16 |
| Source platform | Android 15 |
| Kernel branch | Linux 6.6 |
| Manifest | `oneplus_13_w.xml` |
| Default root solution | BakaSU |
| Filesystem integration | SUSFS |

Only `configs/oos16/OP13.json` is maintained. Other OnePlus, Oppo, Realme, Nord, Ace, Pad, and Open devices are outside the scope of this fork.

## My build profile

Enabled in the current OnePlus 13 configuration:

- BakaSU and SUSFS integration
- ThinLTO and the repository's optimization patch set
- BBR v1 and BBR v3 networking support
- TTL target support
- IP set and IPv6 NAT support
- Unicode bypass fix
- A device-specific module blacklist

Intentionally disabled in this profile:

- HMBIRD/SCX
- Baseband Guard (BBG)
- Droidspaces
- NTSync

These choices are deliberate: this fork favors the features I actually use and avoids optional components that add unwanted behavior, overhead, heat, or maintenance for my OnePlus 13 setup.

## Engineering Changelog

Key engineering modifications maintained in this fork relative to upstream `OnePlus_ReSukiSU_SUSFS`:

| ID | Target Path | Category | Summary |
| --- | --- | --- | --- |
| **CHG-01** | `.github/actions/build-kernel/action.yml` | CI Reliability | Added a 3-tier fallback (`aria2c` → `curl` → `git clone`) when fetching `susfs4ksu` from GitLab to prevent Cloudflare/rate-limit build aborts. |
| **CHG-02** | `.github/workflows/build-kernel-release.yml` | Toolchain ABI | Pinned GitHub Actions runners to `ubuntu-24.04` (preserving `libxml2.so.2` required by prebuilt AOSP `ld.lld`) and replaced obsolete `liblz4-tool` with `lz4`. |
| **CHG-03** | `.github/actions/build-kernel/action.yml`<br>(patches `security/selinux/ss/services.c`, `security/selinux/hooks.c` at build time) | SELinux Consistency | Enforced read-only `backup_sepolicy` validation and AV computation across userspace SELinux query paths under `ReSukiSU` + `SUSFS v2.4.0+`, while preserving kernel-internal domain transitions. |
| **CHG-04** | `.github/actions/build-kernel/action.yml` | Artifact Metadata | Rewrites `kernel.string` in `AnyKernel3/anykernel.sh` during packaging so flashers clearly identify builds from this repository. |

### CHG-03 Technical Design

- **Root Cause**: With `SUSFS v2.4.0` integrated, `susfs_compat.mk` defines `KSU_COMPAT_HAS_SUSFS_FEATURE_SELINUX_HIDE`, delegating query filtering to SUSFS hash tables (`SUS_TYPE_HLIST` / `SUS_AVD_ALLOWED_HLIST`). When userspace frameworks (e.g., LSPosed) reload binary policy via `/sys/fs/selinux/load` during `post-fs-data`, newly injected types/rules bypass KSU's kernel `sepolicy.c` hooks, and policy symbol re-indexing invalidates earlier numeric `type->value` entries.
- **Implementation**:
  1. Added a read-only baseline validator `susfs_check_context_in_backup()` in `security/selinux/ss/services.c` (never mutating `backup_sepolicy->sidtab`) gating `susfs_security_context_to_sid()` and `security_context_str_to_sid()` for unprivileged app UIDs (`>= 10000`, non-`ksu` domain, `ksu_selinux_hide_running == true`);
  2. Updated `susfs_security_compute_av_user()` to map source/target contexts and `tclass` by symbol name against `backup_sepolicy->policydb` with baseline `avd->seqno = 1`;
  3. Filtered non-baseline types by name in `security_get_user_sids()` and extended `handle_selinux_setprocattr()` in `security/selinux/hooks.c` to cover all 5 `procattr` attributes;
  4. **Invariant**: `security_context_to_sid()` (invoked internally via `security_secctx_to_secid()`) remains intentionally ungated so `transive_to_domain("u:r:ksu:s0", cred)` succeeds prior to `commit_creds()`, keeping manager `RootService` and authorized `su` transitions intact.

## Building with GitHub Actions

1. Fork this repository or use it from your own GitHub account.
2. Open **Actions → Build and Release OnePlus 13 Kernel**.
3. Select **Run workflow**.
4. Keep `BakaSU` as the KSU type unless you specifically need standard KernelSU.
5. Enable the release option if you want the completed package published under GitHub Releases.
6. Download the generated AnyKernel3 ZIP from the workflow artifacts or release page.

The workflow is intentionally restricted to the OnePlus 13 configuration. A clean build is slower but useful after toolchain, patch, or source changes; normal builds can reuse ccache.

## Installation and safety

Before flashing:

- Confirm that the package was built for **OnePlus 13, OxygenOS 16, Android 15 / kernel 6.6 source**.
- Back up your boot-critical partitions and important data.
- Keep a known-good boot image and a working recovery method available.
- Do not assume compatibility after a major OTA or kernel source change.
- Read the build log and release notes before using the package.

Flashing a custom kernel can cause boot loops, data loss, broken hardware features, or an unbootable device. You are responsible for understanding and accepting those risks.

## Repository layout

```text
configs/oos16/OP13.json                 OnePlus 13 build profile
manifests/oos16/oneplus_13_w.xml        OnePlus 13 source manifest
.github/actions/build-kernel/           Reusable kernel build action
.github/workflows/build-kernel-release.yml
                                        Manual build and release workflow
```

## Credits

This personal fork builds on work from the Android kernel community, especially:

- [huangdihd/OnePlus_ReSukiSU_SUSFS](https://github.com/huangdihd/OnePlus_ReSukiSU_SUSFS)
- [WildKernels/OnePlus_KernelSU_SUSFS](https://github.com/WildKernels/OnePlus_KernelSU_SUSFS)
- [ReSukiSU](https://github.com/ReSukiSU/ReSukiSU)
- [KernelSU](https://github.com/tiann/KernelSU)
- [SUSFS by simonpunk](https://gitlab.com/simonpunk/susfs4ksu)
- [SUSFS userspace module by sidex15](https://github.com/sidex15)
- [AnyKernel3](https://github.com/osm0sis/AnyKernel3)

All credit for upstream projects and patches belongs to their respective maintainers. My changes focus on the OnePlus 13 build profile, repository cleanup, and the fixes needed by this personal build.

## Disclaimer

This project is provided **as is**, without warranty or a guarantee of support. It is not affiliated with or endorsed by OnePlus. If you choose to build or flash it, you accept full responsibility for the result.
