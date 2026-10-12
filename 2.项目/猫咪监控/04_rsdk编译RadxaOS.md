---
type: 项目
status: 整理中
tags:
  - 技术/Linux
project: 猫咪监控
created: "2026-08-11"
updated: "2026-08-11"
---

# 04 rsdk 编译 RadxaOS（定制系统的基础）

> 官方文档：https://docs.radxa.com/rock5/rock5c/low-level-dev/rsdk
> rsdk 详细文档：https://radxaos-sdk.github.io/rsdk/（本地源码版在 `/home/wbh808/rsdk/docs/`）
> 本地目录：`/home/wbh808/rsdk`
> 学习目标：理解 rsdk 的"用 Debian 软件源组装系统"原理，掌握 ROCK 5C 的 RadxaOS 构建，以及通过 `packages` / `customize-hooks` / `--debs` 定制自己的发行版。

---

## 1. 原理：rsdk 是怎么"拼"出一个 RadxaOS 的

### 1.1 技术栈

```text
rsdk build rock-5c bookworm cli
  │
  ├─ jsonnet 模板（声明式配置）
  │     rootfs.jsonnet  → 决定装哪些包、跑哪些定制 hook
  │     image.jsonnet   → 生成 guestfish 脚本（磁盘分区/格式化）
  ├─ bdebstrap / mmdebstrap
  │     → 从 Debian/Radxa 软件源直接解包安装，生成 rootfs.tar
  ├─ guestfish（libguestfs）
  │     → 把 rootfs 写进磁盘镜像并配置分区/bootloader
  ├─ bash 胶水：rsdk-build（CLI 前端）
  │     → 调度上面所有步骤，输出 out/<产品>_<套件>_<版本>/output.img
  └─ devenv（Nix 系，仅开发环境用）
        → 容器里预装好所有构建工具，保证环境一致
```

### 1.2 为什么这条路线最适合"快速定制发行版"

- **Debian 生态**：ROCK 5C 的 RadxaOS 基于 Debian bookworm，`apt` 能装的包（包括 RKNN/RKLLM 板端运行时）都能进镜像。
- **模块化**：修改 `rootfs.jsonnet` 一处，即可增删包、增删定制脚本，改完重跑即可。
- **产物即镜像**：`output.img` 是可直接烧录的 SD/eMMC 镜像。
- **与 BSP 联动**：`rsdk build --debs <目录>` 能把你用 bsp 编译的自定义内核/u-boot deb 直接装进新系统——这正是"只含所需驱动 + AI"主推路线。

### 1.3 本地状态

```text
/home/wbh808/rsdk/
├── src/bin/rsdk / src/libexec/rsdk/rsdk-build   # CLI 与核心逻辑
├── src/share/rsdk/
│   ├── configs/products.json                    # 已支持产品（含 rock-5c）
│   ├── configs/socs.json                        # SoC 定义
│   └── build/rootfs.jsonnet                     # rootfs 模板（定制入口）
├── docs/                                        # 完整文档（mdBook）
├── debs/                                        # 可放本地 deb
├── .devcontainer/                               # 开发容器配置
└── /home/wbh808/rsdk_0.1.0_all.deb              # 已构建好的安装包
```

ROCK 5C 在 `products.json` 中的定义：

```json
{
  "product": "rock-5c",
  "product_name": "ROCK 5C",
  "product_full_name": "ROCK 5C",
  "soc": ["rk3588s2"],
  "sector_size": [512],
  "supported_suite": ["bookworm"],
  "supported_edition": ["kde", "cli"]
}
```

即：ROCK 5C 支持 Debian bookworm，有 `kde`（桌面）和 `cli`（命令行，最小）两个版本。

---

## 2. 安装 rsdk（两种方式）

### 方式一：安装本机已构建的 deb（最快）

```bash
cd /home/wbh808
sudo apt install ./rsdk_0.1.0_all.deb
```

若提示缺依赖，先补齐：

```bash
sudo apt update
sudo apt install -y bdebstrap mmdebstrap jsonnet guestfs-tools
```

### 方式二：DevContainer（推荐用于开发/二次开发）

