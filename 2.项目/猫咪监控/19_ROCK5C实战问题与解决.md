---
type: 项目
status: 整理中
tags:
  - 技术/Linux
project: 猫咪监控
created: "2026-05-21"
updated: "2026-10-12"
up: "[[00_总览与学习路线]]"
---

# ROCK 5C 实战问题与解决

> [!info] 一句话结论
> 本文把两份按日期流水记录（2026-05-21 ~ 2026-09-21）重组为 ROCK 5C 从环境搭建、源码编译、内核驱动调试到镜像构建与烧录验证的实战问题集，保留可复用的命令、报错原文与解决过程。

## 要点

1. **代理分三层**：宿主机全局代理、容器局域网代理（容器内不能用 `127.0.0.1`）、包管理器专有代理，需分别配置。
2. **源码拉取失败三大主因**：未递归子模块、弱网 TLS 低速断连、编译期自动下载被 403 拒绝，各有对应兜底手段。
3. **自编译镜像异常先看灯**：蓝灯不闪＝引导链/内核没起来；蓝灯闪但不进用户态，优先查串口 console 与内核 bug。
4. **Rockchip 启动链有固定扇区**：`idbloader.img` 在扇区 64、`u-boot.itb` 在扇区 16384，GPT 分区从 16 MiB（扇区 32768）起。
5. **SD→eMMC 迁移**只搬普通分区不够，引导区（boot0/boot1）不同步会无法启动。
6. **板端 RKNN 依赖要锁版本**：如 `onnx==1.16.1`（1.20 起删除了 `onnx.mapping`）。

## 环境与工具链搭建

### 串口与硬件连接

- 串口波特率：**1500000**。
- 接线：一般用 TTL 转串口，GND 接第 6 脚，RX、TX 依次接第 8、10 脚。

### VMware 共享文件夹

```bash
# 创建挂载点并手动挂载（共享文件夹名以实际为准）
sudo mkdir -p /mnt/hgfs
sudo vmhgfs-fuse .host:/VMShare /mnt/hgfs -o allow_other

# 永久挂载
echo '.host:/VMShare /mnt/hgfs fuse.vmhgfs-fuse allow_other 0 0' | sudo tee -a /etc/fstab
```

> [!warning] Linux 目录直接复制到共享文件夹会因软链接出错
> 用 `rsync` 排除链接文件：
> ```bash
> sudo rsync -av --no-links --ignore-errors ~/rock-5c/ /mnt/hgfs/rock-5c/
> ```

### 一键换源（apt）

ROCK 5C 用 `/etc/apt/sources.list.d/50-*.list` 管理软件源，只备份并替换 `50-` 开头的文件：

```bash
# 备份所有 50- 开头的源文件
sudo mkdir -p /etc/apt/sources.list.d/backup
sudo cp /etc/apt/sources.list.d/50-* /etc/apt/sources.list.d/backup/

# 主源
sudo tee /etc/apt/sources.list.d/50-bookworm.list > /dev/null << EOF
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm main contrib non-free non-free-firmware
deb-src https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm main contrib non-free non-free-firmware
EOF

# 同理替换 updates / security / backports，仅把 URL 段换成对应目录
sudo apt update
```

- `> /dev/null` 用于丢弃 `tee` 回显到屏幕的内容。
- 清华源一般不同步 Radxa 这类小众厂商的私有仓库，板端专有包（如 `python3-rknnlite2-example`）需换用能收录的源。

### pip / conda 换源

```bash
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
pip config set global.extra-index-url https://mirrors.tuna.tsinghua.edu.cn/pytorch-wheels/torch/
pip config set global.timeout 120
```

conda 走清华源（写入 `~/.condarc`）：

```yaml
channels:
  - conda-forge
show_channel_urls: true
default_channels:
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/r
custom_channels:
  conda-forge: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  bioconda: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  pytorch: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
solver: libmamba
```

> [!tip] conda 专有代理可与全局代理解耦
> conda 有独立的 `proxy_servers` 配置，用 alias 一键开关：走境外源时 `conda-proxy-on`，走清华源时 `conda-proxy-off`。

### 代理配置

