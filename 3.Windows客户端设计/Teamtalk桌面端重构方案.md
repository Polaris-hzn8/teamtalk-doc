# Teamtalk桌面端重构方案

---

当一个解决方案中既有.exe可执行文件、又有.dll动态库时，目录结构的设计需要兼顾编译时依赖关系和运行时的查找路径。

# 现有架构分析

### 企业级IM系统架构设计考虑因素

1. 模块化设计：每个功能应该独立模块，便于单独编译和测试
2. 低耦合高内聚：减少模块间依赖，避免循环引用
3. 可扩展性：新功能可以容易地添加而不影响现有功能
4. 部署灵活性：核心IM功能应该独立，支持按需启动其他服务

对于音视频、云盘、远程控制这些功能，采用插件化架构会更合适。通过抽象接口定义模块边界，利用依赖注入和工厂模式来管理模块生命周期，这样能在不影响主程序稳定性的前提下实现功能扩展。

### 现有架构分析

```shell
teamtalk.sln
├── 3rdparty/                    # 第三方库
│   ├── httpclient/              # HTTP客户端
│   ├── libogg/                  # Ogg音频库
│   ├── libspeex/                # Speex音频编解码
│   ├── GifSmiley/               # GIF表情库
│   └── DuiLib/                  # UI界面库
├── TTClient/                    # 主客户端
│   ├── Modules.vcxproj         # 业务模块DLL（MFC扩展）
│   │   ├── 依赖：utility.lib, network.lib, httpclient.lib, DuiLib.lib
│   │   └── 包含：cxImage图像库、protobuf协议、libsecurity安全库
│   │   └── 包含：Login/Message/Session/Database等业务模块
│   ├── Network.vcxproj         # 网络库DLL
│   │   └── 依赖：ws2_32.lib
│   │   └── 包含：BaseSocket/EventDispatch/ImPduBase等底层网络
│   ├── Utility.vcxproj          # 工具库DLL（MFC扩展）
│   │   ├── 依赖：sqlite3.lib, network.lib, Netapi32.lib
│   │   └── 包含：jsoncpp/libsecurity/sqlite3/工具函数
│   └── teamtalk.vcxproj        # 主程序EXE
│       └── 依赖：utility.lib, network.lib, DuiLib.lib, Modules.lib
```

### 核心设计亮点

| 方面     | 评价                                                         |
| :------- | :----------------------------------------------------------- |
| 模块化   | 业务模块（Modules）、网络（Network）、工具（Utility）分离较好 |
| 代码复用 | 公共库（sharesrc）被多个项目共享                             |
| 接口设计 | Modules内部有清晰的接口定义（I*Module.h）和实现分离（*_Impl.h） |
| MKO模式  | 模块间通过观察者模式解耦                                     |

1. MKO观察者模式：模块间通过事件解耦，降低耦合度
2. 分层清晰：UI -> modules -> network -> utility
3. 动态库模块化：modules.dll封装了所有的业务模块

### 存在的问题

| 问题                                | 描述                                                         |
| :---------------------------------- | :----------------------------------------------------------- |
| Modules.dll 过于庞大                | 包含：图像编解码(cxImage) + 协议(protobuf) + 安全库 + 所有业务模块 |
| sharesrc 目录混乱                   | cxImage、libsecurity、protocal混在一起，没有明确归属         |
| utility.dll 与 Modules.dll 重复依赖 | libsecurity (aes/base64/md5) 在两边都有编译                  |
| Modules.dll 依赖 MFC                | 限制了使用场景（无法在非MFC项目中使用）                      |
| 网络层和业务层耦合                  | TcpClientModule_Impl 内部直接处理网络逻辑                    |

1. modules、network、utility模块间循环依赖

   ```
   modules ←→ network ←→ utility  ←→ modules (三角循环)
   ```

2. 缺乏清晰的层级边界

   - 哪些是"底层基础库"？
   - 哪些是"业务模块"？
   - 哪些是"界面层"？

3. 模块职责边界模糊

   - `module` 下混杂了 UI 代码（Dialog、Session UI）和业务逻辑

4. UI代码混入modules：Dialog、Layout等界面代码与业务逻辑混在一起



# 改造方案1：分层架构

