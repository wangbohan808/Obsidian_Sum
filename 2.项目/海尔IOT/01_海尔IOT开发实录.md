---
type: 项目
status: 整理中
tags:
  - 工作/海尔
project: 海尔IOT
created: "2026-04-23"
updated: "2026-10-12"
up: "[[00_项目MOC]]"
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


**两者最常用的用法如下：**

```cpp
std::function<int32_t(const BinPack&, BinPack&)> f =
    std::bind(&TaskControl::Cmd_Mode, &task, std::placeholders::_1, std::placeholders::_2);

int32_t ret = f(req, rep);   // 等价于 task.Cmd_Mode(req, rep)
```
- **std::bind绑定的结果使用std::function接收，然后接收后的实例直接加上括号就可以传入参数直接进行调用**


两种语法结构如下：

```cpp
std::bind(&类名::成员函数, 对象指针, 参数1, 参数2, ...)
```
- 绑定后的结果有点类似于函数指针

```cpp
std::function<返回类型(参数类型1, 参数类型2)> 变量 = 可调用对象;
返回类型 结果 = 变量(实参1, 实参2);
```
- 函数类型其实也是作为函数指针的使用方法

- 因此两者可以本质上是一致的
- 使用`std::function`接收`std::bind`绑定的结果



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


### 注册服务函数实现原理

```cpp
  template <typename CLASSPTR, typename REQ, typename REP>
  static void RegisterServer(const std::string& server_key, int32_t (CLASSPTR::*register_func)(REQ, REP), CLASSPTR *obj) {
    ServiceManagerPtr service_worker(new ServiceWorker<REQ, REP>(
            std::bind(register_func, obj, std::placeholders::_1, std::placeholders::_2)));
    service_map_[server_key] = std::move(service_worker);
  };
```
- `std::bind(register_func, obj, std::placeholders::_1, std::placeholders::_2)` 将传入的函数，传入一个对象参数，就会变成需要传入`REQ, REP`两个参数的特殊的可调用对象（函数指针）
- `new ServiceWorker<REQ, REP>`使用特殊的类的构造函数--这种构造函数是传入一个函数指针作为成员变量，后续使用`call`函数就可以进行调用
- `using ServiceManagerPtr = std::unique_ptr<ServiceWorkerBase>;`使用智能指针管理这一个类，构造函数直接构造这个智能指针类型的实例，然后传入这个类型指针进行管理
- `service_map_[server_key] = std::move(service_worker);`将这个后续可以被使用的对象指针，传入map中，方便后续进行调用



### 预约协议

```text
TZ | Number | { Effectiveness | Week | Time(H,M) | N | [id×N] | clean_mode | fan_level | water_level | sweep_count } × Number
```

- 所有预约项一同下发刷新；清扫模式/风机/水量不按房间区分，**预约只能整批选中房间，不能逐房间单独设置**。


### 句柄的理解

```cpp
handler_ = AposSdkInit(authentication_key, server_ip);
```
- 将authentication_key、IP地址写进apossdk，会返回一个int32的句柄；在apossdk中，会进行维护一个类似于键值对，使得传入句柄就知道注册表与IP地址信息，从而与指定进程建立通信
- `authentication_key`用来进行鉴权，扫地机本机进行鉴权之后，决定此次连接是否允许调用扫地机本机的相关参数

```cpp
#define APOS_SDK_HANDLER  apos_iot_sdk::AposSdkIot::Instance()->GetAposHandler()
```
- apossdkiot初始化获得句柄之后，使用宏进行储存，方便后续进行调用

使用场景：
凡是进程之间的通信，第一个参数都是句柄用来定位。如`AposSdkSetEventSubscribe`、`AposSdkSendEvent`、`AposSdkRead` / `AposSdkWrite`



