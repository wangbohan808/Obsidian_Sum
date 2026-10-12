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

# 02 Yocto 镜像构建

> 官方文档：https://docs.radxa.com/rock5/rock5c/other-os/yocto（快速上手 / 构建系统 / 安装系统）
> 本地目录：`/home/wbh808/yocto-rockchip-sdk`
> 学习目标：理解 OpenEmbedded/Yocto 的构建原理，掌握 ROCK 5C 的 Yocto 镜像构建，以及如何通过 recipe / layer / 内核配置片段做定制。

---

## 1. 原理：Yocto 是怎么构建一个 Linux 系统的

### 1.1 整体架构

Yocto 项目 = OpenEmbedded 构建框架 + Poky 参考发行版。核心是 **BitBake** 任务调度器：

```text
                    ┌─────────────────────────────────┐
                    │        BitBake（任务调度器）      │
                    └─────────────────────────────────┘
        layer 1: poky（meta / meta-poky）     基础工具、distro 策略
        layer 2: meta-openembedded（meta-oe） 大量软件包 recipe
        layer 3: meta-rockchip                板级 BSP（machine、kernel recipe）
        layer 4: meta-clang / meta-browser... 可选功能
                    │
        local.conf（MACHINE、IMAGE_INSTALL、并行度、镜像）
        bblayers.conf（启用哪些 layer）
                    │
        输出：build/tmp/deploy/images/<machine>/*.img
```

### 1.2 关键概念

| 概念 | 含义 | 本机例子 |
| --- | --- | --- |
| recipe（.bb） | 描述一个软件包：从哪里下载、怎么编译、装什么 | `meta-rockchip/recipes-kernel/linux/linux-rockchip_6.1.bb` |
| layer | 一组相关 recipe/配置的集合，按优先级叠加 | `meta-rockchip` |
| machine | 硬件平台定义（SOC、设备树、bootloader） | `rockchip-rk3588s-rock-5c` |
| distro | 发行版策略（libc、init 系统、特性） | `poky` |
| image recipe | 最终生成什么系统镜像 | `core-image-minimal` |
| local.conf | 本地构建配置（最重要入口） | `build/conf/rockchip-rk3588s-rock-5c.conf` |
| sstate-cache | 编译缓存，避免重复编译 | `build/tmp/sstate-cache` |
| bbappend | 对已有 recipe 的增量补丁文件（`.bbappend`） | 定制内核最常用 |

### 1.3 构建生命周期

每个 recipe 会经历：`do_fetch`（下载）→ `do_unpack` → `do_patch` → `do_configure` → `do_compile` → `do_install` → `do_package` → `do_rootfs`（组装 rootfs）→ `do_image`（生成镜像文件）。

ROCK 5C 最终镜像由 `core-image-minimal` + machine 配置共同决定：

- `*.rootfs.wic`：eMMC 用的 GPT 磁盘镜像
- `*.rootfs.update.img`：SD 卡启动用的 RK 格式镜像

---

## 2. 本机工程结构

```text
/home/wbh808/yocto-rockchip-sdk/
├── poky/                      # OpenEmbedded 核心（bitbake、oe-init-build-env）
├── meta-rockchip/             # Rockchip BSP layer
│   ├── conf/machine/rockchip-rk3588s-rock-5c.conf
│   └── recipes-kernel/linux/linux-rockchip_6.1.bb
├── meta-openembedded/         # 常用软件包 layer（meta-oe）
├── meta-clang / meta-browser / meta-lts-mixins
├── build/                     # 构建目录（产物在此）
│   ├── conf/
│   │   ├── local.conf -> rockchip-rk3588s-rock-5c.conf   # 已链接 5C 配置
│   │   ├── bblayers.conf
│   │   └── include/*.conf     # audio/display/wifibt 等模块化配置
│   └── tmp/deploy/images/rockchip-rk3588s-rock-5c/       # 已有构建产物！
└── oe-init-build-env -> poky/oe-init-build-env
```

本机已经成功构建过 `core-image-minimal`，产物示例：

```text
build/tmp/deploy/images/rockchip-rk3588s-rock-5c/
├── core-image-minimal-rockchip-rk3588s-rock-5c.rootfs.update.img
├── core-image-minimal-rockchip-rk3588s-rock-5c.rootfs.wic
├── core-image-minimal-rockchip-rk3588s-rock-5c.rootfs.ext4
├── boot.img
└── core-image-minimal-...rootfs.manifest   # 已装包清单（裁剪时很有用）
```