1. 可以考虑将network和utility提升到独立的基础库级别，让各业务模块都依赖它们，从而解决循环依赖问题。
2. 另一种方案是建立更清晰的层级结构：基础层处理核心功能、业务层承载具体模块、公共层提供共享代码。在TTClient下采用include/src目录分离头文件和实现，能让项目结构更加规范。

```shell
teamtalk-win/
├── 3rdparty/                 # 第三方库（不改）
├── dist/                      # 打包资源（不改）
│
├── framework/                # 核心框架层 - 纯接口，不依赖任何业务
│   ├── include/
│   │   ├── imcore/            # 网络核心接口 (ITcpClient, IHttpClient, IPacketParser)
│   │   ├── immodule/          # 业务模块抽象基类 (IModule, IModuleManager)
│   │   └── imcommon/          # 公共基础类型 (IMessage, IUser, IGroup)
│   └── src/
│
├── modules/                   # 业务模块层 - 依赖 framework，可互相独立
│   ├── im-core/               # IM核心模块（用户、消息、群组）
│   ├── im-audio/              # 音视频模块（预留）
│   ├── im-cloud/               # 云盘模块（预留）
│   ├── im-remote/              # 远程协助模块（预留）
│   └── im-common/             # 所有模块共享的工具类
│
├── TTClient/                   # 客户端进程
│   ├── UI/                     # 界面层（只依赖 modules 的接口）
│   └── Main/                   # 程序入口
│
└── sharelib/                   # 导出库/配置文件
```

# 改造方案2：插件化架构/长期演进

```cpp
// 核心框架定义模块接口
class IModule {
public:
    virtual const char* getName() = 0;
    virtual bool initialize(IModuleContext* ctx) = 0;
    virtual void shutdown() = 0;
};

// 模块管理器
class ModuleManager {
    std::map<std::string, IModule*> m_modules;
public:
    void registerModule(const char* name, IModule* module);
    void loadModule(const char* name);      // 运行时加载
    void unloadModule(const char* name);
};
```

```shell
modules/
├── module-core/               # 用户、消息、群组
│   └── plugin.json            # 声明依赖
├── module-audio/              # 音视频
│   └── plugin.json
├── module-cloud/              # 云盘
│   └── plugin.json
└── module-remote/            # 远程控制
    └── plugin.json
```

### 循环依赖解决方案

```shell
# 修改前的依赖关系
modules/          ←→  network/
    ↓                     ↓
 包含网络接口         包含业务定义
 # 修改后的依赖关系
framework/          # 无任何依赖，是最底层
    ↓
network/            # 只依赖 framework
    ↓
modules/            # 依赖 framework + network 接口
    ↓
TTClient UI/        # 依赖 modules 接口
```

解决思路：提取公共抽象层

```cpp
// framework/imcore/include/IConnection.h
// 网络层只定义连接接口，不涉及任何业务
class IConnection {
    virtual void connect(const std::string& ip, int port) = 0;
    virtual void sendPacket(const char* data, size_t len) = 0;
    virtual void disconnect() = 0;
    virtual void setCallback(IConnectionCallback* cb) = 0;
};

// framework/imcommon/include/Message.h
// 公共消息定义，业务和网络都引用这个
struct MessageHeader {
    uint32_t    type;
    uint32_t    length;
    std::string senderId;
};
```

目录结构调整：推荐的 include/src 分离

```shell
TTClient/
├── include/                    # 对外公开的头文件
│   ├── Modules/               # 模块接口 (ILoginModule.h, IMessageModule.h)
│   ├── Network/                # 网络接口
│   └── Utility/               # 工具接口
│
├── src/                        # 源代码
│   ├── Modules/                # 模块实现
│   ├── Network/                # 网络实现
│   └── Utility/               # 工具实现
│
├── Modules/                    # 【保留】资源文件(xml/图片)
└── UI/                         # 【保留】界面代码
```

```shell
TTClient/
├── include/                     # $(ProjectDir)\include
├── src/                         # $(ProjectDir)\src
├── sharesrc/                     # $(SolutionDir)\sharesrc
└── framework/include/            # $(SolutionDir)\framework\include
```

### 改进建议

| 改进项                | 优先级 | 说明                   |
| :-------------------- | :----- | :--------------------- |
| 提取 framework 层     | 🔴 高   | 解决循环依赖的根本方案 |
| 模块接口归入 include/ | 🔴 高   | 清晰的对内对外边界     |
| 模块实现归入 src/     | 🟡 中   | 符合现代C++项目惯例    |
| 引入插件机制          | 🟡 中   | 为未来扩展预留         |
| 按功能拆分 modules    | 🟢 低   | 当前功能简单，可暂缓   |