| API | 作用 | 是否等待对方结果 | 主要参数 | 本项目中的例子 |
|---|---|---|---|---|
| `AposSdkInit` | 建立到本机 `apos_server` 的连接，把套接字、鉴权结果、IP 写入连接表 | 调用期间完成连接 | 许可证、IP | `AposSdkInit(key, "127.0.0.1")`，返回值存入 `handler_` |
| `AposSdkSetEventSubscribe` | 在这条连接上登记要接收的事件和回调 | 登记完成后返回；事件以后异步回调 | 句柄、回调、事件名数组、数组长度 | 订阅 `sub_topic`，回调 `apos_sdk_event_handle` |
| `AposSdkSendEvent` | 发出事件，不等执行结果 | 否，发出去就返回 | 句柄、事件名、`BinPack` | `"/TaskManager/PauseClean"` |
| `AposSdkService` | 发出事件并等待返回数据 | 是，直到收到 `out_msg` 或超时 | 句柄、事件名、输入包、输出包、超时 | `"/WifiModule/GetMacAddr"`，超时 5000 毫秒 |
| `AposSdkRead` | 按路径读取属性或文件 | 是，直到读到数据或超时 | 句柄、输入包、输出包、超时 | 读 `"/WifiModule/WifiMode"`、`"/ParamManager/Data"` |
| `AposSdkWrite` | 按路径写入属性、参数、文件或日志 | 是，直到写完或超时 | 句柄、输入包、超时 | 写 `"/ParamManager/Data"`、`"/System/FileOp"`、`"/System/CustomLog"` |
| `AposSdkDeInit` | 关闭句柄对应的套接字，删除这条连接记录 | 关闭完成后返回 | 句柄 | 仅在 `apos_iot_stop` 中调用 |

返回值约定：`0` 表示成功，小于 `0` 表示失败。除 `AposSdkInit` 返回的是句柄以外，其余接口的第一个参数都是这个句柄。


### 函数指针

**函数指针与成员函数指针还是不同的，使用方式上也有一些区别与限制。**



### 数据传输编解码

**无论是Json还是BinPack，各种协议、文件格式都是为了方便记录以及传输：**
定义一种数据格式-->编码为字节流进行传输-->接收方进行解码进行还原


```txt
JSON 字符流
{"Volume":30,"IsPlay":true}
分隔符是 { } " : ,
数字 30 是字符 '3' 和 '0'，不是 4 字节整数
true 是四个字母

BinPack 记录流
[ Volume | type=6  | 整数 30 的 4 字节 | 长度 ]
[ IsPlay | type=12 | true 的 1 字节    | 长度 ]
下一条靠本条长度跳转
30 的宽度由标签 6 规定为 int32
```

类型标签（`binpack.h`）：

|标签|值|写入|读出|
|---|---|---|---|
|INT32|6|`SetInt32`|`AsInt32`|
|BOOL|12|`SetBool`|`AsBool`|
|STRING|13|`SetString`|`AsString`|
|BINARY|14|`SetBinary`|`AsBinary`|
|ARRAY|15|`SetArray`|`AsArray`|
|OBJECT|16|`SetObject`|`AsObject`|
|JSON|18|`SetJson`|`AsJson`|

- 有些特殊位置字节含义是固定的，长度会根据上述标签对应的字节进行动态变化
- 至于如何保证字节传输稳定性，可以参考基站红外通讯协议，这个属于传输层的事情



### BinPack数据类型的理解


```text
data_bin = {
  "/MapData/SegmentSlamMapDataDirect": {
    "Width": ...,
    "Height": ...,
    "Data": ...
  }
}
```

- 将上述最初的结构，处理掉最外一层，得到的仍然是BinPack
```cpp
int AsObject(std::string key, BinPack& object);

data_bin.AsObject("/MapData/SegmentSlamMapDataDirect", out_bin);
```


写入以及读取：
```cpp
//往对象中写入一条条键值对
in_msg.SetUint32("offset", offset);
msg1.SetObject("/MapData/SegmentSlamMapDataDirect", in_msg);

//从对象中读取进行输出
out_bin.AsUint64("map_update_time", ts);   // 取出 uint64，写入 ts
out_bin.AsInt32("Width", response.width);  // 取出 int32，写入 width
```


