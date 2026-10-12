---
type: 项目
status: 整理中
tags:
  - 技术/Linux
project: 猫咪监控
created: "2026-08-11"
updated: "2026-08-11"
up: "[[00_总览与学习路线]]"
---

# 03 Buildroot 镜像构建（rk3588_sdk）

> 官方文档：https://docs.radxa.com/rock5/rock5c/other-os/buildroot
> 本地目录：`/home/wbh808/rk3588_sdk`（Rockchip 官方 Linux SDK，Buildroot 为其 rootfs 组件）
> 学习目标：理解 Rockchip SDK 的"整包构建"原理，掌握 ROCK 5C 的 Buildroot 镜像构建，以及 rootfs/内核的定制方法。这条路线最适合"极简专用固件"。

---

## 1. 原理：rk3588_sdk 是怎么工作的

### 1.1 SDK 组成

RK3588 Linux SDK 不是单一工具，而是一个"多组件编排"工程：

```text
/home/wbh808/rk3588_sdk/
├── u-boot/            # U-Boot 源码与构建（rockchip 分支）
├── kernel-6.1/        # 内核源码（kernel -> kernel-6.1 软链接）
├── buildroot/         # Buildroot（rootfs 生成）
├── rkbin/             # Rockchip 闭源二进制（DDR 初始化、miniloader、bl31 等）
├── device/rockchip/   # 板级 defconfig、分区表、打包脚本
│   ├── common/        # 通用构建脚本（build.sh、mk-*.sh）
│   └── rk3588/        # RK3588 各板卡 defconfig
├── external/          # 第三方组件（rkwifibt 等）
├── tools/             # 打包/烧录工具（upgrade_tool 等）
├── output/            # 构建输出（firmware/update.img 等）
├── rockdev -> output/firmware
├── build.sh -> device/rockchip/common/scripts/build.sh
└── Makefile -> device/rockchip/common/Makefile
```

### 1.2 构建流程

```bash
./build.sh
  │
  ├─ 选择 defconfig（rockchip_rk3588s_rock_5c_defconfig）
  │     ├─ 内核设备树：rk3588s-rock-5c
  │     └─ 各组件配置（rootfs 类型、分区、功能开关）
  ├─ 依次构建：
  │     ├─ rkbin（固件）
  │     ├─ u-boot（生成 uboot.img）
  │     ├─ kernel（生成 boot.img，含 dtb + Image + ramdisk）
  │     ├─ buildroot（生成 rootfs.img）
  │     └─ recovery（生成 recovery.img）
  ├─ 生成 misc.img / parameter.txt / oem.img / userdata.img
  └─ mk-updateimg.sh 打包 → rockdev/update.img（RK 格式整包）
```

### 1.3 输出物

本机已构建过，产物在：

```text
/home/wbh808/rk3588_sdk/output/firmware/
├── update.img        # 完整 RK 格式镜像（烧录用）
├── boot.img          # 内核 + dtb + ramdisk
├── uboot.img         # u-boot
├── rootfs.img        # buildroot rootfs
├── MiniLoaderAll.bin # 一级 loader
├── parameter.txt     # 分区表
└── recovery.img / oem.img / userdata.img / misc.img
```

---

## 2. 环境准备与源码获取

### 2.1 依赖（官方要求 Ubuntu 20.04/22.04 x86_64，磁盘 ≥ 120G）

```bash
sudo apt update
sudo apt install python2 git rsync gcc g++ make device-tree-compiler bc flex bison lz4 \
     libssl-dev libgmp-dev libmpc-dev expect file unzip bzip2 fakeroot bsdmainutils
sudo ln -s /bin/python2 /bin/python   # 部分脚本需要 python2
```

### 2.2 获取源码（本机已完成，备查）

本机是从 `/mnt/hgfs/rk3588_linux6.1_rkr4_sdk.repo.tar` 解压得到：

```bash
tar xvf rk3588_linux6.1_rkr4_sdk.repo.tar
cd rk3588_sdk
.repo/repo/repo sync -l     # 仅用本地缓存的 repo 同步（无需网络）
```

### 2.3 添加 ROCK 5C 支持（官方流程，本机已包含）

官方要求从 Radxa 维护的仓库切分支（本机 SDK 已预置，可跳过）：