# 改造方案3：架构改进兼容后续扩展

---

| 扩展功能   | 涉及模块                                        | 改动范围 |
| :--------- | :---------------------------------------------- | :------- |
| 音视频通话 | 网络(需扩展)、Modules(新模块)、UI(新界面)       | 中等     |
| 云盘       | 网络(可能新协议)、Modules(新模块)、数据库(新表) | 中等     |
| 远程控制   | 网络(重载)、Modules(新模块)、屏幕采集           | 较大     |
| 社交娱乐   | Modules(新模块)                                 | 较小     |

### 微服务架构改进

将项目拆分成类似微服务架构的类型，不再把所有功能放在一个teamtalk可执行程序中。可以提高系统的可扩展性、可维护性和模块独立性。

1. 现有架构问题：
   - modules.dll过于臃肿
   - 所有的模块都在一个进程中
   - 模块间耦合程度过高
2. 拆分成微服务架构优势：
   - 独立部署和升级
   - 故障隔离
   - 技术栈灵活
3. 拆分维度：
   - 按功能域拆分（IM、音视频、云盘等）
   - 按技术边界拆分（前端、后端、数据存储）
   - 按部署单元拆分

### 进程拆分架构

```shell
teamtalk-win/
├── bin/
│   ├── teamtalk.exe              # 主进程 - 负责启动和管理子进程
│   ├── im-core.exe               # IM核心服务 - 用户、消息、群组
│   ├── im-audio.exe              # 音视频服务 - 语音、视频通话
│   ├── im-cloud.exe              # 云盘服务 - 文件存储
│   ├── im-remote.exe             # 远程协助服务
│   └── ui-desktop.exe            # 桌面UI进程
```

```shell
┌─────────────────────────────────────────────────────────────────────────┐
│                        TeamTalk Desktop Platform                         │
│                                                                         │
│  ┌──────────────┐                                                     │
│  │ teamtalk.exe │  ← 主进程/进程管理器（Process Manager）               │
│  │              │     - 启动/停止子进程                                  │
│  │  • 日志聚合   │     - 进程监控/自动重启                               │
│  │  • 配置管理   │     - 全局配置分发                                    │
│  │  • 进程调度   │     - 进程间通信协调                                  │
│  └──────────────┘                                                     │
│           │                                                            │
│           │ 进程间通信 (IPC)                                            │
│           │                                                            │
│  ┌────────┴────────────────────────────────────────────────────────┐  │
│  │                    IPC Bus (通信总线)                              │  │
│  │                                                                      │  │
│  │   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐           │  │
│  │   │im-core  │  │im-audio │  │im-cloud │  │im-remote │           │  │
│  │   │ .exe    │  │  .exe   │  │  .exe   │  │   .exe   │           │  │
│  │   └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘           │  │
│  │        │            │            │            │                  │  │
│  │        ▼            ▼            ▼            ▼                  │  │
│  │   ┌─────────────────────────────────────────────────────────┐    │  │
│  │   │              共享消息队列 / Unix Domain Socket           │    │  │
│  │   └─────────────────────────────────────────────────────────┘    │  │
│  └────────────────────────────────────────────────────────────────┘  │
│           │                                                            │
│           │  IPC                                                       │
│           ▼                                                            │
│  ┌─────────────────┐                                                   │
│  │   ui-desktop    │  ← 桌面UI进程（Qt/Duilib）                        │
│  │     .exe         │     - 主窗口                                      │
│  │                 │     - 聊天界面                                    │
│  │  • 聊天界面      │     - 联系人列表                                  │
│  │  • 联系人列表    │     - 设置面板                                    │
│  │  • 设置面板      │     - 音视频通话UI                                │
│  └─────────────────┘                                                   │
└─────────────────────────────────────────────────────────────────────────┘
```

### 进程职责划分

| 进程           | 功能域   | 职责                                   | 依赖              |
| :------------- | :------- | :------------------------------------- | :---------------- |
| teamtalk.exe   | 平台层   | 进程管理、配置分发、日志聚合、启动引导 | 无                |
| im-core.exe    | IM核心   | 用户、消息、群组、TCP长连接            | Network库         |
| im-audio.exe   | 音视频   | 语音/视频编解码、RTP传输               | Network库、音频库 |
| im-cloud.exe   | 云存储   | 文件上传下载、增量同步                 | Network库         |
| im-remote.exe  | 远程控制 | 屏幕采集、RFB协议、远程桌面            | im-audio.exe      |
| ui-desktop.exe | 表现层   | 界面渲染、用户交互、调用各服务         | 所有服务          |

