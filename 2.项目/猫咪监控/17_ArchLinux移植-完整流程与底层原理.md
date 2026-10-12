---
type: 项目
status: 整理中
tags:
  - 技术/Linux
project: 猫咪监控
created: "2026-08-19"
updated: "2026-08-19"
---

# 17 Arch Linux 移植：完整操作流程与底层原理

> 前置：建议先读完 00（总览）、01（BSP：u-boot/内核）与 15（勘误与补充）。
> 本文档解决一个前面 01~07 没覆盖的问题：**不依赖 Debian 系，把 Arch Linux 移植到 ROCK 5C**，
> 并讲清楚从 BootROM 到 rootfs 每一层在干什么、哪些地方是"移植点"。
> 文档中的命令均可在本机（x86_64 Ubuntu/Debian）直接执行，产物烧到 ROCK 5C 的 SD 卡/eMMC。

---

## 1. 先想清楚："给板子移植一个发行版"到底在移什么

### 1.1 移植 = 三层东西全部能跑起来

任何一块 ARM 开发板要跑一个发行版，本质上是把下面三层依次点亮：

```text
┌────────────────────────────────────────────────────────────┐
│ 第 3 层  rootfs（发行版用户态）                              │
│         Arch：systemd + pacman + 自建包仓库 + 你的应用       │
├────────────────────────────────────────────────────────────┤
│ 第 2 层  Linux 内核（Image + dtb + initramfs + 模块）        │
│         识别板卡外设：MMC/USB/网络/NPU/GPU/显示/串口         │
├────────────────────────────────────────────────────────────┤
│ 第 1 层  Bootloader（TPL/SPL → U-Boot → ATF）               │
│         初始化 DDR → 加载内核 → 传递设备树 → 交棒            │
├────────────────────────────────────────────────────────────┤
│ 第 0 层  BootROM（固化在 SoC 内，不可改）                   │
│         上电后按固定偏移从介质读取第一段引导代码             │
└────────────────────────────────────────────────────────────┘
```

所谓"移植 Arch Linux"，不是把 Arch 源码重写一遍，而是：

1. **让第 1 层认这块板**：用 ROCK 5C 的 defconfig/设备树构建 U-Boot，写入正确偏移；
2. **让第 2 层认这块板**：内核开启 Rockchip/RK3588 平台支持，提供 `rk3588s-rock-5c.dtb`，
   编译出 arm64 的 `Image` 与内核模块；
3. **让第 3 层变成"你的发行版"**：用 Arch Linux ARM（ALARM）的 aarch64 用户态，
   再把自己的内核、驱动、AI 运行时打成 pacman 包、放进自建仓库。

这三步每一层都对应着明确的源码与产物，也是本文第 5 节操作流程的骨架。

### 1.2 为什么在 ROCK 5C 上做 Arch（而不是继续用 Debian 系）

| 维度 | rsdk/Buildroot/Yocto（文档 02~04、06） | Arch Linux（本文） |
| --- | --- | --- |
| 包管理 | apt / 源码级（bitbake/make） | pacman，包格式统一、事务型数据库 |
| 更新节奏 | 版本固定（bookworm 等） | 滚动更新，内核/用户态持续跟进 |
| 定制方式 | 改 jsonnet/recipe/defconfig 后重编镜像 | 改 PKGBUILD → 自建仓库 → 镜像只是"快照" |
| 适合场景 | 量产、稳定、最小体积 | 个人/极客、持续演进、完全掌控的 DIY 发行版 |
| 成本 | 工具链重、学习曲线陡（尤其 Yocto） | 概念简单（PKGBUILD + repo-add），但要自己维护 |

Arch 的哲学是"发行版 = 包仓库 + 一个最小安装流程"。这恰恰是**定制发行版最省力**的模型：
你不需要每次改动都重新构建整个 rootfs，只要重建受影响的包、更新仓库，板子上 `pacman -Syu`
即可完成一次"发行版升级"。

### 1.3 ALARM 与 Arch 的关系（重要，避免概念混淆）

- **Arch Linux ARM（ALARM）** 是社区维护的 ARM 移植版，提供 aarch64 的软件仓库和基础 rootfs
  压缩包（`ArchLinuxARM-aarch64-latest.tar.gz`）。它不等于 Arch Linux 官方项目，但包格式、
  pacman、PKGBUILD 机制完全一致。
- ALARM 官方只提供**通用的 aarch64 用户态**，不给每块开发板做官方镜像。是否支持 ROCK 5C，
  取决于你提供的内核与 bootloader——这正是本文要做的"移植"。
- "定制 Arch 发行版" = **ALARM 用户态 + 自编内核 + 自建软件仓库 + 自定义镜像流水线**。
  做到最后，你的产品甚至不需要依赖 ALARM 官方内核包，自己的仓库就是发行版的"上游"。

---

## 2. 底层原理：RK3588 启动链

### 2.1 一图看懂完整启动过程

