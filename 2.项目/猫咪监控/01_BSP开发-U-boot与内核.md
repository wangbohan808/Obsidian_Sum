---
type: 项目
status: 整理中
tags:
  - 技术/Linux
project: 猫咪监控
created: "2026-08-11"
updated: "2026-08-11"
---

# 01 BSP 开发：U-boot 与内核构建、定制

> 官方文档：https://docs.radxa.com/rock5/rock5c/low-level-dev/u-boot
> 本地目录：`/home/wbh808/bsp`
> 学习目标：用 Radxa BSP 工具构建 ROCK 5C 的 U-boot 与内核，学会修改内核配置/驱动、设备树，并把结果打包成 deb 用于后续发行版集成。

---

## 1. 原理：bsp 是怎么工作的

### 1.1 定位

`bsp` 是 Radxa 维护的"硬件层（Bootloader + Kernel）构建工具"，解决两个问题：

1. **标准化**：所有 Radxa 板卡的 u-boot / kernel 构建流程统一（拉源码 → 打补丁 → 配置 → 交叉编译 → 打 deb 包）。
2. **可集成**：产物是 Debian 包，能直接进 RadxaOS，也能被 rsdk 通过 `--debs` 装进你自制的发行版。

### 1.2 核心概念

| 概念 | 说明 | 本机例子 |
| --- | --- | --- |
| product | 产品名 | `rock-5c`、`rock-5c-spi`（SPI Flash 版） |
| profile | 一套源码/分支/补丁的组合 | u-boot 用 `rk2410`，linux 用 `rk2410` |
| fork.conf | 定义仓库地址、分支、支持的产品、额外配置 | `/home/wbh808/bsp/u-boot/rk2410/fork.conf` |
| kconfig.conf | 内核 Kconfig 覆盖（可视为"内核配置片段"） | `/home/wbh808/bsp/linux/rk2410/kconfig.conf` |
| 补丁目录 | 按 `NNNN-名字/` 组织，构建时按序应用 | `/home/wbh808/bsp/linux/rk2410/0001-common/` |
| .src/ | 实际源码工作树 | `/home/wbh808/bsp/.src/{u-boot,linux,rkbin,arm-trusted-firmware,radxa-overlays}` |

### 1.3 构建流程（内部发生了什么）

```bash
./bsp linux rk2410 rock-5c
  │
  ├─ 1. 检查/启动容器（Podman 或 Docker），挂载源码目录
  ├─ 2. 准备源码：按 fork.conf 同步/检出对应仓库分支（默认浅克隆）
  ├─ 3. 按顺序应用 profile 下的补丁目录
  ├─ 4. 加载 defconfig + 应用 kconfig.conf（内核）
  ├─ 5. 交叉编译（arm64）
  ├─ 6. 用容器内的 debian 打包工具生成 deb 包（输出到 bsp 根目录）
  └─ 7. （可选）./bsp install 把 deb 装到 SD 卡分区
```

> 注意：默认每次构建会**重置源码并重新同步**，会覆盖你的本地修改。想保留本地修改必须加 `--no-prepare-source`（或 `--dirty`），详见第 4 节。

---

## 2. 环境准备

### 2.1 依赖安装（Ubuntu 22.04 / Debian 12）

```bash
sudo apt update
sudo apt install -y git qemu-user-static binfmt-support

# Podman（Radxa 推荐）
sudo apt install -y podman podman-docker
sudo touch /etc/containers/nodocker

# 或用 Docker：

# sudo apt install -y docker.io

# sudo usermod -aG docker $USER   # 重新登录生效

# 次要功能的可选依赖
sudo apt install -y systemd-container
```

### 2.2 获取/确认代码

官方获取方式（本机已下载，跳过克隆）：

```bash
git clone --recurse-submodules https://github.com/radxa-repo/bsp.git
```

本机目录已就绪，确认关键内容：

```bash
cd /home/wbh808/bsp
./bsp                # 查看内置帮助
./bsp json product   # 查看支持的 product（含 rock-5c）
ls .src/             # 已拉好的源码：u-boot linux rkbin arm-trusted-firmware radxa-overlays
```

---

## 3. 构建 U-boot（ROCK 5C）

### 3.1 命令

