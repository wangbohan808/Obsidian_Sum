---
type: 项目
status: 整理中
tags:
  - 工作/海尔
project: 海尔IOT
created: "2026-04-23"
updated: "2026-10-12"
---

# 海尔 IOT 开发实录（network_haier_normal）

> [!info] 一句话结论
> **network_haier_normal 是连接乐动 APOS 机器人与海尔 UHSD（OSDK）云平台的双向协议桥接进程**，本文按主题归并 2026-04～10 期间的架构、源码、调试、编译与踩坑记录。

## 要点

- **职责**：向下连扫地机 APOS（`CallServer` / `Subscriber`），向上连海尔云（`uhsd_*`），做属性/JSON/hex 协议翻译。
- **入口**：`main.cpp` → `HaierSdkNetwork::Init()` → `apos_iot_init` + `HAIER_PARAM_INS.Init()` + `HAIER_ADAPTER.Init()`。
- **通讯**：RPC push 端口 8801、sub 端口 8802；云端回调 → `Cmd_xxx` → `CallServer`。
- **调日志**：MobaXterm ssh `root/autopack1688`，`killall autopack_manager` → `killall network*` → `./network`。
- **编译**：WSL Ubuntu 18.04 + 交叉工具链 `ssd222d_sdk.tar.gz`，`make.sh` 或 build 目录 `make install`。
- **踩坑**：条件变量虚假唤醒/唤醒丢失、`unique_ptr` 只能 move、禁止拷贝类重复释放、`explicit` 暴露编译期错误。

## 项目背景与架构

### 项目定位

**本质是一个协议桥接/联网模块：一头连扫地机本体系统（乐动 APOS），一头连海尔 IoT 云平台（OSDK/UHSD）。**

- OSDK = 海尔 UHSD（UHome SmartDevice）SDK，即 `third_party/haier_sdk`，头文件以 `uhsd_` 开头。
- APOS = 乐动机器人的机器人操作系统/通信框架。
- RPC：像调本地函数一样调用另一进程的函数。Push 端口 8801（读/写属性、调 Service）、Sub 端口 8802（订阅事件）。

```text
GitLab：https://gitlab.ldrobot.com
登录邮箱：bohan.wang@ce-link.com  用户名：celink-dev-wangbohan
仓库：gitlab.ldrobot.com/ldrobot/autopack/network_haier_normal.git
```

### APOS 分层与本机进程

```text
Linux 内核 4.9.84（管硬件）
  └─ BusyBox + 进程管理
       └─ ★ APOS（AutopackOS）：节点编排 autopack_manager / 事件总线 /event/* /
          SDK（libapos_sdk_v2、libapos_iot_sdk、libldprotocol）/
          平台服务 apos_server 8801、8802 / OTA（otaunpack、updater）
```

本机 RAM 仅约 120MB，进程按故障隔离、实时分级、内存隔离拆分：

- 核心业务调度 `app_sweeper2`（清扫主逻辑，内存最大）；导航规划 `navigator_pl`（算力密集）。
- 定位建图 `slam_pose_provider_pl`、`apos_server`（高频位姿，地图有内存累积风险）。
- 传感器采集 `sensor_node_carrier`、`sensor_node_point_laser`（按外设拆，IO 易阻塞）。
- 网络通信 `network`（网络 IO 阻塞、云端长连接）；辅助层 `debug_proxy`、`autopack_manager`、`play_daemon`；系统层 `wpa_supplicant`、`dropbear`、`sh`、`udhcpc`、`init`。

> [!tip] 并行上限由 CPU 逻辑线程数决定，不是物理核心也不是进程数；进程只做资源隔离。相同线程总量下进程越少越好。

### 通讯两条链路

- 主动做事 `CallServer`（读数据、控任务、设属性）：

```cpp
CallServer(SRV_TASK_CONTROL_STOP_CLEAN, nullptr, nullptr);   // 纯命令
CallServer(SRV_MAP_GET_MAP_ID, nullptr, &srv.response);      // 只读
CallServer(SRV_MAP_MERGE_AUTO_AREA, &req, &res);             // 读写
```

- 被动响应 `Subscriber`（按键、故障、地图变更、清扫记录、OTA 进度），模块 `Init()` 时注册一次，之后等事件回调。

### 单例模式

WiFi、地图、OTA、设备状态在物理上各只有一份，用单例匹配“全局唯一资源”。