### 进程间通信设计

各进程间通过消息队列、共享内存或gRPC等高效通信机制交互。主进程充当服务编排器，负责管理各子进程的启动、监控和生命周期。

UI进程专注于界面展示，IM核心进程处理用户和消息逻辑，音视频进程独立处理多媒体流，云盘和远程协助进程各司其职。

考虑到即时通讯的低延迟需求，采用共享内存实现高速数据交换，同时通过Unix Domain Socket或命名管道处理控制命令。

gRPC作为跨平台通信框架也可考虑。

具体来看，IM服务应涵盖用户管理、消息收发、群组操作和TCP连接维护；音视频服务负责音频编解码、视频采集和RTP/RTCP协议；云盘服务处理文件上传下载和存储管理；远程协助则包含屏幕采集、远程控制和实时音视频交互。

不过实施这一架构需要权衡三个关键问题：C++环境下缺乏像Java那样成熟的RPC框架，IM系统对实时性要求极高IPC可能成为瓶颈，以及团队可能缺乏微服务架构的实践经验。我应该为用户提供一份既务实又可落地的详细改造方案。

```
┌─────────────────────────────────────────────────────────────────┐
│                        IPC 通信架构                               │
│                                                                  │
│  ┌──────────┐        ┌──────────┐        ┌──────────┐         │
│  │  服务A   │        │ IPC Bus  │        │  服务B   │         │
│  └─────┬────┘        └────┬────┘        └────┬────┘         │
│        │                   │                   │              │
│        │    同步RPC调用    │                   │              │
│        ├──────────────────▶│◀──────────────────┤              │
│        │                   │                   │              │
│        │    异步消息发布   │                   │              │
│        ├──────────────────▶│◀──────────────────┤              │
│        │                   │                   │              │
│        │    共享内存      │                   │              │
│        ├──────────────────▶├◀─────────────────┤              │
│        │                   │                   │              │
└────────┴───────────────────┴───────────────────┴──────────────┘
```

```
┌─────────────────────────────────────────────────────────────┐
│                  进程间通信方式对比                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  方式           速度      复杂度    适用场景                  │
│  ─────────────────────────────────────────────              │
│  WM_COPYDATA    快        低        Win32原生，适合UI进程    │
│  命名管道       中        中        可靠的请求-响应模式       │
│  Socket(TCP)    中        中        跨机器/稳定性要求高      │
│  共享内存       最快      高        大数据共享（文件传输）     │
│  消息队列       快        低        异步事件通知              │
│                                                             │
└─────────────────────────────────────────────────────────────┘

推荐组合：
- UI进程 ↔ 核心进程：命名管道 + 自定义消息协议
- 大文件传输：共享内存 + Socket通知
```

#### 通信消息格式设计

```cpp
// IPC 消息头
struct IPCMessageHeader {
    uint32_t    magic;          // 魔数 0x54546541 ("TTEA")
    uint16_t    version;        // 版本号
    uint16_t    msg_type;      // 消息类型
    uint32_t    sender_pid;    // 发送方进程ID
    uint32_t    receiver_pid;   // 接收方进程ID
    uint64_t    seq_num;       // 序列号（用于追踪）
    uint32_t    body_len;      // 消息体长度
    uint8_t     flags;         // 标志（压缩/加密等）
    uint8_t     reserved[3];   // 保留
};

// 消息类型枚举
enum IPCMsgType : uint16_t {
    // 控制消息
    IPC_MSG_HEARTBEAT       = 0x0001,
    IPC_MSG_REGISTER        = 0x0002,  // 服务注册
    IPC_MSG_SUBSCRIBE       = 0x0003,  // 订阅消息
    
    // IM消息
    IPC_MSG_IM_SEND         = 0x1001,  // 发送消息
    IPC_MSG_IM_RECV         = 0x1002,  // 接收消息
    IPC_MSG_IM_STATUS       = 0x1003,  // 用户状态
    
    // 音视频消息
    IPC_MSG_AV_START        = 0x2001,  // 开始通话
    IPC_MSG_AV_STOP         = 0x2002,  // 结束通话
    IPC_MSG_AV_DATA         = 0x2003,  // 媒体数据
    
    // 云盘消息
    IPC_MSG_CLOUD_UPLOAD    = 0x3001,  // 上传文件
    IPC_MSG_CLOUD_DOWNLOAD  = 0x3002,  // 下载文件
    IPC_MSG_CLOUD_SYNC      = 0x3003,  // 同步状态
    
    // 远程控制消息
    IPC_MSG_REMOTE_REQUEST  = 0x4001,  // 请求远程
    IPC_MSG_REMOTE_CONTROL  = 0x4002,  // 控制指令
    IPC_MSG_REMOTE_SCREEN   = 0x4003,  // 屏幕数据
};
```

