# 卓越之剑 OPUS 项目分析结论

**整理日期**：2026-04-03

---

## 一、项目整体架构

### 1.1 整体架构图

```mermaid
graph TB
    Client["客户端"]
    ASP["ASP 服务集群 (.NET 8)"]
    CPP["C++ 服务集群"]

    subgraph ASP["ASP 服务集群"]
        Login["LoginServer<br/>登录鉴权"]
        GeAsp["GeAsp<br/>业务API网关"]
        Chat["ChatServer<br/>公告聚合"]
        Auction["AuctionServer<br/>拍卖行"]
        Billing["BillingServer<br/>计费"]
        WorldQ["WorldQueueServer<br/>排队"]
        Auth["AuthenticationServer<br/>认证"]
    end

    subgraph CPP["C++ 服务集群"]
        Global["GlobalServer<br/>全局控制面"]
        World["WorldServer<br/>世界协调"]
        Comm["CommunityServer<br/>社区/社交"]
        DBAgent["DBAgentServer<br/>数据库代理"]
        Game["GameServer<br/>核心玩法"]
        Instance["InstanceServer<br/>副本执行"]
        InstPool["InstancePoolServer<br/>副本池调度"]
        Log["LogServer<br/>日志"]
        War["WarServer<br/>战场"]
    end

    Client --> ASP
    Client --> Game
    ASP -->|HTTP/TCP| CPP
```

### 1.2 服务职责矩阵

| 服务 | 类型 | 职责 |
|------|------|------|
| GlobalServer | C++ | 全局控制面，跨world总入口 |
| WorldServer | C++ | world级协调中心（频道、跨服、用户注册） |
| CommunityServer | C++ | 社区/关系（组队、公会、社交协调） |
| GameServer | C++ | 核心玩法（角色、地图、战斗、道具） |
| InstancePoolServer | C++ | 副本实例池调度 |
| InstanceServer | C++ | 具体副本执行节点 |
| DBAgentServer | C++ | 数据库访问中介（多连接分片） |
| LogServer | C++ | 日志聚合/持久化 |
| LoginServer | C# | 登录鉴权入口 |
| GeAsp | C# | 综合业务API网关 |
| ChatServer | C# | 世界公告聚合发送 |
| AuctionServer | C# | 拍卖行HTTP接口 |
| BillingServer | C# | 支付/计费通道 |

---

## 二、启动编排

### 2.1 启动顺序流程

```mermaid
flowchart LR
    subgraph Phase1["阶段一：全局层"]
        G["GlobalServer"]
        L["LoginServer"]
    end

    subgraph Phase2["阶段二：World层"]
        Log["LogServer"]
        DB["DBAgentServer"]
        W["WorldServer"]
        C["CommunityServer"]
        IP["InstancePoolServer"]
        Ge["GeAsp"]
        Ch["ChatServer"]
        WQ["WorldQueueServer"]
        Au["AuctionServer"]
        Ar["ArenaServer"]
    end

    subgraph Phase3["阶段三：Game服层"]
        Ga["GameServer"]
        Inst["InstanceServer"]
    end

    subgraph Phase4["阶段四：War服层"]
        Wa["WarServer"]
    end

    Phase1 --> Phase2 --> Phase3
    Phase2 --> Phase4
```

---

## 三、通信机制

### 3.1 IOCP + 工作线程池模型

```mermaid
sequenceDiagram
    participant Client as 客户端
    participant IOCP as IOCP网络线程
    participant Session as Session
    participant WT as WorkThreadManager
    participant Work as WorkThread
    participant PP as PacketProcedure

    Client->>IOCP: 发送数据包
    IOCP->>Session: 接收数据包
    Session->>WT: InsertCommand()
    WT->>Work: 投递到工作队列
    Work->>PP: packetProc()
    PP->>PP: 路由到具体Handler
```

### 3.2 服务注册与重连流程

```mermaid
flowchart TD
    Start["服务启动"] --> Connect["connect()"]
    Connect --> Post["postConnect()"]
    Post --> Reg["sendConnectOKSign()<br/>发送RegServer注册包"]
    Reg --> Add["SRV_CONNECT_MGR.addSrv()"]
    Add --> Monitor["SrvConnectThread<br/>每100ms轮询"]

    Monitor --> Check["checkConnect()"]
    Check --> Condition{"!m_bActive<br/>&& !isAlive()"}

    Condition -->|是| Reconnect["重新连接"]
    Reconnect --> Connect
    Condition -->|否| Alive["保持连接"]
```