```bash
cd /home/wbh808/rsdk
npm install @devcontainers/cli
export PATH="$PWD/src/bin:$PWD/node_modules/.bin:$PATH"
rsdk devcon up
rsdk devcon        # 进入开发容器，出现 RSDK ASCII logo 即成功
```

依赖：`git qemu-user-static binfmt-support npm docker.io`，且用户需加入 docker 组（`sudo usermod -aG docker $USER` 后重新登录）。

> 国内网络提示：GitHub 慢时可改用 `git clone https://gitcode.com/radxaos-sdk/rsdk.git`，Radxa 源可换 `https://mirrors.sdu.edu.cn/radxa-repo/bookworm/`。

### 方式三：源码构建 deb（Debian 12）

```bash
cd /home/wbh808/rsdk
sudo apt-get install devscripts dpkg-buildpackage build-essential
sudo apt build-dep .
dpkg-buildpackage -us -uc -b
sudo apt install ../rsdk_*.deb
```

---

## 3. 使用 rsdk

### 3.1 TUI 向导（入门）

```bash
rsdk
```

选择 `Build system image` → 用**空格键**勾选 `rock-5c` → 选择套件/版本（bookworm + cli）→ 配置镜像（国内建议选镜像源）→ 确认构建。

### 3.2 CLI（推荐，可脚本化）

```bash
# 最简：用产品默认值（bookworm，cli）
rsdk build rock-5c

# 显式指定套件与版本
rsdk build rock-5c bookworm cli
rsdk build rock-5c bookworm kde

# 国内镜像加速（可选）
rsdk build -M https://mirrors.ustc.edu.cn/radxa-deb \
           -m https://mirrors.ustc.edu.cn \
           rock-5c bookworm cli

# 第三方 radxa-deb 镜像不带 pkgs.json 时，禁用 pkgs.json
rsdk build -P rock-5c bookworm cli
```

参数：

- `-M <radxa-mirror>`：Radxa 软件源镜像
- `-m <distro-mirror>`：Debian/Ubuntu 源镜像
- `-P / --no-pkgs-json`：不下载嵌入 pkgs.json
- `--debs <目录>`：集成本地 deb（见第 4.3 节）

### 3.3 产物

```text
/home/wbh808/rsdk/out/rock-5c_bookworm_cli/
├── output.img      # 最终镜像（烧录用）
├── rootfs.tar      # rootfs 归档
├── config.yaml     # 本次构建配置
├── debs/           # 集成进去的本地 deb（如有）
└── manifest / build-image / seed.tar.xz
```

---

## 4. 定制方法（核心）

> 所有定制都建议先阅读 `/home/wbh808/rsdk/docs/tutorials/customize.md` 与 `local_pkg.md`。

### 4.1 往镜像加包：修改 rootfs.jsonnet

文件：`/home/wbh808/rsdk/src/share/rsdk/build/rootfs.jsonnet`

在 `mmdebstrap+` 的 `packages+` 数组里追加（文件里已有注释示例位置）：

```jsonnet
packages+:
[
    // Adding additional packages to the system
    // For example:
    "nano",
    "python3-pip",
    "python3-venv",
    "git",
    "curl",
    "i2c-tools",
    "rknpu2-rk3588",     // NPU 用户态（Radxa 源提供，AI 部署必需）
],
```

> 注意：`packages` 里的包必须能被配置的软件源提供（Debian 源 + Radxa radxa-deb 源）。

### 4.2 加定制动作：customize-hooks

同一个文件里，`customize-hooks+` 数组追加命令。hook 在包安装完成后、镜像打包前执行，`$1` 是挂载的 rootfs 根目录，可用 `chroot "$1"` 执行板内命令：

```jsonnet
"customize-hooks"+:
[
    // 示例：写入自启动服务
    |||
        set -e
        cat > "$1/etc/systemd/system/my-ai.service" <<'EOF'
[Unit]
Description=My Edge AI Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/my-ai-app
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
        chroot "$1" systemctl enable my-ai.service
    |||,
    // 示例：从网络拉一个 deb 并安装
    // |||
    //     set -e
    //     wget -O "$1/tmp/x.deb" https://example.com/x.deb
    //     chroot "$1" dpkg -i /tmp/x.deb
    //     rm "$1/tmp/x.deb"
    // |||,
],
```

