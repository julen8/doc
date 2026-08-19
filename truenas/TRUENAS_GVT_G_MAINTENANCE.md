# TrueNAS GVT-g 与 PVE vGPU 操作手册

本文适用于以下实测环境：

- TrueNAS SCALE `25.04.1`
- Intel UHD Graphics P630，PCI `0000:00:02.0`，设备 ID `8086:3e9a`
- Jailmaker PVE jail 位于 `/mnt/app/jailmaker/jails/pve`

GVT-g 已停止维护且存在安全逃逸风险，只应分配给可信虚拟机。物理 GPU 必须始终绑定 `i915`，不要将 `0000:00:02.0` 整体直通或绑定到 `vfio-pci`。

## 第一部分：在 TrueNAS 上启用 GVT-g

### 1. 检查硬件和内核

连接 TrueNAS：

```shell
ssh nas
```

列出主机上的显示设备：

```shell
lspci -Dnn | grep -Ei 'VGA compatible controller|Display controller|3D controller'
```

本机 P630 对应的输出类似：

```text
0000:00:02.0 Display controller [0380]: Intel Corporation Coffee Lake-S GT2 [UHD Graphics P630] [8086:3e9a]
```

其中：

- 行首的 `0000:00:02.0` 是完整 PCI 地址，格式为 `域:总线:设备.功能`。
- 行尾的 `8086:3e9a` 是 `厂商 ID:设备 ID`；`8086` 代表 Intel，`3e9a` 代表这块 P630。
- 如果列出了多块显卡，应选择 Intel 核显对应的行，并通过下面的命令确认其驱动为 `i915`。

将查到的 PCI 地址传给 `lspci -s`，查看设备和驱动详情：

```shell
lspci -Dnnk -s 0000:00:02.0
```

确认 GPU、IOMMU 参数和内核模块：

```shell
cat /proc/cmdline
grep -E 'CONFIG_DRM_I915_GVT|CONFIG_DRM_I915_GVT_KVMGT|CONFIG_VFIO_MDEV' \
  /boot/config-$(uname -r)
find /lib/modules/$(uname -r) -type f \
  | grep -E '/(kvmgt|mdev|vfio.*)\.ko'
```

本机应满足：

- `0000:00:02.0` 使用 `i915` 驱动
- `/proc/cmdline` 已有 `intel_iommu=on iommu=pt`
- 内核包含 `CONFIG_DRM_I915_GVT`、`kvmgt`、`mdev` 和 VFIO 模块

不要添加 `pcie_acs_override=downstream,multifunction`。

### 2. 创建回滚启动环境并启用

检查当前配置和启动环境：

```shell
midclt call system.advanced.config | jq -r .kernel_extra_options
midclt call boot.environment.query | jq -c '.[] | {id,active,activated,keep}'
cat /proc/cmdline
```

创建回滚启动环境：

```shell
midclt call boot.environment.clone \
  '{"id":"25.04.1","target":"25.04.1-pre-gvt"}'

midclt call boot.environment.keep \
  '{"id":"25.04.1-pre-gvt","value":true}'
```

如果 `25.04.1-pre-gvt` 已存在，不要重复执行 clone。

通过 TrueNAS middleware 写入额外内核参数：

```shell
midclt call system.advanced.update \
  '{"kernel_extra_options":"i915.enable_gvt=1 i915.enable_guc=0"}'
```

确认配置已写入：

```shell
midclt call system.advanced.config | jq -r .kernel_extra_options
grep -RHE 'enable_gvt|enable_guc' /etc/default/grub.d/truenas.cfg
```

从 TrueNAS Web UI 正常重启。确认 VM、jail 和存储任务都已停止后，也可以执行：

```shell
sudo reboot
```

不要直接编辑 `/etc/default/grub` 或 `/etc/default/grub.d/truenas.cfg`。

### 3. 重启后验证

#### 内核参数和模块

```shell
cat /proc/cmdline

sudo cat /sys/module/i915/parameters/enable_gvt
sudo cat /sys/module/i915/parameters/enable_guc

lsmod | grep -E '^(kvmgt|mdev|vfio|i915)'
```

预期结果：

- `/proc/cmdline` 包含 `i915.enable_gvt=1 i915.enable_guc=0`
- `enable_gvt` 为 `Y`
- `enable_guc` 为 `0`
- 已加载 `kvmgt`、`mdev`、`vfio`、`vfio_iommu_type1` 和 `i915`

#### mdev 类型和容量

```shell
find /sys/bus/pci/devices/0000:00:02.0/mdev_supported_types \
  -mindepth 1 -maxdepth 1 -type d -printf '%f\n'

cat /sys/bus/pci/devices/0000:00:02.0/mdev_supported_types/\
i915-GVTg_V5_4/available_instances
cat /sys/bus/pci/devices/0000:00:02.0/mdev_supported_types/\
i915-GVTg_V5_4/description

cat /sys/bus/pci/devices/0000:00:02.0/mdev_supported_types/\
i915-GVTg_V5_8/available_instances
cat /sys/bus/pci/devices/0000:00:02.0/mdev_supported_types/\
i915-GVTg_V5_8/description
```