```cpp
// framework/common/im_protocol.h

// 进程间消息头
struct IPCMessageHeader {
    uint32_t    magic;          // 魔数 0x5454454E ("TTEN")
    uint16_t    version;       // 协议版本
    uint16_t    cmd_type;      // 命令类型
    uint32_t    src_process;   // 源进程ID
    uint32_t    dst_process;   // 目标进程ID
    uint32_t    seq_id;        // 序列号（用于请求-响应匹配）
    uint32_t    body_length;   // 消息体长度
};

// 进程ID定义
enum ProcessID {
    PROCESS_CORE   = 1,
    PROCESS_UI     = 2,
    PROCESS_VIDEO  = 3,
    PROCESS_CLOUD  = 4,
    PROCESS_REMOTE = 5,
    PROCESS_SOCIAL = 6,
};

// 跨进程命令类型
enum IPCmdType {
    // UI → Core
    IPCMD_UI_LOGIN_REQUEST     = 1001,
    IPCMD_UI_SEND_MESSAGE      = 1002,
    IPCMD_UI_CREATE_GROUP      = 1003,
    
    // Core → UI
    IPCMD_CORE_MESSAGE_RECEIVED = 2001,
    IPCMD_CORE_USER_STATUS_CHANGED = 2002,
    IPCMD_CORE_LOGIN_SUCCESS    = 2003,
    
    // 音视频相关
    IPCMD_VIDEO_CALL_REQUEST   = 3001,
    IPCMD_VIDEO_CALL_ACCEPTED  = 3002,
    IPCMD_VIDEO_CALL_END       = 3003,
    
    // 云盘相关
    IPCMD_CLOUD_UPLOAD_PROGRESS = 4001,
    IPCMD_CLOUD_DOWNLOAD_COMPLETE = 4002,
    
    // 远程控制相关
    IPCMD_REMOTE_CONTROL_START  = 5001,
    IPCMD_REMOTE_CONTROL_STOP   = 5002,
    IPCMD_REMOTE_SCREEN_DATA    = 5003,
};
```



#### 服务注册与发现

```cpp
// 服务注册表
struct ServiceInfo {
    std::string     name;           // 服务名称 "im-core"
    uint32_t        pid;            // 进程ID
    std::string     endpoint;       // 通信端点 "/tmp/tt-im-core.sock"
    std::vector<std::string> capabilities;  // 能力列表
    uint32_t        status;         // 服务状态
    uint64_t        heartbeat_ts;  // 心跳时间戳
};

// IPC Bus 核心接口
class IIPCBus {
public:
    // 服务管理
    virtual bool registerService(const ServiceInfo& info) = 0;
    virtual void unregisterService(const std::string& name) = 0;
    virtual ServiceInfo* getService(const std::string& name) = 0;
    
    // 消息发送
    virtual bool sendMessage(uint32_t target_pid, const IPCMessage& msg) = 0;
    virtual bool publishMessage(const std::string& topic, const IPCMessage& msg) = 0;
    
    // 订阅
    virtual void subscribe(const std::string& topic, IIPCSubscriber* cb) = 0;
    virtual void unsubscribe(const std::string& topic, IIPCSubscriber* cb) = 0;
    
    // RPC调用
    virtual std::future<IPCMessage> callRPC(
        uint32_t target_pid, 
        const IPCMessage& request,
        uint32_t timeout_ms = 5000
    ) = 0;
};
```

### 目录结构设计