### 4.3 集成本地 deb（BSP 产物、自研包）——关键功能

把 bsp 编译出的内核/u-boot deb、以及你自研的 AI 运行时 deb 放进一个目录，然后：

```bash
mkdir -p /home/wbh808/rsdk/debs
cp /home/wbh808/bsp/linux-image-*_arm64.deb /home/wbh808/rsdk/debs/
cp /home/wbh808/bsp/linux-headers-*_arm64.deb /home/wbh808/rsdk/debs/
cp /home/wbh808/bsp/linux-dtb-*_arm64.deb /home/wbh808/rsdk/debs/
cp /home/wbh808/bsp/u-boot-rk2410_*_arm64.deb /home/wbh808/rsdk/debs/

rsdk build --debs /home/wbh808/rsdk/debs rock-5c bookworm cli
```

行为说明：

- 目录内 deb 会被复制进目标系统并建成**本地 apt 仓库**（`/srv/local-apt-repository`）。
- 本地仓库 pin 优先级为 **1999**，只要包名匹配，就会**覆盖官方源**——这正是"替换成自己的内核/驱动"的实现方式。
- 若你希望某个 deb **默认安装**（而不是仅作为候选），需要在 `rootfs.jsonnet` 的 `packages` 里显式列出包名。
- 注意：本地 apt 仓库会保留在 rootfs 里，可能阻止后续在线升级同名包；如不希望如此，构建后再清理 `/srv/local-apt-repository`。

### 4.4 让默认安装包含你的包

如果包不在默认安装列表里，必须修改 `rootfs.jsonnet` 的 `packages`（4.1）或相关的类别文件：

```text
src/share/rsdk/build/mod/packages/
├── categories/core.libjsonnet   # 核心包（init、sudo、网络…）
├── cli.libjsonnet               # CLI 版本附加包
├── kde.libjsonnet / gnome...    # 桌面版本附加包
└── packages.libjsonnet          # 汇总入口
```

例如把 `rknpu2-rk3588`、`librkllm` 加进 `categories/core.libjsonnet`，则所有版本都包含。

### 4.5 新增产品（进阶，理解即可）

如果要为"新板卡/新硬件"制作发行版，需要动 4 处：

1. 内核元包：`radxa-repo/bsp` 的 linux profile（`fork.conf` 加产品）
2. 固件/u-boot 元包：bsp 的 u-boot profile
3. 产品元包：`radxa-pkg/radxa-profiles`（拉取设备特定驱动与配置）
4. `products.json`：注册产品及其默认配置（soc、suite、edition）

新 SoC 还需改 `socs.json`、新增 SoC 专属 apt 源等。ROCK 5C 已在支持列表内，一般不需要。

---

## 5. 常见问题

**Q1：devcon 启动失败？**

编辑 `.devcontainer/devcontainer.json` 的 `runArgs`；确认 docker 服务正常、当前用户在 docker 组。

**Q2：GitHub 速率限制？**

开发容器初始化提示 rate limited 时，按输出提示配置 GitHub token 或更换网络。

**Q3：下载软件包慢？**

构建时配置镜像：`-M`（radxa-deb）与 `-m`（Debian）都换国内源。

**Q4：自定义内核没生效？**

确认 `--debs` 目录里有完整的 `linux-image` / `linux-headers` / `linux-dtb`，且 `rootfs.jsonnet` 的 `packages` 里包含对应的 `linux-image-*` 包名；检查构建日志中本地仓库是否被采用（pin 1999）。

---

## 6. 本节产出与下一步

完成本节后：

- 能一键构建 ROCK 5C 的 RadxaOS CLI/桌面镜像
- 能往镜像加包、加定制脚本、集成本地 deb
- 具备"以 rsdk 为骨架打造发行版"的全部基础能力

下一步：

- 端侧 AI 软件栈 → 文档 05
- 综合实战（定制内核 + rsdk 组装 + AI）→ 文档 06
