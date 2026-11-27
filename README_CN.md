<div align="center">

# 一加 13 BakaSU + SUSFS 内核

**我自己使用的一加 13 内核构建仓库**

[English](README.md) · 简体中文

</div>

## 关于本仓库

这是我为 **一加 13** 精简和维护的个人内核构建仓库，不再定位为通用的多机型构建项目。上游仓库中其他设备的配置与清单已被有意移除，使整个仓库只有一个明确目标和一套可复现的构建配置。

当前构建保留了我需要的 BakaSU、SUSFS、网络功能与部分优化，同时关闭了对我的设备或使用方式没有必要的可选组件。仓库公开是为了让构建过程透明、可复现，但这里产生的内核属于个人自定义构建，并非 OnePlus、BakaSU 或 WildKernels 的官方版本。

## 支持目标

| 项目 | 内容 |
| --- | --- |
| 设备 | 一加 13 |
| SoC | 骁龙 8 至尊版（`sun` / SM8750） |
| 固件目标 | OxygenOS 16 |
| 源码平台 | Android 15 |
| 内核分支 | Linux 6.6 |
| Manifest | `oneplus_13_w.xml` |
| 默认 Root 方案 | BakaSU |
| 文件系统集成 | SUSFS |

本仓库只维护 `configs/oos16/OP13.json`。其他一加、OPPO、Realme、Nord、Ace、Pad 和 Open 设备均不在这个分支的支持范围内。

## 我的构建配置

当前一加 13 配置中启用的功能：

- BakaSU 与 SUSFS 集成
- ThinLTO 与仓库中的优化补丁集
- BBR v1 与 BBR v3 网络支持
- TTL Target 支持
- IP Set 与 IPv6 NAT 支持
- Unicode 绕过修复
- 针对本机使用方式设置的模块黑名单

当前配置中有意关闭的功能：

- HMBIRD/SCX
- Baseband Guard（BBG）
- Droidspaces
- NTSync

这些并非遗漏，而是我自己的取舍：只保留实际使用的能力，并避免在一加 13 上引入不需要的行为、额外开销、发热或维护成本。

## 定制变更说明（Engineering Changelog）

相对于上游 `OnePlus_ReSukiSU_SUSFS` 构建模板，本仓库引入以下工程化变更：

| 编号 | 涉及路径 | 变更类别 | 核心内容 |
| --- | --- | --- | --- |
| **CHG-01** | `.github/actions/build-kernel/action.yml` | CI 可靠性 | SUSFS 源码拉取增加 `aria2c` → `curl` → `git clone` 三级降级回退，避免 GitLab/Cloudflare 并发限流中断构建 |
| **CHG-02** | `.github/workflows/build-kernel-release.yml` | 工具链兼容 | 将 Runner 显式锁定为 `ubuntu-24.04`（确保 AOSP 预编译 `ld.lld` 所需的 `libxml2.so.2` 可用），并将废弃包 `liblz4-tool` 替换为 `lz4` |
| **CHG-03** | `.github/actions/build-kernel/action.yml`<br>（构建期修补 `security/selinux/ss/services.c`、`security/selinux/hooks.c`） | SELinux 策略一致性 | 在 `ReSukiSU` + `SUSFS v2.4.0+` 组合下，强制用户态 SELinux 查询统一基于开机原厂策略备份 `backup_sepolicy` 执行校验与访问向量计算，同时保持内核内部域切换入口无损放行 |
| **CHG-04** | `.github/actions/build-kernel/action.yml` | 产物元数据 | 打包阶段自动覆写 `AnyKernel3/anykernel.sh` 中的 `kernel.string`，使刷机工具明确识别当前仓库构建产物 |

### CHG-03 技术细节说明