```
teamtalk-win/
│
├── bin/                         # 可执行文件和动态库
│   ├── teamtalk.exe              # 主进程
│   ├── im-core.exe               # IM核心服务
│   ├── im-audio.exe              # 音视频服务
│   ├── im-cloud.exe              # 云盘服务
│   ├── im-remote.exe             # 远程控制服务
│   └── ui-desktop.exe            # 桌面UI进程
│
├── services/                     # 服务源代码
│   ├── im-core/                  # IM核心服务
│   │   ├── include/              # 服务接口
│   │   │   ├── iim-service.h    # IM服务接口定义
│   │   │   └── im-protocol.h    # IM协议定义
│   │   ├── src/                  # 实现
│   │   │   ├── UserManager.cpp
│   │   │   ├── MessageStore.cpp
│   │   │   ├── GroupManager.cpp
│   │   │   └── TcpConnection.cpp
│   │   └── CMakeLists.txt
│   │
│   ├── im-audio/                 # 音视频服务
│   │   ├── include/
│   │   │   ├── iaudio-service.h
│   │   │   ├── AudioCodec.h
│   │   │   └── RtpSession.h
│   │   └── src/
│   │       ├── AudioCapture.cpp
│   │       ├── SpeexCodec.cpp
│   │       └── RtpTransport.cpp
│   │
│   ├── im-cloud/                 # 云盘服务
│   │   ├── include/
│   │   │   └── icloud-service.h
│   │   └── src/
│   │
│   └── im-remote/                # 远程控制服务
│       ├── include/
│       │   └── iremote-service.h
│       └── src/
│           ├── ScreenCapture.cpp
│           └── RfbProtocol.cpp
│
├── ui/                           # UI进程
│   ├── desktop/                  # 桌面UI
│   │   ├── include/
│   │   │   ├── MainWindow.h
│   │   │   ├── ChatView.h
│   │   │   ├── ContactList.h
│   │   │   └── ServiceProxy.h   # 服务代理（调用后端服务）
│   │   ├── src/
│   │   │   ├── MainWindow.cpp
│   │   │   ├── ChatView.cpp
│   │   │   └── ServiceProxy.cpp # IPC客户端封装
│   │   └── CMakeLists.txt
│   │
│   └── common/                   # UI公共组件
│       ├── include/
│       │   ├── IUIBridge.h     # UI回调接口
│       │   └── IPCClient.h     # IPC客户端封装
│       └── src/
│
├── framework/                     # 核心框架
│   ├── ipc/                      # IPC通信框架
│   │   ├── include/
│   │   │   ├── IPCBus.h
│   │   │   ├── IPCMessage.h
│   │   │   ├── SharedMemory.h
│   │   │   └── UnixSocket.h
│   │   └── src/
│   │
│   ├── process/                  # 进程管理框架
│   │   ├── include/
│   │   │   ├── ProcessManager.h
│   │   │   ├── IService.h
│   │   │   └── ServiceContext.h
│   │   └── src/
│   │       ├── ProcessManager.cpp
│   │       └── ServiceContext.cpp
│   │
│   └── common/                   # 公共基础设施
│       ├── include/
│       │   ├── Logger.h
│       │   ├── ConfigManager.h
│       │   └── EventLoop.h
│       └── src/
│
├── sharesrc/                     # 共享源码
│   ├── protocal/                 # 协议定义
│   ├── libsecurity/              # 安全库
│   └── GlobalDefine.h            # 全局类型
│
├── thirdparty/                    # 第三方库
│   ├── protobuf/
│   ├── speex/
│   └── duilib/
│
├── dist/                         # 打包资源
│   ├── configs/                  # 配置文件
│   ├── skins/                    # 界面主题
│   └── sounds/                   # 提示音
│
└── scripts/                      # 构建脚本
    ├── build-all.bat
    └── package.bat
```

### 进程生命周期管理