```text
上电
 │
 ▼
BootROM（SoC 固化代码，不可修改）
 │  按介质固定偏移读取 → SD/eMMC 第 64 扇区（0x8000）
 ▼
idbloader.img  = TPL（DDR 初始化 blob）+ SPL
 │  TPL：用 rkbin 的 DDR 固件初始化 LPDDR5（本机 rkbin 里有 2400MHz 版）
 │  SPL：初始化时钟/存储/串口，加载下一段
 ▼
u-boot.itb  = U-Boot proper + ATF BL31 + U-Boot 设备树（FIT 打包）
 │  BL31 进入 EL3 提供 PSCI（电源/CPU 管理）
 │  U-Boot 扫描各分区，找 /boot/extlinux/extlinux.conf（bootstd）
 ▼
内核 Image + initramfs + rk3588s-rock-5c.dtb
 │  U-Boot 把 dtb 放到内存指定位置，用 booti 跳转
 ▼
Linux 内核
 │  按 dtb 枚举外设 → 挂载 initramfs → 挂载真正的 rootfs
 ▼
Arch 用户态（systemd → getty/sshd/你的服务）
```

### 2.2 每一阶段的职责与"移植点"

| 阶段 | 代码/产物 | 职责 | 移植点（你要做的事） |
| --- | --- | --- | --- |
| BootROM | SoC 固化 | 从固定偏移读 idbloader | 不可改，只能遵守它的约定 |
| TPL | rkbin 的 DDR bin（或 U-Boot TPL） | 初始化 DDR/时钟 | 选对 DDR 固件（LPDDR5 2400MHz，见 5.2） |
| SPL | idbloader 的一部分 | 初始化存储/串口，加载 u-boot.itb | U-Boot 构建参数 |
| ATF BL31 | rkbin 预编译 elf，打进 u-boot.itb | EL3 运行时、PSCI | 用 rkbin 提供的 `rk3588_bl31_*.elf` |
| U-Boot proper | u-boot.itb | 引导策略、加载内核 | `rock-5c-rk3588s_defconfig`、extlinux.conf |
| 内核 | Image / initramfs / dtb | 驱动外设、挂载 rootfs | 内核 config + `rk3588s-rock-5c.dtb` |
| rootfs | Arch 用户态 | systemd、pacman、应用 | 包清单、自建仓库、systemd 单元 |

### 2.3 启动介质布局与偏移（为什么是这些数字）

RK3588 的 BootROM 与 Rockchip 的固件约定如下（512B/扇区）：

| 介质 | 内容 | 偏移 | 说明 |
| --- | --- | --- | --- |
| SD/eMMC | idbloader | 扇区 64（0x8000，32KB） | TPL+SPL，BootROM 固定读取点 |
| SD/eMMC | u-boot.itb | 扇区 16384（0x800000，8MB） | U-Boot+BL31+dtb，SPL 固定加载点 |
| SD/eMMC | 第一个分区 | 扇区 32768（16MB） | GPT 分区从此开始，留足 bootloader 空间 |
| SPI NOR（rock-5c-spi） | idbloader | 0x0 | 与 eMMC/SD 不同，见 U-Boot 文档 |
| SPI NOR（rock-5c-spi） | u-boot.itb | 0x100000（以实际 config 为准） | — |

原理要点：

1. **前 16MB 是"禁区"**：BootROM/SPL 只认绝对偏移，因此分区表（GPT 头在 LBA1）与 bootloader
   可以共存——GPT 分区 1 从 16MB 开始，loader 区域（32KB~8MB）正好落在分区表之后、分区 1 之前。
   这也是为什么 `dd` 必须写到**整个设备**（`/dev/sdX`），而不是某个分区。
2. **TPL 与 SPL 的分工**：上电时 DDR 尚未初始化，芯片只能把一小段代码搬进 SRAM 执行，
   这段代码（TPL/DDR blob）负责"点亮内存"；内存可用后 SPL 才能把 8MB 处的 u-boot.itb 载入 DDR。
3. **u-boot.itb 是 FIT 打包**：一个文件里装着 U-Boot 本体、BL31、设备树，SPL 用 `bootm`/`booti`
   解析后分发到不同 EL（异常等级）执行。

### 2.4 U-Boot 如何找到内核：bootstd / extlinux

现代 U-Boot（v2023+）用 **bootstd** 统一引导流程：

```text
bootflow scan -lb
  → 扫描所有存储设备的所有分区
  → 查找 /boot/extlinux/extlinux.conf（或 EFI 启动项）
  → 按配置加载 Image / initramfs / dtb
  → booti（arm64 Image 专用引导命令）跳转
```

所以我们给 ROCK 5C 做 Arch 时，关键文件是：

```text
/boot/extlinux/extlinux.conf      ← U-Boot 的"引导菜单"
/boot/Image                       ← arm64 内核（EFI stub，booti 可直接跳）
/boot/initramfs-linux.img         ← mkinitcpio 生成
/boot/dtbs/rockchip/rk3588s-rock-5c.dtb
```

