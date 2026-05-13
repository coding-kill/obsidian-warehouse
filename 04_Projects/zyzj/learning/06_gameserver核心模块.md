# 第六阶段：GameServer 核心模块

> 理解游戏核心逻辑：Actor、Item、Skill、Battle

---

## 目录

1. [Actor 系统](#1-actor-系统)
2. [Item 系统](#2-item-系统)
3. [Skill 系统](#3-skill-系统)
4. [Session 管理](#4-session-管理)
5. [Packet 处理流程](#5-packet-处理流程)

---

## 1. Actor 系统

### Actor 是什么

Actor = 游戏中的任何实体（玩家角色、怪物、NPC、召唤物等）

```
Actor (基类)
├── Character (玩家角色)
├── Monster (怪物)
├── Npc (非玩家角色)
├── Summon (召唤物)
└── Object (场景物体)
```

### Actor 类结构

```cpp
// 来源: GameServer/Actor.h

class Actor : public iActor
{
public:
    Actor(ACTOR_ID _actorId);

    // 基本属性
    ACTOR_ID getActorId() { return m_actorId; }

    // HP/SP
    HP getMaxHp() { return getStatValue_MHP(); }
    HP getHp() { return m_hp; }
    void setHp(HP hp) { m_hp = hp; }

    // 战斗
    bool isDead();
    bool isActive();
    void addExp(EXP exp);

    // 技能
    bool useNextSkill(TABLE_IDX skillIdx, LEVEL skillLevel);
    bool onApplySkill(iActor* caster, TABLE_IDX skillIdx, LEVEL skillLevel);

    // Buff
    void addBuff(Buff* buff);
    void removeBuff(BUFF_ID buffId);

private:
    ACTOR_ID m_actorId;
    HP m_hp;
    SP m_sp;
    LEVEL m_level;
    EXP m_exp;

    // 战斗属性
    std::atomic<uint32_t> m_attackResultBitFlag;

    // Buff 管理
    Buff* m_pMezBuff;  // 硬直/眩晕等控制 buff
};
```

### ActorManager

```cpp
// 来源: GameServer/ActorManager.h

#define ACTOR_MGR ActorManager::GetInstance()

class ActorManager {
public:
    static ActorManager* GetInstance();

    bool initialize();
    void release();

    // 创建 Actor
    Character* CreateCharacter(CharacterInfo* info, Terrain* terrain);
    Monster* CreateMonster(ST_MOB_BASIC_TABLE_PROTO* proto, const ST_VEC3& pos);

    // 查找 Actor
    Character* GetCharacter(ACTOR_ID actorId);
    Actor* GetActor(ACTOR_ID actorId);

    // 广播 (向周围玩家发送)
    void SendToVisiblePlayers(Actor* actor, Packet* packet);

private:
    ObjectPool<Character>* m_characterPool;
    ObjectPool<Monster>* m_monsterPool;

    std::unordered_map<ACTOR_ID, Character*> m_characters;
    std::unordered_map<ACTOR_ID, Monster*> m_monsters;
};
```

### Character 特有属性

```cpp
// Character = 玩家控制的角色
class Character : public Actor
{
public:
    bool initCharacter(CharacterInfo* info, Terrain* pTerrain);

    // 背包/装备
    Inventory m_inventory;
    EquipContainer m_equipContainer;

    // 技能
    CharacterSkillManager* m_skillManager;

    // 成就
    AchievementManager* m_achievementManager;

    // 工会
    GUILD_ID m_guildId;
    std::string m_guildName;

    // 用户信息
    uint64_t m_userUid;
    std::string m_familyName;
    int m_worldId;
};
```

---

## 2. Item 系统

### ItemObject

```cpp
// 来源: CommonLib/CommonItemObject.h

class IItemObject {
public:
    virtual ~IItemObject() {}

    ITEM_IDX getItemIdx() { return m_itemIdx; }
    OBJECT_ID getObjectId() { return m_objectId; }
    ITEM_TYPE getItemType() { return m_itemType; }

    virtual bool CanUse(Actor* actor) = 0;
    virtual void Use(Actor* actor) = 0;

protected:
    ITEM_IDX m_itemIdx;
    OBJECT_ID m_objectId;
    ITEM_TYPE m_itemType;
};
```

### Item 类型

| Item 类型 | 说明 | 示例 |
|----------|------|------|
| 消耗品 | 使用后消失 | 药水、复活币 |
| 装备 | 可穿戴 | 武器、防具、饰品 |
| 时装 | 外形装饰 | 皮肤、翅膀 |
| 任务物品 | 不可交易 | 剧情道具 |
| 货币 | 堆叠 | 金币、钻石 |

### Inventory 管理

```cpp
// 来源: GameServer/Inventory.h

class Inventory {
public:
    bool AddItem(ITEM_IDX itemIdx, COUNT count);
    bool RemoveItem(ITEM_IDX itemIdx, COUNT count);
    bool UseItem(OBJECT_ID objectId, Actor* actor);

    // 背包容量
    int GetCapacity() { return m_capacity; }
    int GetCurrentCount() { return m_items.size(); }

private:
    std::vector<ItemSlot> m_items;  // 背包槽位
    int m_capacity;  // 容量上限
};
```

### 物品操作流程

```cpp
// 使用物品
void HandleUseItem(Session& session, Packet* packet)
{
    auto req = C2G_UseItem.GetRootAs(packet->payload());

    Character* character = SESSION_MGR->GetCharacter(session.GetActorId());
    if (!character)
        return;

    // 查找物品
    ItemSlot* slot = character->m_inventory.FindSlot(req->objectId());
    if (!slot)
        return;

    // 使用
    IItemObject* item = ITEM_MGR->GetItemObject(slot->itemIdx);
    if (item->CanUse(character))
    {
        item->Use(character);
        character->m_inventory.RemoveItem(slot->objectId(), 1);
    }
}
```

---

## 3. Skill 系统

### Skill 类结构

```cpp
// 来源: GameServer/Skill.h

class Skill {
public:
    Skill(SKILL_ID skillId, TABLE_IDX tableIdx, LEVEL level);

    TABLE_IDX getTableIdx() { return m_tableIdx; }
    LEVEL getLevel() { return m_level; }
    SKILL_TYPE getSkillType() { return m_skillType; }

    // 冷却
    bool CanUse();
    void StartCooldown();
    int GetCooldownRemain();

private:
    SKILL_ID m_skillId;
    TABLE_IDX m_tableIdx;      // 表索引
    LEVEL m_level;
    SKILL_TYPE m_skillType;

    Tick m_lastUseTick;
    int m_cooldown;            // 冷却时间(ms)
};
```

### 技能使用流程

```cpp
// 来源: GameServer/Actor.cpp

bool Actor::useNextSkill(TABLE_IDX skillIdx, LEVEL skillLevel)
{
    // 1. 获取技能表数据
    auto skillTable = DATA_TABLE_MGR->GetSkillLevelTable(skillIdx, skillLevel);
    if (!skillTable)
        return false;

    // 2. 检查冷却
    if (!m_cooldownManager.CanUse(skillIdx))
        return false;

    // 3. 检查 SP
    if (getSp() < skillTable->spCost)
        return false;

    // 4. 检查目标
    if (!CheckTarget(skillTable))
        return false;

    // 5. 消耗
    useSp(skillTable->spCost);
    StartCooldown(skillIdx, skillTable->cooldown);

    // 6. 执行技能效果
    return onApplySkill(this, skillIdx, skillLevel);
}
```

### Skill 类型

| 类型 | 说明 |
|------|------|
| 普通攻击 | 自动释放 |
| 主动技能 | 手动释放 |
| 被动技能 | 永久生效 |
| 特性 | 属性加成 |
| 召唤技能 | 召唤怪物/NPC |

---

## 4. Session 管理

### Session 结构

```cpp
// 来源: CommonLib/Session.h

class Session {
public:
    SOCKET GetSocket() { return m_socket; }
    uint64_t GetActorId() { return m_actorId; }

    // 发送数据
    void Send(Packet* packet);
    void Send(byte* buf, int len);

    // 设置关联的 Actor
    void SetActorId(uint64_t actorId) { m_actorId = actorId; }

    // 连接状态
    bool IsConnected() { return m_isConnected; }

private:
    SOCKET m_socket;           // Socket 句柄
    uint64_t m_actorId;        // 关联的 Actor ID
    bool m_isConnected;        // 是否已连接

    // 接收缓冲
    byte* m_recvBuffer;
    int m_recvPos;
    int m_recvremain;

    // 发送缓冲
    CRITICAL_SECTION m_sendCs;
    std::queue<Packet*> m_sendQueue;
};
```

### SessionManager

```cpp
// 来源: CommonLib/SessionMgr.h

class SessionManager {
public:
    bool initialize(int poolSize, int bufferSize);

    // 查找
    Session* GetSessionByActorId(ACTOR_ID actorId);
    Session* GetSessionBySocket(SOCKET socket);

    // 管理
    void AddSession(Session* session);
    void RemoveSession(Session* session);

    // 广播
    void Broadcast(Packet* packet, std::vector<ACTOR_ID>* excludeList = nullptr);

private:
    std::unordered_map<SOCKET, Session*> m_sessionsBySocket;
    std::unordered_map<ACTOR_ID, Session*> m_sessionsByActorId;

    ObjectPool<Session>* m_sessionPool;
};
```

---

## 5. Packet 处理流程

### 完整流程

```
客户端                         GameServer
   │                              │
   │──── TCP 数据包 ──────────────►│
   │                              │
   │                              ├─► IOCP 接收
   │                              ├─► SessionManager 找到 Session
   │                              │
   │                              ├─► PacketManager 解析
   │                              │       │
   │                              │       └─► 验证 Checksum
   │                              │       └─► 查找 Handler
   │                              │
   │                              ├─► Handler 处理
   │                              │       │
   │                              │       ├─► 权限检查
   │                              │       ├─► 业务逻辑
   │                              │       └─► 修改 Actor 状态
   │                              │
   │                              ├─► 数据库操作 (通过 DBAgent)
   │                              │
   │                              └─► 发送响应
   │                              │
   │◄──── TCP 响应包 ──────────────│
```

### PacketHandler 示例

```cpp
// 移动处理
void HandleMove(Session& session, Packet* packet)
{
    // 1. 解析
    auto moveReq = C2G_Move.GetRootAs(packet->payload());

    // 2. 验证
    uint64_t actorId = session.GetActorId();
    Actor* actor = ACTOR_MGR->GetActor(actorId);
    if (!actor || actor->isDead())
        return;

    // 3. 检查移动冷却/速度
    if (!actor->CanMove())
        return;

    // 4. 更新位置
    Vector3 newPos(moveReq->x(), moveReq->y(), moveReq->z());
    actor->SetPosition(newPos);
    actor->SetDirection(moveReq->dir());

    // 5. 广播给周围玩家
    auto notify = G2C_MoveNotify(actorId, newPos, moveReq->dir());
    ACTOR_MGR->SendToVisiblePlayers(actor, &notify);
}
```

### DB 操作流程

```cpp
// 数据库操作通过 DBAgent

// 发送 DB 请求
void SendToDBAgent(Session& session, uint16_t pid, FlatBuffersBuilder& fbb)
{
    DBAGENT_SRV_MGR->SendPacket(session.GetActorId(), pid, fbb);
}

// DBAgent 响应处理
void HandleDBAgentResponse(Session& session, Packet* packet)
{
    auto response = DB2G_LoadCharacter.GetRootAs(packet->payload());

    uint64_t userUid = session.GetActorId();
    CharacterInfo* info = new CharacterInfo();

    // 填充数据
    info->userUid = userUid;
    info->level = response->level();
    info->exp = response->exp();
    // ...

    // 创建 Character
    Character* character = ACTOR_MGR->CreateCharacter(info, terrain);
    session.SetActorId(character->GetActorId());
}
```

---

下一章: [第七阶段：数据库层 + DBAgent](./07_数据库层_dbagent.md)