```cpp
// 主进程 - Process Manager
class ProcessManager {
private:
    std::map<std::string, ServiceProcess> m_services;
    IIPCBus* m_ipcBus;
    
public:
    // 启动所有服务
    bool startup() {
        // 1. 初始化IPC总线
        m_ipcBus = IPCBus::create();
        m_ipcBus->initialize();
        
        // 2. 启动各服务进程
        startService("im-core",   "./im-core.exe");
        startService("im-audio",  "./im-audio.exe");
        startService("im-cloud",  "./im-cloud.exe");
        startService("im-remote", "./im-remote.exe");
        startService("ui-desktop", "./ui-desktop.exe");
        
        // 3. 等待服务就绪
        waitForServicesReady();
        
        // 4. 启动UI
        return true;
    }
    
    // 监控服务健康状态
    void monitorLoop() {
        while (m_running) {
            for (auto& [name, svc] : m_services) {
                if (!checkProcessAlive(svc.pid)) {
                    LOG__(WARN, "Service %s died, restarting...", name);
                    restartService(name);
                }
            }
            std::this_thread::sleep_for(1s);
        }
    }
    
    // 优雅退出
    void shutdown() {
        // 发送优雅退出信号
        for (auto& [name, svc] : m_services) {
            sendShutdownSignal(svc.pid);
        }
        // 等待进程退出
        for (auto& [name, svc] : m_services) {
            waitForExit(svc.pid, 5000);
            if (isAlive(svc.pid)) {
                killProcess(svc.pid);  // 强制终止
            }
        }
    }
};
```

### 服务进程基类

```cpp
// IService - 所有服务的基类
class IService {
protected:
    ServiceContext* m_context;      // 服务上下文
    IIPCBus* m_ipcBus;             // IPC通信
    EventLoop* m_eventLoop;         // 事件循环
    
public:
    virtual const char* getName() = 0;
    
    // 服务生命周期
    virtual bool onInit() = 0;      // 初始化
    virtual void onRun() = 0;        // 运行
    virtual void onStop() = 0;      // 停止
    virtual void onRelease() = 0;   // 释放资源
    
    // 消息处理
    virtual void handleMessage(const IPCMessage& msg) = 0;
    
public:
    void run() {
        // 1. 注册服务
        registerService();
        
        // 2. 初始化
        if (!onInit()) {
            LOG__(ERR, "Service %s init failed", getName());
            return;
        }
        
        // 3. 进入事件循环
        m_eventLoop->loop();
        
        // 4. 停止
        onStop();
        onRelease();
    }
    
    void registerService() {
        ServiceInfo info;
        info.name = getName();
        info.pid = getCurrentPid();
        info.endpoint = getEndpoint();
        m_ipcBus->registerService(info);
    }
};

// IM核心服务示例
class IMService : public IService {
public:
    const char* getName() override { return "im-core"; }
    
    bool onInit() override {
        // 初始化数据库
        m_db.init("im-core.db");
        
        // 初始化网络连接
        m_tcpClient.connect(serverIP, serverPort);
        
        // 订阅相关消息
        m_ipcBus->subscribe("ui.*", this);
        
        return true;
    }
    
    void handleMessage(const IPCMessage& msg) override {
        switch (msg.header.msg_type) {
            case IPC_MSG_IM_SEND:
                handleSendMessage(msg);
                break;
            case IPC_MSG_IM_STATUS:
                handleStatusChange(msg);
                break;
        }
    }
};
```

### UI进程与服务交互

```cpp
// ServiceProxy - UI调用服务的代理
class IMServiceProxy {
private:
    uint32_t m_imCorePid;
    IIPCBus* m_ipcBus;
    
public:
    // 发送消息
    bool sendMessage(const std::string& toUser, const std::string& content) {
        IPCMessage msg;
        msg.header.msg_type = IPC_MSG_IM_SEND;
        msg.body = packMessage(toUser, content);
        return m_ipcBus->sendMessage(m_imCorePid, msg);
    }
    
    // 获取历史消息
    std::future<std::vector<Message>> getHistory(
        const std::string& sessionId, 
        int limit
    ) {
        IPCMessage req;
        req.header.msg_type = IPC_MSG_IM_HISTORY;
        req.body = packHistoryRequest(sessionId, limit);
        
        return m_ipcBus->callRPC(m_imCorePid, req, 3000)
            .then([](IPCMessage&& resp) {
                return unpackMessages(resp.body);
            });
    }
    
    // 订阅消息回调
    void subscribeMessages(IMessageCallback* cb) {
        m_ipcBus->subscribe("im.message.*", cb);
    }
};
```

### 数据共享策略

方案1：消息传递（推荐）

```
┌─────────┐      IPC消息       ┌─────────┐
│ IM服务  │ ─────────────────▶ │  UI进程  │
│         │ ◀───────────────── │          │
└─────────┘    消息回调         └─────────┘
```

方案2：共享内存（可选）