本机实测结果：

| 类型 | 最大分辨率 | 可用实例数 | 设备 API |
| --- | ---: | ---: | --- |
| `i915-GVTg_V5_4` | 1920x1200 | 1 | `vfio-pci` |
| `i915-GVTg_V5_8` | 1024x768 | 2 | `vfio-pci` |

`available_instances` 会随 vGPU 创建而减少。

#### GPU 和渲染节点

```shell
lspci -Dnnk -s 0000:00:02.0
ls -l /dev/dri /dev/dri/by-path
sudo dmesg | grep -Ei 'gvt|kvmgt|i915.*error|i915.*fail'
```

预期 P630 继续使用 `i915` 驱动，并保留：

```text
/dev/dri/renderD128
```

### 4. 禁用或回滚

先停止所有使用 vGPU 的 VM，并删除它们的 `hostpciN` 配置。确认没有 mdev：

```shell
find /sys/bus/mdev/devices -mindepth 1 -maxdepth 1 -type l -printf '%f\n'
```

如果 `kernel_extra_options` 只有本文设置的两个参数，清空并重启：

```shell
midclt call system.advanced.update '{"kernel_extra_options":""}'
```

如果还有其他参数，只删除 `i915.enable_gvt=1` 和 `i915.enable_guc=0`。重启后确认：

```shell
cat /proc/cmdline
test -d /sys/bus/pci/devices/0000:00:02.0/mdev_supported_types \
  && echo gvt_still_enabled || echo gvt_disabled
```

如果启用后无法正常启动，从 TrueNAS 启动菜单选择 `25.04.1-pre-gvt`，或通过 SSH 激活：

```shell
midclt call boot.environment.activate \
  '{"id":"25.04.1-pre-gvt"}'
sudo reboot
```

## 第二部分：把 vGPU 分配给 Jailmaker PVE 虚拟机

PVE jail 的 sysfs 是只读的，因此由 TrueNAS 宿主创建和删除 mdev，PVE 只通过 `/dev/vfio` 使用它。

### 1. 确定 VM、profile 和 UUID

先检查剩余容量：

```shell
ssh nas
cat /sys/bus/pci/devices/0000:00:02.0/mdev_supported_types/\
i915-GVTg_V5_4/available_instances
cat /sys/bus/pci/devices/0000:00:02.0/mdev_supported_types/\
i915-GVTg_V5_8/available_instances
```

本机 profile：

| profile | 最大分辨率 | 空闲时最大实例数 |
| --- | ---: | ---: |
| `i915-GVTg_V5_4` | 1920x1200 | 1 |
| `i915-GVTg_V5_8` | 1024x768 | 2 |

不同 profile 共享 GPU 资源，以 `available_instances` 的当前值为准。

Proxmox 使用固定 UUID：

```text
<8 位 hostpci 索引>-0000-0000-0000-<12 位 VMID>
```

示例：

| VM 配置 | UUID |
| --- | --- |
| VM 102 的 `hostpci0` | `00000000-0000-0000-0000-000000000102` |
| VM 107 的 `hostpci0` | `00000000-0000-0000-0000-000000000107` |
| VM 107 的 `hostpci1` | `00000001-0000-0000-0000-000000000107` |

下面以已部署的 VM 102、`hostpci0`、`i915-GVTg_V5_8` 为例。

### 2. 配置 PVE jail

备份并编辑 Jailmaker 配置：

```shell
ssh nas
cd /mnt/app/jailmaker
sudo cp -a jails/pve/config jails/pve/config.before-gvt-vgpu
sudo ./jlmkr.py edit pve
```

在 `systemd_nspawn_user_args` 中增加：

```text
  --bind=/dev/vfio
```

保持与该配置块其他参数相同的缩进。不要映射可写 `/sys`，也不要给 PVE jail 添加 `/dev/dri`。

在现有 `pre_start_hook` 末尾增加：

```shell
  modprobe kvmgt
  modprobe vfio_iommu_type1
  test -e /sys/bus/mdev/devices/00000000-0000-0000-0000-000000000102 || echo 00000000-0000-0000-0000-000000000102 > /sys/bus/pci/devices/0000:00:02.0/mdev_supported_types/i915-GVTg_V5_8/create
```

增加或扩展 `post_stop_hook`：

```shell
post_stop_hook=#!/usr/bin/bash
  set -euo pipefail
  test ! -e /sys/bus/mdev/devices/00000000-0000-0000-0000-000000000102/remove || echo 1 > /sys/bus/mdev/devices/00000000-0000-0000-0000-000000000102/remove
```

首次添加 `/dev/vfio` bind 时，需要重启 PVE jail。先在 PVE 中停止 VM：

