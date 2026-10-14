# PVE 启用 Intel GVT-g 并分配给 VM 101

记录日期：2026-10-14。

本文记录 `pvemini` 上实际完成的 GVT-g 配置，供后续维护和重新部署参考。宿主机配置由助手完成，重启及 VM 101 的显卡分配由用户完成；用户已确认虚拟机内正常使用。

## 1. 环境与最终状态

| 项目 | 实际配置 |
| --- | --- |
| SSH 入口 | `ssh pvemini` |
| PVE 版本 | 8.4.5，`proxmox-ve` 包版本 8.4.0 |
| 运行内核 | `6.8.12-13-pve` |
| CPU | Intel Core i3-6100T |
| 核显 | Intel HD Graphics 530，PCI ID `8086:1912` |
| 核显 PCI 地址 | `0000:00:02.0` |
| 宿主机核显驱动 | `i915` |
| 引导方式 | UEFI，通过 Proxmox 的 SHIM / GRUB 引导 |
| 虚拟机 | 101，名称 `win8-srv` |
| 虚拟机固件 / 机器类型 | OVMF / `pc-q35-9.0` |
| 分配类型 | `i915-GVTg_V5_4` |
| 最终状态 | 宿主机启用成功，101 已挂载 vGPU；用户确认正常使用 |

VM 配置中的 `ostype: win8` 是 PVE 的系统类型标记，并不能单独证明实际 Windows 版本。初次检查时 guest agent 未运行，未通过它确认客体系统或驱动版本。

## 2. 初始检查

进入宿主机：

```bash
ssh pvemini
```

以下命令均在 PVE 宿主机执行：

```bash
pveversion -v
uname -r
lscpu | grep -E 'Model name|Virtualization'
lspci -nnk -s 00:02.0
cat /proc/cmdline
qm status 101
qm config 101

grep -E 'CONFIG_DRM_I915_GVT|CONFIG_VFIO_MDEV|CONFIG_VFIO_IOMMU_TYPE1' \
  /boot/config-$(uname -r)
modinfo kvmgt
modinfo -p i915 | grep -E 'gvt|guc'
journalctl -k -b --no-pager | grep -Ei 'DMAR|IOMMU|gvt|kvmgt'
```

本机初始检查结果：

- Skylake 核显 HD 530 属于 GVT-g 支持范围。
- 当前内核包含 `CONFIG_DRM_I915_GVT=y`、`CONFIG_DRM_I915_GVT_KVMGT=m`、`CONFIG_VFIO_MDEV=m` 和 `CONFIG_VFIO_IOMMU_TYPE1=m`。
- VT-d / IOMMU 已工作，核显存在 IOMMU 分组。
- 核显绑定 `i915`，无需换驱动。
- `enable_gvt` 初始为 `N`，尚未暴露 mediated device 类型。
- 不需要降级内核或安装第三方 GVT-g 模块。

还检查了 `/etc/default/grub`、`/etc/default/grub.d/`、`/etc/modprobe.d/`、`/etc/modules` 和 `/etc/modules-load.d/`，未发现与本次配置冲突的设置。新环境应重新检查，不能直接假设结果相同。

## 3. 修改前备份

本次实际备份目录位于 **PVE 宿主机**：

```text
/root/gvt-g-backup-QEJt52Jk
```

保存了修改前的：

- `/etc/default/grub` 和 `/etc/default/grub.d/`
- `/etc/modules` 和 `/etc/modules-load.d/`
- `/boot/grub/grub.cfg`
- 两个已安装内核的 initramfs：`6.8.12-13-pve`、`6.8.4-2-pve`
- `/etc/pve/qemu-server/101.conf`

备份目录还保留了本次新增配置文件的副本 `99-gvt-g.cfg` 和 `gvt-g.conf`；这两个副本是新配置，不是修改前的文件。

重新部署时可先创建新的备份目录，而不是覆盖本次备份：

```bash
backup=$(mktemp -d /root/gvt-g-backup-XXXXXXXX)
chmod 700 "$backup"
cp -a /etc/default/grub "$backup/grub"
cp -a /etc/default/grub.d "$backup/grub.d"
cp -a /etc/modules "$backup/modules"
cp -a /etc/modules-load.d "$backup/modules-load.d"
cp -a /boot/grub/grub.cfg "$backup/grub.cfg"
cp -a /etc/pve/qemu-server/101.conf "$backup/101.conf"
printf 'Backup: %s\n' "$backup"
```