宿主机局域网代理（Clash 开「局域网连接」后，用宿主机局域网 IP，而不是 `127.0.0.1`）：

```bash
hostname -I                      # 查宿主机局域网 IP，如 192.168.203.129

# 清除旧代理
git config --global --unset http.proxy
git config --global --unset https.proxy
unset http_proxy https_proxy all_proxy socks_proxy HTTP_PROXY HTTPS_PROXY ALL_PROXY

# 设置新代理
export HTTP_PROXY=http://192.168.203.129:7897
export HTTPS_PROXY=http://192.168.203.129:7897
git config --global http.proxy http://192.168.203.129:7897
git config --global https.proxy http://192.168.203.129:7897
```

验证代理是否可用：

```bash
curl -v --max-time 8 --connect-timeout 3 -x http://192.168.203.129:7897 https://github.com
```

> [!danger] 容器内不能用宿主机回环地址
> rsdk 等容器启动时要显式传局域网 IP 代理，并设置 `NO_PROXY`：
> ```bash
> HTTP_PROXY=http://192.168.203.129:7897 HTTPS_PROXY=http://192.168.203.129:7897 \
> ALL_PROXY=socks://192.168.203.129:7897 \
> NO_PROXY=127.0.0.1,localhost,172.17.0.1,192.168.203.* rsdk devcon up
> ```

### 交叉编译工具链

```bash
sudo tar -xf gcc-arm-11.2-2022.02-x86_64-aarch64-none-linux-gnu.tar.xz -C /opt/

# 写入 ~/.bashrc 后 source
export PATH=$PATH:/opt/gcc-arm-11.2-2022.02-x86_64-aarch64-none-linux-gnu/bin
aarch64-none-linux-gnu-gcc -v
```

> [!note] 工具链版本取决于板子镜像版本、板子芯片与宿主机芯片的匹配关系。

### 板卡 AI 开发环境（Windows）

```powershell
nvidia-smi   # 查驱动版本与 CUDA 上限，本机 GPU 为 RTX 3050（sm_86）
py -0p       # 列出本机已安装的 Python 版本

# GPU 环境（按显卡选对应 CUDA 版本）
py -3.14 -m venv torch_gpu_py314_cu126
.\torch_gpu_py314_cu126\Scripts\Activate.ps1
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu126

# CPU 环境
py -3.11 -m venv torch_cpu_py311
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cpu
```

- 显卡 → CUDA → 唯一对应的 PyTorch，先查 `nvidia-smi` 的 CUDA 上限再装。
- 驱动包台式机（desktop）与笔记本（notebook / NSD）不可交叉安装。
- **Python 解释器可留旧版本，但 pip 新建环境必须升级**。

### VM ↔ Windows 免密 SSH

```powershell
# Windows 管理员 PowerShell：安装并启动 OpenSSH Server，放行 22
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
New-NetFirewallRule -Name "OpenSSH-Server-In-TCP" -DisplayName "OpenSSH Server (sshd)" `
  -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22 -Profile Any
```

```bash
# Linux 虚拟机生成密钥并配置别名
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_winhost -C "vm-to-winhost" -N ""
```

```ini
# ~/.ssh/config
Host winhost
    HostName 192.168.40.1
    User wangb
    IdentityFile ~/.ssh/id_ed25519_winhost
    IdentitiesOnly yes
```

> [!warning] 管理员账户的公钥位置不同
> 若 `whoami /groups` 显示属于 Administrators 组，OpenSSH 优先读 `C:\ProgramData\ssh\administrators_authorized_keys`，而不是用户目录下的 `authorized_keys`。

```bash
# 传输（Windows 路径用正斜杠）
scp /home/wbh808/cat-monitor/arch/output/rock5c-arch.img winhost:"D:/Data/Resource/Linux/ROCK-5C/Output/"
```

> [!error] Windows 端没有 rsync
> `rsync ... winhost:"D:/..."` 会报 `connection unexpectedly closed (code 12)`，改用 `scp`。

### 编程助手配置（Claude / codex）

```bash
# Claude Code
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
npm install -g @anthropic-ai/claude-code

