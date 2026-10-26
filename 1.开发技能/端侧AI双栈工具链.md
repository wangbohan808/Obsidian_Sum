---
type: 原子
status: 整理中
mastery: 1
tags:
  - 技术/端侧AI
  - 技术/Rockchip
  - 技术/Qualcomm
created: "2026-10-20"
updated: "2026-10-20"
up: "[[00_开发技能MOC]]"
---

# 端侧 AI 双栈工具链：ROCK 5C 与 Dragon Q8B

> [!info] 一句话结论
> 手里这两块板子**不是同一个生态**：ROCK 5C 是 Rockchip（RKNN/RKLLM），Dragon Q8B 是 Qualcomm（QAIRT/QNN/Genie）。
> 这恰好构成端侧 AI 求职里最稀缺的组合——**同一件事在两种芯片上各做一遍**。
> 但学习顺序不能颠倒：先用熟的 ROCK 5C 打通闭环，再用 Q8B 做"概念迁移"。

---

## 1. 平台对照（先认清这是两套东西）

| 项目 | Radxa ROCK 5C | Radxa Dragon Q8B |
| --- | --- | --- |
| SoC | Rockchip RK3588S | Qualcomm Snapdragon 8cx Gen 3（SC8280XP） |
| 制程 | 8nm | 5nm |
| CPU | 4×A76 + 4×A55 | 4×Kryo Prime 3.0GHz + 4×Kryo Gold 2.4GHz |
| NPU | RKNPU，**6 TOPS** | Hexagon HTP，Qualcomm AI Engine **29+ TOPS** |
| NPU 架构版本 | — | **v68**（工具链路径/模型后缀多为 `v68`） |
| 内存 | LPDDR4x/5，最高 32GB | LPDDR4x 128-bit，最高 32GB |
| 定位 | 通用 SBC / 学习练手，资料厚、生态熟 | 旗舰边缘 AI 主板，双 2.5GbE、三路 4K120、UFS 3.1 |
| LLM 能力 | RKLLM，可跑 1.5B~7B（w4a16） | Genie，官方称可商用部署 **≤3B** 小模型 |
| 发布/成熟度 | 成熟，社区踩坑记录多 | **2026-06 新发布**，主线内核仍在推进中 |
| 学习曲线 | 平缓（已有自己的 19 号踩坑库） | 陡（高通工具链术语多、资料以官方文档为主） |

> [!warning] 关键差异
> RKNN 的 `.rknn` 与 QNN 的 `context.bin` **完全不能互通**。换板子 = 换整套工具链，不是改配置。
> 这正是它值钱的原因，也是它费时间的原因。

---

## 2. 概念映射表（最重要的一张表）

把两边当成同一件事的不同方言来记，迁移成本会低一个数量级：

| 环节           | Rockchip（ROCK 5C）                         | Qualcomm（Q8B）                                                              |
| ------------ | ----------------------------------------- | -------------------------------------------------------------------------- |
| 内核侧 NPU 通路   | `rknpu` 驱动 → `/dev/rknpu`                 | `fastrpc` + DSP 固件（需设 `ADSP_LIBRARY_PATH`）                                 |
| 板端运行时（C API） | `librknnrt.so`                            | `libQnnHtp.so` 等后端库                                                        |
| 模型产物         | 单个 `.rknn` 文件                             | 模型库 `.so` + `context.bin`（HTP 专用二进制）                                       |
| PC 端转换       | RKNN-Toolkit2                             | QNN Converter / `qnn-model-lib-generator` / `qnn-context-binary-generator` |
| 量化           | Toolkit2 内置 PTQ（`normal` / `mmse` 校准）     | **AIMET**（PTQ、QAT、AdaRound、CLE、层级敏感度分析）                                    |
| Python 快速上手  | RKNN-Toolkit-Lite2                        | **QAI AppBuilder**（Python / C++ 双 API）                                     |
| 高层易用运行时      | —（RKNN 只有一层）                              | **SNPE**（`.dlc` 格式，自动调度 CPU/GPU/HTP）                                       |
| 底层精细控制       | RKNN 自定义算子                                | **QNN**（算子级控制、逐层 profiling、可写 HTP 原生算子）                                    |
| LLM 运行时      | **RKLLM**（`.rkllm`）                       | **Genie**（消费 QNN context binary，管 tokenizer/KV Cache/采样/流式）                |
| 官方模型库        | `rknn_model_zoo` / `rkllm_model_zoo`      | **AI Hub Models**（`qai-hub-models`）                                        |
| 预编译模型        | SDK 包内 + Model Zoo                        | ModelScope `radxa/*_qairt`、AI Hub Workbench                                |
| 性能分析         | `RKNN_LOG_LEVEL`、`eval_perf_watch_npu.sh` | `qnn-profile-viewer`、QNN optrace、逐层 profiling                              |
| 一句话记忆        | **一层 API，够用就好**                           | **三层：SNPE 简单 / QNN 极致 / Genie 管 LLM**                                      |

