---
type: 项目
status: 整理中
tags: [工作/MES]
project: MES海康相机
created: "2026-04-28"
updated: "2026-10-12"
up: "[[00_项目MOC]]"
---

# MES 系统与海康相机上位机

> [!info] 一句话结论
> Python 上位机：治具经串口走自定义帧协议上报测试步骤，上位机按工位配置比对阈值、驱动海康相机取流检测、引导 UI，并把结果 HTTP POST 到海能 MES。

## 要点
- 一套程序覆盖 80+ 工位，用 `dev`（工站编号）区分，软件职责是「比对阈值 + 引导 UI + 上报 MES/数据库」。
- 治具协议帧：`0xA5 0x5A | LEN | DEV | CMD | DATA | SUM`，靠帧头 + 校验和抗误帧。
- 流程四阶段命令字：`0x66` 触发扫码 → `0x57/0x58` 通知治具开始/失败 → `0x77` 上报步骤结果 → `0x88` 结束、`0x89` 失败。
- 相机 MV-CS200-10UC（200 万像素 / USB3.0 / 彩色），Bayer-RAW 无完整 RGB，去马赛克后原始采样不可逆。
- MES 仅两个接口：`HN_CheckLotSN` 校验 SN、`HN_SubmitTestData` 上报结果，返回 `ResultCode=OK/NG`。
- 海康 VM「通信管理」无 HTTP 设备类型，最后一跳 HTTP 一律由上位机发。

## 项目背景
- 工位规模：80+ 工位（`dev` 区分），分 6 大类——主机 PCBA、主机组件、主机成品、基站 PCBA、基站组件、基站成品，分类便于 UI 区分。
- 流水线语义：一条线测多个 dev 形成完整流程，**MES 规定测试流程与顺序，顺序不对返回错误**。
- 软件定位：把治具传来的结果与预存配置比较，UI 引导工人操作并实时显示各项结果与数值；结果上传 MES 与数据库服务器。
- 日志：本地保留通讯与配置修改记录，按测试时间/单次分类、易读；密码修改不入本地日志（日志任何人可读）。
- 治具两种流程：
  - 有二维码 → 上位机等扫码，再通知治具开始测试；
  - 无二维码 → 治具发 `0x66` + dev 直接跳过扫码，等 `0x77` 结果帧。
- 工具链：Windows 工控机 + 本地 MySQL；UI 文档规划 QT6（现有代码基于 wxPython）；Python 开发，PyInstaller 打包为单 exe。
- 部署：数据库、MES 各在同局域网内一台机器；第一阶段目标为实现 7 个治具的通讯逻辑。

## 上位机架构

### 线程模型
- 两个线程：work 线程循环调 `test_run_process`；serial 线程收发串口数据入/出队列。停止用 `threading.Event`。

```python
work_thread = None
serial_thread = None
work_stop_event = threading.Event()
serial_stop_event = threading.Event()

def thread_test_running():
    while not work_stop_event.is_set():
        test.test_run_process()

def start_work_thread():
    global work_thread
    if work_thread is None or not work_thread.is_alive():
        work_stop_event.clear()
        work_thread = threading.Thread(target=thread_test_running)
        work_thread.start()

def stop_work_thread():
    work_stop_event.set()
    if work_thread is not None:
        work_thread.join()
```

- `test_serial_rx_run` 把读到的字节入 `test_rx_q`，处理函数从队列取出后逐字节分发。

### 配置管理
- `config.yaml` + `LoadCfg` dataclass，`load_config()` 读入后写入全局 `load_cfg`。

```python
@dataclass
class LoadCfg:
    dev: str = "001"        # 测试类型编码（工站编号）
    com: str = ""           # 串口端口，如 'COM1'
    mes: str = "3"          # 是否使用 MES
    mcu_ver: str = ""       # 集尘桶/PCB 软件版本
    test_tool: str = ""     # 治具名称或编码
    parts_sn_head: str = "" # 配件纸盒条码头，前 7 位
    project_name: str = ""
```

```python
def resource_path(relative_path):
    """兼容开发与 PyInstaller 打包的绝对路径"""
    try:
        base_path = sys._MEIPASS   # PyInstaller 临时目录
    except AttributeError:
        base_path = os.path.abspath(".")
    return os.path.join(base_path, relative_path)
```

- 取值：`config['key']` 缺键即抛 `KeyError`（必填项）；`config.get('key', 默认)` 安全返回默认（可选项）。