设备树在这里是**内核的"硬件清单"**：U-Boot 把它加载进内存并传给内核，内核按 dtb 里的节点
枚举外设。所以"板级差异"几乎都收敛在 dtb + 内核 config 两层。

### 2.5 内核怎么知道自己"是 ROCK 5C"

1. dtb 根节点 `compatible = "radxa,rock-5c", "rockchip,rk3588"`；
2. 内核的 `arch/arm64/boot/dts/rockchip/rk3588s-rock-5c.dts` 定义了板级外设
   （PMIC、以太网 PHY、eMMC、USB、串口、WiFi 的 SDIO 等），编译产物即 dtb；
3. 内核 config 决定哪些驱动**编进内核/编成模块**，哪些干脆不编。

所以"移植内核"最少要保证三样：`Image`（arm64 内核）、`rk3588s-rock-5c.dtb`（板级设备树）、
`/usr/lib/modules/<版本>/`（与板载硬件匹配的模块）。缺一样就启动失败或外设不工作。

---

## 3. 技术现状核对（2026-08，均已核实）

| 组件 | 现状 | 说明 |
| --- | --- | --- |
| 主线 Linux 内核 | **v6.13 起**自带 `rk3588s-rock-5c.dtb` | 2024-11 合入，dtb 名为 `rk3588s-rock-5c` |
| 主线 U-Boot | **v2025.04 起**支持 `rock-5c-rk3588s_defconfig` | 2025-01 合入；master 文档明确列出 ROCK 5C/5C Lite |
| 本机 bsp 的 Radxa 分支 | U-Boot `rock-5c-rk3588s_defconfig`、内核 dts `rk3588s-rock-5c.dts` 均已存在 | 见 `/home/wbh808/bsp/.src/` |
| ALARM aarch64 rootfs | 官方通用包：`ArchLinuxARM-aarch64-latest.tar.gz` | 含 base 用户态 + `linux-aarch64` 内核包 + 全部 dtb |
| 板载 WiFi6/BT（AIC8800D80） | **不在主线内核**，需树外驱动 | Radxa 提供 `radxa-pkg/aic8800`（WiFi+BT）与固件；AUR 有 `aic8800d80-dkms` |
| NPU（rknpu） | **不在主线内核**，树外驱动 | 本机已有 `rknpu_driver_0.9.8_20241009.tar.bz2`；厂商内核已内置 |
| GPU Mali-G610 | 主线驱动 `panthor` | 无桌面需求可不管 |
| 视频编解码 VPU/MPP | 主线仅部分支持 | 要完整硬编建议用厂商内核 + `rockchip-mpp` 用户态 |
| 摄像头 ISP/CIF | 主线有 rockchip CIF/ISP 驱动；ROCK 5C dtb 无 camera 节点 | 结论同文档 15：MIPI 摄像头需自写 overlay，先用 USB UVC |

### 3.1 本机已有、可直接复用的资源

```text
交叉工具链
  /mnt/hgfs/gcc-arm-10.2-2020.11-x86_64-aarch64-none-linux-gnu.tar.xz
  /mnt/hgfs/gcc-arm-11.2-2022.02-x86_64-aarch64-none-linux-gnu.tar.xz

固件（rkbin，bsp 已拉好）
  /home/wbh808/bsp/.src/rkbin/bin/rk35/rk3588_bl31_v1.54.elf
  /home/wbh808/bsp/.src/rkbin/bin/rk35/rk3588_ddr_lp4_2112MHz_lp5_2400MHz_v1.22.bin
  /home/wbh808/bsp/.src/rkbin/bin/rk35/rk3588_spl_loader_v1.22.114.bin   ← MaskROM 用 loader

内核/U-Boot 源码（bsp 工作树，可直接复用或参考）
  /home/wbh808/bsp/.src/linux      # Radxa 分支（rk2410 / 6.1）
  /home/wbh808/bsp/.src/u-boot     # Radxa 分支（next-dev-v2024.10）

NPU 内核驱动（v0.9.8，RKLLM 要求 ≥0.9.8）
  /home/wbh808/rknn-llm-release-v1.2.3/rknpu-driver/rknpu_driver_0.9.8_20241009.tar.bz2

NPU/LLM 用户态库
  /home/wbh808/rknn-toolkit2-2.3.2/rknpu2/runtime/Linux/librknn_api/aarch64/librknnrt.so
  /home/wbh808/rknn-llm-release-v1.2.3/rkllm-runtime/Linux/librkllm_api/aarch64/librkllmrt.so
```

---

## 4. 方案选型：四条路，先选再动手

