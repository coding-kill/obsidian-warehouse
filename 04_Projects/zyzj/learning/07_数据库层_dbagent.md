# 第七阶段：数据库层 + DBAgent

> 理解 MySQL、Redis 架构和 DBAgent 代理模式

---

## 目录

1. [整体架构](#1-整体架构)
2. [MySQL 表结构](#2-mysql-表结构)
3. [Redis 使用场景](#3-redis-使用场景)
4. [DBAgent 代理模式](#4-dbagent-代理模式)
5. [数据访问流程](#5-数据访问流程)

---

## 1. 整体架构

### 分层结构

```
┌─────────────────────────────────────────────────────────────┐
│                      客户端 (Unity)                         │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              Asp 服务器 (C# HTTP/SignalR)                   │
│  LoginServer │ ChatServer │ BillingServer │ ArenaServer    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              TCPServer (C++ 游戏核心)                       │
│  GameServer │ WorldServer │ GlobalServer │ CommunityServer │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      DBAgent (C++)                          │
│              所有数据库访问的统一入口                        │
└─────────────────────────────────────────────────────────────┘
                    │                    │
                    ▼                    ▼
┌─────────────────────────┐  ┌─────────────────────────────┐
│         MySQL           │  │           Redis             │
│  t_account │ t_character │  │  会话缓存 │ 排行榜 │ 聊天  │
└─────────────────────────┘  └─────────────────────────────┘
```

### 为什么不直接访问数据库

```
直接访问 (游戏服务器直连 DB):
  GameServer ──► MySQL
  问题:
  - 每个 GameServer 都要管理连接池
  - 数据库连接数有限，高并发时不够用
  - 游戏逻辑和 DB 耦合

代理模式 (通过 DBAgent):
  GameServer ──► DBAgent ──► MySQL
  优势:
  - 统一连接管理
  - DBAgent 可以做 SQL 过滤/审计
  - 更容易做读写分离/分库分表
```

---

## 2. MySQL 表结构

### 账户表 (t_account)

```sql
CREATE TABLE `t_account` (
    `uid` BIGINT(20) UNSIGNED NOT NULL AUTO_INCREMENT,
    `permission_level` SMALLINT(5) UNSIGNED DEFAULT '3',  -- 权限等级
    `id` VARCHAR(32) NOT NULL,                            -- 账号
    `sns_id` VARCHAR(124) NOT NULL,                      -- SNS ID
    `pwd` VARCHAR(32) NOT NULL,                          -- 密码
    `akey` VARCHAR(32) NOT NULL,                         -- 认证 Key
    `user_type` SMALLINT(5) DEFAULT '1',                  -- 用户类型 (Guest/Google/Apple...)
    `token` VARCHAR(500) DEFAULT '',                     -- Token
    `state` SMALLINT(5) DEFAULT '0',                     -- 状态
    `account_flag` TINYINT(3) DEFAULT '0',                -- 0:正常, 99:退出中
    `last_login_time` DATETIME,                          -- 最后登录时间
    `last_lgout_time` DATETIME,                          -- 最后登出时间
    `create_time` DATETIME DEFAULT CURRENT_TIMESTAMP,     -- 创建时间
    `delete_time` DATETIME,                              -- 删除时间
    `platform_type` TINYINT(3),                          -- 平台类型
    `autoLoginToken1` VARCHAR(255),                       -- 自动登录 Token1
    `autoLoginToken2` VARCHAR(255),                       -- 自动登录 Token2
    `refreshToken` VARCHAR(255),                          -- 刷新 Token
    PRIMARY KEY (`uid`),
    UNIQUE INDEX `userid` (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=100000;
```

### 角色表 (t_character)

```sql
CREATE TABLE `t_character` (
    `char_uid` BIGINT(20) UNSIGNED NOT NULL DEFAULT '0',  -- 角色 UID
    `user_uid` BIGINT(20) UNSIGNED NOT NULL DEFAULT '0', -- 账户 UID
    `char_idx` BIGINT(19) NOT NULL DEFAULT '0',          -- 角色槽位
    `costume_view_flags` BIGINT DEFAULT '0',              -- 时装外观
    `battle_power` INT DEFAULT '0',                        -- 战斗力
    `tier` SMALLINT DEFAULT '1',                          -- 段位
    `exp` BIGINT DEFAULT '0',                             -- 经验值
    `level` SMALLINT UNSIGNED DEFAULT '0',                 -- 等级
    `hp` BIGINT DEFAULT '0',                              -- 生命值
    `sp` INT DEFAULT '0',                                  -- 魔法值
    `createtime` DATETIME DEFAULT CURRENT_TIMESTAMP,
    `updatetime` DATETIME ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (`char_uid`),
    INDEX `index_user_uid` (`user_uid`)
) ENGINE=InnoDB;
```

### 存储过程

```sql
-- 登录存储过程
DELIMITER //
CREATE PROCEDURE s_login(
    IN _id VARCHAR(32),
    IN _pwd VARCHAR(32),
    IN _akey VARCHAR(32),
    IN _device_uid VARCHAR(64)
)
BEGIN
    -- 返回多个结果集:
    -- 1. 结果码 + 用户信息
    -- 2. 封禁信息
    -- 3. 维护公告
END //
DELIMITER ;
```

---

## 3. Redis 使用场景

### Redis 数据库规划

```json
{
  "redis_list": [
    { "db_name": "RDS_CACHE",    "type": 0, "ip": "127.0.0.1", "port": 6003 },
    { "db_name": "RDS_RANKING", "type": 1, "ip": "127.0.0.1", "port": 6003 },
    { "db_name": "RDS_GLOBAL",  "type": 2, "ip": "127.0.0.1", "port": 6003 },
    { "db_name": "RDS_WORLD_Q", "type": 3, "ip": "127.0.0.1", "port": 6003 }
  ]
}
```

### 各数据库用途

| DB | 用途 | Key 示例 |
|----|------|---------|
| `RDS_CACHE` | 会话缓存、聊天黑名单 | `user:{uid}:token`, `block:{uid}:list` |
| `RDS_RANKING` | 排行榜 | `ranking:arena`, `ranking:power` |
| `RDS_GLOBAL` | 全局数据 | `worlds`, `global_config` |
| `RDS_WORLD_Q` | 世界排队 | `queue:{worldId}` |

### 本项目 Redis 操作示例

```csharp
// 来源: LoginServer/DBProc/Redis.Account.cs

// 获取用户 UID
public async Task<ulong> GetUserUidByAccountAsync(string accountId)
{
    var key = $"account:{accountId}";
    var value = await _redis.StringGetAsync(key);

    if (value.IsNull)
        return 0;

    return ulong.Parse(value);
}

// 设置用户 UID
public async Task SetUserUidByAccountAsync(string accountId, ulong userUid)
{
    var key = $"account:{accountId}";
    await _redis.StringSetAsync(key, userUid.ToString(), TimeSpan.FromDays(30));
}

// Token 验证
public async Task<bool> ValidateTokenAsync(ulong userUid, byte[] token)
{
    var key = $"token:{userUid}";
    var storedToken = await _redis.StringGetAsync(key);

    return storedToken == token;
}
```

---

## 4. DBAgent 代理模式

### DBAgent 职责

```
GameServer                          DBAgent                          MySQL
    │                                  │                               │
    │──── 加载角色请求 ─────────────────►│                               │
    │                                  │──── SQL 查询 ─────────────────►│
    │                                  │◄─── 结果 ─────────────────────│
    │                                  │                               │
    │◄─── 角色数据 ─────────────────────│                               │
    │                                  │                               │
    │──── 保存背包请求 ─────────────────►│                               │
    │                                  │──── SQL 更新 ─────────────────►│
    │                                  │                               │
    │◄─── 保存成功 ─────────────────────│                               │
```

### DBAgent 架构

```
DBAgentServer
│
├── ClientSessionMgr     # 管理与 GameServer 的连接
├── DBThreadMgr         # 数据库工作线程池
│   └── DBThread[N]     # 多个工作线程
└── DBProc_*           # 各模块的 DB 处理
    ├── DBProc_Handler_Account.cpp
    ├── DBProc_Handler_Game.cpp
    ├── DBProc_Handler_Guild.cpp
    └── ...
```

### DBAgent 启动流程

```cpp
// 来源: DBAGent/DBAgentServer.cpp

bool startingService()
{
    // 1. 初始化数据包管理器
    PACKET_MGR->initialize(poolSize);

    // 2. 初始化 IOCP (接收 GameServer 连接)
    CLIENT_IOCP->initializeOnlyIOCP(threadCnt);
    CLIENT_IOCP->startService(port);  // 8210

    // 3. 初始化数据库线程
    DB_THREAD_MGR->Initialize();

    return true;
}
```

### 命令注册

```cpp
// 来源: DBAGent/DBProc_RegRecvPID.cpp

void RegisterDBHandlers()
{
    // 账户相关
    REGISTER_DB_HANDLER(G2D_ACCOUNT_LOGIN, HandleAccountLogin);
    REGISTER_DB_HANDLER(G2D_ACCOUNT_CREATE, HandleAccountCreate);

    // 角色相关
    REGISTER_DB_HANDLER(G2D_CHAR_LOAD, HandleCharLoad);
    REGISTER_DB_HANDLER(G2D_CHAR_SAVE, HandleCharSave);
    REGISTER_DB_HANDLER(G2D_CHAR_CREATE, HandleCharCreate);
    REGISTER_DB_HANDLER(G2D_CHAR_DELETE, HandleCharDelete);

    // 物品相关
    REGISTER_DB_HANDLER(G2D_ITEM_SAVE, HandleItemSave);
    REGISTER_DB_HANDLER(G2D_ITEM_UPDATE, HandleItemUpdate);
}
```

---

## 5. 数据访问流程

### 完整登录流程

```
客户端              LoginServer              DBAgent              MySQL
   │                    │                      │                    │
   │──POST /Login──────►│                      │                    │
   │                    │                      │                    │
   │                    │──查询账户────────────►│                    │
   │                    │                      │──SELECT───────────►│
   │                    │                      │◄──结果─────────────│
   │                    │                      │                    │
   │                    │◄──用户信息────────────│                    │
   │                    │                      │                    │
   │◄─登录成功──────────│                      │                    │
   │   (含世界列表)     │                      │                    │
   │                    │                      │                    │
   │                    │                      │                    │
   │──连接GameServer────►│                      │                    │
   │                    │                      │                    │
   │                    │◄──转发连接────────────│                    │
   │                    │                      │                    │
   │                    │                      │                    │
   │──G2D_CHAR_LOAD────►│                      │                    │
   │                    │────加载角色请求──────►│                    │
   │                    │                      │──SELECT───────────►│
   │                    │                      │◄──角色数据─────────│
   │                    │                      │                    │
   │◄─D2G_CHAR_LOAD────│◄──角色数据───────────│                    │
```

### 数据库分库分表

```cpp
// 分库策略：基于 userUid 的哈希
// 来源: CommonLib/DataBase/MySqlProc.cs

#if _DB_SHARDING_
public partial class AccountDBProc : MySqlProc
{
    // 根据 userUid 确定数据库
    public int GetDbId(ulong userUid)
    {
        return (int)((userUid + 1) % (ulong)_dicConnString.Count());
    }

    // 获取连接
    protected override MySqlConnection GetConnection(int worldId = 0, int dbId = 0)
    {
        var key = new KeyValuePair<int, int>(worldId, dbId);
        if (false == _dicConnString.ContainsKey(key))
            return null;
        return new MySqlConnection(_dicConnString[key]);
    }
}
#endif
```

### 对象池与内存管理

```cpp
// C++ 对象池 - 减少 new/delete 开销
// 来源: CommonLib/ObjectPool.h

template<typename T>
class ObjectPool {
public:
    void initialize(int initialSize)
    {
        for (int i = 0; i < initialSize; ++i)
            _pool.push(new T());
    }

    T* allocate()
    {
        if (_pool.empty())
            return new T();  // 扩容
        T* obj = _pool.top();
        _pool.pop();
        return obj;
    }

    void deallocate(T* obj)
    {
        _pool.push(obj);  // 回收
    }

private:
    std::stack<T*> _pool;
};

// 使用
ObjectPool<Character>* CHARACTER_POOL;

Character* char = CHARACTER_POOL->allocate();
char->initialize(info);
CHARACTER_POOL->deallocate(char);  // 放回池，不释放内存
```

---

## 快速参考

| 组件 | 职责 | 关键文件 |
|------|------|---------|
| DBAgent | 数据库统一入口 | `DBAGent/*.cpp` |
| AccountDBProc | 账户 CRUD | `LoginServer/DBProc/AccountDBProc.cs` |
| Redis | 缓存/会话 | `LoginServer/DBProc/Redis.*.cs` |
| 分库分表 | 水平扩展 | `#if _DB_SHARDING_` |

---

## 学习建议

1. 从 `DBAgentServer.cpp` 的 `startingService()` 理解启动流程
2. 看 `DBProc_Handler_Account.cpp` 理解 SQL 生成
3. 对比 C++ DBAgent 和 C# AccountDBProc 的实现

---

## 学习路径总结

```
✅ 第一阶段：C# 语法速成（Java 对照）
✅ 第二阶段：Asp 服务器入口（LoginServer）
✅ 第三阶段：HTTP + FlatBuffers 协议
✅ 第四阶段：SignalR 实时通信（ChatServer）
✅ 第五阶段：TCPServer (C++) 概览
✅ 第六阶段：GameServer 核心模块
✅ 第七阶段：数据库层 + DBAgent
```

所有学习文档已完成！建议按顺序阅读，也可以根据需要选择性深入某个模块。
