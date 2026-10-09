# 飞牛 GVT-g 来宾 GuC 初始化故障与持久修复记录

记录日期：2026-10-09。

## 1. 实际环境

| 项目 | 实际值 |
| --- | --- |
| TrueNAS 宿主内核 | `6.12.15-production+truenas` |
| 物理核显 | Intel UHD 630，`8086:3e92`，`0000:00:02.0` |
| 宿主驱动及参数 | `i915`，`i915.enable_gvt=1 i915.enable_guc=0` |
| PVE | Jailmaker jail，PVE 8.4.1 |
| 来宾 | VM 102，FNOS，地址 `192.168.123.10` |
| 来宾内核 | `6.18.18.c1107-trim` |
| 来宾内核软件包版本 | `6.18.18.c1107-trim-1107` |
| mdev profile | `i915-GVTg_V5_4` |
| mdev UUID | `00000000-0000-0000-0000-000000000102` |
| 来宾虚拟核显 PCI 地址 | `0000:01:00.0` |

这是本次故障机器的实际环境，不应直接套用维护手册中 P630、`8086:3e9a` 和 `V5_8` 的示例值。

## 2. 故障现象与原因

来宾 PCI 列表能看到虚拟核显，但没有 `Kernel driver in use: i915`，也没有 `/dev/dri/renderD128`。因此看到 PCI 设备不等于硬件编解码可用。

默认参数 `enable_guc=-1` 表示由驱动自动选择。对照测试中，该飞牛模块将其解析成 `2`，出现：

```text
GT0: enable_guc=2 (guc:yes submission:no huc:yes slpc:no)
Failed to reserve top of GGTT for GuC
Device initialization failed (-28)
```

`2` 表示启用 HuC 加载/认证而不启用 GuC submission；Coffee Lake 的 HuC 认证仍需要 GuC。因此驱动仍会尝试保留 GuC 固件地址空间。

本次 GGTT 总大小为 4 GiB，以下区间使用左闭右开表示：

| 用途 | 地址区间 |
| --- | --- |
| GVT-g balloon 保留，来宾不可使用 | `[0x60000000, 0x100000000)` |
| GuC 顶部保留，18 MiB | `[0xFEE00000, 0x100000000)` |

GuC 保留区完全落在不可使用的 balloon 区间中，分配返回 `-ENOSPC`，即 `-28`。这不是磁盘空间不足，也不能用增加 profile 容量简单解释。

上游 Linux `6.12.15` 和 `6.18.18` 的默认策略都对 Gen12 以前的平台关闭 GuC/HuC。反汇编本机飞牛 i915 模块，确认其默认选择分支包含针对 Coffee Lake 等平台写入 `2` 的逻辑。这是发行版模块与上游策略的差异；尚未取得飞牛对应源码，不能确定具体补丁及修改动机。

来宾使用 `enable_guc=0` 后驱动绑定及 VAAPI 恢复；恢复 `-1` 会重现同一错误，再设为 `0` 又恢复。宿主参数不会自动传递给来宾，必须在来宾单独设置。

## 3. 本次已执行的持久修改

只修改飞牛 VM 102，不修改 TrueNAS、PVE、mdev profile、GRUB 或内核模块二进制。

新增来宾配置 `/etc/modprobe.d/i915-gvtg.conf`，权限 `0644`：

```text
options i915 enable_guc=0
```

更新前备份当前内核 initramfs：

```text
/boot/initrd.img-6.18.18.c1107-trim.before-gvtg-guc0
```

然后更新当前内核启动镜像。更新后的镜像及备份均约 90 MiB。仅更新当前运行内核，其他已安装内核镜像没有更新。

## 4. 手动复现方法

以下命令全部在飞牛来宾执行，不是在 TrueNAS 或 PVE jail 执行。

先检查现有配置、工具和空间：

```bash
uname -r
sudo grep -RnsE '^[[:space:]]*options[[:space:]]+i915.*enable_guc' \
  /etc/modprobe.d /usr/lib/modprobe.d
cat /proc/cmdline
command -v update-initramfs
df -h /boot /
```

若已有相关设置，先核对来源并避免冲突；还需检查启动参数是否显式传入其他值。本次没有发现相关持久设置。下面示例遇到已有配置或备份时会停止，不覆盖它们：

```bash
set -e
kernel=$(uname -r)
image="/boot/initrd.img-$kernel"
backup="$image.before-gvtg-guc0"
test ! -e /etc/modprobe.d/i915-gvtg.conf
test ! -e "$backup"
sudo cp -a "$image" "$backup"
printf '%s\n' 'options i915 enable_guc=0' |
  sudo tee /etc/modprobe.d/i915-gvtg.conf
sudo chmod 0644 /etc/modprobe.d/i915-gvtg.conf
sudo update-initramfs -u -k "$kernel"
```

i915 可能在 initramfs 阶段加载，所以只写配置而不更新启动镜像不足以保证下次启动生效。如果更新失败，不要直接重启；先保留错误输出并检查启动镜像。

更新成功后核对配置与镜像：