```shell
ssh root@192.168.123.3
qm shutdown 102 --timeout 180
qm shutdown 107 --timeout 180
exit
```

然后在 TrueNAS 上重启 jail：

```shell
cd /mnt/app/jailmaker
sudo ./jlmkr.py stop pve
sudo ./jlmkr.py start pve
```

确认 mdev 和 VFIO 节点：

```shell
readlink -f \
  /sys/bus/mdev/devices/00000000-0000-0000-0000-000000000102/iommu_group
sudo ls -l /dev/vfio
```

### 3. 在 PVE 中分配并启动 VM

```shell
ssh root@192.168.123.3
qm set 102 --hostpci0 \
  '0000:00:02.0,mdev=i915-GVTg_V5_8,pcie=1'
qm config 102 | grep '^hostpci0:'
qm start 102
```

如果提示 mdev 已存在并正在复用，这是正常行为：

```text
mdev instance '00000000-0000-0000-0000-000000000102' already existed, using it.
```

### 4. 验证

在 PVE 中确认 QEMU 使用正确 UUID：

```shell
qm status 102
qm showcmd 102 --pretty | grep -E 'vfio-pci|sysfsdev'
```

Linux 来宾中确认 GPU 和渲染节点：

```shell
lspci -nn | grep -Ei 'display|vga|3e9a'
ls -l /dev/dri
dmesg | grep -Ei 'i915|drm'
```

本机 VM 102 的预期结果是识别 `8086:3e9a`，并出现 `/dev/dri/renderD128`。

确认 Docker/Jellyfin 路径未受影响：

```shell
ssh nas
lspci -Dnnk -s 0000:00:02.0
ls -l /dev/dri/renderD128
systemctl cat jlmkr-docker.service | grep -- '--bind=/dev/dri'
```

### 5. 给另一台 VM 增加 vGPU

以下以 VM 107 的 `hostpci0` 和第二个 `i915-GVTg_V5_8` 为例。

1. 检查 `i915-GVTg_V5_8/available_instances` 至少为 `1`。
2. 在 `pre_start_hook` 添加 VM 107 的创建命令。
3. 在 `post_stop_hook` 添加 VM 107 的删除命令。
4. 如果 PVE jail 已经映射 `/dev/vfio`，无需重启 jail，可在宿主立即创建 mdev。

宿主钩子中需要增加：

```shell
# pre_start_hook
test -e /sys/bus/mdev/devices/00000000-0000-0000-0000-000000000107 || echo 00000000-0000-0000-0000-000000000107 > /sys/bus/pci/devices/0000:00:02.0/mdev_supported_types/i915-GVTg_V5_8/create

# post_stop_hook
test ! -e /sys/bus/mdev/devices/00000000-0000-0000-0000-000000000107/remove || echo 1 > /sys/bus/mdev/devices/00000000-0000-0000-0000-000000000107/remove
```

当前不重启 jail 时，在 TrueNAS 宿主创建：

```shell
UUID=00000000-0000-0000-0000-000000000107
TYPE=i915-GVTg_V5_8
test -e /sys/bus/mdev/devices/$UUID || \
  echo "$UUID" | sudo tee \
  /sys/bus/pci/devices/0000:00:02.0/mdev_supported_types/$TYPE/create
```

然后在 PVE 中分配：

```shell
ssh root@192.168.123.3
qm shutdown 107 --timeout 180
qm set 107 --hostpci0 \
  '0000:00:02.0,mdev=i915-GVTg_V5_8,pcie=1'
qm start 107
qm showcmd 107 --pretty | grep -E 'vfio-pci|sysfsdev'
```

### 6. 删除 VM 的 vGPU

先在 PVE 中停止 VM 并删除配置：

```shell
qm shutdown 107 --timeout 180
qm set 107 --delete hostpci0
```

从 Jailmaker 配置的 `pre_start_hook` 和 `post_stop_hook` 删除该 UUID 对应的两行，再在 TrueNAS 宿主删除 mdev：

```shell
UUID=00000000-0000-0000-0000-000000000107
test ! -e /sys/bus/mdev/devices/$UUID/remove || \
  echo 1 | sudo tee /sys/bus/mdev/devices/$UUID/remove
```

如果要完全撤销 PVE vGPU，还应删除所有 mdev 钩子和 `--bind=/dev/vfio`，然后重启 PVE jail。

### 7. 常见错误

- `no available instances`：profile 资源已用完，删除不用的 mdev 或选择有容量的 profile。
- `Read-only file system`：PVE jail 正在尝试写 sysfs；应由 TrueNAS 宿主创建 mdev，不要把宿主 `/sys` 改为全局可写。
- `/dev/vfio/<group>` 不存在：确认宿主 mdev 已创建，且 PVE jail 启动参数包含 `--bind=/dev/vfio`。
- VM 启动失败：执行 `qm set VMID --delete hostpciN` 回滚 PVE 配置，再清理对应宿主钩子和 mdev。