# 指向 DeepSeek 兼容端点（示例，token 自填）
export ANTHROPIC_BASE_URL=https://api.deepseek.com/anthropic
export ANTHROPIC_AUTH_TOKEN=<your-token>
export ANTHROPIC_MODEL=deepseek-v4-flash
export CLAUDE_CODE_SUBAGENT_MODEL=deepseek-v4-flash
```

```bash
# codex（板端从国内镜像装）
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo bash -
sudo apt install -y nodejs
sudo npm install -g @openai/codex --registry=https://registry.npmmirror.com
```

## 源码获取与编译

### Git 子模块

```bash
# 只 clone 主仓库 → 子模块文件夹为空
git clone https://github.com/radxa-repo/bsp.git

# 主仓库 + 全部子模块（BSP 开发必须）
git clone --recurse-submodules https://github.com/radxa-repo/bsp.git

# 忘记递归时补拉
cd bsp && git submodule update --init --recursive
```

> [!warning] 直接解压 zip 丢进子模块目录不可靠
> 缺少 `.git` 记录时 git 认不出子模块，`git submodule status` 会报错。只应对 CMake `FetchContent` 这类非子模块依赖用「手动下载源码 → 注释掉在线拉取」的办法。

### 弱网拉取优化

```bash
# 缓冲区过小会分包、乱序报错；调大到 2G
git config --global http.postBuffer 2147483648

# HTTP/1.1 串行短连接，弱网比 HTTP/2 稳
git config http.version HTTP/1.1

# 给低速网络更多容错空间（默认 TLS 低速会直接断连）
git config http.sslLowSpeedLimit 1000
git config http.sslLowSpeedTime 300

# 只要最新提交 / 之后补齐历史
git fetch --depth=1 radxa
git fetch --unshallow radxa
```

### repo sync 与离线兜底

```bash
# 多线程同步，失败即停（比默认更稳）
.repo/repo/repo sync -l -j1 --fail-fast
repo sync -j8 --no-tags --fail-fast
```

编译期自动下载失败（如 buildroot 拉 fakeroot 源码）时手动下载放入缓存目录：

```bash
wget https://sources.buildroot.net/fakeroot/fakeroot_1.32.1.orig.tar.gz
# 手动放入 buildroot/dl/<包名>/ 后重编，buildroot 优先读本地文件、跳过联网
```

> [!note] 403 拒绝访问多因地址不是完整文件路径或权限不足，浏览器粘完整文件 URL 也可手动保存。

### BSP / U-Boot / 内核编译

BSP 构建工具语法：

```bash
bsp [options] <linux|u-boot> <profile> [product]
```

- **profile**：代码大版本 / 基线（哪套 u-boot 源码、哪些补丁），由 SoC 平台决定。
- **product**：具体开发板（在 profile 这套代码里编出这块板子的固件）。

修改内核时需同时改**编译配置文件**与**设备树**：

```bash
cat /home/wbh808/bsp/linux/rk2410/fork.conf
cat /home/wbh808/bsp/linux/rk2410/kconfig.conf
ls  /home/wbh808/bsp/.src/linux/arch/arm64/boot/dts/rockchip/rk3588s-rock-5c.dts
```

U-Boot 单独编译：

```bash
export CROSS_COMPILE=aarch64-linux-gnu-
make distclean                                # 换板必须，否则沿用上一块板的 .config
make rock-5c-rk3588s_defconfig                # 生成 .config
grep '^CONFIG_DEFAULT_DEVICE_TREE=' .config   # 确认板级配置正确
make -j$(nproc)
```

> [!warning] `export` 只对当前终端及其子进程有效，新开终端即失效。

### 内核裁剪缩体积

```bash
# 1. 编辑 kconfig.conf，关闭不需要的 CONFIG_*

