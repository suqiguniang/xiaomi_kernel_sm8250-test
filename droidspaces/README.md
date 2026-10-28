# Droidspaces 支持资产（红米 K40S / munch，kernel 4.19）

本目录存放给 `munch15-base` 内核树启用 **Droidspaces** 所需的东西，CI（`.github/workflows/main.yml`）会自动套用。

| 文件 | 作用 |
|---|---|
| `droidspaces.config` | 按官方 non-GKI 清单整理的内核配置块（带 `DROIDSPACES-CONFIG-BEGIN/END` 标记，CI 用它做幂等追加） |
| `required-config.txt` | 构建后逐项校验 `out/.config` 里必须是 `=y` 的选项（缺一个就 fail，防止配置悄悄回退） |
| `patches/` | non-GKI 内核补丁，CI 用 `patch -p1 --forward` 打，已打过会自动跳过 |

## 版本选择

- Droidspaces：**v6.6.0**（按需求不使用 7.x）。官网 <https://www.droidspaces.org/>，源码 <https://github.com/ravindu644/Droidspaces-OSS>。
- 官网文档：`Documentation/Kernel-Configuration.md`（v6.6.0 tag 内的版本，不是网页上的最新版）。
- 4.19 属于 **non-GKI** 路径：只需要「配置项 + non-GKI 补丁」，**不需要** GKI 的 kABI 补丁（那是 5.4+ 才要）。
- 官网对 4.4–4.19 的定位：Stable，完整支持，可跑嵌套 Docker/Podman；systemd 需 < 258（4.19 没有完整 cgroup v2 语义）。
- 6.6.0 的清单里没有 7.x 才加的项（`MEMCG_SWAP` 强制、`CFS_BANDWIDTH`、`IP6_NF_NAT`、`MACVLAN` 等），这里也就不引入，减少改动面。

## 补丁取舍

v6.6.0 的 `Documentation/resources/kernel-patches/non-GKI/` 里有两个补丁：

| 补丁 | 本项目 | 原因 |
|---|---|---|
| `01.fix_kernel_panic_in_xt_qtaguid.patch` | **未收录** | 本树没有 `net/netfilter/xt_qtaguid.c`（全树 grep 无此文件），打了必然失败 |
| `02.fix_restore cgroup file prefix handling.patch` | **收录** | 修复 LXC/Droidspaces 的 cgroup 文件前缀；已在本树 `patch --dry-run` 通过（`kernel/cgroup/cgroup.c` 偏移 +419，上下文匹配） |

## 内核配置

`droidspaces.config` 覆盖：IPC（`SYSVIPC`/`POSIX_MQUEUE`）、命名空间（`PID_NS`/`UTS_NS`/`IPC_NS`/`NET_NS`）、seccomp、cgroups（`CGROUP_DEVICE`/`CGROUP_PIDS`/`CGROUP_NET_PRIO`/`MEMCG`…）、`DEVTMPFS`、`OVERLAY_FS`、`TMPFS_XATTR/POSIX_ACL`、固件加载，以及 NAT 网络隔离（`VETH`/`BRIDGE`/`NF_TABLES`/`MASQUERADE`/`XT_MATCH_ADDRTYPE`/`NF_CT_NETLINK`）。

- **`CONFIG_USER_NS` 保持关闭**（安全性取舍，机主决定）。代价：容器里跑 Docker 会出现 procfs unsafe 相关告警，需要额外处理；Droidspaces 本体功能不受影响。
- 本树里不存在的符号已剔除：`ANDROID_PARANOID_NETWORK`（已移除）、`FW_LOADER_COMPRESS`、`NF_CONNTRACK_IPV4`、`NETFILTER_XT_TARGET_MASQUERADE`（4.19 用 `IP_NF_TARGET_MASQUERADE`）、`NF_CONNTRACK_NETLINK`（4.19 叫 `NF_CT_NETLINK`）。
- 已用 `make O=out ARCH=arm64 munch_defconfig` 实测：清单里全部落地为 `=y`，无新增 unmet dependency。

## KernelSU-Next 钉版本

- 4.19 是非 GKI 老内核，走上游的 **`*-legacy`** 线（tag：`v3.4.0-legacy`）。
  上游 `v3.4.2` 已重构为 GKI/KMI 外部模块构建（`kernel/Makefile` 里 `check_symbol` + vmlinux 符号校验、没有 `compat/kernel_compat.o` 兼容层），**不能**用在 4.19 树内集成。
- 集成方式与上游 `kernel/setup.sh` 等价：`drivers/kernelsu -> ../KernelSU-Next/kernel` 软链 + `drivers/Makefile` 追加 `obj-$(CONFIG_KSU) += kernelsu/` + `drivers/Kconfig` 加一行 source。
- 老内核用 **manual hook**：`CONFIG_KSU_MANUAL_HOOK=y`，配合 CI 打的 `SukiSU_patch/4.19/ksu_hooks_sukisu_4.19.patch`。
- 内核侧版本号由 KSU 自己的 Kbuild 算：`KSU_VERSION = 30000 + git rev-list --count + 289`，构建日志里有 `-- KernelSU-Next version: NNNNN`。
- **管理器**：只能用 KernelSU-Next 官方签名 APK（内核里写死 `KSU_NEXT_MANAGER_SIZE=0x3e6`、`KSU_NEXT_MANAGER_HASH=79e59011…9821bf7`）。选 **v3.4.x 系列的 KernelSU-Next 管理器**（与本 tag 同源），不要用原版 KernelSU / SukiSU 管理器。
- 回退方案：把 workflow 输入 `ksu_repo=https://github.com/Winkmoon/KernelSU-Next`、`ksu_ref=next` 即可回到原来的 fork 分支构建（那条线当前也能用，只是版本会漂移）。

## 怎么用

GitHub → Actions → `构建小米 sm8250 (munch) HyperOS 内核` → Run workflow，三个输入：

| 输入 | 默认 | 说明 |
|---|---|---|
| `kernel_branch` | `munch15-base` | 内核源码分支；本仓库没有就自动回退 Winkmoon 仓库 |
| `ksu_repo` | `https://github.com/KernelSU-Next/KernelSU-Next` | KSU 源码仓库 |
| `ksu_ref` | `v3.4.0-legacy` | tag / 分支 / commit 都可以 |

流程：取源码 → 装 KSU（钉版本）→ 打 Droidspaces 补丁 → 合配置（幂等）→ `make munch_defconfig` → **校验配置**（缺项直接 fail）→ 编译 → AnyKernel3 打包 → Release。

## 刷机后验证

```bash
su -c 'droidspaces check'
```

或 Droidspaces App → Settings → Requirements → Check Requirements，`must have`/`recommended` 全绿。
再跑一个容器实测：`/dev` 正常（devtmpfs）、`--volatile`（overlayfs）、`--net=nat` 能上网（veth+bridge+masquerade）、`--memory` 上限（memcg）。

## 已知限制

- 4.19 无完整 nftables：容器内网络工具用 `iptables-legacy` / `ip6tables-legacy`；嵌套 Docker 用 `cgroupfs` + `vfs`。
- `CONFIG_USER_NS=n`：容器内 Docker 的 procfs 安全检查会报警（不影响 Droidspaces 起容器）。
- AnyKernel3 里 `do.modules=0`（模块不刷入），所以功能必须内建（`=y`）——本目录的配置块全部是 `=y`。
- 回滚：刷回原 boot.img 即可；本方案只改 defconfig + `kernel/cgroup/cgroup.c` 5 行补丁，不动 ABI 结构。