> [!tip] 迁移心法
> RKNN ≈ QNN（都是底层直连 NPU）；RKLLM ≈ Genie（都是 LLM 专用封装）。
> SNPE 在 RKNN 侧**没有对应物**——它是"不想调硬件细节时"的高层快捷方式，可以先跳过。

---

## 3. ROCK 5C 资源清单

### 3.1 官方文档

| 内容 | 链接 |
| --- | --- |
| ROCK 5C 总文档 | https://docs.radxa.com/rock5/rock5c |
| AI 应用开发（RKNN/RKLLM 安装与示例） | https://docs.radxa.com/rock5/rock5c/app-development/ai |
| 底层开发（U-boot / Kernel / 编译 RadxaOS） | https://docs.radxa.com/rock5/rock5c/low-level-dev |

### 3.2 源码与 SDK（GitHub）

| 仓库 | 用途 | 备注 |
| --- | --- | --- |
| https://github.com/airockchip/rknn-toolkit2 | PC 端模型转换 + 量化 | 本地已有 2.3.2 |
| https://github.com/airockchip/rknn-llm | LLM 转换工具链 + 板端 runtime | **本地 1.2.3，最新已到 1.3.1**（新增 Qwen3-VL / DeepSeekOCR / InternVL3.5） |
| https://github.com/airockchip/rknn_model_zoo | 视觉模型示例（YOLO/分类/分割/OCR） | 必读：含各模型转换脚本与精度对比 |
| https://github.com/airockchip/rkllm_model_zoo | 已转换好的 `.rkllm` 模型 | 省去自己转的时间 |
| https://github.com/airockchip/rknpu | NPU 内核驱动源码 | 想做平台层必读 |

### 3.3 自己已有的资产（别重复造）

- `2.项目/猫咪监控/05_端侧AI部署-RKNN与RKLLM.md` —— 软件栈原理 + 完整流程
- `2.项目/猫咪监控/11_猫咪监控项目-模型裁剪与优化专题.md` —— 量化/剪枝/蒸馏
- `2.项目/猫咪监控/19_ROCK5C实战问题与解决.md` —— **真实踩坑库，面试最值钱的一份**
- 本机 SDK：`/home/wbh808/{bsp,rsdk,rk3588_sdk,yocto-rockchip-sdk}`、`rknn-toolkit2-2.3.2`、`rknn-llm-release-v1.2.3`

---

## 4. Dragon Q8B 资源清单

### 4.1 官方文档（Radxa）

| 内容 | 链接 |
| --- | --- |
| Q8B 总览（规格/接口/系统） | https://docs.radxa.com/dragon/q8b |
| **NPU 开发总目录**（起点） | https://docs.radxa.com/dragon/q8b/app-dev/npu-dev |
| 板端启用 NPU（T2 以上镜像已预装） | https://docs.radxa.com/dragon/q8b/app-dev/npu-dev/fastrpc-setup |
| NPU 开发指南（QAIRT 总体） | https://docs.radxa.com/dragon/q8b/app-dev/npu-dev/qairt-sdk |
| **NPU 快速验证**（resnet50 跑通，先做这个） | https://docs.radxa.com/dragon/q8b/app-dev/npu-dev/quick-example |
| QAIRT SDK 安装（含 ARM64 Docker 镜像） | https://docs.radxa.com/dragon/q8b/app-dev/npu-dev/qairt-install |
| **QAIRT 模型移植**（核心章节） | https://docs.radxa.com/dragon/q8b/app-dev/npu-dev/qairt-usage |
| QAIRT 工具（转换/编译/性能分析） | https://docs.radxa.com/dragon/q8b/app-dev/npu-dev/qairt-tools |
| **AIMET 量化工具** | https://docs.radxa.com/dragon/q8b/app-dev/npu-dev/aimet |
| QAI AppBuilder（Python/C++ 简接口） | https://docs.radxa.com/dragon/q8b/app-dev/npu-dev/qai-appbuilder |
| QAI Hub Models（官方优化模型库） | https://docs.radxa.com/dragon/q8b/app-dev/npu-dev/qai-hub-models |
| 模型下载中心 | https://docs.radxa.com/dragon/q8b/app-dev/npu-dev/model-zoo |
| ONNX Runtime QNN EP | https://docs.radxa.com/dragon/q8b/app-dev/npu-dev/qnn-onnxrt-execution-provider |
| TFLite Delegate 验证 | https://docs.radxa.com/dragon/q8b/app-dev/npu-dev/tflite-delegate-demo |
| **LLM 大模型示例**（Genie） | https://docs.radxa.com/dragon/q8b/app-dev/npu-dev/LLM |

