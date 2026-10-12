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

# 05 端侧 AI 部署：RKNN（视觉）与 RKLLM（大模型）

> 官方文档：https://docs.radxa.com/rock5/rock5c/app-development/ai（RKNN 安装 / RKLLM 安装 / 各模型示例）
> 本地资源：
> - `/home/wbh808/rknn-toolkit2-2.3.2`（RKNN-Toolkit2 / Lite2 / Runtime / NPU 用户态库）
> - `/home/wbh808/rknn-llm-release-v1.2.3`（RKLLM Toolkit / Runtime / NPU 驱动源码）
> - `/home/wbh808/rknn_model_zoo-main`（示例模型）
> - `/mnt/hgfs/DeepSeek-R1-1.5B`、`/mnt/hgfs/Qwen2.5-1.5B-Instruct`、`/mnt/hgfs/DeepSeek-R1-Distill-Qwen-7B`（模型权重）
>
> 学习目标：理解 RKNN/RKLLM 的软件栈，掌握"PC 端模型转换 → 板端推理"全流程，并明确把 AI 运行时打包进自制发行版所需的全部组件。

---

## 1. 原理：RK3588 NPU 与软件栈

### 1.1 硬件

ROCK 5C 的 RK3588S 集成 NPU（6 TOPS 算力），支持 INT4/INT8/INT16/FLOAT 推理，可同时运行视觉模型（RKNN）和大语言模型（RKLLM）。NPU 在系统中的表现：

```text
/dev/rknpu                  # 设备节点
内核模块：rknpu             # lsmod 可见
用户态库：librknnrt.so     # RKNN C API
          librkllmrt.so    # RKLLM C API
```

### 1.2 软件栈总览

```text
                PC 端（x86）                        板端（ROCK 5C, arm64）
        ┌─────────────────────────┐        ┌──────────────────────────────┐
        │ RKNN-Toolkit2 (Python)  │ 转换   │ RKNN-Toolkit-Lite2 (Python)  │
        │ RKLLM-Toolkit (Python)  │ ────→  │ 或 RKNN/RKLLM Runtime (C API)│
        │ 模型: .pt/.onnx/.h5     │ 量化   │ 模型: .rknn / .rkllm         │
        └─────────────────────────┘        └──────────────────────────────┘
                                                     │
                                            用户态: librknnrt / librkllmrt
                                                     │
                                            内核: rknpu 驱动 (0.9.x)
                                                     │
                                            硬件: RK3588S NPU
```

### 1.3 两条产品线

| | RKNN | RKLLM |
| --- | --- | --- |
| 用途 | 视觉/传统模型：YOLO、分类、分割、OCR、语音 | 大语言模型：Qwen、DeepSeek、Llama、Phi |
| 模型格式 | `.rknn` | `.rkllm` |
| PC 端工具 | rknn-toolkit2（Python） | rkllm-toolkit（Python） |
| 板端 API | rknn-toolkit-lite2（Python）/ RKNN Runtime（C） | RKLLM Runtime（C） |
| NPU 驱动要求 | ≥ 0.9.2（推荐 0.9.6+） | ≥ 0.9.8 |
| 本机版本 | 2.3.2 | release-v1.2.3 |

> 关键结论：**做定制发行版时，板端只需打包"NPU 内核驱动 + 用户态 .so + 模型文件 + 推理程序"**，转换工作永远在 PC 端完成，不需要把整条工具链装到板子上。

---

## 2. 环境准备（PC 端）

### 2.1 Miniforge / conda

本机已安装 Miniforge（`/home/wbh808/miniforge3`）。建议按工具分开建环境：

```bash
# RKNN 视觉：Python 3.8（配 Ubuntu 22.04）
conda create -n rknn python=3.8 -y
conda activate rknn

# RKLLM 大模型：Python 3.12
conda create -n rkllm python=3.12 -y
conda activate rkllm
```

### 2.2 安装 RKNN-Toolkit2（PC 端转换）

```bash
conda activate rknn
cd /home/wbh808/rknn-toolkit2-2.3.2/rknn-toolkit2/packages
ls
# 选择对应 cp38 的 x86_64 wheel
pip install rknn_toolkit2-2.3.2-cp38-cp38-manylinux_2_17_x86_64.manylinux2014_x86_64.whl
```

验证：

```bash
python3 -c "from rknn.api import RKNN; print('RKNN-Toolkit2 OK')"
```

### 2.3 安装 RKLLM-Toolkit（PC 端转换）

```bash
conda activate rkllm
cd /home/wbh808/rknn-llm-release-v1.2.3/rkllm-toolkit/packages
pip install rkllm_toolkit-1.2.3-cp312-cp312-linux_x86_64.whl
python3 -c "from rkllm.api import RKLLM; print('RKLLM-Toolkit OK')"
```

### 2.4 交叉编译工具链（编译板端 C 代码）