### 3.3 多通道消息路由

| CommandType | 来源 | 处理 |
|------------|------|------|
| COMMAND_TYPE_USER | 客户端C2S | User_Handler_Packet |
| COMMAND_TYPE_WORLD_SRV | WorldServer下发 | User_Handler_WorldSrv |
| COMMAND_TYPE_COMMUNITY_SRV | CommunityServer | User_Handler_CommunitySrv |
| COMMAND_TYPE_DBAGENT_SRV | DBAgent回包 | User_Handler_DBAgentSrv |

---

## 四、配置体系

### 4.1 配置加载顺序

```mermaid
flowchart LR
    A["my.config<br/>环境目录"] --> B["MyServerInfo.json<br/>本机基础参数"]
    B --> C["ServerInfo_global_*.json<br/>全局层配置"]
    C --> D["ServerInfo_world_{id}_*.json<br/>world层配置"]
    D --> E["按启动参数确定服务实例身份"]
```

### 4.2 数据表加载顺序（技能系统依赖链）

```mermaid
flowchart LR
    A1["SkillUpgradeTable"] --> A2["SkillAbilityTable"]
    A2 --> A3["SkillAbilityValueTable"]
    A3 --> A4["SkillBasicTable"]
    A4 --> A5["SkillLevelTable"]
    A5 --> A6["StanceBasicTable"]
```

### 4.3 数据表加载顺序（角色系统依赖链）

```mermaid
flowchart LR
    B1["CharacterExpTable"] --> B2["CharacterLevelTable"]
    B2 --> B3["CharacterCreationTable"]
    B3 --> B4["CharacterInitialTable"]
    B4 --> B5["CharacterBasicTable"]
    B5 --> B6["CharacterPromotionTable"]
```

---

## 五、用户状态管理

### 5.1 用户生命周期状态机

```mermaid
stateDiagram-v2
    [*] --> Session: 新连接
    Session --> UserConnect: World校验通过
    UserConnect --> ONLINE: usingList
    ONLINE --> WAIT: 断线/踢下线
    ONLINE --> MOVEOUT: 跨服/跳图
    WAIT --> ONLINE: 重连成功
    WAIT --> RETURN: 超时
    MOVEOUT --> Save: saveUser()
    Save --> Remove: 场景移除
    Remove --> [*]
    RETURN --> [*]
```

### 5.2 内存缓存结构

| 缓存 | 含义 |
|------|------|
| m_usingList | 在线用户，完整持有对象 |
| m_waitList | 等待态，断线重连窗口 |
| m_accListByAccountId | 账号→User索引 |
| m_listByFamilyName | 家族名→User索引 |
| m_freeList | User对象池复用 |

### 5.3 跨服一致性保证原则

1. **单主写**：同一时刻只有一个服"主写"该用户内存态
2. **短离线不丢态**：WAIT保留内存对象，允许快速重连
3. **跨服前先收口**：MOVE_OUT前做saveUser + 场景移除
4. **最终落库兜底**：DBAgent异步落地保证最终一致

---

## 六、业务系统详解

### 6.1 游戏核心系统域

```mermaid
graph LR
    subgraph Core["核心系统"]
        Char["角色成长"]
        Skill["技能战斗"]
        Terrain["地图场景"]
        Item["装备道具"]
    end

    subgraph Card["卡牌系统"]
        Pioneer["Pioneer"]
        Collection["Collection"]
        Pet["Pet"]
    end

    subgraph Social["社交经济"]
        Chat["聊天"]
        Guild["公会"]
        Shop["商城"]
    end

    subgraph PVP["PVP竞技"]
        Arena["竞技场"]
        Ranking["排行"]
        Season["赛季"]
    end

    subgraph Event["运营活动"]
        BP["战令"]
        Evt["事件"]
    end
```

### 6.2 技能系统层次