```cpp
template<typename T>
class Singleton {
public:
    static T& Instance() {
        if (NULL == value_) { value_ = new T(); }
        return *value_;                       // 返回引用是无拷贝别名，推荐
    }
private:
    static T* value_;
};
template<typename T> T* Singleton<T>::value_ = NULL;

typedef Singleton<HaierSdkNetwork> sHaierSdkNetwork;
#define HAIER_SDK_NETWORK sHaierSdkNetwork::Instance()
#define IOT_SDK HAIER_SDK_NETWORK.iot_sdk_
```

- 返回实例 `T` 会拷贝、破坏单例。代码占 ROM、指针占 RAM、`new T()` 首次进堆且当前实现不释放（常驻）。

### 乐动 IOT 代码的作用

1. 用封装好的 OSDK 做连云：配网、注册、上线、收写属性、报状态、OTA、传地图资源，对外都走 `uhsd_*`。
2. 当 OSDK 与扫地机 APOS 之间的翻译：下来 OSDK 回调 → `Cmd_xxx` → `CallServer`；上去 APOS 事件/查询 → 转海尔格式 → `uhsd_dev_status_report`。

## network_haier_normal 源码结构

### 文件职责

- `main.cpp`：入口，启动后 `while(1)` 常驻。
- `haier_sdk_network.cpp/.h`：`HaierSdkNetwork` 单例，`Init()` 总入口、`init_param()` 构造鉴权参数。
- `haier_adapter.cpp/.h`：`HaierAdapter`，封装海尔 SDK（`InitOsdk` / `CallbackInit` / `InitAndSetDevHandle` / `SendStausReport` / 命令表 / 各业务 Service）。
- `haier_param`：`HAIER_PARAM_INS`，用乐动 API 取产品 License、SN、MAC、区域。
- `CMakeLists.txt`：交叉编译，产出 `network_haier_normal_wifi`。

### 启动流程

```text
main() → LOGD("TUYA SDK Demo start run ...") → HAIER_SDK_NETWORK.Init()
  ├─ apos_iot_init(param)      // 连 LDRPC 127.0.0.1:8801/8802，订阅 ~142 个机器人事件
  ├─ apos_iot_service_init()   // APOS IoT 服务层初始化
  ├─ HAIER_PARAM_INS.Init()    // 参数初始化
  └─ HAIER_ADAPTER.Init()      // 启动海尔协议适配器
→ while (1) sleep 10s
```

```cpp
int main(int argc, char *argv[]) {
    LOGD("TUYA SDK Demo start run ... ");
    if (HAIER_IOT::HAIER_SDK_NETWORK.Init() != 0) { LOGE("TuyaSdkNetwork Init failed."); }
    while (1) { std::this_thread::sleep_for(std::chrono::milliseconds(1000 * 10)); }
    return 0;
}
```

```cpp
apos_iot_sdk::Apos_Iot_Init_Param_Func init_param() {
    apos_iot_sdk::Apos_Iot_Init_Param_Func env;
    memcpy(env.authentication_key, APOS_SDK_LICENSE, sizeof(APOS_SDK_LICENSE));
    std::string ip = "127.0.0.1";
    memcpy(env.server_ip, ip.c_str(), ip.size());
    return env;
}

int HaierSdkNetwork::Init() {
    apos_iot_sdk::Apos_Iot_Init_Param_Func param_func = init_param();
    iot_sdk_ = apos_iot_sdk::AposSdkIot::Instance();
    try {
        iot_sdk_->apos_iot_init(param_func);
        iot_sdk_->apos_iot_service_init();
    } catch (const std::system_error& e) {
        std::cerr << "Initial sdk: caught system_error " << e.code() << ": " << e.what() << '\n';
    }
    HAIER_PARAM_INS.Init();
    HAIER_ADPATER.Init();
    return 0;
}
```

> [!note] `apos_iot_sdk.cpp` 只提供链接库、不给源码，但能用 `LOGD` 打印同格式日志，是因为工程引用 `apos_iot_sdk.h` 并调用其中声明的 `CallServer`、`Subscriber` 等。与机器通讯的类是 `AposSdkIot`（`Instance` / `apos_iot_init` / `CallServer` / `Subscriber` / `GetAposHandler`）。

### HaierAdapter 初始化

`InitOsdk`：判断区域（国内/海外），海外必须在 `uhsd_init` 前调 `uhsd_set_domain_region_ext`，再传入机器配置、网卡信息初始化并打印版本号、设置 OSDK 日志等级。