ROCK 5C 对应的 profile 是 `rk2410`：

```bash
cd /home/wbh808/bsp

# 只准备源码、不编译（适合先改代码）
./bsp u-boot rk2410 rock-5c --no-build

# 正式构建
./bsp u-boot rk2410 rock-5c -r 999
```

参数说明：

- `-r 999`：指定版本号，让新 deb 优先级高于官方仓库里的版本。
- `--no-build`：只拉源码、打补丁、配置，不编译。
- `--no-prepare-source`：**不重置源码**，用本地修改编译（二次构建必备）。
- `-C`：先 `make distclean`；`-c`：先 `make clean`。
- `--dirty`：等价于 `--no-prepare-source --no-config --no-container-update`，最快增量构建。

### 3.2 产物

构建完成后，bsp 根目录出现：

```text
u-boot-rk2410_2017.09-999_arm64.deb
```

> 该包是**元包/安装包**，安装后会把对应板卡的 u-boot 刷写脚本放到 `/usr/lib/u-boot/rock-5c/`。

### 3.3 安装到板子

把 deb 复制到 ROCK 5C 上执行：

```bash
sudo dpkg -i u-boot-rk2410_2017.09-999_arm64.deb

# 把新 u-boot 写入启动介质（SD 卡通常是 /dev/mmcblk1，eMMC 可能是 /dev/mmcblk0）
cd /usr/lib/u-boot/rock-5c/
sudo ./setup.sh update_bootloader /dev/mmcblk1
sudo reboot
```

重启后观察串口日志，出现你设置的版本号即成功。

---

## 4. 构建内核（ROCK 5C）

### 4.1 官方推荐：dev container + make deb

ROCK 5C 内核官方仓库是 `radxa-pkg/linux-rk2410`，Radxa 为它提供了 VS Code Dev Container 开发环境：

```bash
git clone --recurse-submodules https://github.com/radxa-pkg/linux-rk2410.git
cd linux-rk2410
code .                # 用 VS Code 打开，选择 "Reopen in Container"
# 容器内：
make deb
```

产物：`linux-image-*.deb`、`linux-headers-*.deb` 等。

### 4.2 用 bsp 构建内核（与 u-boot 同一套工具链）

```bash
cd /home/wbh808/bsp

# 准备源码（会应用 linux/rk2410 下的补丁和 kconfig.conf）
./bsp linux rk2410 rock-5c --no-build

# 正式构建
./bsp linux rk2410 rock-5c -r 999
```

产物示例：

```text
linux-image-6.1.75-1-rk2410_6.1.75-999_arm64.deb
linux-headers-6.1.75-1-rk2410_6.1.75-999_arm64.deb
linux-dtb-6.1.75-1-rk2410_6.1.75-999_arm64.deb
```

### 4.3 安装内核到板子 / SD 卡

方式一：dpkg 安装（在板子上）：

```bash
sudo dpkg -i linux-image-*.deb linux-headers-*.deb linux-dtb-*.deb
sudo update-initramfs -c -k all
sudo reboot
```

方式二：直接装进 SD 卡 rootfs（在构建机上）：

```bash
cd /home/wbh808/bsp
./bsp install linux-image-6.1.75-*_arm64.deb /dev/sdb2   # 根据实际设备名修改
```

方式三：用 rsdk 集成进新镜像（见文档 04/06 的 `--debs` 用法）。

---

## 5. 定制内核：驱动裁剪与设备树

这是"制作只含所需驱动的发行版"最核心的一步。内核源码在 `/home/wbh808/bsp/.src/linux`。

### 5.1 工作流总览

```bash
cd /home/wbh808/bsp

# 1. 首次准备源码
./bsp linux rk2410 rock-5c --no-build

# 2. 修改源码/配置

#    .src/linux 下改代码、设备树或 .config

# 3. 用本地修改重新构建
./bsp --no-prepare-source linux rk2410 rock-5c -r 999
```

### 5.2 修改内核配置（两种方式）

**方式 A：临时改 .config（快速验证，不保留）**

```bash
cd /home/wbh808/bsp/.src/linux
make ARCH=arm64 rockchip_linux_defconfig        # 先回到基线
make ARCH=arm64 menuconfig                       # 交互式裁剪（需要 ncurses）
```