---

## 3. 构建步骤

### 3.1 环境要求与依赖

```bash
sudo apt-get update
sudo apt-get install \
    build-essential chrpath cpio debianutils diffstat file gawk gcc git \
    iputils-ping libacl1 locales python3 python3-git \
    python3-jinja2 python3-pexpect python3-pip python3-subunit socat \
    texinfo unzip wget xz-utils zstd liblz4-tool -y

# 生成 en_US.UTF-8 locale（Yocto 必需）
sudo locale-gen en_US en_US.UTF-8
sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
```

要求：32GB+ 内存、90GB+ 可用磁盘（本机首次构建已完成，增量构建需求小得多）。

### 3.2 首次获取源码（本机已完成，备查）

```bash
mkdir -p ~/yocto-rockchip-sdk && cd ~/yocto-rockchip-sdk
repo init -u https://github.com/radxa/yocto-manifests.git -b scarthgap
repo sync
```

### 3.3 配置 local.conf（已配置好）

```bash
cd /home/wbh808/yocto-rockchip-sdk/build/conf
ln -sf rockchip-rk3588s-rock-5c.conf local.conf
```

本机 `rockchip-rk3588s-rock-5c.conf` 已包含：

- `MACHINE = "rockchip-rk3588s-rock-5c"`
- 清华镜像加速（PREMIRRORS / SOURCE_MIRROR_URL）
- wget 重试优化
- 可选的并行度限制（默认注释）

### 3.4 开始构建

```bash
cd /home/wbh808/yocto-rockchip-sdk
source ./oe-init-build-env      # 进入构建环境（会自动 cd 到 build）
bitbake core-image-minimal
```

> 常用变体：
> - 只编译内核（改驱动时快速验证）：`bitbake linux-rockchip -c compile -f` 后 `bitbake core-image-minimal`
> - 只看某个包的依赖/环境：`bitbake -e <recipe>`、`bitbake -s | grep <名字>`
> - 清缓存：`bitbake -c cleanall <recipe>`（慎用，会重新下载/编译）

### 3.5 产物与烧录

```bash
cd /home/wbh808/yocto-rockchip-sdk/build/tmp/deploy/images/rockchip-rk3588s-rock-5c/
ls -lh *.img
```

- SD 卡：烧 `*.rootfs.update.img`（可用文档 07 的 dd / balenaEtcher / Win32DiskImager 方式）
- eMMC：烧 `*.rootfs.wic`（或用 `upgrade_tool` 刷整包）

---

## 4. 定制方法（打造自己的发行版）

Yocto 的定制全部通过"追加配置/追加 recipe"完成，不要直接改 poky 里的文件。

### 4.1 往镜像里加包：IMAGE_INSTALL

编辑 `build/conf/rockchip-rk3588s-rock-5c.conf`（即 local.conf）：

```text
# 追加安装的软件包
IMAGE_INSTALL:append = " python3 python3-pip git openssh-sftp-server i2c-tools"

# 若需要调试/串口工具
IMAGE_INSTALL:append = " strace gdb tmux"
```

### 4.2 定制内核：bbappend + 内核配置片段

新建目录并创建 `linux-rockchip_6.1.bbappend`：

```bash
mkdir -p /home/wbh808/yocto-rockchip-sdk/meta-rockchip/recipes-kernel/linux/linux-rockchip_6.1/
```

在该目录放一个内核配置片段 `my-drivers.cfg`：

```text
# 开启 RKNPU 驱动（RK3588 NPU）
CONFIG_ROCKCHIP_RKNPU=y
CONFIG_ROCKCHIP_RKNPU_DRM_GEM=y

# 关闭用不到的驱动示例

# CONFIG_WLAN_VENDOR_ATH is not set
```

然后创建 `linux-rockchip_6.1.bbappend`：

```text
FILESEXTRAPATHS:prepend := "${THISDIR}/linux-rockchip_6.1:"

SRC_URI:append = " file://my-drivers.cfg"

# 内核配置片段机制：把 .cfg 合并进内核配置
inherit kernel-yocto
do_configure:append() {
    # 实际上通过 KERNEL_FEATURES 或直接合并 .cfg：
    :
}
```