```cpp
bool HaierAdapter::InitOsdk() {
    if (HAIER_PARAM_INS.GetRegion() == 1)      region = UHSD_REGION_SEA;
    else if (HAIER_PARAM_INS.GetRegion() == 0) region = UHSD_REGION_DOMESTIC;
    else {   // License 未明确 region，用 upCodeT 兜底（如 F55241E00 等海外机型）
        region = (HAIER_PARAM_INS.GetProductUpcodet() == "F55241E00") ? UHSD_REGION_SEA : UHSD_REGION_DOMESTIC;
    }
    if (region == UHSD_REGION_SEA) {
        std::string country = HAIER_PARAM_INS.GetCountryCode();
        if (country.empty()) { country = "IN"; }   // 默认印度
        uhsd_set_domain_region_ext(const_cast<uhsd_char *>(country.c_str()), 0, 0);
    }
    uhsd_sysinfo_cfg_t sysinfo_cfg;
    memset(&sysinfo_cfg, 0, sizeof(sysinfo_cfg));
    strncpy(sysinfo_cfg.file_path, "/data/cfg/", sizeof(sysinfo_cfg.file_path) - 1);
    strncpy(sysinfo_cfg.eth, "wlan0", sizeof(sysinfo_cfg.eth) - 1);
    uhsd_linux_syscfg_init(sysinfo_cfg);

    if (uhsd_init()) { LOGE("OSDK init failed"); return false; }
    if (uhsd_version() != NULL) { LOGI("OSDK Version: %s", uhsd_version()); }
    uhsd_log_set(NULL, UHSD_LOG_OFF);
    return true;
}
```

`CallbackInit`：注册各类云端回调，注册后对应事件（状态）发生就回调用户函数。

```cpp
uhsd_dev_status_register(HaierStatusCallback, this);                        // 状态：配网/连云/绑定/连路由
uhsd_dev_set_device_read_cb(HaierDeviceReadCallback);                       // 读属性
uhsd_dev_set_write_cb(HaierWriteCallback);                                  // 写属性
uhsd_dev_set_op_cb(HaierDeviceOperationCallback);                           // 操作
uhsd_dev_set_start_depairing_notify_cb(HaierStartDepairingCallback, NULL);  // 解除配对
uhsd_dev_register_rsc_related_cb(ResourceNotifyCallback, nullptr,
                                 ChannelStatusCallback, nullptr);            // 资源相关
```

`InitAndSetDevHandle`：创建句柄后设置设备信息，再挂载 wifinetcard module。

```cpp
gw_devHandle = uhsd_dev_new_devHandle(UHSD_DEV_ROLE_GENERAL);
uhsd_dev_set_devHandle_typeId(gw_devHandle, HAIER_PARAM_INS.GetProductTypeid().c_str());
uhsd_dev_set_devHandle_devKey(gw_devHandle, HAIER_PARAM_INS.GetProductDevkey().c_str());
uhsd_dev_set_devHandle_upCodeT(gw_devHandle, HAIER_PARAM_INS.GetProductUpcodet().c_str());
uhsd_dev_set_devHandle_dMac(gw_devHandle, HAIER_PARAM_INS.GetMacAddr().c_str()); // mac 必须设置
gw_moduleHandle = uhsd_dev_new_wifinetcard_module(devMac, "ssc9211d", "1.0.0");
uhsd_dev_add_module_to_device(gw_devHandle, gw_moduleHandle);
```

状态上报：`SendStausReport(pairs, len)` 内部调 `uhsd_dev_status_report(gw_devHandle, pairs, len)`，非 0 打 `LOGE`。

### CMake 解读

```bash
cmake_minimum_required(VERSION 3.10)
project(network_haier_normal)
SET(APPLICATION "network_haier_normal")
if(NOT CMAKE_BUILD_TYPE)   # 默认 Release 压体积；调试用 -DCMAKE_BUILD_TYPE=Debug
	set(CMAKE_BUILD_TYPE "Release")
endif()
set(LIB_DIR_NAME "ssd222d")
set(APOS_IOT_SDK_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/../apos_iot_sdk" CACHE PATH "...")
set(APOS_SDK_INSTALL_DIR "${CMAKE_CURRENT_SOURCE_DIR}/../sdk" CACHE PATH "...")
# 把 SDK 编译产物同步进工程第三方库
set(APOS_SDK_LIB_SOURCE "${APOS_SDK_INSTALL_DIR}/lib/libapos_iot_sdk.so")
set(APOS_SDK_LIB_DEST   "${CMAKE_CURRENT_SOURCE_DIR}/third_party/apos_iot_sdk/lib/${LIB_DIR_NAME}/libapos_iot_sdk.so")
```