### 全局实例与模块导入
- 每个类实例用模块级全局变量接收，跨模块直接调方法；UI 更新统一走 `wx.CallAfter` 回主线程。

```python
class MainFrame(wx.Frame):
    def __init__(self, parent):
        global main_frame
        main_frame = self

wx.CallAfter(MainFrame.main_frame.up_notification_ui, second="海能MES，" + ret[1], color=wx.RED)
```

- 模块顶层导入只执行一次（`sys.modules` 缓存）：会执行 `import`、赋值、`class/def` 定义、`LoadCfg()`；函数体不会执行。

### UI 与扫码枪输入
- 扫码枪模拟键盘极快输入 + 末尾 Enter/Tab，用 `wx.EVT_CHAR_HOOK` 全局捕获。

```python
self.Bind(wx.EVT_CHAR_HOOK, self.on_key_event)

def on_key_event(self, event):
    key_code = event.GetUnicodeKey()
    focus = wx.Window.FindFocus()
    if focus and focus != self:
        self.SetFocus()                 # 焦点回到主窗口
    if key_code == wx.WXK_RETURN or key_code == wx.WXK_TAB:
        self.up_sn_ui(self.barcode_data)
        if test.barcode_q.full() is not True:
            test.barcode_q.put(self.barcode_data)
        self.barcode_data = ""          # 清空准备下一次
    elif key_code != wx.WXK_NONE:
        self.barcode_data += chr(key_code)
    event.Skip()
```

- UI 复位统一入口 `reset_ui()`：清版本、SN、三行提示，复位测试项文字/颜色/图标。

## 相机接入与取流

### 工业相机参数
- 型号 MV-CS200-10UC 拆解：`MV` 机器视觉；`CS` 产品系列；`200` 约 200 万像素；`10` 标准版（`60` 增强版）；`U` = USB3.0（`G` = GigE）；`C` = Color 彩色（`M` = Mono 黑白，灵敏度高、适合打光检缺陷）。
- 电气：12–24V⎓ 0.2A / USB 5V 0.55A；单根 USB 即可供电采集，也可用纯数据线 + 6 针航空插头单独供电。
- 定位逻辑：**先定场景，再定场景下的 ID**；同一物理硬件在不同场景使用完全不同的一套 ID。

### 成像与景深
- 8mm 定焦：拧对焦环不改变焦距，只移动镜片改变像距 v，使其匹配物距 u。
- 高斯成像公式：

$$\frac{1}{u}+\frac{1}{v}=\frac{1}{f}$$

- 景深 DOF：物方空间中弥散圆不超允许阈值的前后物距区间，一般后景深 > 前景深。
- 影响景深四因素：F 值（主手段，F 越小景深越浅）、焦距（越长越浅）、物距（越近越浅）、允许弥散圆（像素越小越苛刻）。

### 色彩空间与 Bayer-RAW
- 数组约定 `[H, W, C]`：一张 480 行 × 640 列彩色图 shape 为 `[480, 640, 3]`。
- Bayer-RAW：一个感光单元不能同时采 R/G/B，靠单像素单色采样 + 两倍 G 匹配人眼；去马赛克（Demosaic）插值补全缺失颜色，**跑完原始拜耳采样信息永久丢失**，可在片上 ISP 或 CPU/GPU 软件做。
- 直取灰度的四种方式：YUV 直接取 Y（最省）、`cv2.cvtColor(BGR2GRAY)`（BT.601）、Bayer 不做 Demosaic、单色 CMOS 原生（真实光子计数）。
- 通道变化速查：
  - RGB：R↑红/↓青、G↑绿/↓品红、B↑蓝/↓黄；调亮度需三通道同改，亮度与色彩耦合。
  - YCbCr（Y 16–235，Cb/Cr 中点 128）：Y↑变亮、Cb↑偏蓝/↓偏黄、Cr↑偏红/↓偏绿；**只动 Y 不偏色**。
  - HSV（OpenCV H∈[0,179]）：H 色轮旋转、S 越高越艳（S=0 即灰度）、V 明度（非人眼真实亮度）。
  - 灰度：加权 `Gray = 0.299R + 0.587G + 0.114B`（优先），不要用简单平均。

## 治具通讯协议

### 协议帧结构

```text
偏移:   0      1      2      3      4      5 … N-2        N-1
内容:  0xA5 | 0x5A |  LEN |  DEV |  CMD |  DATA[0..n-1] |  SUM
```

