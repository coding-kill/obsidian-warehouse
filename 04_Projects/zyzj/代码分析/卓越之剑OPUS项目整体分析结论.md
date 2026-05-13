# 卓越之剑 OPUS 项目整体分析结论

> 分析时间：2026-04-03
> 分析范围：TCPServer（C++游戏服集群）+ Asp（C# Web服务集群）

---

## 一、项目整体架构

### 1.1 技术栈概览

| 层级 | 技术选型 | 说明 |
|------|---------|------|
| 游戏核心逻辑 | C++ (VS工程, IOCP模型) | 高性能长连接服务器集群 |
| Web/网关/业务接口 | C# .NET 8 (Kestrel) | 登录、网关、聊天、拍卖等 |
| 数据库 | SQL (按库拆分: account/game/world/guild/log) | 分库分表设计 |
| 缓存/分布式 | Redis | 会话、排行、拍卖、跨服状态等 |
| 配置中心 | JSON配置文件 + my.config | 环境目录切换机制 |
| 通信协议 | FlatBuffers | 服务间二进制序列化 |

### 1.2 C++ 服务集群（TCPServer）

| 服务 | 职责 |
|------|------|
| GlobalServer | 全局控制面，跨world总入口 |
| WorldServer | world级协调中心（频道、跨服状态、用户注册） |
| CommunityServer | 社区/关系域（组队、公会、社交协调） |
| GameServer | 核心玩法逻辑（角色、地图、战斗、任务、道具等） |
| InstancePoolServer | 副本实例池调度 |
| InstanceServer | 具体副本执行节点 |
| DBAgentServer | C++服务访问数据库的中介层（多连接分片） |
| LogServer | 日志聚合/持久化通道 |

### 1.3 C# 服务集群（Asp）

| 服务 | 职责 |
|------|------|
| LoginServer | 登录鉴权入口 |
| GeAsp | 综合业务API网关（world/web侧） |
| ChatServer | 世界公告聚合发送 |
| AuctionServer | 拍卖行HTTP接口 |
| ArenaServer | 竞技场接口 |
| WorldQueueServer | 登录排队 |
| BillingServer | 支付/计费通道 |
| AuthenticationServer | 第三方认证校验 |
| LogServer | 日志服务 |

### 1.4 启动编排顺序

```
# 全局层
GlobalServer + LoginServer

# World层（每个world一套）
LogServer + DBAgentServer + WorldServer + CommunityServer +
InstancePoolServer + GeAsp + ChatServer + WorldQueueServer +
AuctionServer + (ArenaServer可选)

# Game服层
GameServer (按terrain_group分组，可横向扩容)
InstanceServer (按实例副本需求)

# War服层
WarServer (按terrain_group_id=20001等战场地形)
```

---

## 二、配置体系

### 2.1 配置加载机制

配置入口：`TCPServer/Bin/config/`

```
my.config                    → 只存"当前环境目录名"（dev/live）
MyServerInfo.json            → 本机默认world/server、日志、线程池等基础参数
ServerInfo_global_*.json     → 全局层服务 + 全局DB/Redis + 地图组
ServerInfo_world_{id}_*.json → 某个world下的服务、DB、Redis、容量参数
```

加载顺序：`my.config` → `MyServerInfo.json` → `ServerInfo_global_*.json` → `ServerInfo_world_*.json` → 根据启动参数确定服务实例身份。

### 2.2 核心配置字段

| 字段 | 含义 |
|------|------|
| server_id | 服务唯一标识 |
| world_id | 世界/区服ID |
| terrain_group_id | 地图分组（决定哪些game/war服承载哪些地图） |
| ip/port | 网络地址 |
| server_list | 该层所有服务实例列表 |

---

## 三、通信机制

### 3.1 服务间通信（C++ TCP长连接）

- **网络模型**：IOCP + 工作线程池 + 会话池 + 包池
- **协议**：FlatBuffers（GEM_PACKET_xxx, SEND_PID_xxx, RECV_PID_xxx）
- **收包流程**：IOCP → Session → 投递WorkThread → PacketProcedure
- **发包流程**：构建FB builder → Session send

GameServer 启动后主动连接：
- WorldServer、CommunityServer、DBAgentServer（多连接）、LogServer、ChatServer、AuctionServer、BillingServer、ArenaServer（条件）

### 3.2 服务注册与重连机制

所有TCP长连服务使用统一重连框架：
- `iSrvConnect::checkConnect()` 接口
- `SrvConnectThread`：每100ms轮询所有已注册服务
- `SrvConnectManager::addSrv()` 把服务纳入重连列表
- **重连条件**：`!m_bActive && !isAlive()`
- **重连后**：再次发送 RegServer 包重新注册

不走统一重连的服务：Auction（HTTP）、Billing（HTTP队列）、Chat（HTTP聚合）、Authentication（一次性HTTPS校验）。

### 3.3 消息处理模型

```
客户端包 → IOCP网络线程接收 → WorkThreadManager.InsertCommand() →
WorkThread工作线程 → packetProcedure.packetProc() →
User::packetProcedure() → 按pid+用户状态分发到OnClient_xxx
```

**关键点**：
- 网络线程只负责收包和投递，不跑核心业务逻辑
- 业务逻辑在工作线程（WorkThread）执行
- 同一用户尽量路由到固定工作线程，减少锁竞争

### 3.4 多通道消息路由

Game内部通过 `User::addAllProcedure()` 注册多通道处理器：
- `COMMAND_TYPE_USER` → 客户端C2S包
- `COMMAND_TYPE_WORLD_SRV` → WorldServer下发包
- `COMMAND_TYPE_COMMUNITY_SRV` → CommunityServer包
- `COMMAND_TYPE_DBAGENT_SRV` → DBAgent回包