```bash
cd device/rockchip
git remote add radxa https://github.com/radxa/device-rockchip.git
git fetch radxa
git checkout -b rk3588-linux-6.1 remotes/radxa/rk3588-linux-6.1

cd ../u-boot && git remote add radxa https://github.com/radxa/u-boot.git
git fetch radxa && git checkout -b next-dev-buildroot remotes/radxa/next-dev-buildroot

cd ../kernel && git remote add radxa https://github.com/radxa/kernel.git
git fetch radxa && git checkout -b linux-6.1-stan-rkr4.1-buildroot remotes/radxa/linux-6.1-stan-rkr4.1-buildroot

cd ../external/rkwifibt && git remote add radxa https://github.com/radxa/rkwifibt.git
git fetch radxa && git checkout -b develop remotes/radxa/develop

cd ../buildroot && git remote add radxa https://github.com/radxa/buildroot.git
git fetch radxa && git checkout -b rockchip/2024.02 remotes/radxa/rockchip/2024.02
```

---

## 3. 构建步骤

### 3.1 首次配置并构建

```bash
cd /home/wbh808/rk3588_sdk
./build.sh
```

脚本会列出可选 defconfig，选择：

```text
rockchip_rk3588s_rock_5c_defconfig
```

对应文件：`/home/wbh808/rk3588_sdk/device/rockchip/rk3588/rockchip_rk3588s_rock_5c_defconfig`

内容（极简）：

```text
RK_KERNEL_DTS_NAME="rk3588s-rock-5c"
RK_USE_FIT_IMG=y
```

### 3.2 增量/重建常用命令

```bash
cd /home/wbh808/rk3588_sdk

# 只重编内核
./build.sh kernel

# 只重编 u-boot
./build.sh uboot

# 只重编 rootfs
./build.sh rootfs

# 重新打包整包镜像
./build.sh updateimg

# 全部重来
./build.sh cleanall
```

> 提示：实际入口 `device/rockchip/common/scripts/build.sh`，可用 `./build.sh help` 查看全部子命令。

### 3.3 保存自己的配置

顶层 kconfig 选择完成后执行：

```bash
make savedefconfig
```

这样你的选择会固化到 defconfig，之后 `./build.sh` 可重复使用。

---

## 4. 定制方法

### 4.1 rootfs 定制（Buildroot 本体）

Buildroot 的配置在 `/home/wbh808/rk3588_sdk/buildroot/`：

```bash
cd /home/wbh808/rk3588_sdk/buildroot
make rockchip_rk3588_defconfig        # 使用 Rockchip 基础配置
make menuconfig                        # 交互式选择软件包
make savedefconfig                     # 保存
```

然后回 SDK 顶层执行 `./build.sh rootfs` 重新生成 rootfs.img。

常用定制项：

| 想做的事 | 配置项示例 |
| --- | --- |
| 加 Python | `BR2_PACKAGE_PYTHON3=y`（含 `BR2_PACKAGE_PYTHON3_*` 子项） |
| 加网络工具 | `BR2_PACKAGE_DROPBEAR=y`（SSH）、`BR2_PACKAGE_IW=y`、`BR2_PACKAGE_WPA_SUPPLICANT=y` |
| 加文件系统工具 | `BR2_PACKAGE_E2FSPROGS=y`、`BR2_PACKAGE_DOSFSTOOLS=y` |
| WiFi 固件（ROCK 5C 的 AIC8800） | `BR2_PACKAGE_RKWIFIBT_AIC8800D80_USB_FIRMWARE=y` |
| 系统工具 | `BR2_PACKAGE_BUSYBOX`（默认）、`BR2_PACKAGE_UTIL_LINUX` |

> ROCK 5C WiFi 说明：在 SDK 顶层 `make menuconfig` 里搜索 `aic`，把 supported modules 选为 `AIC8800D80 USB`，同时在 `buildroot/configs/rockchip_rk3588_defconfig` 里加 `BR2_PACKAGE_RKWIFIBT_AIC8800D80_USB_FIRMWARE=y`，驱动和固件才会进 rootfs。

### 4.2 内核定制

内核源码在 `/home/wbh808/rk3588_sdk/kernel-6.1`，基线配置：

```text
kernel-6.1/arch/arm64/configs/rockchip_linux_defconfig
```