更标准的做法是直接用 Yocto 内核配置机制：

```text
FILESEXTRAPATHS:prepend := "${THISDIR}/linux-rockchip_6.1:"
SRC_URI:append = " file://my-drivers.cfg"
```

配合在 recipe 里启用 `kernel-yocto` 的 `merge_config` 逻辑（Radxa 的 6.1 kernel recipe 通常已支持），`.cfg` 会自动生效。若不确定，最简单可靠的方式是：

```bash
# 进入构建环境后，手工验证内核配置是否生效
bitbake -c menuconfig linux-rockchip    # 交互式调整（改动会写到源树，需固化）
bitbake -c diffconfig linux-rockchip   # 导出差异到 build/tmp/.../fragment.cfg
```

把 `fragment.cfg` 复制为 bbappend 的 `.cfg` 文件即可固化。

### 4.3 创建自己的 layer（推荐的组织方式）

```bash
cd /home/wbh808/yocto-rockchip-sdk
source oe-init-build-env
bitbake-layers create-layer ../meta-my-distro
bitbake-layers add-layer ../meta-my-distro
```

在 `meta-my-distro/recipes-core/images/my-image.bb` 里定义自己的镜像：

```text
SUMMARY = "My ROCK 5C AI distro image"
LICENSE = "MIT"

inherit core-image

# 从基础镜像继承 + 追加自己的包
IMAGE_INSTALL:append = " \
    rknpu2 \
    rkllm-runtime \
    python3-rknnlite \
    my-ai-app \
    "
```

然后：

```bash
bitbake my-image
```

### 4.4 添加自研软件包（recipe 示例）

把自己的 AI 应用做成 recipe，例如 `meta-my-distro/recipes-ai/my-ai-app/my-ai-app_1.0.bb`：

```text
SUMMARY = "Edge AI inference service"
LICENSE = "CLOSED"

SRC_URI = "file://my-ai-app"

do_install() {
    install -d ${D}${bindir}
    install -m 0755 ${WORKDIR}/my-ai-app ${D}${bindir}/
    install -d ${D}${systemd_system_unitdir}
    install -m 0644 ${WORKDIR}/my-ai-app.service ${D}${systemd_system_unitdir}/
}

SYSTEMD_SERVICE:${PN} = "my-ai-app.service"
inherit systemd
```

`files/` 目录放二进制与 unit 文件。这样镜像里就自带自启动的 AI 服务。

### 4.5 常用调试命令

```bash
bitbake -e linux-rockchip | grep -E 'SRC_URI|KERNEL_FEATURES'   # 看 recipe 实际展开值
bitbake -s | grep -i rknn                                       # 搜索是否有 rknn 相关包
bitbake -c devshell linux-rockchip                             # 进入 recipe 的 shell 环境
bitbake core-image-minimal -c populate_sdk                     # 生成交叉编译 SDK
```

---

## 5. 常见问题

**Q1：构建时下载超时/失败？**

本机 local.conf 已配置清华源镜像。若仍失败，可加大 `FETCHCMD_wget` 的重试次数，或检查代理。

**Q2：构建过程中系统崩溃/内存不足？**

在 local.conf 中降低并行度：

```text
BB_NUMBER_THREADS = "4"
PARALLEL_MAKE = "-j 4"
```

**Q3：Weston 构建报 PAM 错误？**

```bash
sudo apt-get install libpam0g-dev -y
```

并在 local.conf 加 `DISTRO_FEATURES:append = " pam"`，然后 `bitbake -c cleansstate weston` 重试。

**Q4：改了内核配置不生效？**

确认改动在正确的 layer/bbappend 里，且执行了 `bitbake -c cleanall linux-rockchip`（或至少 `-c cleansstate`）后再重新构建。只改 `.cfg` 而 recipe 没触发重配时，可用 `bitbake -c configure -f linux-rockchip` 强制重配。

---

## 6. 本节产出与下一步

完成本节后：

- 能构建 ROCK 5C 的 Yocto 镜像（已具备）
- 能用 `IMAGE_INSTALL` / bbappend / 自定义 layer 控制镜像内容
- 能定制内核配置并固化

下一步：

- 如果你想要"Debian 生态 + AI 包即装即用" → 文档 04（rsdk）
- 如果你想要"最小体积嵌入式固件" → 文档 03（Buildroot）
- 综合定制方案 → 文档 06