LLM 示例目前两个：
- Llama3.2-1B：https://docs.radxa.com/dragon/q8b/app-dev/npu-dev/LLM/llama3.2-1b-qairt-v68
- Qwen2.5-0.5B：https://docs.radxa.com/dragon/q8b/app-dev/npu-dev/LLM/qwen2-5-0-5b-qairt-v68

### 4.2 源码仓库

| 仓库 / 资源 | 用途 |
| --- | --- |
| https://github.com/qualcomm/qai-appbuilder | 官方 Python/C++ 推理封装，含 6 个示例 |
| https://github.com/qualcomm/ai-hub-models | **官方模型库**：几百个已优化模型 + 量化/导出配方，可当量化教学读 |
| https://github.com/quic/aimet | AIMET 量化库源码，PTQ/QAT/AdaRound/CLE 实现 |
| https://quic.github.io/aimet-pages/ | AIMET 文档 |
| https://github.com/radxa/kernel | Radxa 内核，按 `sc8280xp` 搜 Q8B 的 dts 提交 |
| https://github.com/armbian/build/pull/10215 | Q8B 的 Armbian edge 内核（主线 7.1）移植 PR，**学平台 bring-up 的活教材** |
| https://docs.qualcomm.com/doc/80-63442-10/topic/general_overview.html | QAIRT SDK 官方文档根入口 |

### 4.3 模型与工具获取途径

- **ModelScope**：`modelscope download --model radxa/resnet50_qairt`（官方 quick example 用这个）
- **AI Hub Workbench**：https://aihub.qualcomm.com/ —— 需注册 Qualcomm ID + 配置 API token（`qai-hub configure --api_token xxx`），可云端编译+真机 profiling
- **QAIRT SDK**：随文档下载，当前文档路径对应版本 `2.42.0.251225`
- **Docker 镜像**（省去本地装 SDK）：
  ```bash
  sudo docker pull radxazifeng278/qairt-npu-v68:v1.2
  sudo docker run --privileged -it -v /dev:/dev -v $(pwd):/workspace --name qairt-sdk radxazifeng278/qairt-npu-v68:v1.2 /bin/bash
  ```

---

## 5. 推荐阅读顺序（三步走，别跳）

### 第 0 步：环境自检（半天）

```bash
# Q8B 上确认 NPU 通了
ls /dev/fastrpc*          # fastrpc 节点在
export PRODUCT_SOC=8280   # Q8B 是 SC8280XP
echo $ADSP_LIBRARY_PATH   # 没设就跑不了 NPU
```

- Q8B 需 **T2 或更高版本系统镜像**（NPU 环境已预装）
- 若用 wheel 方式装 QAI AppBuilder，`ADSP_LIBRARY_PATH` 指向包内 `libs/` 目录：
  ```bash
  export ADSP_LIBRARY_PATH=$(python3 -c "import os, qai_appbuilder; print(os.path.join(os.path.dirname(qai_appbuilder.__file__), 'libs'))")
  ```

### 第 1 步：Q8B 先跑通"官方 demo 不改代码"（1 周）

目标不是学会，是**建立信心 + 摸清文件长什么样**。

1. `quick-example` 的 resnet50：`qnn-net-run --backend ./libQnnHtp.so --retrieve_context ./resnet50_aimet_quantized_8280.bin ...`
2. 观察产物：`context.bin` 有多大？`libQnnHtp.so` 是什么角色？
3. 对照 QAI AppBuilder 的 Python 示例，用 10 行代码把同一个模型跑起来

> 验收：能说出"`.bin` 文件里装的是什么"。

### 第 2 步：Q8B 把一个**自己的**模型跑上去（2–3 周）