| 方案 | bootloader | 内核 | rootfs | 特点 | 适合谁 |
| --- | --- | --- | --- | --- | --- |
| A 快速 | 官方/现成（bsp deb 或 RadxaOS 提取） | ALARM 官方 `linux-aarch64`（≥6.13） | ALARM tarball | 半小时出系统，最小工作量 | 先跑通、验证板子 |
| **B 全源码（本文主线）** | 主线 U-Boot v2025.04+ 自编 | 主线内核自编 | ALARM tarball / pacstrap | 全程可控，真正理解移植 | 学习 + DIY 发行版 |
| C 厂商内核 | Radxa 分支自编或官方 | Radxa `linux-rk2410`（6.1-rkr4.x） | ALARM tarball | 功能最全（NPU/VPU/WiFi 省事） | 要上 AI/多媒体产品 |
| D UEFI（进阶） | EDK2 UEFI 固件 | 通用 aarch64 UEFI 内核 | archiso-aarch64 通用 ISO | 像装 x86 一样装 Arch | 想走标准 UEFI 路线 |

推荐顺序：**A 验证板子 → B 理解并掌控 → 有 AI/多媒体需求时切 C 的内核**。
下面以 B 为主线写完整流程，A/C 的关键差异单独标注。

---

## 5. 完整操作流程（方案 B：全源码主线路线）

### 5.1 环境准备

```bash
# 1) 主机依赖
sudo apt update
sudo apt install -y git wget curl bsdtar rsync binfmt-support qemu-user-static \
    arch-install-scripts gdisk dosfstools e2fsprogs make gcc flex bison bc \
    libssl-dev libncurses-dev device-tree-compiler python3

# 2) 交叉工具链（解压到 /opt，变量名按自己环境改）
sudo mkdir -p /opt
sudo tar -xJf /mnt/hgfs/gcc-arm-11.2-2022.02-x86_64-aarch64-none-linux-gnu.tar.xz -C /opt
export PATH=/opt/gcc-arm-11.2-2022.02-x86_64-aarch64-none-linux-gnu/bin:$PATH
aarch64-none-linux-gnu-gcc --version   # 验证

# 3) 工作目录
mkdir -p ~/rock5c-arch/{bootloader,kernel,rootfs,repo,output}
cd ~/rock5c-arch

# 4) 下载源码（主线）
git clone --depth 1 --branch v2025.04 https://github.com/u-boot/u-boot.git u-boot
git clone --depth 1 --branch v6.13    https://github.com/torvalds/linux.git linux

# rkbin（含 BL31 与 DDR blob；本机 bsp 里已有，直接用符号链接即可）
ln -s /home/wbh808/bsp/.src/rkbin rkbin

# 5) ALARM rootfs 压缩包
wget http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz
```

> 提示：`linux-aarch64` 内核包要 ≥6.13 才有 ROCK 5C 的 dtb；如果 tarball 自带的内核版本偏旧，
> 先 `pacman -Syu` 升级（见 5.6）。

### 5.2 构建 U-Boot（第 1 层）

```bash
cd ~/rock5c-arch/u-boot

export CROSS_COMPILE=aarch64-none-linux-gnu-
export BL31=../rkbin/bin/rk35/rk3588_bl31_v1.54.elf
export ROCKCHIP_TPL=../rkbin/bin/rk35/rk3588_ddr_lp4_2112MHz_lp5_2400MHz_v1.22.bin

make rock-5c-rk3588s_defconfig
make -j$(nproc)

ls -l idbloader.img u-boot.itb u-boot-rockchip.bin
cp idbloader.img u-boot.itb u-boot-rockchip.bin ../bootloader/
```

原理与要点：

- `ROCKCHIP_TPL` 指 DDR 初始化固件。ROCK 5C 是 **LPDDR5（最高 2400MHz）**，
  所以用 `rk3588_ddr_lp4_2112MHz_lp5_2400MHz_*.bin`；选错型号会卡在 DDR 初始化阶段。
- `BL31` 是预编译的 ATF 固件（Rockchip 未开源全部 EL3 代码），U-Boot 构建时把它打进 u-boot.itb。
- 产物三种：`idbloader.img`（TPL+SPL）、`u-boot.itb`（U-Boot+BL31+dtb）、
  `u-boot-rockchip.bin`（binman 把前两者按偏移拼好的**整包**，写入时一条 dd 即可）。
- 想用 Radxa 分支（方案 C）时，命令同样适用，只是源码换成本机
  `/home/wbh808/bsp/.src/u-boot`，defconfig 同名（`rock-5c-rk3588s_defconfig`）。

### 5.3 构建内核（第 2 层）

```bash
cd ~/rock5c-arch/linux

export ARCH=arm64
export CROSS_COMPILE=aarch64-none-linux-gnu-

# 配置基线（二选一）：

# a) 通用基线：arm64 defconfig 已含 ARCH_ROCKCHIP 与大部分主流驱动
make defconfig
# b) 更贴近发行版：直接采用 ALARM linux-aarch64 包的 .config

#    （github.com/archlinuxarm/PKGBUILDs → core/linux-aarch64）

# 关键项自查（scripts/config 可批量开关；menuconfig 可交互）：
grep -E "CONFIG_ARCH_ROCKCHIP|CONFIG_MMC_DW_ROCKCHIP|CONFIG_USB_DWC3|CONFIG_SERIAL_8250_DW" .config

# 编译
make -j$(nproc) Image dtbs modules

# 产物
ls -l arch/arm64/boot/Image
ls -l arch/arm64/boot/dts/rockchip/rk3588s-rock-5c.dtb
```