另将实际已安装内核对应的 `/boot/initrd.img-*` 备份到该目录。本次是配置备份，**不是虚拟机磁盘或 Windows 系统备份**。

## 4. 启用 GVT-g

### 4.1 设置启动参数

本次没有直接改写 `/etc/default/grub`，而是新增：

```text
/etc/default/grub.d/99-gvt-g.cfg
```

完整内容：

```bash
GRUB_CMDLINE_LINUX_DEFAULT="$GRUB_CMDLINE_LINUX_DEFAULT intel_iommu=on i915.enable_gvt=1 i915.enable_guc=0"
```

参数说明：

| 参数 | 用途 |
| --- | --- |
| `intel_iommu=on` | 显式启用 Intel IOMMU；本机修改前已工作 |
| `i915.enable_gvt=1` | 启用 GVT-g 宿主机支持 |
| `i915.enable_guc=0` | 禁用 GuC / HuC，避免使用与 GVT-g 不兼容的 GuC submission |

采用追加方式保留了原来的 `quiet` 等参数。此方案适用于本机的 GRUB 引导方式；其他引导方式应先确认启动参数的实际来源。

### 4.2 设置开机加载模块

新增：

```text
/etc/modules-load.d/gvt-g.conf
```

完整内容：

```text
kvmgt
vfio_iommu_type1
```

`kvmgt` 会通过模块依赖加载 `i915`、`kvm`、`mdev`、`vfio` 等模块，无需重复列出全部依赖。

**不要为本方案屏蔽 `i915` 或把整块 HD 530 绑定给 `vfio-pci`。** GVT-g 依赖宿主机继续使用 `i915` 管理核显，这与整卡直通不同。

本次两个新增文件均设置为 `root:root`，权限 `0644`。

### 4.3 更新启动文件

```bash
update-initramfs -u -k all
update-grub
grub-script-check /boot/grub/grub.cfg
```

本次两个已安装内核的 initramfs 都更新成功，GRUB 语法检查通过。可确认正常启动项已包含参数：

```bash
grep -E '^[[:space:]]*linux[[:space:]]' /boot/grub/grub.cfg
```

配置写入后，当前已加载的 `i915` 不会因此立即启用 GVT-g。不要为了即时生效强行卸载正在使用的核显驱动。

## 5. 重启宿主机


## 6. 重启后验证

```bash
cat /proc/cmdline
cat /sys/module/i915/parameters/enable_gvt
cat /sys/module/i915/parameters/enable_guc
lsmod | grep -E 'i915|kvmgt|vfio|mdev'
lspci -nnk -s 00:02.0
readlink /sys/bus/pci/devices/0000:00:02.0/iommu_group
systemctl --no-pager --full status systemd-modules-load.service

pvesh get /nodes/pvemini/hardware/pci/0000:00:02.0/mdev \
  --output-format json-pretty
qm list
```

本次实际验证结果：

- 启动参数包含 `intel_iommu=on i915.enable_gvt=1 i915.enable_guc=0`。
- `enable_gvt` 为 `Y`，`enable_guc` 为 `0`。
- `kvmgt`、`mdev`、`vfio_iommu_type1`、`vfio` 已加载。
- 核显仍绑定 `i915`，位于 IOMMU group 0。
- `systemd-modules-load.service` 成功退出。
- 内核出现 `MDEV: Registered`，PVE API 能列出 vGPU 类型。
- 101、105、106 均重新运行。

首次检查、尚未分配 vGPU 时，资源如下：

| 类型 | low_gm_size | high_gm_size | 标示分辨率 | 当时可用实例 |
| --- | --- | --- | --- | --- |
| `i915-GVTg_V5_4` | 128 MB | 512 MB | 1920×1200 | 1 |
| `i915-GVTg_V5_8` | 64 MB | 384 MB | 1024×768 | 2 |

两种类型共享同一核显资源，可用数量不能直接相加；分配后数量会变化。这里的显存资源字段也不代表新增了独立物理显存。

## 7. 分配给 VM 101

用户自行完成了分配，并确认正常使用。随后通过 SSH 核实，101 的最终配置包含：

```text
hostpci0: 0000:00:02.0,mdev=i915-GVTg_V5_4,pcie=1
```

如需在相同环境重新配置，可在 PVE 界面的 VM 101 硬件页添加 PCI 设备，选择核显及上述 mediated device 类型，并启用 PCI-Express。