本机已备好，在 `/mnt/hgfs/`：

```bash
cd /mnt/hgfs
tar xf gcc-arm-11.2-2022.02-x86_64-aarch64-none-linux-gnu.tar.xz -C ~/tools  # 建议统一放 ~/tools
export GCC_COMPILER=$HOME/tools/gcc-arm-11.2-2022.02-x86_64-aarch64-none-linux-gnu/bin/aarch64-linux-gnu
```

---

## 3. RKNN 视觉模型：转换与板端部署

### 3.1 用 rknn_model_zoo 快速验证（推荐）

`/home/wbh808/rknn_model_zoo-main` 提供了 YOLO 等现成示例：

```bash
cd /home/wbh808/rknn_model_zoo-main/examples/yolov8/python
# 1) 下载 ONNX 模型（或用自训模型导出 ONNX）

# 2) PC 端转换：将 ONNX 转成 RKNN 并量化
python3 convert.py <onnx模型> rk3588 i8 rknn_model_yolov8.rknn
```

关键 API 流程（PC 端转换脚本模板）：

```python
from rknn.api import RKNN

rknn = RKNN(verbose=True)
# 1. 配置（target_platform='rk3588'，量化 i8/u8/fp）
rknn.config(target_platform='rk3588', mean_values=[[0,0,0]], std_values=[[255,255,255]])
# 2. 加载模型
rknn.load_onnx(model='model.onnx')
# 3. 构建
rknn.build(do_quantization=True, dataset='dataset.txt')
# 4. 导出 .rknn
rknn.export_rknn('model.rknn')
rknn.release()
```

### 3.2 板端推理（Python：rknn-toolkit-lite2）

在 ROCK 5C（或自制镜像）上：

```bash
# arm64 的 lite2 wheel（本机包在 rknn-toolkit2-2.3.2/rknn-toolkit-lite2/packages）
cd /home/wbh808/rknn-toolkit2-2.3.2/rknn-toolkit-lite2/packages
pip install rknn_toolkit_lite2-2.3.2-cp3X-cp3X-manylinux_2_17_aarch64.manylinux2014_aarch64.whl
```

推理脚本：

```python
from rknnlite.api import RKNNLite

rknn = RKNNLite()
rknn.load_rknn('model.rknn')
rknn.init_runtime()          # 默认用 NPU
outputs = rknn.inference(inputs=[img])
rknn.release()
```

### 3.3 板端推理（C：RKNN Runtime）

```bash
# 用户态库位于 rknn-toolkit2-2.3.2/rknpu2/runtime/

# 板端 .so 与头文件：

#   rknpu2/runtime/Linux/librknn_api/aarch64/librknnrt.so

#   rknpu2/runtime/Linux/librknn_api/include/rknn_api.h
```

编译示例：

```bash
export GCC_COMPILER=$HOME/tools/gcc-arm-11.2-2022.02-x86_64-aarch64-none-linux-gnu/bin/aarch64-linux-gnu
cd /home/wbh808/rknn-toolkit2-2.3.2/rknpu2/examples/rknn_yolov5_demo
./build-linux.sh -c rk3588 -a aarch64
```

把生成的二进制 + `librknnrt.so` 拷到板子即可运行。

### 3.4 NPU 驱动检查与安装

```bash
# 板端检查
ls -l /dev/rknpu
lsmod | grep rknpu
sudo dmesg | grep -i "Initialized rknpu"
# 期望类似：Initialized rknpu 0.9.6/0.9.8 ...

# 若缺失（RadxaOS 上）：
sudo apt update && sudo apt install rknpu2-rk3588
sudo reboot
```

> 自制发行版注意：`/dev/rknpu` 需要内核驱动 + 设备树节点（RK3588 默认已带 `rknpu@fdab0000`）。把 `rknpu2-rk3588` 装进镜像（rsdk 的 packages 或 Buildroot 的 overlay）即可，详见文档 06。

---

## 4. RKLLM 大模型：转换与板端部署

### 4.1 转换（PC 端，rkllm 环境）

```bash
conda activate rkllm
cd /home/wbh808/rknn-llm-release-v1.2.3/rkllm-toolkit/examples
```

转换脚本模板（以 Qwen/DeepSeek 为例）：

```python
from rkllm.api import RKLLM

llm = RKLLM()
# 加载 HuggingFace 格式模型目录
llm.load_huggingface(model='/mnt/hgfs/Qwen2.5-1.5B-Instruct')
# 配置量化（w4a16 是推荐的大模型量化档）
llm.build(target_platform='rk3588', optimization_level=1,
          quantized_dtype='w4a16', num_npu_core=3)
# 导出 .rkllm
llm.export_rkllm('/mnt/hgfs/Qwen2.5-1.5B-Instruct.rkllm')
llm.release()
```