# 2. 构建验证
cd /home/wbh808/bsp && ./bsp linux rk2410
# 3. 对比体积
ls -lh .src/linux/arch/arm64/boot/Image
du -sh .src/linux/lib/modules/*/
# 4. 功能测试通过后再继续裁
```

猫监控项目追加的驱动清单（示例）：

```text
CONFIG_ROCKCHIP_RKNPU=y
CONFIG_MMC=y
CONFIG_USB_VIDEO_CLASS=y
CONFIG_VIDEO_ROCKCHIP_CIF=y
CONFIG_VIDEO_ROCKCHIP_ISP=y
CONFIG_VIDEO_IMX219=y
CONFIG_ROCKCHIP_MPP_SERVICE=y
CONFIG_VIDEO_ROCKCHIP_RGA=y
CONFIG_WATCHDOG=y
```

> [!note] 体积由 Kconfig（编了多少驱动/功能）决定，DTB 本身很小。

### RKNN 工具链安装

x86（PC 端转换）用锁定版本的环境：

```bash
conda create -n rknn230_fix python=3.10 -y
conda activate rknn230_fix
pip install protobuf==4.25.4 setuptools==66.0.0 numpy==1.26.4
pip install tqdm scipy ruamel.yaml psutil fast-histogram onnxruntime==1.16.3
cd ~/rknn-toolkit2-2.3.0/rknn-toolkit2/packages/x86_64
pip install ./rknn_toolkit2-2.3.0-cp310-cp310-manylinux_2_17_x86_64.manylinux2014_x86_64.whl --no-deps
pip install torch==2.4.0 onnx==1.14.1 opencv-python==4.8.1.78
```

> [!error] `AttributeError: module 'onnx' has no attribute 'mapping'`
> onnx ≥ 1.20 删除了 `onnx.mapping` 接口，而 rknn-toolkit2 要求 `onnx>=1.16.1`，**唯一兼容版本为 `onnx==1.16.1`**：
> ```bash
> pip uninstall onnx -y
> pip install onnx==1.16.1 --only-binary=onnx
> python -c "import onnx; print(onnx.__version__, hasattr(onnx, 'mapping'))"
> ```

板端（aarch64）安装 Lite2：

```bash
sudo apt update
sudo apt install rknpu2-rk3588 python3-rknnlite2          # SOC 为 RK3588 系列
sudo apt install python3-rknnlite2-example                # 测试例程（仅板端有源）
cd /usr/share/python3-rknnlite2/resnet18 && python3 test.py
```

模型转换（rknn_model_zoo 封装）：

```bash
python3 convert.py ../model/yolov8n.onnx rk3588
# 参数：<onnx模型> <TARGET_PLATFORM> [dtype: i8/fp] [输出路径]

# 可选平台：rk3562/rk3566/rk3568/rk3576/rk3588/rk1808/rv1109/rv1126
```

## 内核与驱动调试

### 编译配置三件套

| 层 | 作用 | 时机 |
| --- | --- | --- |
| 配置（defconfig / kconfig.conf） | 哪些驱动 / 功能编进二进制 | 编译期开关 |
| 设备树（dts → dtb） | 这块板实际接了哪些控制器、地址、引脚 | 运行期地图 |
| 驱动（`.c` / `.ko`） | 按 `compatible` 与 DT 节点配对 | 运行时生效 |

- 设备树不只给内核用，U-Boot 也靠 DTB 初始化硬件。
- 板子跑起来真正干活的只有 `.dtb` 和内核驱动（编进 `Image` 或 `.ko`）。
- BSP 三件套文件：`fork.conf`（上游指针 / 板列表 / rkbin 选型）、`kconfig.conf`（开关覆盖）、`0xxx-*/ *.patch`（源码差量）。

### 串口 console 设备名错配

`fiq_debugger` 把硬件 `ttyS2` 改名成 `ttyFIQ0`，但启动参数仍写 `ttyS2`：

```text
内核日志与串口登录输出指向不存在的设备，看不到 login:，但系统本身没崩溃。
```

### 内核温控 IPA Oops

6.1.84 厂商内核，温控 IPA（power_allocator）访问未初始化的 devfreq 锁：

> [!error] 报错原文
> ```text
> mutex_lock bad magic
> ```
> 系统真死机，`multi-user.target` 挡不住该 bug；需重新编译内核（thermal fix）再刷入。

### 摄像头设备排查

```bash
ls /dev/video*
v4l2-ctl --list-devices           # 列出可控制设备
lsusb | grep -iE 'cam|uvc|video'  # 从 USB 里筛摄像头
ls -l /dev/video-camera0          # rk 为 MIPI 主输出节点建的软链接 → video11
```

- 一个 UVC 摄像头会有多个节点：`/dev/video0` 取图，`/dev/video1` 是元数据设备。
- `media-ctl` 管硬件通路布线，`v4l2-ctl` 管采集参数；必须先搭链路再打开 video 节点。

```bash
media-ctl -p                                                # 查看现有链路
media-ctl --links                                           # 搭建流水线
v4l2-ctl --list-formats-ext                                 # 确认可用格式
v4l2-ctl -d /dev/video0 --stream-mmap --stream-count=100    # 抓 100 帧验证
```

## 镜像构建

### 四套构建方式与资源占用

| 方式 | SDK 目录 | 产出 | 一句话 |
| --- | --- | --- | --- |
| BSP | `~/bsp` | u-boot / linux-image 的 **deb 包** | 容器 + profile 交叉编译打包 |
| rsdk | `~/rsdk` | `output.img` | Debian 源组装 rootfs |
| Buildroot | `~/rk3588_sdk` | `update.img` | 极简嵌入式整包 |
| Yocto | `~/yocto-rockchip-sdk` | `core-image-*.wic` | 源码级构建整个发行版 |

资源实测：`rk3588_sdk` 整目录约 **69G**；Yocto 首次构建划分 300GB 仍不够，32GB 内存实际消耗 40GB+：

```bash
df -Th
du -h -d 1 ~/rk3588_sdk/
```

### Yocto 工程结构

```text
yocto-rockchip-sdk/
├── bitbake               # 构建核心工具（类比 gcc）
├── build                 # 编译工作区（唯一可随意改）
├── meta-poky             # Poky 官方核心发行版层
├── meta-rockchip         # 瑞芯微 RK3588 / ROCK 5C 硬件 BSP 层（关键）
├── meta-openembedded     # 嵌入式通用软件合集
├── poky                  # Yocto 上游原版仓库
└── oe-init-build-env     # 环境初始化脚本（每次编译必需 source）
```

### Buildroot 换源与重编译

> [!warning] 图形化改软件源不生效
> 需直接改对应包的 `.mk`：
> ```bash
> cd buildroot
> vim package/make/make.mk
> # MAKE_SITE = $(BR2_GNU_MIRROR)/make
> MAKE_SITE = https://mirrors.tuna.tsinghua.edu.cn/gnu/make
> ```

```bash
# 清理缓存重编
rm -rf output/rockchip_rk3588
make O=output/rockchip_rk3588 rockchip_rk3588_defconfig
cd .. && ./build.sh all
```

Buildroot 内核换 radxa 分支：

```bash
cd kernel
git remote add radxa https://github.com/radxa/kernel.git
git fetch radxa
git checkout -b linux-6.1-stan-rkr4.1-buildroot remotes/radxa/linux-6.1-stan-rkr4.1-buildroot
```

### VMware 磁盘精简与扩容

```powershell
# 厚置备 → 精简（生成一份拷贝后换盘）
.\vmware-vdiskmanager -r "E:\VMWare\Ubuntu_2204\Ubuntu-2204-64-bit.vmdk" -t 1 "E:\VMWare\Ubuntu_2204\Ubuntu-2204-64-bit-thin.vmdk"
# 只能扩容、不能缩容
.\vmware-vdiskmanager -x 500GB "E:\VMWare\Ubuntu_2204\Ubuntu-2204-64-bit-thin.vmdk"
```

```bash
sudo apt install -y cloud-guest-utils
sudo growpart /dev/sda 3
sudo resize2fs /dev/sda3
```

## 烧录与启动验证

### 启动链与灯显判断

```text
BootROM(片内 SRAM)  →  读扇区 64    → idbloader.img  （TPL 训 DDR + SPL）
                    →  读扇区 16384 → u-boot.itb       （含 BL31/ATF + U-Boot 本体）
                    →  加载 Image + dtb → Linux 内核 → 挂 rootfs → user space