- `message()` 编译时输出；`CACHE PATH` 变量跨构建持久化。

## 关键功能逻辑

### 初始化命令表（map + std::function + bind）

```cpp
std::map<std::string, std::function<int(const std::string&)>> cmd_obj_api_table_;
cmd_obj_api_table_[HAIER_Y_CLEAN_MODE] = std::bind(&TaskControl::Cmd_YMop, &TASK_CTL_INS, std::placeholders::_1);
```

- `std::function<int(const std::string&)>` 统一“入参 std::string、返回 int”的接口，`std::bind` 生成可调用对象（绑定对象指针 + 占位符），调用时才执行。意义：统一接口、按键值动态路由、避免大量 if-else。

```cpp
int HaierAdapter::HaierWriteHandle(const std::string &cmd, const std::string &cmd_value) {
    std::lock_guard<std::mutex> locker(write_mutex_);
    auto iter = cmd_obj_api_table_.find(cmd);
    if (iter != cmd_obj_api_table_.end()) {
        return iter->second(cmd_value);   // first=键, second=值
    }
    LOGD("HaierWriteHandle cmd:%s not found", cmd.c_str());
    return -1;
}
```

无参命令用 `std::map<int, std::function<int()>>`（`cmd_map_table_`：`DUST_COLLECTION → Cmd_StartDustCollection`、`START_SWEEP → Cmd_StartTotalClean` 等，即采集-组包-上报）。

### 写回调（云端下发 → 本地执行 → 应答）

```cpp
uhsd_s32 HaierAdapter::HaierWriteCallback(uhsd_devHandle devHandle, uhsd_s32 req_sn,
        const uhsd_char *property_name, const uhsd_char *property_value, const uhsd_char *traceId) {
    int result = HAIER_ADPATER.HaierWriteHandle(std::string(property_name), std::string(property_value));
    return uhsd_dev_write_resp(devHandle, req_sn, result, 0, traceId);
}
```

### 回调-订阅机制（EventWorker）

```cpp
template<typename MsgType>
class EventWorker : public EventWorkerBase {
 private:
    typedef std::function<int32_t(const MsgType&)> SubFunc;
    SubFunc subscriber_func_;
 public:
    EventWorker(SubFunc func) : subscriber_func_(std::move(func)) {};
    int32_t Call(const MsgType& in_msg) { return subscriber_func_(in_msg); };
};

EventWorkerInnerPtr worker_ptr(new EventWorker<const StdStringMsg &>(
        std::bind(&DevStatus::StatusForceReport, this, std::placeholders::_1)));
IOT_SDK->Subscriber(TOPIC_SWEEP_DEV_STATUS, std::move(worker_ptr));
```

原理：注册时把“回调对象（函数地址 + this）”按话题名存表；事件来时按话题名查表调用。<const MsgType&> 偏特化优先级高于主模板，据入参不同生成不同类。

### 生产者-消费者线程队列

队列用锁 + 条件变量 + 原子标志保证线程安全，消费者阻塞等待、生产者异步入队；成员为 `std::mutex queue_mutex_`、`cond_mutex_`、`std::condition_variable condition_var_`、`std::queue<T> event_queue_`、`std::atomic<bool> is_set_flag_`。

```cpp
void DevStatus::DevStatusLoop() {
    while (true) {
        bool b_flag = false;
        request_signal_.PopSignal(b_flag, 300);   // 300ms 超时
        UpdateDevAllStatus(b_flag);
    }
}
```

- 会动 `event_queue_` 的 `PushSignal` / `PopSignal` / `Size` 共用同一把 `queue_mutex_`，避免并发读写竞争。

### promise / future（等异步结果）

上传地图等接口是“登记回调 → 阻塞等待”，用 promise/future 解决“线程 A 依赖线程 B 结果”：回调里 `static_cast<CallbackWrapper*>(cb_param)` 还原结构指针，`set_value(result)` 唤醒等待方（`wait_for` 退出）。

```cpp
auto wrapper_ptr = static_cast<CallbackWrapper*>(cb_param);
if (wrapper_ptr && *wrapper_ptr) {
    PushFileResult result;
    result.errNo = errNo;
    if (file_name_c) result.file_name = file_name_c;
    (*wrapper_ptr)->set_value(result);
    delete wrapper_ptr;
}
```