- 帧头筛假帧 + 校验和校验，误帧概率约 1/(256³)；按 3min/次、每次约 10 帧估算，出现一帧错误帧约需连续测 10 年。
- `LEN`：帧长不定，用于正确解码；`DEV`：设备号，决定各工站执行逻辑；`CMD`：控制测试流程。

### 四阶段状态机
- 阶段一：工人放料按开始，治具周期性发 `0x66`，上位机 UI 提示「请扫码」。
- 阶段二：上位机收到扫码数据后提示「等待治具测试结果」，持续发 `0x57 + SN`；治具处理后停止发 `0x66` 进入测试。

> [!warning] 风险与解决
> 扫码后再收到 `0x66` 会清掉 SN。解决：收到 SN 后切换状态，只接收 `0x77`/`0x88`，并设超时（报「等待治具回复超时」）。

- 阶段三：治具按 `0x77 + 步骤 + 结果` 上报（只报结果不报过程量）；上位机比对阈值后更新项状态与提示，只接收后续步骤与结束指令。
- 阶段四：全部步骤走完，治具连发三帧 `0x88 03`，上位机最终判定并上报；若上位机判失败，向治具连发三帧 `0x89 03`。

### SN 队列与扫码使能
- `barcode_q` 为唯一队列：扫码入队、流程夺取消费、异常/重开串口时清空。
- 治具分支（dev < 100）收到 `0x66` 后 `check_sn_enable = True` 并提示「请扫码」；扫码消费后按校验结果回 `0x57`（OK）或 `0x58`（NG）。

```python
if check_sn_enable and (barcode_q.empty() is not True):
    sn = barcode_q.get()
    str_list = [int(byte) for byte in sn.encode('utf-8')]
    res = mes_run.check_sn_is_ok(sn)
    if int(load_cfg.dev) < 100:
        if res:
            ser_send_data(dev=int(load_cfg.dev), cmd=0x57, data=str_list)
        else:
            ser_send_data(dev=int(load_cfg.dev), cmd=0x58, data=str_list)
    check_sn_enable = False
```

- `[int(byte) for byte in sn.encode('utf-8')]`：把 SN 逐字符转成字节（0–255）供串口发送。
- 非治具流程（dev 100–105）由扫码触发，`check_process_run_state` 要求队列恰有 1 条 SN，否则报「获取条码数量异常」并回 idle。
- `start_sn_collect` 按 dev 决定 SN 槽位数（如 100 需主机/电池/前撞三条），未填满才置 `sn_up_enable = True`。
- RV30（dev=50）特殊：门闸校验失败先发 `0x58` 再发 `0x89 03`，立即上报 MES NG、不等 `0x88`；dev 字节由 `rv30_proto_tx_dev_byte()` 动态给出（方雄规定为 `0x50`）。

## 关键逻辑与代码块

### MES 通讯链路与 URL
- 接口 URL：`http://192.168.7.90/CelinkHNSDJWCFService/OrBitWebAPI.ashx`。
- 结构 = 协议(`http`) + 主机(内网 IP) + 端口(默认 80，可省) + 路径(`.ashx` 物理文件，传统 WebService) + query + fragment（`#` 不发服务器）。
- 上报链路：扫码 → 上位机调度 → 相机/VM 检测 → UI 显示 → 上位机 HTTP POST → MES。
- 内网 IP 仅 10/8、172.16–31/12、192.168/16 三段为私有；MES `192.168.7.90` 属私有段，外网直连不可达，需 VPN 或端口映射。

### SN 校验
```python
def check_sn_is_ok(sn=""):
    dev = test.load_cfg.dev
    test_dev = test.load_cfg.test_tool
    user_parameter = station_sn_code.get(dev, "")
    if user_parameter == "":        # 该设备不需要过海能工站
        return True
    dat_dirt = {'API': 'HN_CheckLotSN', 'UserParameter': user_parameter}
    user_data = {"LotSN": sn, "TestDevice": test_dev}
    dat_dirt["UserData"] = json.dumps(user_data, ensure_ascii=False, indent=4)
    ret = celink_mes_communication(dat_dirt)
    ...
```

- `station_sn_code` 把「本地设备编号」映射到「海能 MES 用户码」，为空表示不过站。

> [!tip] 实测行为
> 若 `UserParameter` 是 MES 未定义的值，接口不报错且返回值仍为 `True`。