```
┌─────────┐      共享内存       ┌─────────┐
│ IM服务  │ ◀═══════════════▶ │  UI进程  │
│         │                    │          │
└─────────┘                    └─────────┘
     │                              │
     ▼                              ▼
   数据库                        UI本地缓存
```

适用于：大文件二进制数据、高频更新数据

| 阶段    | 目标         | 工作量                  |
| :------ | :----------- | :---------------------- |
| Phase 1 | 进程拆分框架 | 搭建IPC框架、主进程管理 |
| Phase 2 | IM核心独立   | 将im-core.exe独立       |
| Phase 3 | UI解耦       | UI进程通过IPC调用IM     |
| Phase 4 | 音视频模块   | 开发im-audio.exe        |
| Phase 5 | 云盘模块     | 开发im-cloud.exe        |





# 其他问题

### 改造方案对比

| 维度     | 方案1（清晰分层）    | 方案2（插件化）     | 微服务架构            |
| :------- | :------------------- | :------------------ | :-------------------- |
| 进程数   | 单进程               | 单进程/可选多进程   | 多进程                |
| 模块隔离 | 编译时隔离           | 运行时加载          | 进程级隔离            |
| 部署     | 整体部署             | 可选热更新          | 独立部署升级          |
| 通信方式 | 函数调用+MKO         | 函数调用+MKO        | IPC通信               |
| 崩溃影响 | 一个模块挂导致全部挂 | 可选隔离            | 单进程崩溃不影响其他  |
| 适用场景 | 中小型应用           | 中型应用/部分插件化 | 大型应用/需要高可靠性 |

### MKO模式的保留

| 场景          | 原 MKO 模式  | 通信方式              | 演进后方案                        |
| :------------ | :----------- | --------------------- | :-------------------------------- |
| 进程内模块间  | ✅ MKO 观察者 | 函数调用、内存事件    | ✅ 保留，`LocalEventBus`模块间解耦 |
| 进程内→跨进程 | ❌ 不适用     | IPC、Socket、消息队列 | ✅ `IPCEventBus` 适配              |
| 跨进程服务间  | ❌ 不适用     |                       | ✅ gRPC / TCP / NamedPipe          |

```shell
┌─────────────────────────────────────────────────────────────┐
│                    UI 进程 (teamtalk.exe)                    │
│                                                             │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐    │
│   │ MainUI  │  │SessionUI│  │ContactUI│  │SettingsUI│    │
│   └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘    │
│        │            │            │            │            │
│        └────────────┴────────────┴────────────┘            │
│                         │                                    │
│              ┌──────────▼──────────┐                       │
│              │    本地事件总线       │  (MKO模式 - 进程内)     │
│              │   LocalEventBus     │                       │
│              └──────────┬──────────┘                       │
└─────────────────────────┼───────────────────────────────────┘
                          │ IPC / TCP localhost / Named Pipe
┌─────────────────────────┼───────────────────────────────────┐
│                    事件网关层  (EventGateway)                │
│              ┌──────────▼──────────┐                       │
│              │   进程间事件总线      │  (跨进程通信)           │
│              │   IPCEventBus       │                       │
│              └──────────┬──────────┘                       │
└─────────────────────────┼───────────────────────────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
        ▼                 ▼                 ▼
┌───────────────┐ ┌───────────────┐ ┌───────────────┐
│  IM 服务进程   │ │  文件服务进程   │ │  音视频服务   │
│  im-server.exe│ │ file-server.exe│ │av-server.exe │
│               │ │               │ │              │
│ - 消息收发     │ │ - 文件上传下载 │ │ - 音视频通话 │
│ - 用户管理     │ │ - 云盘存储     │ │ - 屏幕共享   │
│ - 群组管理     │ │               │ │              │
│               │ │               │ │              │
│ ┌───────────┐ │ │ ┌───────────┐ │ │              │
│ │本地事件总线│ │ │ │本地事件总线│ │ │              │
│ └───────────┘ │ │ └───────────┘ │ │              │
└───────────────┘ └───────────────┘ └───────────────┘
```

1. MKO 观察者模式在同进程内的价值：
   - 模块间解耦，通过事件通信
   - 避免直接依赖
   - 但它本质上是进程内的通信机制
2. 跨进程通信（IPC）选项：
   - 命名管道（Named Pipe）
   - TCP Socket（本地回环）
   - gRPC
   - 共享内存 + 消息队列
   - Windows Message（仅限同桌面会话）