## 语法学习

### 对象的生命周期

```cpp
class AposSdkIot {
 private:
   AposSdkIot* apos_iot_sdk_;
   
   
   ServiceManager* service_manager_;
```

```cpp
AposSdkIot *AposSdkIot::Instance() {
    static AposSdkIot apos_iot;
    return &apos_iot;
}


OperateCode AposSdkIot::apos_iot_init(Apos_Iot_Init_Param_Func init_param_func) {
    // 先创建单列
    LOGD("AposSdkSetEventSubscribe----------------------------");
    
    
    apos_iot_sdk_ = Instance();
```
- 声明只是方便编译，真正实现的时候才会决定对象的性质、生命周期
- 局部静态变量的生命周期也是整个程序


### 拷贝赋值与拷贝构造

```cpp
T a;      // 默认构造：a 已被创建，占有一块存储
T b;
b = a;    // 赋值：不创建新的 b，只改 b 里已有内容

# 其实本质相当于下面操作：
b.operator=(a);   // 拷贝赋值运算符
```
- 本质：两个对象早已存在，赋值的意思是——让左边对象（LHS）的存储内容，在语义上变成与右边（RHS）一致，通常不释放 LHS 的名字绑定（还是同一个 `b`，还是同一块「对象身份」）


```cpp
T b = a;    // 或 T b(a);
```
- 拷贝构造，空间还没诞生
- 这是在 创建 `b` 时用 `a` 初始化一块新存储，是 constructor，不是 assignment。



### 构造函数语法知识

```cpp
    HaierSdkNetwork()                                  = default;
    ~HaierSdkNetwork()                                 = default;
    HaierSdkNetwork(const HaierSdkNetwork&)            = delete;
    HaierSdkNetwork& operator=(const HaierSdkNetwork&) = delete;
```
- 如果开发者没有书写自己的构造函数，拷贝构造、拷贝赋值、移动构造、移动赋值都可以自动存在
- 只要类里出现了任何一个你自己声明的构造函数，编译器就不再自动生成无参的默认构造。
- 删除拷贝，本身也是一个构造函数声明；后续有“`new T()`，必须有无参构造”
- 所以第一行要把被挤掉的默认构造加回来


### C++指针

**裸指针必须手动释放，智能指针会根据作用阈范围自动释放：**

```cpp
UiMapProtocol* map_ = new UiMapProtocol();
map_->Init();
delete map_;     // 有人记得写，才和 unique_ptr 等价
```

```cpp
std::unique_ptr<UiMapProtocol> map_(new UiMapProtocol());
map_->Init();
```

智能指针常见的有两种：
1. 独享空间的智能指针，一个指针作用域结束直接释放资源： `std::unique_ptr`
2. 共享指针，所有资源结束作用域，才会进行释放指针： `std::share_ptr`


### explicit关键字理解

- 智能指针是一个类模板，传入“基类”类型后变为模板类
```cpp
template<typename T>
class unique_ptr {
public:
    using pointer = T*;

    explicit unique_ptr(pointer p) noexcept;   // explicit 在这里

    template<typename U>
    unique_ptr(unique_ptr<U>&& other) noexcept; // 这个没有 explicit
};
```

```cpp
class unique_ptr<ServiceWorkerBase> {
public:
    using pointer = ServiceWorkerBase*;

    // 构造函数 A：参数是基类原始指针，标了 explicit
    explicit unique_ptr(ServiceWorkerBase* p) noexcept;

    // 构造函数 B：参数是另一个 unique_ptr 临时对象，没有 explicit
    template<typename U>
    unique_ptr(unique_ptr<U>&& other) noexcept;
};
```
- 上面两种构造函数，实际使用如下：
- `noexcept` 写在参数列表后面，表示这个构造函数不会抛出异常。
- `&&` 表示这个参数绑定右值；在你的那行代码里，等号右边的智能指针没有变量名，所以它是临时对象