安装到 rootfs（先有 rootfs 再执行，见 5.4）：

```bash
# 假设 rootfs 已解包到 ~/rock5c-arch/rootfs
make INSTALL_MOD_PATH=../rootfs modules_install
mkdir -p ../rootfs/boot/dtbs/rockchip
cp arch/arm64/boot/Image ../rootfs/boot/Image
cp arch/arm64/boot/dts/rockchip/rk3588s-rock-5c.dtb ../rootfs/boot/dtbs/rockchip/
```

可选：NPU 驱动（AI 部署必需，详见 6.4）。主线内核默认没有 rknpu 驱动，需要把树外驱动
合入内核树后重编；如果暂时不需要 AI，先跳过不影响启动。

### 5.4 构建根文件系统（第 3 层）

**方法一：解包 ALARM 官方 tarball（最快，推荐起步）**

```bash
mkdir -p ~/rock5c-arch/rootfs
# 用 bsdtar 解包，保留扩展属性/ACL；root 用户直接执行
sudo bsdtar -xpf ArchLinuxARM-aarch64-latest.tar.gz -C ~/rock5c-arch/rootfs

# 进入 chroot 需要的准备
sudo cp /usr/bin/qemu-aarch64-static ~/rock5c-arch/rootfs/usr/bin/
sudo arch-chroot ~/rock5c-arch/rootfs
```

tarball 自带内容（ALARM 官方说明）：base 用户态、`linux-aarch64` 内核包、openssh、haveged、
默认 root 密码 `root`、普通用户 `alarm/alarm`、systemd-networkd 自动 DHCP（eth0/en\*）、
sshd 已启用。解包后第一件事（在 chroot 内）：

```bash
pacman-key --init
pacman-key --populate archlinuxarm
pacman -Syu     # 升级内核/用户态；国内可先配好 mirrorlist
```

**方法二：pacstrap 从零组装（做"自己的发行版"时用）**

在一台已装 ALARM 的机器（或上面的 chroot）里执行：

```bash
# -C 指定临时 pacman.conf，-c 使用缓存
pacstrap -C /tmp/pacman.conf -c /mnt/rootfs base linux-aarch64 \
    linux-firmware openssh haveged systemd-resolvconf
```

这样包清单完全由你控制，是"定制发行版"的正统做法——最终 rootfs 里装什么，
就是 pacstrap 的包列表 + 之后的显式安装，一清二楚。

**方法三（进阶）：自建仓库后再 pacstrap**——把 6.2 的自建仓库加进 pacman.conf，
pacstrap 就能直接装你自己的内核包和驱动包。这是"发行版"与"改了一台机器"的分水岭。

### 5.5 组装可启动镜像

分区规划（与 Radxa 布局一致，分区 1 从 16MB 开始，给 bootloader 留足空间）：

```text
分区 1  boot  512MB  ext4   ← /boot：extlinux.conf、Image、initramfs、dtb
分区 2  root  剩余    ext4   ← /
```

```bash
cd ~/rock5c-arch
IMG=output/rock5c-arch.img

# 1) 建 8GB 空镜像与 GPT
truncate -s 8G $IMG
sgdisk -Z $IMG
sgdisk -n 1:32768:+512M -t 1:8300 -c 1:boot $IMG
sgdisk -n 2:0:0        -t 2:8300 -c 2:root $IMG

# 2) 挂为 loop 设备并格式化
sudo losetup -P /dev/loop0 $IMG
sudo mkfs.ext4 /dev/loop0p1
sudo mkfs.ext4 /dev/loop0p2
sudo mkdir -p /mnt/archroot

# 3) 写 bootloader —— 注意是绝对偏移、写到整个设备，不是分区！
sudo dd if=bootloader/idbloader.img of=/dev/loop0 seek=64    conv=fsync
sudo dd if=bootloader/u-boot.itb    of=/dev/loop0 seek=16384 conv=fsync
# 或用 binman 整包（主线 U-Boot 推荐，效果等同上面两条）：

# sudo dd if=bootloader/u-boot-rockchip.bin of=/dev/loop0 seek=64 conv=fsync

# 4) 拷 rootfs
sudo mount /dev/loop0p2 /mnt/archroot
sudo mkdir -p /mnt/archroot/boot
sudo mount /dev/loop0p1 /mnt/archroot/boot
sudo rsync -aHAX ~/rock5c-arch/rootfs/ /mnt/archroot/

# 5) extlinux 引导配置
sudo mkdir -p /mnt/archroot/boot/extlinux
sudo tee /mnt/archroot/boot/extlinux/extlinux.conf >/dev/null <<'EOF'
TIMEOUT 10
DEFAULT arch

LABEL arch
    LINUX ../Image
    INITRD ../initramfs-linux.img
    FDT ../dtbs/rockchip/rk3588s-rock-5c.dtb
    APPEND root=PARTUUID=<root 分区 PARTUUID> rw rootwait console=ttyS2,1500000
EOF

# 6) fstab
sudo tee -a /mnt/archroot/etc/fstab >/dev/null <<'EOF'
# <file system> <dir> <type> <options> <dump> <pass>
PARTUUID=<boot 分区 PARTUUID> /boot ext4 defaults 0 2
PARTUUID=<root 分区 PARTUUID> /    ext4 defaults 0 1
EOF

# 7) 主机名与初始配置（可选）
echo "rock5c-arch" | sudo tee /mnt/archroot/etc/hostname

# 8) 生成 initramfs（在 chroot 内，针对自编内核版本）
sudo arch-chroot /mnt/archroot mkinitcpio -k $(ls /mnt/archroot/usr/lib/modules | head -1) \
    -g /boot/initramfs-linux.img

# 9) 收尾
sudo umount /mnt/archroot/boot /mnt/archroot
sudo losetup -d /dev/loop0
```