等价命令如下，**仅在确认 `hostpci0` 未被其他设备占用时使用**：

```bash
qm set 101 --hostpci0 '0000:00:02.0,mdev=i915-GVTg_V5_4,pcie=1'
```

变更后让 VM 完整关机再启动，使新的 QEMU 进程应用设备配置；仅在 Windows 内重启不一定能应用 PVE 的待生效硬件变更。此步骤不需要再次重启 PVE。

本次保留原有虚拟显示器，没有设置 `vga: none`、`x-vga=1`、额外 ROM 或自定义 QEMU 参数。

PVE 自动创建和管理 mdev 实例，不需要手工生成 UUID、写入 sysfs 的 `create` 文件或增加开机创建脚本。**`mdev=` 的值是类型名称，不是 UUID。**

最终检查：

```bash
qm status 101
qm config 101
ls -l /sys/bus/mdev/devices
```

本次 101 处于 `running`，存在活动实例：

```text
00000000-0000-0000-0000-000000000101
```

Windows 内应检查 Intel 显卡驱动、设备管理器状态，并验证所需应用的硬件加速。用户已反馈正常使用，但本次未记录实际安装的 Intel 驱动版本。

## 8. 已观察到的警告与限制

### GVT-g 初始化警告

首次重启的内核日志中出现：

- `RPM raw-wakeref not held`
- `RPM wakelock ref not held during HW access`
- `golden_hw_state` 文件加载失败，错误 `-2`

随后仍成功出现 `MDEV: Registered`，PVE 正常列出类型，用户也成功使用了 vGPU。因此这些日志没有阻止本次配置成功，但不能据此保证所有工作负载都稳定。后续遇到 VM 启动失败、驱动错误或 GPU 挂起时，应检查：

```bash
journalctl -k -b --no-pager | grep -Ei 'gvt|kvmgt|i915|vfio|mdev'
```

### 备用 EFI 引导器提示

更新 initramfs 时提示 `/boot/efi/EFI/BOOT/BOOTX64.efi` 是备用引导器，但 GRUB 包未配置自动更新它；同时提示没有 `/etc/kernel/proxmox-boot-uuids`，跳过 ESP 同步。

本机实际启动入口为 `\EFI\PROXMOX\SHIMX64.EFI`，后续重启已证明 GVT-g 参数生效。本次没有重装 GRUB，也没有修改备用引导器。以后若处理引导器升级或恢复，应单独核查，不要把该提示当成本次启用失败。

### 使用限制

- GVT-g 是共享核显，不是整卡直通，不能把此配置当作 VM 独占宿主机 HDMI 输出的方案。
- 保留原有虚拟显示器有利于继续使用 noVNC，但 noVNC 能显示画面不等于应用已使用 Intel GPU 加速。
- GVT-g 属于旧技术。升级 PVE 或内核前应检查新内核是否仍支持它，并保留已验证可用的内核。
- Windows 加速取决于客体系统和匹配驱动，本记录不推荐未经验证的具体驱动版本。

## 9. 停用与回退

以下是后续需要停用时的方案，**本次未执行**。

1. 正常关闭 VM 101，确认 `hostpci0` 仍是本文的 vGPU 配置，再删除：

   ```bash
   qm set 101 --delete hostpci0
   ```

2. 确认没有其他 VM 依赖 GVT-g，并确认下列文件仍是本文新增内容。若已有后续修改，应先整合，不能直接删除。

3. 删除本次两个独立配置文件，重新生成启动文件：

   ```bash
   rm /etc/default/grub.d/99-gvt-g.cfg
   rm /etc/modules-load.d/gvt-g.conf
   update-initramfs -u -k all
   update-grub
   grub-script-check /boot/grub/grub.cfg
   ```

4. 在维护窗口重启 PVE，再检查 `enable_gvt` 和 VM 状态。

因为本次没有改写原始 GRUB 配置或其他模块配置，移除两个新增文件即可撤销本次启用设置。不建议直接覆盖旧的 VM 配置、GRUB 配置或 initramfs，以免丢失后续变更。

## 10. 参考

- [Proxmox PCI(e) Passthrough 文档](https://pve.proxmox.com/wiki/PCI%28e%29_Passthrough)

本文以本机实际配置和验证结果为准；其他硬件、内核或引导方式不能直接照搬。