- **构造函数A--原始指针--代表着在括号里使用`new`创建指针**
- **构造函数B--智能指针--构造一个临时对象进行接收**


- 两种构造函数书写形式如下：
```cpp
using ServiceManagerPtr = std::unique_ptr<ServiceWorkerBase>;

  template<typename CLASSPTR>
  static void RegisterServer(const std::string& server_key, int32_t (CLASSPTR::*register_func)(), CLASSPTR *obj) {
    ServiceManagerPtr service_worker = std::unique_ptr<ServiceWorker<void, void>>(new ServiceWorker<void, void>(std::bind(register_func, obj)));
    service_map_[server_key] = std::move(service_worker);
  };

  template <typename CLASSPTR, typename REQ, typename REP>
  static void RegisterServer(const std::string& server_key, int32_t (CLASSPTR::*register_func)(REQ, REP), CLASSPTR *obj) {
    ServiceManagerPtr service_worker(new ServiceWorker<REQ, REP>(
            std::bind(register_func, obj, std::placeholders::_1, std::placeholders::_2)));
    service_map_[server_key] = std::move(service_worker);
  };
```

1. **传参方式进行构造有两种：括号是直接初始化，等号是拷贝初始化。**
2. 根据传参方式选择对应的构造函数


### 继承与虚函数

前两句的方向是对的，第三句把函数放进了对象的数据布局里。成员变量按“基类在前、派生类在后”排列；函数放在代码段里，不占用对象中基类数据后面的空间。

**继承时对象里有什么**

**单继承时，派生类对象的开头是完整的基类部分，派生类自己的成员变量接在后面。**`GetShowMapData` 那个对象就是：

```text
基类部分：vptr、service
派生类部分：register_func_
```

`vptr` 属于基类部分，因为虚函数是 `ServiceWorkerBase` 先声明的。`register_func_` 是派生类新增的数据，加在基类部分之后。基类原有成员的相对位置不变。

成员函数不在这段内存里。基类的 `Call` 和派生类的 `Call` 都是程序代码段中的函数，对象里没有一份“函数数据”可以往后追加。

**虚函数替换的是表里的地址**

派生类不会改基类那张虚函数表，也不会改基类函数的代码。编译器给派生类另做一张表。签名相同的虚函数，在派生类这张表的对应格子里，写入派生类函数的地址。

`ServiceWorkerBase` 的表：

```text
第 0 格：没有 IMessage 版本的函数体
第 1 格：ServiceWorkerBase::Call(BinPack&, BinPack&)
```

`ServiceWorker<const REQ &, REP &>` 的表：

```text
第 0 格：这个派生类的 Call(IMessage*, IMessage*)
第 1 格：仍是 ServiceWorkerBase::Call(BinPack&, BinPack&)
```

对象里的 `vptr` 指向第二张表。通过基类指针调用 `Call(IMessage*, IMessage*)` 时，读到的是第 0 格的派生类地址。第 1 格没有换，所以 `BinPack` 版本仍执行基类的 `return 0`。

这里的“同名、同参数”指的是虚函数签名一致，派生类这份函数才算重写，才能占住同一个格子。

**非虚函数不会放在数据结构下面**

派生类再写一个同名、同参数的普通函数，基类函数的代码还在，对象内存也不会多出一个函数指针。这是两个独立函数，派生类的名字把基类的名字隐藏起来。

调用哪一个，编译器看的是指针或对象的声明类型：

```cpp
ServiceWorkerBase* base = derived_object;
base->普通函数();      // 编译器直接调用基类的函数
derived_object->普通函数(); // 编译器直接调用派生类的函数
```

`base` 指向的对象里即使有派生类新增的 `register_func_`，普通函数调用也不会去查 `vptr`。基类函数因此碰不到派生类后半段里的数据。

虚函数才把选择权放到对象的 `vptr` 上。基类指针调用时，先进入派生类那份 `Call`，这份函数按派生类编译，所以能使用排在基类数据后面的 `register_func_`。


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