### Service 注册与调用

```cpp
// 编译器从函数签名推断 REQ/REP，创建 ServiceWorker 放进 map
RegisterServer("/Map/GetSegmentEmpty", &UiMapProtocol::GetSegemntEmptyServer, this);
```

- 嵌套类 `StdIntSrv::Request/Response` 继承 `apos_serialize::IMessage`；数据经共享内存回填 `response.area_info`。

### 预约协议

```text
TZ | Number | { Effectiveness | Week | Time(H,M) | N | [id×N] | clean_mode | fan_level | water_level | sweep_count } × Number
```

- 所有预约项一同下发刷新；清扫模式/风机/水量不按房间区分，**预约只能整批选中房间，不能逐房间单独设置**。

## 调试与问题排查

### 日志格式与关键词

```text
# 格式1：本工程业务日志（可定位到源码文件:行号）
[日期 时间.毫秒] [PID-TID/标签] [级别/文件名:行号] 消息
2026/7/3 18:33:29.276 1873-1873/apos_iot_sdk D/main.cpp:39 TUYA SDK Demo start run ...
# 格式2：海尔 OSDK 内部日志（预编译库，一般改不了）
[OSDK-版本][级别][时间 t:线程名_线程ID s:序号] 模块 (函数名) 消息
[OSDK-5.6.0][E][2026-07-03 18:33:31:376 t:misc_1918 s:508] iwimp (wpactrl_connect) wpactrl open failed, 2.
# 格式3：LDRPC / APOS 底层 RPC 日志
[2026/7/3 18:33:29.275 F][apos_client.cpp:257] <<LDRPC>> AposSdkWrite AposClient not valid
```

```cpp
uhsd_s32 level = UHSD_LOG_OFF;   // 0关 1Verbose 2Debug 3Info 4Warn 5Error 6Fatal
uhsd_log_set(NULL, level);
```

### 抓设备端日志

```bash
1. USB转网口连接待测机，为其分配固定IP
2. 乐动上位机软件找到该IP，发送开始调试指令
3. MobaXterm ssh 上位机：用户名 root / 密码 autopack1688
4. cd /tmp/approm ; killall autopack_manager   # 杀自动挂起进程，否则重启冲突
5. killall network* ; ./network                # 杀旧IOT进程并重启（后台常驻可加 &）
```

- 过程可用 `procrank` 查看进程。
- 配网按键差异：印度机型点击后**不杀** `network` 可直接取日志；泰国水洗机点击后**自动杀掉** `network`，需重新 `./network`。
- 用指令重启配网：

```bash
cd /tmp/approm/
./event_tool -p -t "/event/keyboard" -e "network_config" -d "{}"
```

### 安卓 App 抓日志

```bash
# 开发者选项 -> USB调试；手机全程亮屏并允许调试

# 管理员 CMD 一键装 ADB（chocolatey）
@"%SystemRoot%\System32\WindowsPowerShell\v1.0\powershell.exe" -NoProfile -InputFormat None -ExecutionPolicy Bypass -Command "iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))" && SET "PATH=%PATH%;%ALLUSERSPROFILE%\chocolatey\bin"
choco install adb -y && adb version
adb shell logcat > D:\app_log\HW2T_android.log   # ctrl+C 结束
```

### 查找设备 IP 与网络排查

```bash
# Windows：先 ping 网段广播缓存 ARP，再读表
for ($i=1;$i -le 254;$i++){ping -w 1 -n 1 10.38.27.$i | Out-Null} ; arp -a

# 按目标网段选网卡：0.0.0.0 是默认路由，Gateway=0.0.0.0 表示直连，Metric 越小优先级越高
route -n
# 找哪些文件在配置网卡：-r 递归 -i 忽略大小写 -E 扩展正则；2>/dev/null 丢弃 stderr
grep -riE 'eth0|udhcpc|ifconfig' /config/etc /etc /oem/bin 2>/dev/null | head -30
```

- 网卡三件套：`eth0`（有线）、`wlan0`（WiFi）、`lo`（内核虚拟回环，非物理）；`flags` 中 `UP` 已启用、`RUNNING` 链路连通。
- DHCP 分配识别依据是**网卡 MAC**，不是网口位置。DORA 四步：Discover / Offer / Request / Ack。

> [!warning] 固定 IP 两种方案别混淆：**DHCP MAC 预留**（设备仍走 DHCP，服务器承诺固定 IP，换路由器失效）vs **客户端静态 IP**（设备硬写 IP、不发 DHCP、换网段就不通）。