推荐用猫咪监控的 YOLOv8n —— 这样你有了**同一个模型在两个 NPU 上的实测数据**，这是简历上的黄金素材。

路径：`PyTorch → ONNX → QNN Converter（或 AIMET 量化）→ context.bin → qnn-net-run / QAI AppBuilder`

关键动作：
- 用 AIMET 做 PTQ，对比 `INT8` / `w8a16` 的精度与延迟
- 用 `qnn-profile-viewer` 看逐层耗时，和 RKNN 的 profiling 结果对照
- 记录：**同一模型，RK3588（6 TOPS）vs SC8280XP（29+ TOPS）的 ms 与 mAP**

> 验收：产出一张双平台对比表，能解释"为什么算力差 5 倍，延迟没差 5 倍"（答案指向内存带宽、算子支持度、图切分）。

### 第 3 步：Q8B 上跑 Genie 的 LLM（1–2 周）

用官方 Qwen2.5-0.5B 或 Llama3.2-1B 示例，理解 Genie 的三个核心概念：**Dialog / KV Cache / 流式回调**。

对照你已经跑过的 RKLLM：两者是同一层东西，把这层抽象吃透，以后换任何芯片（地平线、寒武纪、爱芯）都是同一套提问方式。

> 验收：能讲清"端侧 LLM 的瓶颈在哪三个地方"（首 token 延迟、decode 吞吐、KV Cache 内存）。

### 5.4 源码阅读：只挑一份的话，读 `rknn-llm`

> [!important] 结论（2026-10-20 定）
> **第一份读 `airockchip/rknn-llm`**，第二选择 `rknn_model_zoo/examples/yolov8/cpp/`。
> 不先读 `rknpu` 内核驱动——它回答的是"NPU 怎么被驱动"，不是"推理为什么慢"。

**为什么是它：一份仓库覆盖四层**

| 目录 | 内容 | 对应能力层 |
| --- | --- | --- |
| `rkllm-toolkit/` | PC 端转换 + 量化配置（Python） | L2 量化 |
| `rkllm-runtime/Linux/librkllm_api/` | 板端 C API（`rkllm.h`、callback、多实例） | L3 推理引擎 |
| `rknpu-driver/` | NPU 内核驱动源码 | L0（已强，略读即可） |
| `scripts/` + `benchmark.md` | 频率/利用率脚本、各模型实测数据 | **L4 性能工程** |

**为什么不选其他三个**

- `rknn-toolkit2`：核心转换逻辑闭源（只有 whl），仓库里是文档与示例 → 读了学不到
- `rknn_model_zoo`：结构雷同的重复示例，读一个等于读二十个（但 `yolov8/cpp` 例外，见下）
- `rknpu` 内核驱动：真开源、硬核，但距离"推理部署"岗的面试题较远，容易掉进内核泥潭

> [!note] 先认清限制
> `librkllmrt.so` 本体闭源。能读的是 **header + toolkit 的 Python + demo + 驱动**。
> 这不影响学习效果——反而逼你在接口与行为层理解，而不是陷进汇编。

**读法：五个问题定位，不从第一行读**

| # | 问题 | 去哪找 | 读懂的标志 |
| --- | --- | --- | --- |
| 1 | 为什么推理 API 是 callback 异步？ | `rkllm.h` 的 `RKLLM_INFER_ASYNC` + demo callback | 能说清"同步会让 UI 卡在 decode 阶段" |
| 2 | 为什么模型被拆成 prefill 和 decode？ | `rkllm-toolkit` build 流程 | 能解释两者计算特征差异（矩阵乘 vs 访存密集） |
| 3 | KV Cache 存哪、占多少？`max_context_len` 如何影响？ | runtime 的 cache 管理 + `benchmark.md` 内存列 | 能估算 4096 上下文 + 1.5B 模型的内存占用 |
| 4 | 为什么权重 w4、激活 a16？ | toolkit 量化配置 + benchmark 精度对比 | 能讲清"省带宽 vs 保精度"的权衡点 |
| 5 | `rknn_core_num=3` 怎么用上三个 NPU 核？ | runtime 的 core 绑定 + 驱动 | 能说明多实例/多核的调度边界 |

预期投入：**3 天读主干**（不是 3 周）。验收物：一张"从 HF 模型到板端 token"的全链路图 + 一篇笔记。