```

| 阶段 | PC 在哪 | 蓝灯 |
| --- | --- | --- |
| BootROM / TPL 训 DDR / SPL | SRAM | 不会闪 |
| U-Boot | DRAM | 一般不闪 |
| 内核已启动、LED 驱动探测完成 | DRAM | **最早从这里开始闪** |
| 挂 rootfs、systemd、出现 `login:` | 用户态 | 早就在闪 |

> [!note] 心跳灯只代表内核还活着，与 rootfs / 桌面 / SSH 无关。

### GPT 分区布局（16 MiB 保留区）

```text
扇区 0~33      GPT 元数据 + 保护 MBR
扇区 64        idbloader.img 起始        （BootROM 第一跳）
扇区 16384     u-boot.itb 起始           （SPL/TPL 之后加载的 FIT）
扇区 32768     分区 1 开始（16 MiB）      ← Radxa 约定，前面全留给 Bootloader
```

- Radxa 规定第一个 GPT 分区从 16 MiB 起，保证 `mkfs` 只格式化分区内部、不擦掉扇区 64 / 16384 的 Bootloader。
- 这是 Rockchip / Radxa 平台约定，不是 GPT 规范要求。

### 板端分区结构（eMMC 为例）

```text
mmcblk0p1  config   16M  vfat  ← rsetup：before.txt / config.txt / after.txt
mmcblk0p2  efi      300M vfat  ← ESP（extlinux、Image、dtb）
mmcblk0p3  rootfs   232G ext4  ← 完整 Linux 系统
mmcblk0boot0 / boot1  4M       ← eMMC 原生隔离 Boot 分区，存 IDBLoader
```

五级链式启动：**MaskROM → IDBLoader(TPL/SPL) → ATF(BL31) → U-Boot → Kernel → Rootfs**。

### SD → eMMC 迁移

- 分区可灵活划分、数据可搬，但**只搬普通分区不够**：`u-boot` 等引导内容移动不当，eMMC 无法启动。
- `mmcblk0boot0 / boot1` 是 MaskROM 唯一直接读取的区域；普通分区（config / efi / rootfs）MaskROM 完全不识别。

## 踩坑与解决记录

### 自编译镜像蓝灯闪但不进用户态

两个独立根因，**必须同时处理，只修一个无效**：

1. 串口 console 定位错误：`ttyS2` 被改为 `ttyFIQ0`，启动参数未同步 → 看不到 login（系统没崩）。
2. 内核温控 IPA bug：`mutex_lock bad magic` → 系统真死机。

处理：重新编译内核加入 thermal fix，并修正启动参数中的 console 设备名。

### 自编译 Arch 镜像无法启动

- 烧录官方镜像可正常启动 → 板子、电源、串口等硬件无问题。
- 自制镜像蓝色系统灯**完全不闪** → 问题在引导 / 内核阶段；若蓝灯已开始闪但没进用户态，先查上面两条根因。

### Cursor / SSH 连接不稳定

- Cursor 的 ssh 扩展经过一夜会挂掉。
- 开 Clash 会影响 Cursor 的 SSH 连接（TUN / 全局代理干扰局域网），建议开「绕过局域网」或临时关 TUN。

### VMware 相关

> [!warning] 可移动设备 / 虚拟网卡异常
> `services.msc` 确认 **VMware USB Arbitration Service** 正在运行；服务没开会导致读卡器无法挂进虚拟机、虚拟网卡关闭进而 SSH 断连。

- 虚拟机若设为兼容 USB 2.0，读卡器必须插 USB 2.0 口，插 3.0 口识别不了。

### Git 下载与拉取问题

- 无代理直连 GitHub 报：

  ```text
  fatal: unable to access 'https://github.com/airockchip/rknn-toolkit2.git/':
  Failed to connect to github.com port 443 after 21077 ms: Could not connect to server
  ```

  → 用加速工具或镜像；有时挂 VPN 反而不通、关掉才通。
- 大仓库子模块单独下载：`git submodule sync && git submodule update --init`（但子模块最好别用这套）。

### 文件系统与下载 403

- FAT32（vfat）：兼容性最好，单文件最大 4GB、分区最大 2TB。
- ext4：Linux 根文件系统默认；NTFS：Windows 默认；exFAT：Linux 默认不预装驱动。
- 编译期下载报 403：访问地址非完整文件路径或权限不足，改用完整 URL 手动下载。