```bash
cat /etc/modprobe.d/i915-gvtg.conf
modprobe -c | grep -E '^options i915 .*enable_guc'
sudo lsinitramfs "/boot/initrd.img-$(uname -r)" | grep i915-gvtg.conf
```

## 5. 本次验证结果与待验证事项

- 已核对配置内容为 `options i915 enable_guc=0`。
- 已确认原 initramfs 备份存在，新镜像已更新。
- 已解包新 initramfs，读取其中 `etc/modprobe.d/i915-gvtg.conf`，内容与预期一致；解包临时目录已删除。
- 当前来宾参数为 `0`，DRM 调试开关为 `0`，渲染节点存在，VAAPI/iHD 初始化成功。
- 前序临时修复阶段已通过 720p、60 帧 H.264 VAAPI 硬件编码及硬件解码；宿主 Jellyfin VAAPI/硬件编码也通过。
- 本次没有重启 VM，没有重新加载驱动；尚未验证重启后持久设置的实际效果，也未验证飞牛媒体应用完整播放流程。

维护窗口中正常重启飞牛 VM 后验证，不需要为此重启 TrueNAS 或 PVE jail：

```bash
uname -r
cat /sys/module/i915/parameters/enable_guc
lspci -nnk -s 01:00.0
ls -l /dev/dri/renderD128
vainfo --display drm --device /dev/dri/renderD128
sudo journalctl -k -b --no-pager |
  grep -Ei 'GuC|GGTT|Device initialization failed|Initialized i915'
```

预期参数为 `0`、核显绑定 i915、渲染节点存在且 VAAPI 成功。modprobe 配置不是内核命令行参数，所以 `/proc/cmdline` 不出现 `i915.enable_guc=0` 是正常现象。

## 6. 对升级和功能的影响

此方法不修改厂商模块，不要求每次升级重新编译，不应阻止正常软件包升级。常规升级通常保留 `/etc/modprobe.d`，新内核 initramfs 的生成通常也会带入配置，但不能保证飞牛大版本升级、系统修复或镜像替换一定保留这些设置。

每次升级重启后都检查运行内核、参数、渲染节点和 VAAPI。如果安装新内核后准备切换启动，先确认目标内核镜像包含配置；必要时针对目标内核执行 `update-initramfs -u -k <目标内核版本>`。本次只更新了 `6.18.18.c1107-trim`，切换其他已安装旧内核前也应检查它的镜像。

关闭 GuC 同时关闭此路径的 HuC 加载/认证，可能影响依赖 HuC 的部分低功耗编码或码率控制功能。普通 H.264 VAAPI 编解码已实测通过，但不代表所有格式、编码模式和长期并发负载均验证完成。

本配置是当前 GVT-g 环境的兼容性措施。若以后取消 GVT-g、改用独立 GPU 直通或厂商修复驱动，需重新评估，不应无条件沿用。

## 7. 未由此修复的告警

宿主 `RPM raw-wakeref not held` / `RPM wakelock ref not held during HW access` 对应延迟加载 kvmgt 时初始 MMIO 快照路径缺少 PM 引用，不能仅据此证明当时 GPU 已挂起或损坏。

来宾 DC mismatch、power-well fuse 和 SAGV `-110` 对应显示电源初始化与 GVT-g 模拟的不完整匹配。它们在成功编解码时仍出现，与致命 GuC/GGTT 冲突是不同问题；本设置不承诺消除这些告警。

## 8. 回滚

删除本次新增的独立配置并重新生成目标内核镜像：

```bash
sudo rm /etc/modprobe.d/i915-gvtg.conf
sudo update-initramfs -u -k "$(uname -r)"
```

成功后正常重启 VM。若此前已为其他内核生成带有此配置的镜像，回滚时也需更新那些镜像。

当前飞牛内核恢复默认 `-1` 会重新触发已确认的初始化故障，回滚前应准备好修复配置或其他已验证方案。

原镜像备份仅适用于对应的 `6.18.18.c1107-trim`。必要时在配置也已恢复的情况下，将该备份恢复到同版本镜像路径；不要把它复制给其他内核版本。不要删除当前镜像后再等待重新生成。

## 9. 源码核对入口

- [上游 6.18.18 GuC 默认策略](https://github.com/gregkh/linux/blob/v6.18.18/drivers/gpu/drm/i915/gt/uc/intel_uc.c)
- [上游 6.12.15 GuC 默认策略](https://github.com/gregkh/linux/blob/v6.12.15/drivers/gpu/drm/i915/gt/uc/intel_uc.c)
- [上游 6.18.18 GGTT 保留](https://github.com/gregkh/linux/blob/v6.18.18/drivers/gpu/drm/i915/gt/intel_ggtt.c)
- [上游 6.18.18 GVT balloon](https://github.com/gregkh/linux/blob/v6.18.18/drivers/gpu/drm/i915/i915_vgpu.c)

源码链接用于核对上游行为，不代表飞牛或 TrueNAS 完全没有发行版补丁。当前机器默认值差异已有实测日志及模块反汇编证据，具体厂商补丁仍未取得。