**第二选择：`rknn_model_zoo/examples/yolov8/cpp/`**
它回答另一个问题：**一张图从内存到 NPU 再回来，中间被搬了几次**。零拷贝、RGA 预处理、NHWC 布局、前后处理耗时——全是猫咪监控项目里的真实性能项。
选它的条件：想先把视觉主线的性能归因做扎实。选 `rknn-llm` 的条件：想补端侧 LLM 这块空白（**推荐**，因为视觉已有实操直觉）。

---

## 6. 两块板子怎么分工（别互相拖累）

| 板子 | 角色 | 干什么 | 不要干什么 |
| --- | --- | --- | --- |
| **ROCK 5C** | 主力 / 主线 | 继续把猫咪监控闭环做完；练端到端、成本敏感、系统裁剪 | 不要为了追新版本反复重装 |
| **Q8B** | 第二栈 / 加分项 | 只做三件事：跑通 demo、迁一个自有模型、跑一次 Genie | **不要试图在 Q8B 上复刻整个猫咪监控系统** |

> [!important] 时间纪律
> Q8B 的价值在于"第二栈"这个标签，不在于把它做成第二个完整项目。
> 总投入控制在 **4–6 周**，拿到双平台对比数据就收手，回到 ROCK 5C 主线。
> Q8B 平台新（2026-06 发布）、主线内核仍在推进，深挖会掉进 bring-up 的无底洞——那不是你现在的目标。

---

## 7. 已知的坑（Q8B 侧）

1. **NPU 跑不起来先查 `ADSP_LIBRARY_PATH`** —— 这是高通平台最常见的第一道墙。
2. **HTP 架构版本必须对**：Q8B 是 **v68**，模型/镜像后缀都带 `v68`；拿错版本（如 v73 的 QCS9075 产物）不能跑。
3. **`PRODUCT_SOC` 要设 8280**，不是 6490、不是 9075。
4. **镜像版本**：低于 T2 的系统镜像没有预装 NPU 运行环境。
5. **风扇不受 Linux 控制**：Q8B 的风扇由跑在 ADSP 上的 Radxa 固件服务驱动（社区有 `radxa-dragon-q8b-fan` 记录了这个机制）。别在 `/sys` 里找 pwm。
6. **ONNX Runtime 的 QNN EP 在 Linux 上编译要改 CMakeLists**（官方不直接支持），好消息是 Radxa 给了预编译 whl：
   `pip3 install https://github.com/ZIFENG278/onnxruntime/releases/download/v1.23.2/onnxruntime_qnn-1.23.2-cp312-cp312-linux_aarch64.whl`
7. **优先用 Docker 镜像跑 QAIRT**，别在板子上硬装 SDK——依赖地狱。

---

## 8. 这件事对转行的意义

面试时最有说服力的一句话不是"我会 RKNN"，而是：

> "同一个 YOLOv8n，我在 RK3588 的 6 TOPS RKNPU 和 SC8280XP 的 29 TOPS Hexagon 上都部署过，量化方案分别是 RKNN-Toolkit2 的 mmse 校准和 AIMET 的 AdaRound，实测延迟差异主要来自算子支持度和带宽而不是标称算力。"

这句话同时命中了三件事：**跨栈能力、量化深度、性能归因**。而这三样，恰好是前一版规划里 L2/L3/L4 三层要补的东西——**Q8B 就是补这三层最现成的载体。**

关联：[[3.职业规划]] · [[9.半年扎根计划]] · [[00_CW32固件总览]] · `2.项目/猫咪监控/05_端侧AI部署-RKNN与RKLLM.md`

---

## 9. 待办

- [ ] Q8B 烧录 T2+ 镜像，确认 `fastrpc` 节点与 `ADSP_LIBRARY_PATH`
- [ ] **读 rknn-llm：五个问题过一遍（3 天）**
- [ ] 跑通 resnet50 quick example
- [ ] 装 QAI AppBuilder wheel，用 Python 跑通一次推理
- [ ] 把 YOLOv8n 迁到 QNN，出双平台对比表
- [ ] 跑通 Genie 的 Qwen2.5-0.5B，记录首 token 延迟与 token/s
- [ ] 把 rknn-llm 本地 1.2.3 升到 1.3.1
- [ ] 补一篇 `20_Q8B高通栈踩坑.md` 到猫咪监控目录（或独立目录）

```dataview
LIST
FROM "1.开发技能"
WHERE contains(tags, "技术/端侧AI")
  AND file.name != this.file.name
```