---

## 四、用户生命周期与状态管理

### 4.1 用户状态机

```
[新连接 Session]
    ↓
[World校验通过 UserConnect]
    ↓
[ONLINE (usingList在线用户)]
    ↓ 断线/踢下线
[WAIT (waitList, 可重连窗口)]
    ↓ 超时 / 重连成功
[RETURN (freeList对象池)] / [ONLINE]
    ↓ 跨服/跳图
[MOVE_OUT_ZONE + saveUser + 场景移除]
    ↓
[目标服 ONLINE]
```

### 4.2 内存缓存策略

- **在线用户**：`m_usingList` 完整持有（背包、货币、状态、任务等）
- **等待用户**：`m_waitList` 短时保留（断线重连窗口，默认有超时）
- **过渡机制**：离线先移入wait，超时才回收至对象池
- **跨服一致性**：单主服持有 + World协调迁移 + saveUser前置落地

### 4.3 切服过程状态限制

当GameServer收到World下发的MoveServer后：
1. 给客户端发目标服信息（ip/port/connectKey）
2. 用户状态切为 `MOVE_OUT_ZONE`
3. 从当前场景移除
4. 触发 `saveUser()` 保存
5. **普通业务命令被拒绝**，只做迁服收尾回包
6. 客户端连新服后恢复正常处理

---

## 五、核心数据表体系

### 5.1 表加载顺序（按依赖关系）

```
SkillUpgradeTable → SkillAbilityTable → SkillAbilityValueTable
→ SkillBasicTable → SkillLevelTable → StanceBasicTable

CharacterExpTable → CharacterLevelTable → CharacterCreationTable
→ CharacterInitialTable → CharacterBasicTable → CharacterpROMOTIONTable
```

### 5.2 关键业务表分组

| 业务域 | 相关表 |
|--------|--------|
| 角色成长 | CharacterBasicTable, CharacterLevelTable, CharacterExpTable, CharacterPromotionTable |
| 技能战斗 | SkillBasicTable, SkillLevelTable, SkillAbilityTable, StanceBasicTable |
| 装备道具 | ItemBasicTable, ItemAbilityTable, ItemEquipTable |
| 卡牌Pioneer | PioneerTable, PioneerReinTable, PioneerReinProTable |
| 图鉴收集 | CollectionBasicTable, Hero/Pet/Monster/ItemCollectionTable |
| 宠物系统 | PetBasicTable, PetIncubationTable, PetSyntheticTable, PetExchangeTable |
| 副本实例 | InstanceTable, DropItemTable, MonsterAISkillTable |
| 运营活动 | EventDataTable, BattlePassTable, EventFindTable |

---

## 六、排错字典（启动/运行常见报错）

| 报错日志 | 归属模块 | 第一优先检查 |
|---------|---------|-------------|
| `DataManager Init failed!` | GameServer启动 | json语法/字段名/类型是否匹配 |
| `dataTable load fail!` | DataTableManager | 最近改动的json表，逗号/引号/尾逗号 |
| `Item Manager load failed!` | ItemManager | ItemBasicTable新增item重复ID/非法类型 |
| `not found curr Stance` | CharacterSkillManager | 角色默认stance在StanceBasicTable是否有效 |
| `not curr stance skill` | CharacterSkillManager | stance里是否挂了该skill |
| `Fail addCollection` | CollectionManager | CollectionBasicTable行内容合法性 |
| Pioneer强化异常 | PioneerManager | PioneerReinProTable与PioneerReinTable阶段对齐 |
| Pet孵化/合成异常 | PetIncubationManager | egg item存在性、结果pet/grade合法性 |

---

## 七、扩展机制

### 7.1 新增服务实例（横向扩容）

1. 在 `ServerInfo_world_xxx.json` 增加 `server_list` 项
2. 配置对应端口/ID/terrain_group
3. 由批处理或运维脚本拉起

### 7.2 新增业务协议

1. 新增 FlatBuffers 消息定义（PID + schema）
2. 在对应 Session Manager 的 `packetProcedure` 注册处理
3. 在 SendFunc/Manager 中增加发送封装

### 7.3 新增数据表

1. 加 datatable（JSON）和 SQL
2. 在 `DataManager` 侧加载
3. 在业务 manager 初始化时接入

### 7.4 新增 Web API

1. ASP 对应服务新增 Controller/API
2. 用 `ServerInfoManager` 读取配置/目标世界
3. 复用 `CommonLib` 的 DB/Env/加密/证书工具

---

## 八、快速上手路径（建议）

1. **第一天**：跑通最小链路 `Global + Login + World + DBAgent + Game`，确认服务注册关系
2. **第二天**：读 `GameServer.cpp` 的 `startingService()`，对照配置理解依赖顺序
3. **第三天**：读 `WorldSrvManager` 理解Game-World通信收发模型
4. **第四天**：读一个ASP服务（如LoginServer）理解Web面入口
5. **第五天**：跟一遍"登录后进服"完整包流转

---

## 九、结论总结

**卓越之剑OPUS** 是一个典型的**分层多进程MMO RPG集群**：
- C++侧负责高性能游戏逻辑（IOCP长连接、FlatBuffers协议）
- C#侧负责Web/网关/商业化接口（Kestrel + HTTP）
- 配置驱动的多服务编排，支持按地图/实例/战场横向扩容
- 用户状态机 + 内存缓存 + 异步DB落地 保证在线一致性
- 统一的重连框架 + 服务注册机制 提供容错能力

**核心风险点**：
- 配置端口与server_id强绑定，改错会"能启动但互联失败"
- `my.config` 决定环境目录，很多"读不到配置"问题根源在此
- 表驱动设计，改表后需重启GameServer并做完整回归测试
