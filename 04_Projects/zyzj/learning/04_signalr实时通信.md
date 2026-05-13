# 第四阶段：SignalR 实时通信（ChatServer）

> 理解 WebSocket 实时通信和 SignalR Hub 设计

---

## 目录

1. [SignalR 简介](#1-signalr-简介)
2. [Hub 生命周期](#2-hub-生命周期)
3. [并发用户管理](#3-并发用户管理)
4. [聊天频道设计](#4-聊天频道设计)
5. [消息广播](#5-消息广播)
6. [完整流程图](#6-完整流程图)

---

## 1. SignalR 简介

### WebSocket vs HTTP

| 特性 | HTTP | WebSocket (SignalR) |
|------|------|---------------------|
| 连接方向 | 请求-响应 | **双向** |
| 服务器推送 | 需要轮询 | **原生支持** |
| 适用场景 | REST API | **实时聊天、通知** |
| 性能 | 每次新建连接 | 复用连接 |

### SignalR 核心概念

```
客户端 (Unity)
    │
    │  SignalR 连接 (WebSocket)
    ▼
ChatHub
    │
    ├─→ Hub 方法调用 (客户端 → 服务器)
    │       Clients.Caller.SendAsync()   // 返回给调用者
    │       Clients.Group().SendAsync()  // 发送到组
    │
    └─→ 服务器推送 (服务器 → 客户端)
            Clients.All.SendAsync()      // 推送给所有人
```

---

## 2. Hub 生命周期

### Hub 定义

```csharp
// 来源: ChatServer/Hub/ChatHub.cs
public partial class ChatHub : Hub
{
    // 连接建立时
    public override async Task OnConnectedAsync()
    {
        await base.OnConnectedAsync();
        if (0 < App.ShowDebugLevel)
            log.Debug($"OnConnectedAsync ConnectionId:{Context.ConnectionId}");
    }

    // 连接断开时
    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        // 清理用户数据
        App.UsersByConnId.TryRemove(Context.ConnectionId, out var userInfo);
        if (userInfo != null)
        {
            App.Users.TryRemove(userInfo.UserUid, out _);

            // 清理各频道数据
            if (_dicWorld.TryRemove(userInfo.UserUid, out _)) { ... }
            if (_dicGuild.TryRemove(userInfo.UserUid, out _)) { ... }
            if (_dicParty.TryRemove(userInfo.UserUid, out _)) { ... }
        }

        await base.OnDisconnectedAsync(exception);
    }
}
```

### Hub 方法 (客户端调用)

```csharp
public class ChatHub : Hub
{
    // 认证方法
    public async Task AuthWithBlockList(byte[] token, int worldId, string familyName)
    {
        // 验证 Token
        eNetResultType resultType = ValidCheck(token, out var userUid);
        if (resultType != eNetResultType.SUCCESS)
        {
            await Clients.Caller.SendAsync(AUTH_WITH_BLOCKLIST, (int)resultType, "");
            return;
        }

        // 创建用户信息
        var userInfo = new UserInfo(userUid, familyName, 0, worldId, ...);
        App.Users.TryAdd(userUid, userInfo);
        App.UsersByConnId.TryAdd(Context.ConnectionId, userInfo);

        // 返回成功
        await Clients.Caller.SendAsync(AUTH_WITH_BLOCKLIST, (int)resultType, blockListJson);
    }

    // 进入世界频道
    public async Task EnterWorld()
    {
        // 检查用户
        if (false == App.UsersByConnId.TryGetValue(Context.ConnectionId, out var userInfo))
        {
            await Clients.Caller.SendAsync(ENTER_WORLD, (int)eNetResultType.NOT_FOUND_USER_DATA);
            return;
        }

        // 添加到频道
        userInfo.SetWorldKey(userInfo.WorldId, channel);
        _dicWorld.TryAdd(userInfo.UserUid, userInfo);
        await Groups.AddToGroupAsync(Context.ConnectionId, userInfo.WorldKey);

        // 响应客户端
        await Clients.Caller.SendAsync(ENTER_WORLD, (int)eNetResultType.SUCCESS, channel);
    }
}
```

---

## 3. 并发用户管理

### ConcurrentDictionary

```csharp
// 线程安全的字典，用于管理在线用户

// 用户表: ConnectionId → UserInfo
static ConcurrentDictionary<string, UserInfo> UsersByConnId { get; set; }

// 用户表: UserUid → UserInfo
static ConcurrentDictionary<ulong, UserInfo> Users { get; set; }

// 频道计数: ChannelKey → 在线人数
static ConcurrentDictionary<string, ChannelCount> DicChannelCount { get; set; }
```

### UserInfo 模型

```csharp
// 来源: ChatServer/Models/UserInfo.cs
public class UserInfo
{
    public ulong UserUid { get; private set; }
    public string FamilyName { get; private set; }
    public string ConnectionId { get; private set; }  // SignalR 连接 ID
    public int WorldId { get; private set; }
    public eNetPermissionLevelType PermissionLevel { get; private set; }

    // 频道 Key
    public string WorldKey { get; private set; }
    public string GuildKey { get; private set; }
    public string PartyKey { get; private set; }
    public string TerrainKey { get; private set; }
    public string FactionKey { get; private set; }

    // 设置连接信息
    public void SetConnectionId(string connectionId)
    {
        ConnectionId = connectionId;
    }

    // 设置世界频道 Key
    public void SetWorldKey(int worldId, int channel = 1)
    {
        WorldKey = $"W{worldId}_C{channel}";  // 如 "W1_C1"
    }
}
```

### 线程安全操作

```csharp
// 添加用户 (线程安全)
App.Users.TryAdd(userUid, userInfo);
App.UsersByConnId.TryAdd(Context.ConnectionId, userInfo);

// 删除用户 (线程安全)
App.Users.TryRemove(userUid, out _);
App.UsersByConnId.TryRemove(Context.ConnectionId, out _);

// 查询用户
if (App.Users.TryGetValue(userUid, out var userInfo)) { ... }

// ConcurrentDictionary 的 TryRemove 返回是否成功
if (_dicWorld.TryRemove(userInfo.UserUid, out var removed)) { ... }
```

---

## 4. 聊天频道设计

### 频道类型

| 频道 | Key 格式 | 说明 |
|------|----------|------|
| 世界 | `W{worldId}_C{channel}` | 全服聊天 |
| 地形 | `T{worldId}_{terrainId}` | 区域聊天 |
| 工会 | `G{guildUid}` | 工会成员聊天 |
| 队伍 | `P{partyUid}` | 队伍聊天 |
| 阵营 | `F{zoneIdx}_{type}_{factionUid}` | 阵营聊天 |
| 私聊 | (ConnectionId) | 点对点 |

### 进入/离开频道

```csharp
public async Task EnterGuild(ulong guildUid)
{
    // 检查用户
    if (false == App.UsersByConnId.TryGetValue(Context.ConnectionId, out var userInfo))
        return;

    // 设置频道 Key
    userInfo.SetGuildKey(guildUid);
    var key = userInfo.GuildKey;  // "G12345"

    // 确保频道计数存在
    if (false == DicChannelCount.ContainsKey(key))
        DicChannelCount.TryAdd(key, new ChannelCount());

    // 增加计数
    DicChannelCount[key].Increment();

    // 添加到 SignalR 组
    _dicGuild.TryAdd(userInfo.UserUid, userInfo);
    await Groups.AddToGroupAsync(Context.ConnectionId, key);

    // 通知频道内其他人
    await Clients.Group(key).SendAsync(ENTER_GUILD, (int)eNetResultType.SUCCESS,
        userInfo.UserUid, userInfo.FamilyName);
}

public async Task ExitGuild()
{
    // 获取用户
    if (false == App.UsersByConnId.TryGetValue(Context.ConnectionId, out var userInfo))
        return;

    // 移除
    _dicGuild.TryRemove(userInfo.UserUid, out _);
    var key = userInfo.GuildKey;

    // 减少计数
    DicChannelCount[key].Decrement();

    // 从组移除
    await Groups.RemoveFromGroupAsync(Context.ConnectionId, key);

    // 通知
    await Clients.Group(key).SendAsync(EXIT_GUILD, (int)eNetResultType.SUCCESS, userInfo.UserUid);

    userInfo.SetGuildKey(0);
}
```

---

## 5. 消息广播

### 发送消息到频道

```csharp
// 世界频道聊天
public async Task ChatWorld(string msg)
{
    // 验证用户
    if (false == App.UsersByConnId.TryGetValue(Context.ConnectionId, out var userInfo))
        return;

    if (string.IsNullOrEmpty(userInfo.WorldKey))
    {
        await Clients.Caller.SendAsync(CHAT_WORLD, (int)eNetResultType.NOT_ENTERED, ...);
        return;
    }

    // 广播到世界频道
    // Clients.Group = 频道内所有人
    await Clients.Group(userInfo.WorldKey).SendAsync(
        CHAT_WORLD,
        (int)eNetResultType.SUCCESS,
        userInfo.UserUid,
        userInfo.FamilyName,
        msg
    );
}

// 私聊
public async Task ChatWhisper(ulong receiverUid, string msg)
{
    // 获取接收者
    if (false == App.Users.TryGetValue(receiverUid, out var receiverInfo))
    {
        await Clients.Caller.SendAsync(CHAT_WHISPER, (int)eNetResultType.LOGOUT_STATE, ...);
        return;
    }

    // 点对点发送
    await Clients.Clients(receiverInfo.ConnectionId).SendAsync(
        CHAT_WHISPER,
        (int)eNetResultType.SUCCESS,
        userInfo.UserUid,
        userInfo.FamilyName,
        msg
    );
}
```

### Clients API

```csharp
// 发送给调用者
Clients.Caller.SendAsync(method, args);

// 发送给指定连接
Clients.Client(connectionId).SendAsync(method, args);

// 发送给多个连接
Clients.Clients(connectionIdList).SendAsync(method, args);

// 发送给组 (频道)
Clients.Group(groupName).SendAsync(method, args);

// 发送给所有人
Clients.All.SendAsync(method, args);

// 排除某人
Clients.Group(groupName).SendAsync(...);
// 然后在某客户端用 JavaScript 过滤
```

---

## 6. 完整流程图

### 聊天完整流程

```
客户端                           ChatHub                      Redis                    GameServer
   │                               │                           │                         │
   │─── Hub 连接 ──────────────────►│                           │                         │
   │                               │                           │                         │
   │─── AuthWithBlockList() ──────►│                           │                         │
   │   token, worldId, familyName  │──ValidCheck(token)───────►│                         │
   │                               │◄── userUid ───────────────│                         │
   │                               │                           │                         │
   │                               │──BlockList(userUid)───────────────────────────────►│
   │                               │◄── blockList ─────────────────────────────────────│
   │                               │                           │                         │
   │◄── AuthWithBlockList(result)─│                           │                         │
   │                               │                           │                         │
   │─── EnterWorld() ─────────────►│                           │                         │
   │                               │──TryAdd(userUid)─────────►│                         │
   │                               │──Groups.AddToGroupAsync───►│                         │
   │                               │                           │                         │
   │◄── EnterWorld(channel)───────│                           │                         │
   │                               │                           │                         │
   │─── ChatWorld(msg) ───────────►│                           │                         │
   │                               │──Clients.Group(key)───────►│                         │
   │                               │   (广播给同频道所有人)      │                         │
   │◄── ChatWorld(user, msg)───────│                           │                         │
   │                               │                           │                         │
   │─── ExitWorld() ──────────────►│                           │                         │
   │                               │──TryRemove(userUid)──────►│                         │
   │                               │──Groups.RemoveFromGroup──►│                         │
   │                               │                           │                         │
   │◄── ExitWorld(result)──────────│                           │                         │
```

### SignalR 连接建立

```
Unity 客户端
    │
    ├─→ 创建 HubConnection
    │       HubConnectionBuilder("http://chatserver/ChatHub")
    │           .WithUrl("/ChatHub")
    │           .Build();
    │
    ├─→ 连接
    │       await connection.StartAsync();
    │
    ├─→ 调用 Hub 方法
    │       await connection.InvokeAsync("AuthWithBlockList", token, worldId, name);
    │
    └─→ 接收服务器推送
        connection.On("ChatWorld", (uid, name, msg) => {
            // 处理聊天消息
        });
```

---

## 与 Java 对比

| 概念 | Java Spring | C# ASP.NET Core SignalR |
|------|-------------|-------------------------|
| WebSocket | `@EnableWebSocket` | `app.MapHub<ChatHub>()` |
| 连接处理 | `WebSocketHandler` | `Hub.OnConnectedAsync` |
| 消息发送 | `session.send()` | `Clients.Group().SendAsync()` |
| 用户管理 | 自己维护 Map | `ConcurrentDictionary` |
| 组管理 | `WebSocketSession.Group()` | `Groups.AddToGroupAsync()` |

---

## 练习建议

1. 追踪一个玩家从连接，到世界聊天，到离开的完整流程
2. 查看 `ChatServer/Hub/ChatHub.cs` 中的 `ValidCheck` 方法实现
3. 对比 SignalR 的 Group 机制和 Redis Pub/Sub

---

下一章: [第五阶段：TCPServer (C++) 概览](./05_tcpserver_cpp概览.md)