- **根因**：集成 `SUSFS v2.4.0` 后，`susfs_compat.mk` 会启用 `KSU_COMPAT_HAS_SUSFS_FEATURE_SELINUX_HIDE`，改用 SUSFS 哈希表（`SUS_TYPE_HLIST` / `SUS_AVD_ALLOWED_HLIST`）拦截查询。当用户态框架（如 LSPosed）在 `post-fs-data` 阶段通过 `/sys/fs/selinux/load` 重载二进制策略时，新注入类型与规则不会进入该哈希表，且符号表重排会导致早期登记的类型 ID 失配。
- **实现**：
  1. 在 `security/selinux/ss/services.c` 中新增只读基线校验函数 `susfs_check_context_in_backup()`（不写入 `backup_sepolicy->sidtab`，无锁竞争与内存增长），接管 `susfs_security_context_to_sid()` 与 `security_context_str_to_sid()`；
  2. 在 `susfs_security_compute_av_user()` 中将源/目标上下文及 `tclass` 按符号名映射至 `backup_sepolicy->policydb` 计算访问向量，并固定返回基线序列号 `avd->seqno = 1`；
  3. 在 `security_get_user_sids()` 中按符号名过滤非原厂类型，并在 `security/selinux/hooks.c` 的 `handle_selinux_setprocattr()` 中将校验范围扩展至全部 5 类 `procattr` 属性；
  4. **权限隔离不变式**：保持内核内部入口 `security_context_to_sid()`（供 `security_secctx_to_secid()` 调用）不被拦截，确保应用提权路径 `transive_to_domain("u:r:ksu:s0", cred)` 在 `commit_creds()` 之前正常完成域切换，不影响管理器 `RootService` 与授权应用 `su`。
- **运行时前提**：BakaSU 在 `boot-completed` 阶段若检测到 `selinux_hide` 未启用，会主动释放 `backup_sepolicy` 以节省内存（此后动态开启将返回 `-EAGAIN`）。因此首次刷入后需在管理器中开启 `selinux_hide`（或执行 `ksud feature set selinux_hide 1 && ksud feature save`）并**重启一次设备**，使 `backup_sepolicy` 在开机阶段保持驻留。

## 使用 GitHub Actions 构建

1. Fork 本仓库，或直接在你自己的 GitHub 仓库中使用。
2. 打开 **Actions → Build and Release OnePlus 13 Kernel**。
3. 点击 **Run workflow**。
4. 除非明确需要标准 KernelSU，否则保持 KSU 类型为 `BakaSU`。
5. 如果希望构建完成后自动发布到 Releases，请打开 release 选项。
6. 构建结束后，从工作流 Artifacts 或 Releases 页面下载 AnyKernel3 ZIP。

工作流已收敛为只构建一加 13。完整清理构建速度较慢，适合工具链、补丁或源码发生变化后使用；日常构建可以继续复用 ccache。

## 刷入前须知

刷入之前请务必：

- 确认压缩包面向 **一加 13、OxygenOS 16、Android 15 / kernel 6.6 源码**。
- 备份启动关键分区和个人数据。
- 准备可用的原厂启动镜像与救砖方式。
- 系统进行大版本 OTA 或内核源码变化后，不要默认旧构建仍然兼容。
- 使用前阅读完整构建日志和 Release Notes。

刷入自定义内核可能导致卡开机、数据丢失、硬件功能异常，甚至设备无法启动。请在理解风险并具备恢复能力的前提下操作，所有后果由使用者自行承担。

## 仓库结构

```text
configs/oos16/OP13.json                 一加 13 构建配置
manifests/oos16/oneplus_13_w.xml        一加 13 源码清单
.github/actions/build-kernel/           内核构建复用 Action
.github/workflows/build-kernel-release.yml
                                        手动构建与发布工作流
```

## 致谢

这个个人分支建立在 Android 内核社区大量工作的基础上，特别感谢：

- [huangdihd/OnePlus_ReSukiSU_SUSFS](https://github.com/huangdihd/OnePlus_ReSukiSU_SUSFS)
- [WildKernels/OnePlus_KernelSU_SUSFS](https://github.com/WildKernels/OnePlus_KernelSU_SUSFS)
- [ReSukiSU](https://github.com/ReSukiSU/ReSukiSU)
- [KernelSU](https://github.com/tiann/KernelSU)
- [simonpunk 开发的 SUSFS](https://gitlab.com/simonpunk/susfs4ksu)
- [sidex15 维护的 SUSFS 用户空间模块](https://github.com/sidex15)
- [AnyKernel3](https://github.com/osm0sis/AnyKernel3)

所有上游项目和补丁的版权与贡献归各自维护者所有。我的改动主要围绕一加 13 的个人构建配置、仓库精简，以及此构建所需的兼容性修复。

## 免责声明

本项目按**现状**提供，不附带任何保证，也不承诺持续支持。本项目与 OnePlus 没有隶属关系，亦未获得其认可。选择构建或刷入，即代表你自行承担全部风险与后果。
