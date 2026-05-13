# 第五阶段：TCPServer (C++) 概览

> 理解 C++ 游戏服务器的架构、IOCP 模型和启动流程

---

## 目录

1. [C++ vs Java 语法对照](#1-c-vs-java-语法对照)
2. [项目结构](#2-项目结构)
3. [IOCP 模型](#3-iocp-模型)
4. [启动流程](#4-启动流程)
5. [Packet 处理](#5-packet-处理)
6. [C++ 到 C# 过渡建议](#6-c-到-c-过渡建议)

---

## 1. C++ vs Java 语法对照

### 基本类型

| C++ | Java | 说明 |
|-----|------|------|
| `int` | `int` | 相同 |
| `long` | `long` | 可能不同大小 |
| `bool` | `boolean` | C++ 用 `bool` |
| `char` | `char` | C++ 是 1 字节 |
| `uint32_t` | `int` | 推荐用固定宽度类型 |
| `int64_t` | `long` | 64 位整数 |
| `std::string` | `String` | C++ 字符串 |

### 指针和引用

```cpp
// Java: 所有对象都是引用
// C++: 有指针(*)和引用(&)两种

// 指针 - 可以是 nullptr
std::string* pStr = &someString;  // 取地址
std::string* pStr2 = nullptr;     // 空指针
if (pStr != nullptr) { }          // 检查空

// 引用 - 必须初始化，不能是 nullptr
std::string& rStr = someString;   // 必须绑定到对象
rStr = "changed";                 // 直接修改原对象

// C++11 后推荐用智能指针，避免手动内存管理
#include <memory>
std::unique_ptr<std::string> pStr = std::make_unique<std::string>("hello");
std::shared_ptr<std::string> pStr2 = std::make_shared<std::string>("world");
```

### 类定义

```cpp
// C++ 类
class User {
public:  // 默认 private
    // 构造器
    User() : userUid(0), level(1) { }  // 初始化列表
    User(uint64_t uid, int lv) : userUid(uid), level(lv) { }

    // 成员函数
    void SetLevel(int lv) { level = lv; }
    int GetLevel() const { return level; }  // const 表示不修改成员

    // 成员变量
    uint64_t userUid;
    int level;

private:
    std::string name;
};

// 使用
User user;              // 栈上分配
user.SetLevel(10);
User* pUser = new User();  // 堆分配 (不推荐)
delete pUser;           // 必须手动释放
```

### 命名空间

```cpp
// C++
namespace GameServer {
    namespace Packet {
        class Handler { };
    }
}

// 使用
GameServer::Packet::Handler handler;

// 或 using
using GameServer::Packet::Handler;
Handler handler2;
```

### 模板 (泛型)

```cpp
// C++ 模板 (类似 Java 泛型，但更强大)
template<typename T>
class List {
    T* data;
    int size;
public:
    void Add(T item) { /* ... */ }
    T Get(int index) { return data[index]; }
};

// 使用
List<int> intList;           // int 列表
List<std::string> strList;    // string 列表

// Java 风格 - C++11 后也支持 auto
auto it = intList.Get(0);  // 自动推导类型
```

### 容器

| C++ | Java | 说明 |
|-----|------|------|
| `std::vector<T>` | `List<T>` | 动态数组 |
| `std::list<T>` | `LinkedList<T>` | 链表 |
| `std::map<K,V>` | `Map<K,V>` | 有序映射 |
| `std::unordered_map<K,V>` | `HashMap<K,V>` | 哈希映射 |
| `std::set<T>` | `HashSet<T>` | 集合 |
| `std::queue<T>` | `Queue<T>` | 队列 |

```cpp
// C++ 容器使用
#include <vector>
#include <unordered_map>

std::vector<int> nums = {1, 2, 3, 4, 5};
nums.push_back(6);
int first = nums[0];

std::unordered_map<uint64_t, User*> users;
users[12345] = &user;
User* u = users[12345];

// 遍历
for (const auto& num : nums) {
    std::cout << num << std::endl;
}
```

### 预处理指令

```cpp
// C++ 预处理 (Java 没有!)
#define MAX_PLAYER 1000       // 宏定义
#define PI 3.14159

#ifdef DEBUG                 // 条件编译
    std::cout << "debug mode";
#else
    std::cout << "release mode";
#endif

// 本项目使用
#if _DB_SHARDING_
    // 分库分表代码
#else
    // 非分库分表代码
#endif
```

---

## 2. 项目结构

```
TCPServer/
├── CommonLib/              # 公共库
│   ├── CommonStruct.h     # 公共结构体
│   ├── CommonEnum.h       # 枚举定义
│   ├── CommonConst.h      # 常量
│   ├── AutoFbb.h          # FlatBuffers 自动生成
│   ├── DBAgentSrvClient.cpp  # DBAgent 客户端
│   └── Socket/            # Socket 通信
├── GameServer/            # 游戏核心服务器
│   ├── GameServer.cpp     # 入口
│   ├── Actor.cpp          # 角色
│   ├── ActorManager.cpp    # 角色管理
│   ├── PacketHandler/     # 消息处理
│   ├── Skill/             # 技能系统
│   └── Buff/              # Buff 系统
├── WorldServer/           # 世界服务器
├── GlobalServer/          # 全局服务器
├── CommunityServer/       # 社区服务器
├── DBAGent/              # 数据库代理
└── Bin/
    └── config/            # 配置文件
```

### 全局单例模式

```cpp
// C++ 全局单例 (类似 Java 的 static 单例)
// 来源: CommonLib/CommonDefined.h

#define USER_MGR UserManager::GetInstance()
#define ACTOR_MGR ActorManager::GetInstance()
#define PACKET_MGR T3PacketManager::GetInstance()
#define DATA_TABLE_MGR DataManager::GetInstance()

// 使用
ACTOR_MGR->initialize();
ACTOR_MGR->CreateActor(actorId);
```

---

## 3. IOCP 模型

### IOCP 是什么

IOCP (Input/Output Completion Port) = Windows 高性能异步 I/O 模型

```
                        ┌─────────────┐
客户端1 ──Socket──►│             │
客户端2 ──Socket──►│   IOCP      │──► 工作线程池
客户端3 ──Socket──►│   (端口)    │──► 处理函数
    ...             │             │
                    └─────────────┘

特点:
- 少量线程处理大量连接
- 内核级事件通知
- 高性能网络服务器首选
```

### 与 Java NIO 对比

| 概念 | Java NIO | C++ IOCP |
|------|----------|----------|
| 通道 | `SocketChannel` | `SOCKET` (句柄) |
| 选择器 | `Selector` | `IOCP Port` |
| 事件 | `SelectionKey` | `OVERLAPPED` + `GetQueuedCompletionStatus` |
| 缓冲区 | `ByteBuffer` | `WSABUF` + 自定义缓冲池 |

### 本项目的 IOCP 使用

```cpp
// 来源: GameServer/GameServer.cpp

// 初始化 IOCP
if (!USER_IOCP->initializeOnlyIOCP(MY_SRV_THREAD_CNT->iocp))  // 4 个线程
{
    LOG_ERROR << "IOCP Manager Init failed!";
    return false;
}

// 创建工作线程
if (!USER_IOCP->threadCreate(SESSION_MGR))
{
    LOG_ERROR << "IOCP Manager Create Thread failed!";
    return false;
}

// 启动监听
if (!USER_IOCP->startService(MY_SRV_INFO->port))  // 8300 端口
{
    LOG_ERROR << "IOCP Manager startservice failed!";
    return false;
}
```

---

## 4. 启动流程

### GameServer.cpp 入口

```cpp
// 来源: GameServer/GameServer.cpp

bool startingService()
{
    LOG_INFO << "Server Service Start !!!";

    // 1. 初始化数据表
    if (!DATA_TABLE_MGR->Initialize())
        return false;

    // 2. 初始化发送管理器
    SEND_MGR->Initialize();

    // 3. 初始化地形
    if (!TERRAIN_MGR->initTerrains())
        return false;

    // 4. 初始化对象池 (内存池，减少 new/delete)
    BUFF_POOL->initialize();
    CHARACTER_POOL->initialize();
    SKILL_POOL->initialize();
    ITEM_OBJECT_POOL->initialize();
    // ... 更多池

    // 5. 初始化数据包管理器
    if (!PACKET_MGR->initialize(MY_SRV_CONFIG->packetPoolSize))
        return false;

    // 6. 初始化会话管理器
    SESSION_MGR->initialize(poolSize, bufferSize);
    USER_MGR->initialize(poolSize);

    // 7. 初始化 IOCP
    USER_IOCP->initializeOnlyIOCP(threadCnt);
    USER_IOCP->startService(port);  // 8300

    // 8. 连接 DBAgent
    DBAGENT_SRV_MGR->initialize();

    // 9. 连接 Redis
    REDIS_THREAD_MGR->Initialize();

    // 10. 初始化 Actor 管理器
    ACTOR_MGR->initialize();

    // 11. 生成怪物
    TERRAIN_MGR->allSpawnMonster();

    return true;
}
```

### 内存池初始化

```cpp
// C++ 对象池 - 避免频繁 new/delete，减少内存碎片
// 来源: GameServer/GameServer.cpp

BUFF_POOL->initialize();
CHARACTER_POOL->initialize();
SKILL_POOL->initialize();
ITEM_OBJECT_POOL->initialize();
EQUIP_ITEM_OBJECT_POOL->initialize();

// 对比 Java:
// Java 的对象池通常用 ThreadLocal 或 Apache Commons Pool
// C++ 需要自己实现或用 Boost.Pool
```

---

## 5. Packet 处理

### 包结构

```
┌─────────────────────────────────────────────────────┐
│ Header (8 bytes)                                    │
├──────────┬──────────┬──────────┬────────────────────┤
│ Size(2B) │ PID(2B)  │ Checksum │ Options            │
├──────────┴──────────┴──────────┴────────────────────┤
│ Payload (Size - 8 bytes)                            │
│ FlatBuffers Binary Data                              │
└─────────────────────────────────────────────────────┘
```

### PacketManager 处理

```cpp
// 来源: CommonLib/T3PacketManager.h
class T3PacketManager {
public:
    bool initialize(int poolSize);
    void release();

    // 处理收到的数据包
    void onRecvPacket(Session& session, byte* buf, int size);

    // 发送数据包
    void sendPacket(Session& session, uint16_t pid, FlatBuffersBuilder& fbb);

    // 获取包对象池
    PacketPool* getPool() { return m_packetPool; }

private:
    PacketPool* m_packetPool;
    std::unordered_map<uint16_t, PacketHandler> m_handlers;
};
```

### 消息处理器注册

```cpp
// C++ 函数指针注册 (类似 Java 的 Handler Mapping)
// 来源: GameServer/PacketHandler/HandlerRegister.cpp

void RegisterAllHandlers()
{
    // 玩家相关
    REGISTER_HANDLER(C2G_LOGIN, HandleLogin);
    REGISTER_HANDLER(C2G_LOGOUT, HandleLogout);
    REGISTER_HANDLER(C2G_CREATE_CHAR, HandleCreateChar);

    // 战斗相关
    REGISTER_HANDLER(C2G_MOVE, HandleMove);
    REGISTER_HANDLER(C2G_ATTACK, HandleAttack);
    REGISTER_HANDLER(C2G_USE_SKILL, HandleUseSkill);

    // 物品相关
    REGISTER_HANDLER(C2G_USE_ITEM, HandleUseItem);
    REGISTER_HANDLER(C2G_EQUIP_ITEM, HandleEquipItem);
}
```

### Handler 示例

```cpp
// C++ 消息处理函数
// 来源: GameServer/PacketHandler/Handler_Plyaer.cpp

void HandleMove(Session& session, Packet* packet)
{
    // 解析 FlatBuffers
    auto moveReq = C2G_Move.GetRootAsMove(packet->payload());

    uint64_t actorId = session.GetActorId();
    Vector3 pos = { moveReq->x(), moveReq->y(), moveReq->z() };
    float dir = moveReq->dir();

    // 验证移动
    if (!ACTOR_MGR->CanMove(actorId, pos))
        return;

    // 更新位置
    ACTOR_MGR->SetPosition(actorId, pos);

    // 广播给周围玩家
    SendToVisiblePlayers(actorId, G2C_Move(actorId, pos, dir));
}
```

---

## 6. C++ 到 C# 过渡建议

### 学习路线

```
1. C++ 基础 (1-2天)
   - 指针、引用、智能指针
   - STL 容器
   - 类和继承
   - 模板基础

2. 本项目 C++ 模式 (3-5天)
   - 全局单例 (#define)
   - 对象池
   - IOCP 网络模型
   - Packet 处理流程

3. 对比理解 (持续)
   - C++ Actor ↔ C# Actor (如果Asp有)
   - C++ IOCP ↔ C# Socket/SignalR
```

### 本项目 C++ 关键文件

| 文件 | 作用 |
|------|------|
| `GameServer.cpp` | 入口、初始化 |
| `CommonLib/CommonStruct.h` | 公共结构体 |
| `CommonLib/T3PacketManager.h` | 包管理 |
| `CommonLib/UserIOCP.h` | 网络 I/O |
| `CommonLib/SessionMgr.h` | 会话管理 |
| `GameServer/Actor.cpp` | 角色核心 |
| `GameServer/ActorManager.cpp` | 角色管理 |

### 最小阅读量

对于 Java 开发者，建议先掌握：
1. C++ 语法差异（指针、引用、内存管理）
2. 本项目的全局单例模式
3. Packet 的收发流程
4. Actor/Skill/Item 的基本概念

深入细节可以后续按需学习。

---

下一章: [第六阶段：GameServer 核心模块](./06_gameserver核心模块.md)