```bash
cd /home/wbh808/rk3588_sdk/kernel-6.1
make ARCH=arm64 rockchip_linux_defconfig
make ARCH=arm64 menuconfig
make ARCH=arm64 savedefconfig     # 生成精简 defconfig
cp defconfig arch/arm64/configs/rockchip_linux_defconfig
cd /home/wbh808/rk3588_sdk && ./build.sh kernel
```

设备树：`kernel-6.1/arch/arm64/boot/dts/rockchip/rk3588s-rock-5c.dts`（改完同样重编内核）。

### 4.3 rootfs overlay（往镜像里塞自己的文件，最常用）

Buildroot 支持"overlay 目录"：把 `BR2_ROOTFS_OVERLAY` 指向的目录原样拷贝进 rootfs。

在 `buildroot/configs/rockchip_rk3588_defconfig` 中加：

```text
BR2_ROOTFS_OVERLAY="$(BR2_EXTERNAL_ROCKCHIP_PATH)/board/rockchip/rk3588/overlay"
```

然后把你的文件（AI 运行时、模型、脚本、systemd unit）放进该 overlay 目录，重编 rootfs 即可。

### 4.4 post-build / post-image 脚本

在 defconfig 中指定：

```text
BR2_ROOTFS_POST_BUILD_SCRIPT="$(BR2_EXTERNAL_ROCKCHIP_PATH)/board/rockchip/common/post-build.sh"
BR2_ROOTFS_POST_IMAGE_SCRIPT="$(BR2_EXTERNAL_ROCKCHIP_PATH)/board/rockchip/common/post-image.sh"
```

post-build 脚本在 rootfs 生成后、打包前执行，适合：删除多余文件、修改 /etc、设置默认用户等。

---

## 5. 烧录

### 5.1 eMMC（Linux / macOS）

使用 `upgrade_tool`（本机路径 `rk3588_sdk/tools/linux/Linux_Upgrade_Tool/Linux_Upgrade_Tool/upgrade_tool`，或 `rk3588_sdk/rkbin/tools/upgrade_tool`）通过 Type-A 烧录口：

```bash
sudo /home/wbh808/rk3588_sdk/tools/linux/Linux_Upgrade_Tool/Linux_Upgrade_Tool/upgrade_tool \
     uf /home/wbh808/rk3588_sdk/rockdev/update.img
```

Windows 用 RKDevTool 对应操作。

### 5.2 SD 卡

Windows 下用瑞芯微 SDDiskTool（SD_Firmware_Tool.exe），选择"SD 启动"，升级固件选 `update.img`。

如果提示清空 MBR 失败，把 SD 卡标签从 dos 转为 gpt：

```bash
sudo fdisk /dev/sdb      # 设备名按实际
# 输入 p 查看 → 输入 g 转 GPT → 输入 wq 保存
```

Linux 下也可以直接 dd 部分镜像，但 RK 格式整包更推荐 SDDiskTool；若手上有 `*.img` 的裸盘镜像（如 Yocto 的 wic），可：

```bash
sudo dd if=xxx.wic of=/dev/sdb bs=4M conv=fsync status=progress
```

---

## 6. 常见问题

**Q1：./build.sh 报 python2 缺失？**

执行 `sudo ln -s /bin/python2 /bin/python`（Ubuntu 22.04 需要先 `apt install python2`）。

**Q2：WiFi 不工作？**

确认 defconfig 里 AIC8800D80 USB 驱动与固件宏都已开启（见 4.1），重编 kernel + rootfs 后重新烧录。

**Q3：update.img 很小/启动不了？**

检查 `rockdev` 下各 img 是否齐全；确认 `parameter.txt` 分区表与镜像匹配；eMMC 烧录用 upgrade_tool，SD 卡用 SDDiskTool。

**Q4：想用 Debian 的 apt 包？**

Buildroot 不是 Debian 系，apt 包不能直接用。要么选配 Buildroot 包，要么改用 rsdk（文档 04）。

---

## 7. 本节产出与下一步

完成本节后：

- 能一键构建 ROCK 5C 整包 `update.img`
- 能用 defconfig / menuconfig / overlay 定制 rootfs
- 能裁剪内核并固化配置

下一步：

- 想要 Debian 系生态 + AI 包 → 文档 04（rsdk）
- 综合实战（Buildroot 作为"极简固件"备选方案）→ 文档 06