`/home/wbh808/rknn-llm-release-v1.2.3/rkllm-toolkit/examples` 下有 `Llama2`、`Qwen2`、`DeepSeek-R1` 等现成脚本，直接改模型路径即可。`benchmark.md` 里有各模型的耗时数据（ROCK 5C 同 RK3588S）。

### 4.2 板端编译 RKLLM Runtime Demo

```bash
export GCC_COMPILER=$HOME/tools/gcc-arm-11.2-2022.02-x86_64-aarch64-none-linux-gnu/bin/aarch64-linux-gnu
cd /home/wbh808/rknn-llm-release-v1.2.3/rkllm-runtime
ls Linux/librkllm_api/         # 头文件 + aarch64 静态/动态库

# 以官方 demo 为例
cd Linux/librkllm_api/example
./build-linux.sh -t rk3588
```

产物是 `llm_demo` 之类的可执行文件。运行时需要：

```text
librkllmrt.so          # 用户态库（复制到 /usr/local/lib）
librkllm_api/include   # 头文件（编译期）
模型文件 *.rkllm
NPU 驱动 ≥ 0.9.8
```

### 4.3 板端运行

```bash
sudo ldconfig
./llm_demo /mnt/hgfs/xxxx.rkllm   # 或示例自带脚本
```

### 4.4 重要：RKLLM 需要 NPU 驱动 ≥ 0.9.8

Radxa 6.1 官方固件默认 0.9.6，需要：

```bash
sudo rsetup  → System → System Update   # 升级系统与驱动
sudo apt autopurge && sudo reboot
sudo cat /sys/kernel/debug/rknpu/version   # 期望 RKNPU driver: v0.9.8
```

若使用自制内核（bsp/Buildroot/Yocto 构建），NPU 驱动源码在：

```text
/home/wbh808/rknn-llm-release-v1.2.3/rknpu-driver/
```

把 `rknpu` 驱动目录合并进内核源码（`drivers/video/rockchip/...` 对应位置，或按 README 指引），并确保内核配置 `CONFIG_ROCKCHIP_RKNPU=y` 后重编内核。

---

## 5. 打包进自制发行版：组件清单

无论用 rsdk、Buildroot 还是 Yocto 做发行版，AI 能力最终由以下几部分组成：

| 组件 | 来源 | 打包位置 |
| --- | --- | --- |
| 内核 rknpu 驱动 | 内核编译选项 `CONFIG_ROCKCHIP_RKNPU=y`（或 rknn-llm 的 rknpu-driver 源码） | 内核镜像 |
| 设备树节点 | `rk3588s.dtsi` 自带 `rknpu@fdab0000`（一般无需改） | dtb |
| `librknnrt.so` | `rknn-toolkit2-2.3.2/rknpu2/runtime/.../aarch64/` | `/usr/lib` 或 `/usr/local/lib` |
| `librkllmrt.so` | `rknn-llm-release-v1.2.3/rkllm-runtime/Linux/librkllm_api/` | `/usr/lib` 或 `/usr/local/lib` |
| Python 运行时（可选） | `rknn-toolkit-lite2` wheel（arm64） | pip 安装进 rootfs |
| 推理程序 | 自研/示例编译产物 | `/usr/local/bin` |
| 模型文件 `.rknn/.rkllm` | PC 端转换 | `/opt/models`（按需） |
| systemd 服务 | 自写 unit | `/etc/systemd/system` |

> 最小化建议：如果只需要跑固定模型，**只放 librknnrt/librkllmrt + 推理二进制 + 模型**，不要装 Python 工具链，镜像体积与启动速度最优。

---

## 6. 常见问题

**Q1：`failed to open rknpu module` / `failed to open rknn device`？**

内核驱动没加载或用户态库缺失。依次检查：`lsmod | grep rknpu`、`ls -l /dev/rknpu`、`ldconfig -p | grep rknn`。

**Q2：RKLLM 跑起来后速度慢/报版本错误？**

确认 `cat /sys/kernel/debug/rknpu/version` ≥ 0.9.8；量化建议 `w4a16`；`num_npu_core` 可设 3。

**Q3：转换报 `no available devices`？**

RKNN-Toolkit2 转换时不需要板子（纯 x86），检查是否误装了 lite2 或 wheel 与 Python 版本不匹配（cp38 wheel 需 py3.8 环境）。

**Q4：模型转换后精度差？**

检查 `dataset.txt` 的校准图片数量（建议 ≥ 几百张）与预处理参数（mean/std、RGB/BGR）是否与训练时一致。

---

## 7. 本节产出与下一步

完成本节后：

- 能完成 RKNN 视觉模型转换与板端推理
- 能完成 RKLLM 大模型转换、runtime 编译与运行
- 明确自制发行版需要打包的 AI 组件清单

下一步：文档 06 把这些能力与内核定制、rootfs 定制整合成完整方案。