- 扫地机网卡脚本在只读区 `/config/etc/init.d/S40_network`（只配 wlan0）；开机链 `inittab → rcS → S40_network`，`eth0` 需自己加配置。

## 编译与烧录

### IOT 编译环境部署

```bash
wsl --install -d Ubuntu-18.04          # 安装 WSL2 Ubuntu 18.04
sudo apt-get install build-essential   # 装编译工具
lsb_release -a                         # Ubuntu 18.04.2 LTS / bionic
grep "/bin/bash" /etc/passwd           # 查看可登录账户
sudo -i ; su - wangbohan808            # 切换用户
ln -s wangbohan808 peter               # 统一软链接 peter，使源码路径重定向到自己用户

cp /mnt/f/Resource/Linux/Haier_IOT/ssd222d_sdk.tar.gz ~ && tar -zxvf ssd222d_sdk.tar.gz
git clone https://gitlab.ldrobot.com/ldrobot/autopack/network_haier_normal.git
git fetch origin && git branch -r
git switch -c dev origin/dev           # 新版用 switch，旧版用 checkout -b；拉其它分支同理
sudo ln -s /usr/lib/x86_64-linux-gnu/libmpfr.so.6 /usr/lib/x86_64-linux-gnu/libmpfr.so.4  # 库名软连接
```

### 处理海尔 OSDK

```bash
# 1. 编译 Linux 适配层（不再编进 OSDK 库，随 tag 包单独提供）

#    改 linux 目录里的编译器路径，在源码目录直接 make，同时产出 .a 与 .so
sudo chown -R wangbohan808:wangbohan808 <...>/src/linux/output   # 或删掉 output 后重 make
sudo make
# 2. 把适配层产物与 OSDK 原有内容统一打包进 IOT 代码仓库（头文件/编译产物放对应位置）

# 3. 编译技巧：李新写的 make.sh 可直接执行；build 目录 make install 打包要替换的系统文件
```

- 本项目用静态库（`.a`）编译 IOT 源码（`.a` / `.so` 内容一致，二选一）；替换文件到系统后需要**加权**。

## 踩坑记录

### 条件变量：虚假唤醒与唤醒丢失

```cpp
// 陷阱：进入等待前通知发生 → 通知丢失（条件变量仅等待时才收到通知）
// 方案：每 100ms 或收到信号检查一次，未收到就继续 wait_for，直到 timeout
int32_t once_timeout = 0, once_timeout_max = 100;
while (timeout > 0) {
    if (timeout > once_timeout_max) { timeout -= once_timeout_max; once_timeout = once_timeout_max; }
    else { once_timeout = timeout; timeout = 0; }
    signal_status = condition_var_.wait_for(lck, std::chrono::milliseconds(once_timeout),
                                            [&] { return is_set_flag_ == true; });
    if (signal_status && (is_set_flag_.load() || event_queue_.size() > 0)) break;
}
```

### unique_ptr 只能移动，不能拷贝

```cpp
using EventWorkerInnerPtr = std::unique_ptr<EventWorkerBase>;
layer2_event_map_[event] = worker_ptr;             // ❌ 编译错误（禁止拷贝）
layer2_event_map_[event] = std::move(worker_ptr);  // ✅ 只能转移所有权
```

### 禁止拷贝类（重复释放）

```cpp
HaierSdkNetwork(const HaierSdkNetwork&)            = delete;
HaierSdkNetwork& operator=(const HaierSdkNetwork&) = delete;
```

> [!danger] 含指针成员的类默认生成浅拷贝，两指针指向同一地址，析构会**重复释放导致崩溃**；也防“单例被多对象管理”引发时序问题。

### explicit 让错误暴露在编译期

- 单参构造不加 `explicit` 会做隐式转换：`drink(100)`（本意漏写 `drink(int)`）或 `c == 10` 会被悄悄编译通过、逻辑跑偏；加 `explicit` 后按预期编译报错。

### 其它内存/语法陷阱

- **析构函数绝不自动 `delete` 指针成员**——内存泄漏根源；`new` 的对象必须手动 `delete`，栈对象出作用域才自动析构。
- 默认构造只默认构造类类型成员，**不对基本类型（int/指针）初始化**，会是垃圾值。
- 静态成员变量属 `.data/.bss`，程序加载阶段一次性分配、全局唯一，不随对象创建销毁；所有成员函数（含 static）代码都在 `.text` 只读段，只存一份。
