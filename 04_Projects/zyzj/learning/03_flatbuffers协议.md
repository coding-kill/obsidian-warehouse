# 第三阶段：HTTP + FlatBuffers 协议

> 理解本项目的二进制通信协议和数据序列化

---

## 目录

1. [为什么用 FlatBuffers](#1-为什么用-flatbuffers)
2. [FlatBuffers 结构](#2-flatbuffers-结构)
3. [Packet ID 枚举](#3-packet-id-枚举)
4. [FlatBuffersInputFormatter](#4-flatbuffersinputformatter)
5. [序列化/反序列化流程](#5-序列化反序列化流程)
6. [消息流程图](#6-消息流程图)

---

## 1. 为什么用 FlatBuffers

| 特性 | JSON | FlatBuffers | Protobuf |
|------|------|-------------|----------|
| 解析速度 | 慢 | **快** | 快 |
| 内存占用 | 大 | **小** | 小 |
| 格式 | 文本 | **二进制** | 二进制 |
| 随机访问 | 需要解析全部 | **直接访问字段** | 需要解析 |
| 用途 | 通用 | **游戏服务器** | 通用 |

**游戏服务器选 FlatBuffers 的原因：**
- 低延迟：无需解析整个消息体
- 高效：减少 GC 压力（直接操作 ByteBuffer）
- 跨语言：C++ 服务端和 Unity 客户端可以共用 schema

---

## 2. FlatBuffers 结构

### Schema 定义 (`.fbs`)

```flatbuffers
// 本项目的 .fbs 定义位于 CommonLib/Generated/
namespace GEM.NetPacket.C2LS;

table sNetSend_Token {
    UserId: string;
    Authkey: string;
}

table sNetRecv_Token {
    Result: int;
    Token: [ubyte];
}

root_type sNetSend_Token;
```

### 生成的 C# 代码

```csharp
// 来源: Generated/FlatBuffersInputFormatter.Login.Generated.cs
namespace GEM.NetPacket.C2LS
{
    // FlatBuffers table 结构
    public partial struct sNetSend_Token : IFlatbufferObject
    {
        private Table __p;
        public ByteBuffer ByteBuffer => __p.bb;

        // 字段访问 - 惰性解析，不需要完整解析
        public string UserId => __p.__offset(4) != 0
            ? __p.__string(__p.__offset(4) + __p.bb_pos)
            : null;

        public string Authkey => __p.__offset(6) != 0
            ? __p.__string(__p.__offset(6) + __p.bb_pos)
            : null;
    }

    // T 类型 - 用于业务逻辑的普通类
    public partial class sNetSend_TokenT
    {
        public string UserId { get; set; }
        public string Authkey { get; set; }
    }
}
```

### FlatBuffers vs 普通类

```csharp
// 普通 JSON 类 (需要完整解析)
public class User {
    public string Name { get; set; }  // 解析时全部读取
    public int Age { get; set; }
}

// FlatBuffers table (惰性解析)
sNetSend_Token token = GetRootAsNetSend_Token(buffer);
// 只访问 Name 时，不解析 Age
string name = token.UserId;  // 按需读取，不触发 Age 解析
```

---

## 3. Packet ID 枚举

### 协议命令空间

```csharp
// 来源: GEM.NetPacket 命名空间
namespace GEM.NetPacket.C2LS  // Client to Login Server
namespace GEM.NetPacket.L2C   // Login Server to Client
namespace GEM.NetPacket.C2G   // Client to Game Server
namespace GEM.NetPacket.G2C   // Game Server to Client
```

### 常用协议

```csharp
// C2LS (Client → LoginServer)
public enum eNetSendPID
{
    NONE = 0,
    Token = 1,
    Register = 2,
    Login = 3,
    Leave = 4,
    // ...
}

// L2C (LoginServer → Client)
public enum eNetRecvPID
{
    NONE = 0,
    Token = 1,
    Register = 2,
    Login = 3,
    Leave = 4,
    // ...
}
```

---

## 4. FlatBuffersInputFormatter

### HTTP 输入格式化器

```csharp
// 来源: LoginServer/Formatters/FlatBuffersInputFormatter.Login.Generated.cs
public partial class FlatBuffersInputFormatter
{
    // 对象池 - 避免频繁分配内存
    readonly ObjectPool<ByteBuffer> _bytebufferPool = ObjectPool.Create<ByteBuffer>();
    readonly ObjectPool<Verifier> _verifierPool = ObjectPool.Create<Verifier>();

    // 根据 Packet ID 反序列化
    public object OnStub(in string pID, in byte[] bytes)
    {
        var bb = _bytebufferPool.Get();  // 从池获取
        bb.SetBuffer(bytes);

        var verifier = _verifierPool.Get();
        verifier.Buf = bb;
        verifier.depth = 0;
        verifier.numTables = 0;

        object obj = null;
        switch (pID)
        {
            case "Token":
                // 验证 + 反序列化
                obj = sNetSend_TokenVerify.Verify(verifier)
                    ? sNetSend_Token.GetRootAssNetSend_Token(bb).UnPack()
                    : null;
                break;
            case "Login":
                obj = sNetSend_LoginVerify.Verify(verifier)
                    ? sNetSend_Login.GetRootAssNetSend_Login(bb).UnPack()
                    : null;
                break;
            // ... 其他协议
        }

        // 归还池
        _verifierPool.Return(verifier);
        _bytebufferPool.Return(bb);

        return obj;
    }
}
```

### 注册到 MVC

```csharp
// 来源: LoginServer/Program.cs
builder.Services.AddControllers(options =>
{
    // 清除默认的 JSON 格式化器
    options.InputFormatters.Clear();
    options.OutputFormatters.Clear();

    // 添加 FlatBuffers 格式化器
    options.InputFormatters.Add(new FlatBuffersInputFormatter());
    options.OutputFormatters.Add(new FlatBuffersOutputFormatter());
});
```

---

## 5. 序列化/反序列化流程

### 完整请求流程

```
客户端 (Unity)
    │
    │  FlatBuffers 二进制数据
    ▼
HTTP POST /Token
Content-Type: application/octet-stream
    │
    │  字节流
    ▼
Kestrel HTTP Server
    │
    ▼
FlatBuffersInputFormatter.OnStub()
    │
    ├─→ 从 ByteBuffer 读取
    ├─→ Verifier.Verify() 验证数据
    └─→ sNetSend_Token.UnPack() → sNetSend_TokenT
    │
    ▼
LoginController.Token(sNetSend_TokenT req)
    │
    │  业务逻辑
    ▼
sNetRecv_TokenT ret = new() { Result = SUCCESS, Token = tokenBytes }
    │
    │  FlatBuffers 二进制
    ▼
FlatBuffersOutputFormatter
    │
    ▼
HTTP 响应 (二进制)
    │
    ▼
客户端解析
```

### 手动序列化示例

```csharp
// 创建 FlatBuffers 消息
public sNetRecv_TokenT CreateTokenResponse(ulong userId, int randomNo, int expireTime)
{
    // 生成 Token
    var tokenBytes = SimpleToken.Serialize(userId, randomNo, expireTime);

    return new sNetRecv_TokenT
    {
        Result = eNetResultType.SUCCESS,
        Token = tokenBytes
    };
}

// 发送到客户端
return ret;  // FlatBuffersOutputFormatter 自动序列化
```

---

## 6. 消息流程图

### 登录完整流程

```
客户端                              LoginServer                    Redis                MySQL
   │                                   │                            │                    │
   │─── POST /Token ──────────────────►│                            │                    │
   │   {UserId, Authkey}              │                            │                    │
   │                                   │──GetUserUidByAccountAsync─►│                    │
   │                                   │◄───────── userUid ─────────│                    │
   │                                   │                             │                    │
   │                                   │───── UserUid(userId) ──────────────────────────►│
   │                                   │◄────── userUid ────────────────────────────────│
   │                                   │                             │                    │
   │                                   │───── SimpleToken.Serialize ──►                │
   │                                   │◄──── token bytes ──────────│                    │
   │                                   │                             │                    │
   │◄──── Token Response ─────────────│                             │                    │
   │   {Result, Token:bytes[16]}      │                             │                    │
   │                                   │                             │                    │
   │                                   │                             │                    │
   │─── POST /Login ──────────────────►│                            │                    │
   │   {Token, BuildVer, ProtocolVer}  │                            │                    │
   │                                   │──ValidCheck(token)──────────►│                    │
   │                                   │◄── userUid ─────────────────│                    │
   │                                   │                             │                    │
   │                                   │───── PostLogin ───────────────────────────────►│
   │                                   │◄──── WorldServerList ─────────────────────────│
   │                                   │                             │                    │
   │◄──── Login Response ─────────────│                             │                    │
   │   {Result, AuthToken,             │                             │                    │
   │    WorldServerList}              │                             │                    │
```

### FlatBuffers Schema 示例

```flatbuffers
// C2LS: Client to LoginServer
namespace GEM.NetPacket.C2LS;

table sNetSend_Token {
    UserId: string;
    Authkey: string;
}

table sNetSend_Login {
    Token: [ubyte];           // 16字节 SimpleToken
    ProtoclVer: int;
    BuildVer: int;
    UserPlatformType: byte;
    UserIp: string;
    UserType: byte;          // eNetSNSAccessType
    OsType: byte;             // eNetClientOsType
}

table sNetSend_TestLogin {
    UserId: string;
    Pwd: string;
    DeviceUid: string;
}
```

---

## 练习建议

1. 查看 `CommonLib/GEM/NetPacket/` 下的 FlatBuffers schema 文件
2. 对比 `FlatBuffersInputFormatter.Login.Generated.cs` 和 `FlatBuffersInputFormatter` 基类
3. 理解 ObjectPool 的作用：减少 GC 压力

---

下一章: [第四阶段：SignalR 实时通信（ChatServer）](./04_signalr实时通信.md)