### 结果上报
```python
data = {
    "API": "HN_SubmitTestData",
    "UserParameter": station_report_code.get(dev),
    "UserData": json.dumps({
        "LotSN": ck_sn,
        "TestDevice": test.load_cfg.test_tool,
        "UserCode": "001",              # 工号，非必填
        "TestResult": result,           # OK / NG
        "StartTime": start_time.strftime("%Y-%m-%d %H:%M:%S"),
        "EndTime": end_time.strftime("%Y-%m-%d %H:%M:%S"),
        "TestDataItem": mes_run.celink_report,
    }, ensure_ascii=False, indent=4),
}
```

- `add_report()` 把每项结果翻译成 MES 约定项（`ItemName/ItemValue/ItemResult/Description`），累积进 `celink_report` 随上报发送。
- 返回值仅 `ResultCode`（OK/NG）+ `ResultMessage`（UI 提示）；请求用 `requests.post(url, data=mes_data, timeout=5)`，区分超时与通讯错误。

### 海康 VM 与上位机通讯
- 前提：VM「通信管理」只有 TCP/UDP/串口/ModBus/PLC，**无内置 HTTP 设备类型**；全局脚本（C#）可引用 `System.Net.Http` 自行 POST，但属二次开发，非产品化协议。

| 方式 | 触发 | 谁发 HTTP 给 MES | 适用 |
| --- | --- | --- | --- |
| ① SDK 集成 | 上位机调 VMSDK `Run()`/软触发 | 上位机 | 数据最全、端到端可控（首选） |
| ② 通信转发 | 上位机发 `TRIG,<job>,<SN>` | 上位机 | VM 独立跑、改方案不动上位机 |
| ③ 全局脚本直发 HTTP | VM 流程照常跑 | VM 脚本 | 临时联调，MES 逻辑耦合，不推荐 |

- ② 推荐拓扑：上位机开 TCP Server，VM 作客户端自动重连；VM→上位机回 `RST,001,SN123,OK,250\n`。
- HTTP 只是 TCP 之上的一种应用层封装；无 HTTP 也能发数据，TCP/UDP 靠 `IP:端口` 定向，HTTP 用 URL 表达「机器 + 路径」。

## 调试与问题排查

### COM 口
- COM 口是 Windows 给串口实例的编号，不等于硬件；USB 转串口与蓝牙 SPP 都会被包装成 COM。
- USB 转串口芯片 PID 优先表（自动选口用）：CH340=29987、CP2102/2104=60000、PL2303=24577、FT232=24576。
- 蓝牙配对后生成一对 COM：传出（客户端，上位机一般选这个）+ 传入（服务器）。
- 称重工位收发：`WEIGH_CMD_READ = b"R\r\n"`（硬件必须用 bytes，`b` 与不加 `b` 类型不同）；解析 `re.compile(r"[+-]\s*([\d.]+)\s*kg")`。

### 网络与抓包
- HTTP 明文连 MES 四类风险：窃听、冒充、篡改、重放；「内网所以安全」不成立。
- 能被抓到包的原因：无线/共享介质天然广播、交换机端口镜像/ARP 欺骗/MAC 泛洪、路由器三层必须拆包、网卡混杂模式。防御的现实做法是 HTTPS/TLS 让截获内容变密文。
- 抓视频真实地址：地址在「播放状态」DOM 的 `currentSrc`（前端 JS 动态请求的一次性签名直链）；下载需带 `Referer`、`User-Agent`，`token/sign` 一次一签、过期即失效。

```powershell
curl.exe -L "https://...直链..." -H "Referer: https://channels.weixin.qq.com/" -o "D:\Media\out.mp4"
ffprobe.exe -v error -show_entries stream=codec_name,width,height -of default=noprint_wrappers=1 out.mp4
```

### 常见报错与提示
- `等待治具回复超时`——扫码后长时间收不到治具指令（需切状态 + 超时）。
- `获取条码数量异常`——非治具流程队列 SN 数不等于 1。
- `串口断开连接` / `请启动治具开始测试`——串口插拔与连接状态变化。
- MES 返回 NG：用 `format_message_with_line_breaks` 每两个逗号换行后显示在 UI。
- 异常处理用 `try / except 类型 as 变量` 区分超时与通讯错误。

### 开发经验（AI 辅助 SOP）
- 优势：需求文档写清楚后由 AI 执行，开发者不必精通 UI 等非核心细节。
- 对开发者的要求：技术审美（能判断方案可行）+ 业务理解（脑中要有程序完整运行过程）。

> [!note] 需求反例
> 原设计「先选工位再运行」不可行——产线工人多数不识字。改为双击直接运行，左上角留配置按钮，保存后复用上次配置。

- 其他约定：密码不明文保存，配置修改需密码校验。