然后在 bsp 根目录执行带 `--no-prepare-source --no-config` 的构建，避免 bsp 重置 `.config`：

```bash
cd /home/wbh808/bsp
./bsp --no-prepare-source --no-config linux rk2410 rock-5c -r 999
```

**方式 B：用 kconfig.conf 固化（推荐，可复现）**

编辑 `/home/wbh808/bsp/linux/rk2410/kconfig.conf`，追加形如：

```text
# 示例：确保 RKNPU 驱动开启（RK3588 的 NPU）
CONFIG_ROCKCHIP_RKNPU=y

# 示例：关闭用不到的驱动以缩小内核

# CONFIG_BT_HCIBTUSB is not set
```

> 注意：`kconfig.conf` 的条目会被 bsp 在 defconfig 之上执行 `scripts/kconfig/merge_config.sh` 合并，所以写 `CONFIG_XXX=y` 或 `# CONFIG_XXX is not set` 都可以。

修改后重新构建：

```bash
./bsp linux rk2410 rock-5c -r 999   # 这次会重新应用 kconfig.conf
```

### 5.3 修改设备树（DTS）

RK3588 的设备树源文件在：

```text
/home/wbh808/bsp/.src/linux/arch/arm64/boot/dts/rockchip/
    rk3588s-rock-5c.dts       # ROCK 5C 板级设备树
    rk3588s.dtsi / rk3588.dtsi  # SoC 级
    rk3588s-rock-5c-*.dtsi    # 扩展
```

改完 DTS 后只需重新构建（`--no-prepare-source` 保留修改）。如果只想快速验证设备树，可以：

```bash
./bsp --no-prepare-source --dtb linux rk2410 rock-5c
```

产物是 dtb 包，适合反复迭代硬件适配。

### 5.4 把修改固化成补丁（提交/分享/复用）

bsp 的补丁机制：`linux/rk2410/NNNN-名字/*.patch`，按目录顺序应用。

```bash
cd /home/wbh808/bsp/.src/linux
git add -A
git commit -m "my driver customization"

# 导出补丁到 bsp profile 目录（起始编号注意与已有目录错开）
mkdir -p /home/wbh808/bsp/linux/rk2410/0500-my-custom
git format-patch -M -N --start-number 1 --zero-commit -o ../../linux/rk2410/0500-my-custom HEAD~1
```

之后每次 `./bsp linux rk2410 rock-5c` 都会自动应用这些补丁，换机器也能复现。

---

## 6. 常见问题

**Q1：构建报容器相关错误？**

确认容器后端可用：`podman info` 或 `docker ps`。bsp 默认自动探测，也可指定 `-b docker` / `-b podman`。装了 Docker 且没建 `/etc/containers/nodocker` 时，bsp 可能误用 podman。

**Q2：为什么我的源码修改总是丢？**

默认 `bsp` 每次构建前会重新同步源码。请用 `--no-prepare-source`（保留本地修改）或 `--dirty`（最快增量）。

**Q3：想用自定义 make 命令手动编译？**

参考 bsp 官方文档的 Prepare working tree 部分：先用 `./bsp -C --no-build linux rk2410` 准备好源码，然后自己进入 `.src/linux` 执行 make。注意 bsp 是唯一仍在维护、且会额外修改源码的官方流程，尽量以 bsp 为准。

**Q4：构建很慢？**

首次构建需要拉源码 + 编译，属正常。后续用 `--dirty` 增量构建；把容器镜像更新关掉（默认已带 `--no-container-update` 的效果可通过 `--no-container-update` 显式指定）。

---

## 7. 本节产出与下一步

完成本节后，你应该拥有：

- 可安装的 `u-boot-rk2410_*.deb`（自定义 bootloader）
- 可安装的 `linux-image-*.deb` / `linux-dtb-*.deb`（自定义内核）
- 一套"可复现"的内核配置修改（kconfig.conf + 补丁目录）

下一步：

- 想快速出可用系统 → 读文档 04（rsdk），用 `--debs` 把这些 deb 装进 RadxaOS。
- 想压缩系统体积 → 读文档 03（Buildroot）或 02（Yocto），把内核源码/配置移植过去。