```mermaid
graph TB
    subgraph Stance["StanceBasicTable"]
        S1["Stance[0]"]
        S2["Stance[1]"]
    end

    subgraph Skill["技能层"]
        SK1["SkillBasicTable"]
        SK2["SkillLevelTable"]
    end

    subgraph Ability["效果层"]
        AB1["SkillAbilityTable"]
        AB2["SkillAbilityValueTable"]
    end

    subgraph Cool["冷却层"]
        CD["CooltimeManager"]
    end

    S1 --> SK1
    SK1 --> SK2
    SK1 --> AB1
    AB1 --> AB2
    S1 --> CD
```

---

## 七、排错字典

### 7.1 启动期报错

| 报错日志 | 归属 | 第一优先检查 |
|---------|------|-------------|
| `DataManager Init failed!` | GameServer启动 | json语法/字段名/类型 |
| `dataTable load fail!` | DataTableManager | 最近改动的json表 |
| `Item Manager load failed!` | ItemManager | ItemBasic表重复ID/非法类型 |
| `check relation` 失败 | 跨表引用校验 | 等级/阶段不连续 |

### 7.2 运行期报错

| 报错日志 | 归属 | 第一优先检查 |
|---------|------|-------------|
| `not found curr Stance` | CharacterSkillManager | StanceBasic表是否包含stanceIdx |
| `not curr stance skill` | CharacterSkillManager | stance里是否挂了skill |
| `Fail addCollection` | CollectionManager | CollectionBasic字段合法性 |
| Pioneer强化异常 | PioneerManager | 强化表阶段对齐 |
| Pet孵化/合成异常 | PetIncubationManager | egg item/结果pet存在性 |

### 7.3 改表防炸服作业单

| 业务域 | 涉及表 | 固定回归 |
|-------|--------|---------|
| 角色 | CharacterBasic/Level/Exp/Promotion | 新建→进游戏→切stance |
| 技能 | SkillBasic/Level/Ability/Stance | 技能释放/升级/冷却 |
| Pioneer | PioneerTable/Rein/Pro | pioneer强化 |
| Collection | CollectionBasic/*Collection | 图鉴注册/领奖 |
| Pet | PetBasic/Incubation/Synthetic/Exchange | 孵化/合成/兑换 |

---

## 八、核心流程图汇总

### 图1：整体架构图

```mermaid
graph TB
    Client["客户端"] --> ASP
    Client --> GameS["GameServer"]
    ASP --> CPP["C++集群"]
```

### 图2：用户状态机

```mermaid
stateDiagram-v2
    [*] --> ONLINE
    ONLINE --> WAIT: 断线
    ONLINE --> MOVEOUT: 跨服
    WAIT --> ONLINE: 重连
    WAIT --> RETURN: 超时
```

### 图3：跨服流程

```mermaid
sequenceDiagram
    participant Old as 当前GameServer
    participant Client as 客户端
    participant New as 目标GameServer

    Old->>Client: NtfyMoveServer(ip/port/key)
    Client->>New: 连接目标服
    Old->>New: saveUser()数据
    Note over Old: MOVE_OUT_ZONE
    New->>Client: 接管用户
```

### 图4：消息处理流程

```mermaid
flowchart LR
    A["客户端"] --> B["IOCP"]
    B --> C["Session"]
    C --> D["WorkThreadManager"]
    D --> E["WorkThread"]
    E --> F["PacketProcedure"]
    F --> G["Handler"]
```

---

## 九、快速上手计划

| 天数 | 目标 | 关键动作 |
|------|------|---------|
| Day 1 | 跑通最小链路 | 启动Global+Login+World+DBAgent+Game，观察注册日志 |
| Day 2 | 理解启动依赖 | 读GameServer.cpp startingService() |
| Day 3 | 理解Game-World通信 | 读WorldSrvManager收发模型 |
| Day 4 | 理解Web服务入口 | 读LoginServer/Program.cs |
| Day 5 | 串联完整链路 | 跟一遍登录→进服完整包流转 |

---

## 十、扩展机制

### 新增服务实例
1. 在 `ServerInfo_world_xxx.json` 增加 `server_list` 项
2. 配置端口/ID/terrain_group
3. 运维脚本拉起

### 新增业务协议
1. FlatBuffers消息定义（PID + schema）
2. 在SessionManager注册packetProcedure
3. 在SendFunc/Manager增加发送封装

### 新增数据表
1. 加datatable(JSON)和SQL
2. 在DataManager侧加载
3. 业务manager初始化时接入
