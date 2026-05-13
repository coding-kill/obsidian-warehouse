# 第二阶段：Asp 服务器入口（LoginServer）

> 以 LoginServer 为例，深入理解 ASP.NET Core 启动流程、依赖注入、配置加载

---

## 目录

1. [整体架构](#1-整体架构)
2. [Program.cs 入口解析](#2-programcs-入口解析)
3. [依赖注入 (DI) 容器](#3-依赖注入-di-容器)
4. [配置加载体系](#4-配置加载体系)
5. [Kestrel HTTP 服务器配置](#5-kestrel-http-服务器配置)
6. [控制器 (Controller)](#6-控制器-controller)
7. [数据库层](#7-数据库层)
8. [启动流程总结](#8-启动流程总结)

---

## 1. 整体架构

```
LoginServer
│
├── Program.cs          # 入口：服务启动配置
├── Libs/
│   └── App.cs          # 静态 App 对象（全局访问点）
├── Controllers/
│   └── LoginController.cs  # HTTP 端点处理
├── DBProc/
│   └── AccountDBProc.cs    # 账户数据库操作
├── Api/
│   └── LoginApiBase.cs     # 平台 API 抽象
└── Data/
    └── TAccount.cs         # 数据模型
```

**与其他 Java 框架对比：**

| 概念 | Java Spring Boot | C# ASP.NET Core |
|------|------------------|-----------------|
| 入口 | `main()` + `@SpringBootApplication` | `main()` + `WebApplication.CreateBuilder()` |
| Bean 容器 | `ApplicationContext` | `IServiceProvider` (DI) |
| 全局访问 | `@Autowired` 注入 | 静态 `App` 类 |
| REST 控制器 | `@RestController` | `[ApiController]` + `ControllerBase` |
| 配置 | `application.yml` / `@ConfigurationProperties` | `appsettings.json` / `IOptions<T>` |

---

## 2. Program.cs 入口解析

### 完整流程概览

```csharp
// 来源: LoginServer/Program.cs (简化版)
public static void Main(string[] args)
{
    int serverId = 20;
    if (args.IsNotNullOrEmpty())
        serverId = int.Parse(args[0]);

    // 1. 日志配置
    var builder = WebApplication.CreateBuilder(args);
    builder.Logging.ClearProviders().AddLog4Net("login.log4net.config", true);

    // 2. 初始化 App 配置
    if (App.Init(serverId) == false) return;

    // 3. 服务器间认证 (发布版本)
    Authentication auth = new Authentication();
    if (auth.Authenticate(loginServer, authServer) == false) return;

    // 4. 初始化基础组件
    App.InitBase(builder.Configuration);
    App.Table.LoadPatchTables(eServerType.Login);

    // 5. 初始化数据库
    App.DB.Init(App.ServerConfig.GlobalDBs, ...);

    // 6. 配置 HTTP 服务器 (Kestrel)
    builder.Services.Configure<KestrelServerOptions>(options => {
        options.ListenAnyIP(httpPort);
        options.ListenAnyIP(httpsPort);
    });

    // 7. 注册服务
    builder.Services.AddControllers();  // 注册所有控制器
    builder.Services.AddHealthChecks(); // 健康检查

    // 8. 构建应用
    var app = builder.Build();

    // 9. 配置中间件管道
    app.UseRouting();
    app.UseAuthentication();
    app.UseAuthorization();

    // 10. 映射路由
    app.MapControllers();
    app.MapHealthChecks("/healthz");

    // 11. 启动
    app.Run();
}
```

### 预处理指令 (Conditional Compilation)

```csharp
// Java 没有等价物！这是 C/C++ 的特性

#if DEBUG              // DEBUG 模式编译
#elif SG_RELEASE       // SG 发布版本编译
#elif ID_RELEASE       // ID 发布版本编译
#else                  // 默认编译
#endif

// 本项目使用场景：
// - DEBUG: 本地开发，启用 TestLogin 等
// - SG_RELEASE / ID_RELEASE / TW_RELEASE / JP_RELEASE: 不同渠道发布
// - 平台差异化代码通过预处理指令隔离
```

---

## 3. 依赖注入 (DI) 容器

### C# 内置 DI vs Java Spring

**Java Spring:**
```java
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;
}
```

**C# ASP.NET Core:**
```csharp
// Java: @Service 注解自动注册
// C#: 需要显式注册到容器

// Program.cs 中注册
builder.Services.AddSingleton<ScheduleService>();           // 单例
builder.Services.AddScoped<IMyService, MyService>();        // 请求作用域
builder.Services.AddTransient<IHttpClientFactory, ...>();   // 每次请求新建

// 但本项目用的是静态 App 类，不是 DI
// 这是项目特殊设计，不是 ASP.NET Core 标准做法
```

### 本项目的"伪 DI"模式

```csharp
// 来源: LoginServer/Libs/App.cs
public class App : BaseApp
{
    // 静态属性 = 全局单例
    static public Define Define { get; set; } = new();
    static public LoginApiBase LoginApi { private set; get; }  // 子类赋值
    static public DBManager DB { private set; get; } = new();
    static public RedisDBProc Redis => DB.Redis;              // 延迟访问
}

// 任何地方都可以直接访问
var result = await App.DB.Account.UserUid(userId, authKey);
```

**为什么不用标准 DI？**
- 历史原因：项目早期设计
- TCPServer (C++) 没有 DI 概念，保持一致
- 游戏服务器通常用静态单例更简单直接

---

## 4. 配置加载体系

### 配置文件结构

```
Bin/config/
├── my.config              # 标记当前配置目录 (如 "dev")
├── dev/                   # 开发环境
│   ├── MyServerInfo.json  # 本服务器信息
│   ├── ServerInfo_global_Dev.json   # 全局服务器列表
│   └── ServerInfo_world_1_Dev.json # 世界1配置
└── prod/
    └── ...
```

### 配置加载流程

```csharp
// 来源: CommonLib/Config/ServerInfoManager.cs

public static ServerInfoManager Create()
{
    // 1. 读取 my.config 确定环境
    var folderName = File.ReadLines("config/my.config").First(); // "dev"

    var serverInfo = new ServerInfoManager();
    serverInfo.Build = folderName.ToUpper();

    // 2. 加载本服务器信息
    string json = File.ReadAllText($"config/{folderName}/MyServerInfo.json");
    serverInfo.FromMyServerInfo(JsonConvert.DeserializeObject<stMyServerInfo>(json));

    // 3. 加载全局配置
    json = File.ReadAllText($"config/{folderName}/ServerInfo_global.json");
    var serverList = JsonConvert.DeserializeObject<stGlobal>(json);
    serverInfo.FromGlobal(serverList);

    // 4. 加载世界配置
    foreach (var worldId in serverInfo.WorldServerIdList)
    {
        json = File.ReadAllText($"config/{folderName}/ServerInfo_world_{worldId}.json");
        var world = JsonConvert.DeserializeObject<stWorld>(json);
        serverInfo.FromWorld(world);
    }

    // 5. 解密密码
    serverInfo.Env.Load(pathConfig);  // 加载 .env
    serverInfo.DecryptDBPassword();    // 解密数据库密码

    return serverInfo;
}
```

### stMyServerInfo 结构

```csharp
// 典型的服务器配置 JSON 结构
public class stMyServerInfo
{
    public int serverId { get; set; }
    public string server_name { get; set; }
    public string public_ip { get; set; }
    public int tcp_port { get; set; }
    public int http_port { get; set; }
    public int https_port { get; set; }
    public string config_file_subfix { get; set; }  // 如 "_Dev"
}
```

### .env 文件与密码加密

```csharp
// .env 文件格式
DB_PASSWORD=encrypted_base64_string
REDIS_PASSWORD=xxx
SSL_KEY=xxx

// 解密流程
serverInfo.Env.Load(pathConfig);
serverInfo.DecryptDBPassword();  // AES 解密
```

---

## 5. Kestrel HTTP 服务器配置

### Kestrel 简介

Kestrel = ASP.NET Core 内置跨平台 HTTP 服务器（类似 Java 的 Undertow / Tomcat）

```csharp
// 来源: LoginServer/Program.cs
builder.Services.Configure<KestrelServerOptions>(options =>
{
    // HTTP 端口
    options.ListenAnyIP(httpPort, listenOptions => {
        listenOptions.Protocols = HttpProtocols.Http1;  // HTTP/1.1
    });

    // HTTPS 端口
    options.ListenAnyIP(httpsPort, listenOptions => {
        listenOptions.Protocols = HttpProtocols.Http1AndHttp2;  // HTTP/1.1 + HTTP/2
        SetSSL(listenOptions, builder);  // 配置 SSL 证书
    });

    // 平台专用认证端口
    options.ListenAnyIP(authPort, listenOptions => {
        listenOptions.Protocols = HttpProtocols.Http1AndHttp2;
        SetSSL(listenOptions, builder);
    });
});
```

### HTTP 协议版本

| 协议 | 说明 |
|------|------|
| `HttpProtocols.Http1` | HTTP/1.1，仅 |
| `HttpProtocols.Http2` | HTTP/2，仅 |
| `HttpProtocols.Http1AndHttp2` | 两者都支持 |

### SSL 配置

```csharp
private static void SetSSL(ListenOptions listenOptions, WebApplicationBuilder builder)
{
    // 从配置读取 SSL 信息
    var sectionSSL = builder.Configuration.GetSection("ssl");
    var filename = sectionSSL["filename"];
    var pwd = sectionSSL["pwd"];

    // 如果密码被加密，从环境变量解密
    if (bool.TryParse(sectionSSL["useEnc"], out bool useEnc) && useEnc)
    {
        var sslKey = App.ServerConfig.Env.Value("SSL_KEY");
        pwd = AesEncryption.Decrypt(sslKey, sectionSSL["pwd_enc"]);
    }

    // 加载证书
    var cert = App.GetCerificate2(filename, pwd);
    if (cert != null)
        listenOptions.UseHttps(cert);
}
```

---

## 6. 控制器 (Controller)

### 标准 ASP.NET Core 控制器

```csharp
// 来源: LoginServer/Controllers/LoginController.cs
[ApiController]           // 启用自动模型绑定和验证
[Route("")]               // 根路由 = /LoginServer
public partial class LoginController : ControllerBase
{
    static readonly log4net.ILog log = ...;

    // HTTP POST /Token
    [HttpPost("Token")]
    public async Task<object> Token(sNetSend_TokenT req)
    {
        // req 是自动从请求体反序列化的 FlatBuffers 对象
        sNetRecv_TokenT ret = new();
        // 业务逻辑...
        return ret;  // 自动序列化为 FlatBuffers 返回
    }

    // HTTP POST /Login
    [HttpPost("Login")]
    public async Task<object> Login(sNetSend_LoginT req) { ... }

    // HTTP GET /Ping (特殊，非异步)
    [HttpGet("Ping")]
    [RequestTimeout(milliseconds: 1000)]
    public void Ping() { }
}
```

### 路由特性 (Route Attributes)

```csharp
[Route("")]                    // 类级别路由
public class LoginController : ControllerBase { }

// 方法级别
[HttpPost("Token")]            // POST /Token
[HttpGet("Ping")]              // GET /Ping
[HttpPut("Update")]            // PUT /Update
[HttpDelete("Delete")]        // DELETE /Delete

// 路由模板
[Route("api/[controller]")]   // api/login
[HttpGet("api/v{version}/users")]  // api/v1/users
```

### 模型绑定

```csharp
// ASP.NET Core 自动将 JSON/form-data 绑定到参数
[HttpPost("Login")]
public async Task<object> Login(sNetSend_LoginT req)
{
    // req.UserId, req.AuthKey 等自动从请求体填充
    var userId = req.UserId;  // string
    var authKey = req.Authkey;  // string
}

// 本项目使用 FlatBuffersInputFormatter 替换默认 JSON 绑定
// 所以 req 类型是 FlatBuffers T 类 (如 sNetSend_TokenT)
```

### 异步返回模式

```csharp
// 标准异步控制器
public async Task<IActionResult> GetUsers()
{
    var users = await _userService.GetUsersAsync();
    return Ok(users);  // 返回 200 + JSON
}

// 本项目的简化模式 (直接返回对象)
public async Task<object> Token(sNetSend_TokenT req)
{
    return ret;  // 框架自动处理序列化
}
```

---

## 7. 数据库层

### MySqlProc 基类

```csharp
// 来源: CommonLib/DataBase/MySqlProc.cs
public partial class MySqlProc
{
    protected string _connString;  // 连接字符串

    // 构造器
    public MySqlProc(string connString)
    {
        _connString = connString;
    }

    // 获取连接
    protected MySqlConnection GetConnection()
    {
        return new MySqlConnection(_connString);
    }
}
```

### AccountDBProc 使用示例

```csharp
// 来源: LoginServer/DBProc/AccountDBProc.cs
public partial class AccountDBProc : MySqlProc
{
    // 查询返回元组 (Result, UserInfo, Ban, MaintenanceList)
    public async Task<(eNetResultType, TAccount, sNetBanT?, List<sNetMaintenanceT>?)>
        AccountInfo_select(string _id, string _pwd = null)
    {
        using (var conn = GetConnection())  // using = try-finally 自动关闭
        {
            conn.Open();  // 打开连接

            // 调用存储过程
            var cmd = new MySqlCommand("s_account_info_select", conn);
            cmd.CommandType = CommandType.StoredProcedure;
            cmd.Parameters.Add("_id", MySqlDbType.String).Value = _id;

            // 读取结果
            using (var reader = await cmd.ExecuteReaderAsync())
            {
                if (await reader.ReadAsync())
                {
                    userInfo.User_ID = _id;
                    userInfo.User_Type = (eNetSNSAccessType)reader.GetInt16(0);
                    // ...
                }
            }

            return (result, userInfo, ban, maintenanceList);
        }
    }

    // 简单插入/更新
    public async Task<eNetResultType> AccountInfo_insert(...)
    {
        using (var conn = GetConnection())
        {
            conn.Open();
            var cmd = new MySqlCommand("s_account_info_insert", conn);
            cmd.CommandType = CommandType.StoredProcedure;
            // 参数赋值...
            await cmd.ExecuteNonQueryAsync();  // 无返回结果
            return eNetResultType.SUCCESS;
        }
    }
}
```

### 调用数据库

```csharp
// 在控制器中调用
[HttpPost("TestLogin")]
public async Task<object> TestLogin(sNetSend_TestLoginT req)
{
    sNetRecv_TestLoginT ret = new();

    // 调用 DBproc，传入连接字符串
    (ret.Result, userInfo, ret.Ban, ret.MaintenanceList) =
        await App.DB.Account.test_Login(req.UserId, req.Pwd, req.DeviceUid);

    if (ret.Result == eNetResultType.SUCCESS)
    {
        // 业务逻辑
    }

    return ret;
}
```

---

## 8. 启动流程总结

```
main(args)
    │
    ├─→ App.Init(serverId)           # 加载配置文件
    │       └─→ ServerInfoManager.Create()
    │           ├─→ 读取 my.config 确定环境
    │           ├─→ 加载 MyServerInfo.json
    │           ├─→ 加载 ServerInfo_global.json
    │           ├─→ 加载 ServerInfo_world_{id}.json
    │           └─→ 解密数据库密码
    │
    ├─→ Authentication.Authenticate()  # 服务器间认证 (发布版本)
    │
    ├─→ App.InitBase()               # 初始化基础组件
    │       ├─→ AppsettingsManager.Create()
    │       └─→ Table.LoadPatchTables()
    │
    ├─→ App.DB.Init()                # 初始化数据库
    │       └─→ 创建 AccountDBProc, RedisDBProc 等
    │
    ├─→ Kestrel 配置                 # HTTP 服务器端口
    │       ├─→ HTTP: httpPort
    │       ├─→ HTTPS: httpsPort
    │       └─→ AUTH: authPort
    │
    ├─→ builder.Services.AddControllers()  # 注册控制器
    │
    └─→ app.Run()                   # 启动服务器
            └─→ 等待 HTTP 请求
```

---

## 练习建议

1. **追踪一次完整的 Token 请求：**
   - 客户端 POST `/Token` → FlatBuffersInputFormatter → LoginController.Token()
   - → App.Redis.GetUserUidByAccountAsync()
   - → App.DB.Account.UserUid()
   - → SimpleToken.Serialize()
   - → 返回 sNetRecv_TokenT

2. **对比 Java Spring Boot：**
   - 画出 LoginServer 的请求处理流程
   - 对比 Spring Boot 的 Filter → Controller → Service → Repository 流程

---

下一章: [第三阶段：HTTP + FlatBuffers 协议](./03_flatbuffers协议.md)
