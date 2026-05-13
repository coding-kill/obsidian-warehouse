# C# 语法速成：Java 开发者指南

> 基于本项目真实代码，Java → C# 快速上手

---

## 目录

1. [类型系统对照](#1-类型系统对照)
2. [类与接口](#2-类与接口)
3. [异步编程 (async/await)](#3-异步编程-asyncawait)
4. [委托与事件](#4-委托与事件)
5. [LINQ 查询](#5-linq-查询)
6. [泛型](#6-泛型)
7. [特性 (Attribute)](#7-特性-attribute)
8. [集合框架](#8-集合框架)
9. [本项目常用模式](#9-本项目常用模式)

---

## 1. 类型系统对照

### 基本类型

| Java | C# | 说明 |
|------|-----|------|
| `int` | `int` | 相同，32位有符号 |
| `long` | `long` | 相同，64位有符号 |
| `boolean` | `bool` | C# 用 `bool`，不是 `boolean` |
| `String` | `string` | C# 用小写 `string`，引用类型 |
| `void` | `void` | 相同 |

### 无符号类型 (C# 特有)

```csharp
// Java 没有无符号类型，C# 有
uint ui = 100;        // 无符号 32 位
ulong ul = 1000L;     // 无符号 64 位
byte b = 255;         // 0~255
```

### 可空类型 (C# 特有)

```csharp
// Java 的包装类型需要用 Integer，且没有 null 安全机制
// C# 可以让任何类型可空
int? nullableInt = null;      // 可空 int，等价于 Nullable<int>
string? nullableStr = null;    // 可空引用类型
```

### 全局静态访问 (本项目典型用法)

```csharp
// Java: 静态单例通过 getInstance() 访问
// C#: 直接用 static 属性，全项目共享

// 来源: CommonLib/BaseApp.cs
static public string ADMIN_KEY { get; } = "adminsupervisor";
static public long UnixTimeSecond => DateTimeOffset.UtcNow.ToUnixTimeSeconds();
static public Random Random => Random.Shared;  // C# 6+ 的 expression-body 语法
```

---

## 2. 类与接口

### 类定义

```csharp
// C# 类默认 internal (程序集内可见)，Java 默认 package-private
public class SimpleToken          // = Java public class
{
    // C# 属性 (Property) = Java getter/setter 的简化写法
    public ulong UserUid { get; private set; }  // 外部只读，内部可写
    public int Key { get; private set; }
    public int UnixTime { get; private set; }

    // C# 可以定义无参构造器
    public SimpleToken() { }

    // 带参构造器
    public SimpleToken(ulong userId, int key, int unixTime)
    {
        UserUid = userId;
        Key = key;
        UnixTime = unixTime;
    }

    // C# 6+ 支持 expression-body 成员
    static public int TOKEN_LENTH => 16;  // 等价于 { get { return 16; } }
}
```

### 静态导入 (C# 10+ / Java 5+)

```csharp
// Java: import static java.lang.Math.PI;
// C#: using static 指令
using static System.Math;
double r = PI * 2;  // 不需要 Math.PI
```

### 接口

```csharp
// C# 接口可以包含默认实现 (C# 8+)
public interface IAuthentication
{
    bool Authenticate(IPAddress ipAddress, string hash, out string response);
}

// 默认实现 (本项目未使用，但 C# 特有)
public interface ILogger
{
    void Log(string msg) => Console.WriteLine(msg);  // 默认实现
}
```

### 命名空间 (相当于 Java 包)

```csharp
namespace CommonLib.Appsettings.Manager
{
    public class AppsettingsManager { }
}
// 使用: CommonLib.Appsettings.Manager.AppsettingsManager
// 或顶部 using CommonLib.Appsettings.Manager;
```

---

## 3. 异步编程 (async/await)

**这是 Java 和 C# 差异最大的地方，也是最重要的。**

### 基本语法

```csharp
// Java: CompletableFuture<T> + thenApply()
// C#: async Task<T> + await

// C# 方法必须返回 Task/Task<T>，不能用 void (除非是事件处理器)
public async Task<string> FetchDataAsync()
{
    // await 等待异步操作完成，不阻塞线程
    string result = await httpClient.GetStringAsync("http://api.example.com/data");
    return result;
}

// 调用方
var data = await FetchDataAsync();  // Java: future.get() 会阻塞
```

### 本项目典型异步模式

```csharp
// 来源: LoginController.cs
[HttpPost("Token")]
public async Task<object> Token(sNetSend_TokenT req)
{
    sNetRecv_TokenT ret = new();

    // 等价 Java: redis.get(key).toFuture().thenCompose(...)
    ulong userUid = await App.Redis.GetUserUidByAccountAsync(userId);

    // 等价 Java: db.query().toFuture().thenApply(...)
    (ret.Result, userUid) = await App.DB.Account.UserUid(userId, authKey).Result;

    // 返回值直接是对象，不需要包装
    return ret;
}
```

### 多异步操作

```csharp
// 并行执行多个异步操作
var task1 = GetUserAsync(1);
var task2 = GetUserAsync(2);
await Task.WhenAll(task1, task2);  // 等全部完成

// 或等待任意一个完成
await Task.WhenAny(task1, task2);

// 串行等待
await Step1Async();
await Step2Async();
```

### Task 返回模式 (本项目常见)

```csharp
// C# async 方法返回 Task<T>
// 非 async 方法可以用 .Result 同步等待 (阻塞线程，不推荐)

public async Task<(eNetResultType Result, ulong UserUid)> UserUid(string id, string authKey)
{
    // 返回元组类型，等价于 Java 的 record 或 Pair
    await Task.Delay(1);
    return (eNetResultType.SUCCESS, 12345UL);
}

// 调用
var (result, userUid) = await App.DB.Account.UserUid(id, authKey);
// 或
var tuple = await App.DB.Account.UserUid(id, authKey);
if (tuple.Result == eNetResultType.SUCCESS) { ... }
```

---

## 4. 委托与事件

### 委托 (Delegate) = 函数指针

```csharp
// 声明委托类型
public delegate int Compare(int a, int b);

// 使用 (类似 Java 函数式接口)
Compare cmp = (a, b) => a - b;  // Lambda 表达式

// C# 内置泛型委托
Action<string> logFunc = (msg) => Console.WriteLine(msg);  // 返回 void
Func<int, int, int> add = (a, b) => a + b;               // 返回 int
Predicate<int> isPositive = (n) => n > 0;                 // 返回 bool
```

### 事件 (Event)

```csharp
// 来源: CommonLib/BuildTime/BuildTimeAttribute.cs
public class BuildTimeAttribute : Attribute
{
    // 事件声明，外部只能 += / -=，不能直接调用
    public static event Action OnBuildTimeRetrieved;
}

// 触发事件 (只在类内部)
OnBuildTimeRetrieved?.Invoke();
```

### Lambda 表达式

```csharp
// Java: (a, b) -> a + b
// C#: (a, b) => a + b

// 本项目典型用法
ThreadPool.GetMinThreads(out int minWorker, out int minIO);
ThreadPool.GetMaxThreads(out int maxWorker, out int maxIO);

// LINQ 查询 (见下一节)
var list = new List<int> { 1, 2, 3, 4, 5 };
var evens = list.Where(n => n % 2 == 0).ToList();
```

---

## 5. LINQ 查询

**LINQ = Language Integrated Query，C# 特有的数据库风格查询语法**

### 语法风格

```csharp
var numbers = new[] { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };

// 方法语法 (更常用)
var evens = numbers.Where(n => n % 2 == 0).ToList();
var count = numbers.Count(n => n > 5);
var first = numbers.FirstOrDefault(n => n > 100);

// 查询语法 (SQL 风格)
var evens2 = from n in numbers
             where n % 2 == 0
             select n;

// 本项目典型用法 - 来源: LoginController.cs
var worldDB = App.DB.WorldList.Values.FirstOrDefault();
var serverInfo = App.ServerConfig.BillingServerList.First().Value;

// 链式操作
var result = list
    .Where(x => x.IsValid)
    .OrderBy(x => x.Priority)
    .Select(x => x.Name)
    .ToList();
```

### 常用 LINQ 方法

| LINQ 方法 | 说明 |
|-----------|------|
| `Where(predicate)` | 过滤 = Java Stream.filter() |
| `Select(selector)` | 转换 = Java Stream.map() |
| `First/FirstOrDefault` | 第一个元素 |
| `Any/All` | 是否有任何/所有满足条件 |
| `OrderBy/OrderByDescending` | 排序 |
| `GroupBy` | 分组 |
| `Join` | 连接两个集合 |
| `ToList/ToArray/ToDictionary` | 收集到集合 |

---

## 6. 泛型

### 基本用法

```csharp
// 与 Java 类似，但语法更简洁
var list = new List<int>();        // C#
List<Integer> list = new ArrayList<>();  // Java

var dict = new Dictionary<string, int>();  // C#
Map<String, Integer> map = new HashMap<>();  // Java
```

### 泛型约束

```csharp
// C# 泛型约束语法
public T Get<T>(string key) where T : class, new()
{
    // T 必须是引用类型，且有无参构造器
    return new T();
}

// 多种约束
where T : class           // 必须是引用类型
where T : struct          // 必须是值类型
where T : new()           // 必须有无参构造器
where T : BaseClass       // 必须继承自 BaseClass
where T : IInterface      // 必须实现 IInterface
```

### 协变与逆变 (C# 特有)

```csharp
// out = 协变 (返回类型可以更具体)
// in = 逆变 (参数类型可以更宽泛)

IEnumerable<Base> bases = new List<Derived>();  // IEnumerable<out T> 协变
Action<Base> act = (Derived d) => { };           // Action<in T> 逆变
```

---

## 7. 特性 (Attribute) = Java 注解

### 内置特性

```csharp
// 来源: LoginController.cs
[ApiController]           // ASP.NET Core: 自动模型绑定验证
[Route("")]               // 路由路径
[HttpPost("Token")]       // HTTP POST 方法，路径 = /Token

// 来源: ChatHub.cs
[HttpPost]                // 默认 POST
[RequestTimeout(1000)]     // 请求超时 1000ms
```

### 自定义特性

```csharp
// 来源: CommonLib/BuildTime/BuildTimeAttribute.cs
[AttributeUsage(AttributeTargets.Assembly)]  // 只能用于程序集
public class BuildTimeAttribute : Attribute
{
    static public DateTime? GetAssemblyBuildTime(Assembly assembly)
    {
        // 特性用于反射获取元数据
    }
}

// 使用
[BuildTime]  // 应用于程序集级别
class Program { }
```

### 反射获取特性

```csharp
// Java: clazz.getAnnotation(MyAnnotation.class)
// C#:
var attr = assembly.GetCustomAttribute<BuildTimeAttribute>();
```

---

## 8. 集合框架

### 常用集合

| Java | C# | 说明 |
|------|-----|------|
| `ArrayList` | `List<T>` | 动态数组，首选 |
| `HashMap` | `Dictionary<TKey, TValue>` | 键值对 |
| `HashSet` | `HashSet<T>` | 不重复集合 |
| `LinkedList` | `LinkedList<T>` | 双向链表 |
| `Queue/Deque` | `Queue<T>/ConcurrentQueue<T>` | 队列 |
| `BlockingQueue` | `Channel<T>` / `ConcurrentQueue<T>` | 并发队列 |

### 并发集合

```csharp
// Java: ConcurrentHashMap
// C#:
ConcurrentDictionary<ulong, UserInfo> _dicWorld = new();

// 线程安全操作
_dicWorld.TryAdd(userUid, userInfo);
_dicWorld.TryRemove(userUid, out var removed);
_dicWorld.TryGetValue(userUid, out var info);

// ConcurrentQueue (线程安全队列)
ConcurrentQueue<int> queue = new();
queue.Enqueue(1);
queue.TryDequeue(out var value);
```

### 初始化器

```csharp
// C# 集合初始化器，比 Java 更简洁
var list = new List<int> { 1, 2, 3, 4, 5 };
var dict = new Dictionary<string, int>
{
    ["key1"] = 1,
    ["key2"] = 2
};
```

---

## 9. 本项目常用模式

### 静态 App 访问点 (类似 Spring @Autowired)

```csharp
// 来源: LoginServer/Controllers/LoginController.cs
// C# 没有 Spring 那样的依赖注入容器，但项目用静态类代替

// 等价于 Java: @Autowired static App app;
// 任何地方都可以 App.Redis, App.DB, App.ServerConfig 访问
public class App
{
    static public IRedis Redis { get; set; }      // 注入的 Redis 客户端
    static public DBManager DB { get; set; }      // 数据库管理器
    static public ServerInfoManager ServerConfig { get; set; }
}
```

### 扩展方法 (C# 特有)

```csharp
// 来源: CommonLib/Const/Extensions.cs
// 为现有类型添加方法，无需继承

public static class StringExtensions
{
    public static bool IsNullOrEmpty(this string str)  // 为 string 添加方法
    {
        return string.IsNullOrEmpty(str);
    }
}

// 使用
string s = "hello";
if (s.IsNullOrEmpty()) { }  // 像调用实例方法一样
```

### record 类型 (C# 9+)

```csharp
// 简洁的不可变数据类型，类似 Java record (16+)
public record UserInfo(ulong UserUid, string FamilyName, int Level, int WorldId);

// 使用
var user = new UserInfo(12345, "Player1", 10, 1);
Console.WriteLine(user.UserUid);  // 12345
```

### null 条件运算符

```csharp
// Java: obj != null ? obj.getName() : null
// C#:
string name = obj?.Name;           // 如果 obj 为 null，返回 null
string name = obj?.Name ?? "Unknown";  // null 合并

// 来源: ChatHub.cs
var result = await App.DB.ChatDB.BlockList(userUid);
await Clients.Caller.SendAsync(BLOCK_LIST, (int)result.eNetResultType,
    result.blockUserUidList.IsNullOrEmpty() ? "[]" : "[" + string.Join(",", result.blockUserUidList) + "]");
```

### 模式匹配 (C# 9+)

```csharp
// switch 表达式
public string GetResultType(eNetResultType result) => result switch
{
    eNetResultType.SUCCESS => "成功",
    eNetResultType.FAIL => "失败",
    eNetResultType.TOKEN_EXPIRED => "Token过期",
    _ => "未知"
};

// 类型模式
object obj = GetSomeObject();
if (obj is string s && s.Length > 5)
{
    Console.WriteLine(s);
}
```

### 文件级别命名空间 (C# 10+)

```csharp
// 新写法: 不用缩进行
namespace LoginServer.Controllers;

public class LoginController : ControllerBase { }
```

---

## 快速参考表

| Java | C# |
|------|-----|
| `extends/implements` | `:` (冒号) |
| `@Override` | `override` (关键字，不是注解) |
| `instanceof` | `is` |
| `(Type) cast` | `(Type) cast` 或 `as Type` |
| `synchronized` | `lock` |
| `volatile` | `volatile` |
| `package` | `namespace` |
| `import static` | `using static` |
| `CompletableFuture` | `Task` |
| `Stream.filter().map()` | `Where().Select()` 或 LINQ |
| `Optional` | `T?` 可空类型 |
| `var` | `var` (相同语法) |
| `record` | `record` (相同语法) |

---

## 练习建议

1. 阅读 `LoginServer/Controllers/LoginController.cs` — 掌握 async/await、Task、控制器模式
2. 阅读 `CommonLib/SimpleToken.cs` — 理解 struct、属性、static
3. 阅读 `ChatServer/Hub/ChatHub.cs` — 理解 async Task、SignalR Hub、ConcurrentDictionary
4. 对照 `LoginServer/Program.cs` — 理解依赖注入、配置、Kestrel

---

下一章: [第二阶段：Asp 服务器入口（LoginServer）](./02_asp服务器入口.md)