PARTUUID 获取方式：

```bash
sudo blkid /dev/loop0p1 /dev/loop0p2   # 复制 PARTUUID 填进上面的配置
```

> 内核包自带的 `mkinitcpio` 默认 hooks（`base udev autodetect modconf block filesystems ...`）
> 已覆盖 SD/eMMC 启动；自编内核只要模块装进 `/usr/lib/modules/<版本>`，按上面命令生成即可。

### 5.6 首次启动与验证

烧录与硬件准备：

```bash
# 镜像写入 SD/eMMC（确认设备名！）
sudo dd if=output/rock5c-arch.img of=/dev/sdX bs=4M conv=fsync status=progress
```

接线：Type-C 5V 供电、HDMI 或串口（UART2，1500000 波特）、网线。
串口接线注意 3.3V 电平，命令示例：

```bash
sudo screen /dev/ttyUSB0 1500000
# 或 minicom -D /dev/ttyUSB0 -b 1500000
```

启动后逐项验证：

```bash
uname -a                                  # 内核版本应为自编版本
cat /proc/device-tree/compatible          # 应为 radxa,rock-5c
lsblk                                     # 分区挂载
ip a                                      # systemd-networkd 应已拿到 IP
cat /etc/os-release                       # 你的发行版身份
pacman -Syu                               # 首次完整升级
```

方案 A 与 C 的差异：

- **方案 A**：跳过 5.2/5.3，rootfs 直接用 tarball 自带的 `linux-aarch64` 内核与 dtb，
  只需 5.5 的分区 + 写 bootloader + extlinux.conf。bootloader 可直接取 bsp 构建的
  u-boot deb（文档 01）里 `/usr/lib/u-boot/rock-5c/` 的 `idbloader.img`、`u-boot.itb`，
  或从官方 RadxaOS 镜像对应偏移提取。
- **方案 C**：内核源码换成 `/home/wbh808/bsp/.src/linux`（Radxa 6.1-rkr4.x，NPU/MPP 已内置），
  U-Boot 换成本机 bsp 分支；其余流程相同。

---

## 6. 从"能启动"到"定制发行版"

### 6.1 把自编内核做成 pacman 包

手动拷贝内核不是发行版的做法。参照 ALARM 的 `linux-aarch64` PKGBUILD 写一个自己的：

```bash
# PKGBUILD（示意，字段按实际裁剪）
pkgname=linux-rock5c
pkgver=6.13
pkgrel=1
arch=('aarch64')
license=('GPL2')
depends=('mkinitcpio' 'linux-firmware')
makedepends=('bc' 'perl' 'python')
source=("linux-6.13.tar.xz" "config-rock5c")

build() {
  cd linux-6.13
  cp "$srcdir/config-rock5c" .config
  make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- olddefconfig
  make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- Image dtbs modules
}

package() {
  cd linux-6.13
  make ARCH=arm64 INSTALL_MOD_PATH="$pkgdir" modules_install
  install -Dm644 arch/arm64/boot/Image "$pkgdir/boot/Image"
  install -Dm644 arch/arm64/boot/dts/rockchip/rk3588s-rock-5c.dtb \
    "$pkgdir/boot/dtbs/rockchip/rk3588s-rock-5c.dtb"
  install -Dm644 "$srcdir/linux-rock5c.preset" \
    "$pkgdir/etc/mkinitcpio.d/linux-rock5c.preset"
}
```

要点：

- `mkinitcpio.d/linux-rock5c.preset` 参照 `linux-aarch64` 包的 preset，定义内核版本与
  initramfs 输出名，升级时自动触发 `mkinitcpio -P` 重新生成；
- 交叉编译在 x86 上要传 `CROSS_COMPILE`（原生 x86 的 makepkg 不会自动交叉编译）；
- 更省事的方式：**在板子上原生 `makepkg`**（Arch 生态最正统），或 qemu-user-static 的
  aarch64 chroot 里编。产品化流水线再考虑交叉编译。

### 6.2 自建软件仓库（"发行版"的灵魂）

```bash
# 1) 在镜像/服务器上建仓库目录
mkdir -p /srv/archrepo/os/aarch64

# 2) 放入自建包（内核、驱动、AI 运行时、你的应用）
cp ~/rock5c-arch/repo/*.pkg.tar.zst /srv/archrepo/os/aarch64/

# 3) 生成仓库索引
repo-add /srv/archrepo/os/aarch64/rock5c.db.tar.gz \
    /srv/archrepo/os/aarch64/*.pkg.tar.zst

# 4) 客户端 pacman.conf 增加

# [rock5c]

# SigLevel = Optional TrustAll        # 起步期；正式发布用 pacman-key 签名

# Server = https://your-server/archrepo/os/aarch64

# 本地镜像可：Server = file:///srv/archrepo/os/aarch64
```

之后板子上 `pacman -S linux-rock5c my-ai-runtime` 就能安装/升级你的"发行版组件"，
和官方仓库无缝共存。再配合 `/etc/os-release`、hostname、默认用户的定制，
"自己的 Arch 发行版"就成立了。

### 6.3 定制 initramfs

编辑 `/etc/mkinitcpio.conf` 的 hooks 与 modules 行即可定制启动阶段加载的内容：

```text
HOOKS=(base udev autodetect modconf block filesystems keyboard fsck)
MODULES=(dw_mmc ...)          # 需要提前加载的模块
```

改完重新生成：`mkinitcpio -P`。原理：initramfs 是"临时根"，负责加载存储/文件系统驱动、
找到真正的 rootfs 并切换过去；裁剪它是减小启动体积、加快启动的主要手段。

### 6.4 集成 AI 运行时（RKNN / RKLLM，对应文档 05/06 的 Arch 版）

**内核驱动（rknpu）**：

- 方案 C（厂商内核）：已内置 `CONFIG_ROCKCHIP_RKNPU=y`，跳过。
- 方案 B（主线内核）：把本机 `rknpu_driver_0.9.8_20241009.tar.bz2` 合入内核树：

```bash
cd ~/rock5c-arch/linux
tar xjf /home/wbh808/rknn-llm-release-v1.2.3/rknpu-driver/rknpu_driver_0.9.8_20241009.tar.bz2 \
    -C drivers/          # 解出 drivers/rknpu/
# 在 drivers/Makefile 与 drivers/Kconfig 中注册 rknpu（参照厂商内核的集成方式），

# 然后 make menuconfig 开启 CONFIG_ROCKCHIP_RKNPU，重新编译
```

**用户态库与 udev**：

```bash
# 装进 rootfs（或打成 6.2 的自建包）
install -Dm755 /home/wbh808/rknn-toolkit2-2.3.2/rknpu2/runtime/Linux/librknn_api/aarch64/librknnrt.so \
    rootfs/usr/lib/librknnrt.so
install -Dm755 /home/wbh808/rknn-llm-release-v1.2.3/rkllm-runtime/Linux/librkllm_api/aarch64/librkllmrt.so \
    rootfs/usr/lib/librkllmrt.so

cat > rootfs/etc/udev/rules.d/99-rknpu.rules <<'EOF'
KERNEL=="rknpu", MODE="0666", GROUP="video"
EOF
```

验证（板子上）：

```bash
ls -l /dev/rknpu
cat /sys/kernel/debug/rknpu/version        # 需 ≥ 0.9.8（RKLLM 要求）
```

推理程序参考文档 05/15：示例在 `/home/wbh808/rknn-llm-release-v1.2.3/examples/rkllm_api_demo`。

### 6.5 WiFi / BT（AIC8800D80）

板载 WiFi6 芯片不在主线内核，两种做法：

- 用 AUR/自建包：`aic8800d80-dkms`（SDIO 版）+ 固件包（Radxa 的
  `radxa-pkg/aic8800` releases 提供 driver + firmware，含 WiFi 与 BT）；
- 或把树外驱动直接编进自编内核（同 rknpu 的集成思路）。

> 排错提示：AIC8800 用 SDIO 接口，先确认 dtb 里 SDIO 节点正常枚举（`dmesg | grep -i sdio`）。

### 6.6 镜像流水线（一条命令出镜像）

把 5.5 的步骤收敛成脚本，作为发行版"构建系统"的雏形：

```text
rock5c-arch/
├── bootloader/               # 构建产物 idbloader.img / u-boot.itb
├── linux-rock5c/             # 内核 PKGBUILD + config + rknpu/aic8800 补丁
├── packages/                 # my-ai-runtime、aic8800 等自建包
├── repo/                     # repo-add 输出
├── rootfs/                   # pacstrap/解包出来的 rootfs
├── scripts/
│   ├── build-uboot.sh
│   ├── build-kernel.sh       # 产出 pkg.tar.zst 并 repo-add
│   ├── build-rootfs.sh       # pacstrap + 定制 hook
│   └── make-image.sh         # 分区/写 loader/rsync/生成 extlinux.conf
└── output/rock5c-arch-<ver>.img
```

这与文档 06 里 rsdk 的 `rootfs.jsonnet + customize-hooks + guestfish` 是**同一个思想**：
只是"包生态"从 Debian 换成了 Arch，`--debs` 换成了自建 pacman 仓库。

---

## 7. 常见问题与排错

**Q1：串口完全没有输出（黑屏无日志）**

多半是 bootloader 没写对：确认 `dd` 目标是整个设备、偏移为 64/16384；确认 DDR blob 是
LPDDR5 2400MHz 版本；确认供电稳定、串口 3.3V 电平、波特率 1500000。

**Q2：U-Boot 起来了，但找不到内核**

检查 `/boot/extlinux/extlinux.conf` 的路径（相对 `extlinux/` 目录的 `../Image`）；
在 U-Boot 命令行用 `ls mmc 1:1 /boot` 等命令确认分区编号；或 `bootflow scan -lb` 手动扫描。

**Q3：内核起来了，但 rootfs 挂载失败**

- `APPEND` 里 `root=PARTUUID=...` 是否正确（用 `blkid` 核对）；
- 是否需要 `rootwait`（SD/eMMC 建议保留）；
- initramfs 是否生成（`/boot/initramfs-linux.img` 存在且 hooks 含 `block filesystems`）。

**Q4：能进系统但没网络**

ALARM tarball 默认用 systemd-networkd 的 DHCP（eth0/en\*）。`ip link` 看网卡名，
`systemctl status systemd-networkd` 看状态；板载 WiFi 需要 6.5 的树外驱动。

**Q5：`pacman-key --populate archlinuxarm` 失败 / 校验错误**

先 `systemd-timesyncd` 校时（时间错误会破坏签名验证），再换镜像源重试；
必要时 `rm -rf /etc/pacman.d/gnupg` 后重新 init/populate。

**Q6：NPU 设备节点不存在 / 版本太低**

确认内核编译时开启 rknpu 驱动、dtb 保留 `rknpu@fdab0000` 节点、udev 规则已安装；
`/sys/kernel/debug/rknpu/version` 低于 0.9.8 时升级驱动（本机已有 0.9.8）。

**Q7：升级内核后启动失败**

内核包升级会触发 mkinitcpio 重生成 initramfs；若 /boot 是独立分区，确认 fstab 正确挂载。
保留上一个内核与 initramfs 副本作为回滚（改 extlinux.conf 的 KERNEL/INITRD 行即可）。

**Q8：AUR 包在 aarch64 上装不了**

很多 AUR PKGBUILD 只声明 x86_64；用 `makepkg -A`（忽略 arch 检查）或用
yay/paru 构建时加 `--ignorearch`（等价于 makepkg -A），或干脆自建 PKGBUILD——
这本来就是做发行版的日常。

---

## 8. 与文档 00~16 的衔接

| 已有文档 | 与本文的关系 |
| --- | --- |
| 01 BSP | 同一套"板级移植"思想：bsp 把 u-boot/内核打成 deb，本文把同样的源码打成 pacman 包；bsp 的 kconfig.conf/补丁机制可整体搬进 PKGBUILD |
| 06 定制发行版实战 | rsdk 路线（Debian 组装）与本文（Arch 组装）互为镜像：packages ↔ pacstrap 包清单，customize-hooks ↔ chroot 定制脚本，--debs ↔ 自建 pacman 仓库 |
| 05/10/11 AI 部署 | 内核 rknpu 驱动 + librknnrt/librkllmrt 用户态与 RKNN/RKLLM 转换流程完全一致，只是集成方式从 dpkg 换 pacman |
| 09/15 摄像头 | 结论同样适用：主线 dtb 无 camera 节点，MIPI 需自写 overlay；先 USB UVC |
| 07 烧录排错 | 串口 1500000、SD/eMMC 设备名、loader 写偏移等结论通用 |

---

## 9. 成果物清单与下一步

按本文执行完，你将得到：

```text
output/rock5c-arch.img
├── 自编 U-Boot（主线 v2025.04+，rock-5c-rk3588s）
├── 自编内核（≥6.13，含 rk3588s-rock-5c.dtb；可选 rknpu/aic8800）
├── ALARM aarch64 rootfs（或 pacstrap 自组）
├── /boot/extlinux/extlinux.conf
└── （进阶）自建 pacman 仓库：linux-rock5c、my-ai-runtime 等包
```

建议的下一步：

1. 先跑方案 A（官方内核 + tarball）确认板子与串口/网络；
2. 按方案 B 自编 U-Boot 与内核，验证"全源码可控"；
3. 需要 AI/多媒体时切方案 C 的 Radxa 内核，集成 rknpu 0.9.8 与 RKNN/RKLLM；
4. 把内核、AI 运行时、应用逐个 PKGBUILD 化并 repo-add，形成自己的 Arch 发行版；
5. 用文档 08~14 的猫咪监控项目作为第一个"产品"，产出 `rock5c-arch-<ver>.img`。
