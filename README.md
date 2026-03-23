# Astral Party Security Research

> 吉星派对（星引擎Party）客户端安全研究与游戏数据逆向工程

[![Game](https://img.shields.io/badge/Game-Astral%20Party%20v3.0.1-blue)]()
[![Engine](https://img.shields.io/badge/Engine-Unity%20IL2CPP%20+%20HybridCLR-green)]()
[![UI](https://img.shields.io/badge/UI-FairyGUI-orange)]()
[![Network](https://img.shields.io/badge/Network-WebSocket%20+%20Protobuf-purple)]()

---

## 项目概述

本仓库记录了对 **Astral Party（吉星派对/星引擎Party）** 的客户端安全研究成果，包括：

- **12 个安全漏洞**的完整分析（含 GM 接口、协议漏洞、内存操控）
- **游戏数据逆向**：22 个角色、55+ 张卡牌、151 个筹码、13 张地图
- **RPC 协议分析**：45 个 C2S 命令、完整的网络栈架构
- **内存布局**：关键单例模式、字段偏移、UI 树结构
- **262 个 UI 按钮**的功能分类与 RPC 安全性分析

## 技术架构

```
┌─────────────────────────────────────────────────────┐
│                   Astral Party Client                │
├─────────────────────────────────────────────────────┤
│  UI Layer          │  FairyGUI (GRoot → GComponent)  │
│  Hot-Update        │  HybridCLR (解释执行)            │
│  Game Logic        │  GameLogicManager (40+ 子系统)    │
│  RPC Layer         │  C2SRPC → RPCMsgManager          │
│  Protocol          │  Protocal → Frame → EnOrDecode   │
│  Transport         │  WebSocket (ManagedWebSocket)     │
│  Runtime           │  Unity IL2CPP (AOT + Interpreter) │
└─────────────────────────────────────────────────────┘
```

## 漏洞概览

| # | 漏洞名称 | 严重等级 | 类型 | 状态 |
|---|---------|---------|------|------|
| 1 | [CheatItemC2S GM 物品接口](#1-cheatitemc2s) | 🔴 严重 | RPC 协议 | 未验证 |
| 2 | [DevChargeC2S 开发者充值](#2-devchargec2s) | 🔴 严重 | RPC 协议 | 未验证 |
| 3 | [GmC2S 通用 GM 命令](#3-gmc2s) | 🔴 严重 | RPC 协议 | 未验证 |
| 4 | [GMPlayerSettingC2S 属性设置](#4-gmplayersettingc2s) | 🟡 高危 | RPC 协议 | 未验证 |
| 5 | [TransferStarDiscC2S 星盘转移](#5-transferstardiscc2s) | 🟡 高危 | RPC 协议 | 未验证 |
| 6 | [devPoint_ 骰子控制](#6-devpoint) | 🟡 高危 | 协议字段 | 部分验证 |
| 7 | [GMData.SetDicePoint 单机骰子](#7-gmdatasetdicepoint) | 🟢 低危 | 内存调用 | 可利用 |
| 8 | [UIGM 面板系统](#8-uigm-面板) | 🟡 高危 | UI 注入 | 未验证 |
| 9 | [SysRoomAddExpC2S 房间经验](#9-sysroomaddexpc2s) | 🟡 高危 | RPC 协议 | 未验证 |
| 10 | [SysRecoupItemC2S 补偿物品](#10-sysrecoupitemc2s) | 🟡 高危 | RPC 协议 | 未验证 |
| 11 | [SysSendMailC2S 系统邮件](#11-syssendmailc2s) | 🟡 高危 | RPC 协议 | 未验证 |
| 12 | [MoveC2S.forceDir_ 强制方向](#12-movec2sforcedir) | 🟢 低危 | 协议字段 | 未验证 |

---

## 漏洞详情

### 1. CheatItemC2S

**GM 物品生成接口** — 允许向服务器发送物品创建请求

```
命名空间: party.protocol
RPC 类:   Core.Net.CheatItemC2SRPC
方法:     CheatItemC2SCall(CheatItemC2S msg)
```

| 偏移 | 字段 | 类型 | 说明 |
|------|------|------|------|
| +0x18 | IsAll | bool | 获取全部物品 |
| +0x1C | ItemId | int32 | 目标物品 ID |
| +0x20 | ItemCount | int32 | 数量 |
| +0x24 | IsGacha | bool | 抽卡模式（绕过正常抽卡流程） |
| +0x28 | Exp | int32 | 经验值 |

**风险评估**：服务端可能有 GM 权限校验。如果校验缺失或可绕过，可无限生成任意物品。

---

### 2. DevChargeC2S

**开发者充值接口** — 生产环境可能未关闭的调试接口

```
命名空间: party.protocol
RPC 类:   Core.Net.DevChargeC2SRPC
回调处理: StoreLogic.OnDevChargeS2CServerCallBack
```

**风险评估**：🔴 如果生产环境未禁用此接口，可实现免费充值。直接影响付费系统。

---

### 3. GmC2S

**通用 GM 命令接口** — 功能未完全逆向

```
命名空间: party.protocol
RPC 类:   Core.Net.GmC2SRPC
关联:     SysGmChangeNameC2S (GM 改名)
配置:     Core.GMConfig
```

---

### 4. GMPlayerSettingC2S

**GM 玩家属性设置** — 修改玩家属性

```
偏移 +0x18: RoomActionLimitAllOpen (bool) — 解除房间行为限制
```

---

### 5. TransferStarDiscC2S

**星盘转移** — 可能存在负数/溢出漏洞

```
偏移 +0x18: Count (int32) — 转移数量
RPC 类: Core.Net.TransferStarDiscC2SRPC
回调:   StoreLogic.OnTransferStarDiscS2CServerCallBackAsync
```

---

### 6. devPoint_

**骰子点数控制** — 多个 C2S 消息中存在的调试字段

| 消息类型 | 偏移 | 说明 |
|---------|------|------|
| ThrowDiceC2S | +0x20 | 移动骰子（0=随机, 1-6=指定） |
| BattleChoiceC2S | +0x20 | 战斗骰子 |
| ChoiceDirectionC2S | +0x24 | 方向选择骰子 |

**验证状态**：客户端可正常构造消息，但服务端生产环境可能忽略此字段。

---

### 7. GMData.SetDicePoint

**单机模式骰子控制** — 零风险，仅影响本地

```
类:   SinglePlayer.GamePlay.GMData
方法: SetDicePoint(int value)
其他: SetLockGameProgress(bool), SetScore(int)
```

通过 `il2cpp_runtime_invoke` 在单机模式（SceneType=6）下直接调用即可。

---

### 8. UIGM 面板

**完整的 GM 工具 UI 系统**

| 类名 | 功能 |
|------|------|
| UIGMWindow | GM 主窗口（29 个方法） |
| UIGM_Button_Operate | 操作按钮 |
| UIGM_Button_Server | 服务器操作 |
| UIGM_Button_Single | 单机操作 |
| UIGM_Com_SetAttr | 属性修改 |
| UIGM_Com_Set_Attr_Buff | Buff 设置 |

**GMWindow 关键方法**：`CallExp`, `CallItemChange`, `CallAllItemChange`, `CallGacha`, `CallGameOver`, `CallPerform`, `RefreshCards`, `SetSlotValue`

---

### 9-11. Sys* 系列接口

| 接口 | 功能 | 风险 |
|------|------|------|
| SysRoomAddExpC2S | 房间加经验 | 可能需要 GM 权限 |
| SysRecoupItemC2S | 物品补偿发放 | 可能需要 GM 权限 |
| SysSendMailC2S | 发送系统邮件（可附带奖励） | 可能需要 GM 权限 |

---

### 12. MoveC2S.forceDir_

**强制指定移动方向** — 绕过岔路口限制

```
MoveC2S:
  +0x18: info_ (ActionInfo)
  +0x20: direction_ (int32)    — 方向值
  +0x24: forceDir_  (bool)     — 强制方向标志
  +0x28: path_      (RepeatedField<Int32>)
```

---

## 技术限制

### HybridCLR RPC 调用问题（Error 11018）

通过 `il2cpp_runtime_invoke` 调用 HybridCLR 热更新的 C2SRPC 方法时，服务器返回"错误命令号 11018"。

**原因分析**：
- HybridCLR 解释器在非原生上下文中执行时，命令号（Handle）字段可能未正确初始化
- RPC 消息经过 `RPCMsgManager → Protocal → Frame → EnOrDecode → WebSocket` 链路，任何一环出错都会导致服务端拒绝

**受影响的操作**：
- FightWindow（战斗选择）
- LandGambleWindow（猜大小）
- LandDivinationWindow（占卜）
- 所有直接 C2SRPC.C2SCall 调用

**不受影响的操作**：
- 移动按钮（ThrowDice 通过 FairyGUI EventListener.Call 触发）
- 商店/事件/彩票等 UI 弹窗的关闭按钮

---

## 内存布局

### 单例发现模式

```
SimpleSingletonProvider<T>:
  class → parent(+0x58) → static_fields(+0xB8) → _inst(+0x00)
```

### GameLogicManager 子系统偏移

| 偏移 | 子系统 | 说明 |
|------|--------|------|
| +0x10 | initialize | 是否在对局中 (bool) |
| +0x20 | RoomLogic | 房间管理 |
| +0x28 | BattleLogic | 战斗逻辑 |
| +0x30 | CardLogic | 卡牌管理 |
| +0x40 | FightLogic | 战斗处理 |
| +0x48 | LandLogic | 地块事件 |
| +0x68 | MatchLogic | 匹配系统 |
| +0x88 | ActionLogic | 玩家行动 |
| +0xB0 | MailLogic | 邮件系统 |
| +0xB8 | StoreLogic | 商店系统 |
| +0xD8 | TaskLogic | 任务系统 |
| +0xE0 | ActivityLogic | 活动系统 |

### BattleProperty 字段偏移（ReactiveProperty，值在 obj+0x10）

| 偏移 | 字段 | 类型 |
|------|------|------|
| +0x18 | Level | ReactiveProperty\<int\> |
| +0x20 | Gold | ReactiveProperty\<int\> |
| +0x28 | Position | int（直接值） |
| +0x30 | HP | ReactiveProperty\<int\> |
| +0x38 | ATK | ReactiveProperty\<int\> |
| +0x40 | DEF | ReactiveProperty\<int\> |
| +0x58 | Stars | ReactiveProperty\<int\> |

### ActionLogic 字段

| 偏移 | 字段 | 说明 |
|------|------|------|
| +0x18 | playerAction | PlayerActionEnum |
| +0x20 | usableCards | RepeatedField\<Int32\> |
| +0x30 | throwDiceSn | Int64（序列号） |

---

## RPC 协议

### 网络栈

```
应用层  party.protocol.XxxC2S    (Protobuf 消息)
  ↓
RPC层   Core.Net.XxxC2SRPC       (RPC 包装器)
  ↓
管理层  Core.Net.RPCMsgManager   (会话/回调/序列号)
  ↓
协议层  Core.Net.Protocal        (命令号 + 加密)
  ↓
帧层    Core.Net.Frame           (长度 + 载荷)
  ↓
加密层  Core.Net.EnOrDecode      (加解密)
  ↓
传输层  Core.Net.USocket → WebSocket
```

### 完整 C2S 命令列表（45 个）

```
AbandonCard      AffirmHero       AskBattle        BattleChoice
BattleThrowDice  BattleUseCard    BombThrowDice    ChangeRoom
CheatItem        ChoiceDirection  Connect          CreateRoom
EventThrowDice   ExitRoom         GambleThrowDic   Gm
Heartbeat        JoinRoom         LandChoiceTarget LandNo
LotteryChoice    Move             MoveAgain        PlayerUseItem
Pursuit          QueryRoom        RefreshRoomState RollGold
SearchRoom       SendChat         ShopBuy          StartGamble
StartGame        StopOrContinue   SyncRoom         ThrowDice
ThrowDiceResult  TriggerDestiny   TriggerDivination
TriggerEvent     TriggerHospital  UseEffectCard    UseQuickCard
```

### 关键 C2S 消息结构

```protobuf
// 掷骰子
ThrowDiceC2S {
  ActionInfo info = 1;   // +0x18 (sn + useTime)
  int32 devPoint = 2;    // +0x20 (0=随机, 1-6=指定)
  bool isNoOper = 3;     // +0x24
}

// 战斗选择
BattleChoiceC2S {
  ActionInfo info = 1;   // +0x18
  int32 devPoint = 2;    // +0x20
  bool dodge = 3;        // +0x24 (闪避)
  bool noDodge = 4;      // +0x25 (不闪避)
}

// 移动方向
MoveC2S {
  ActionInfo info = 1;   // +0x18
  int32 direction = 2;   // +0x20
  bool forceDir = 3;     // +0x24
  repeated int32 path = 4; // +0x28
}

// 使用效果卡
UseEffectCardC2S {
  ActionInfo info = 1;   // +0x18
  int32 cardId = 2;      // +0x28
  repeated int64 targetIds = 3; // +0x30
  bool useSkill = 4;     // +0x44
  int32 skillId = 5;     // +0x48
}
```

---

## 游戏数据

### 角色（22 个已收录）

| 角色 | HP | ATK | DEF | 主动技能 | 被动技能 |
|------|----|----|-----|---------|---------|
| 帕露南 | 10 | 1 | 2 | 网络购物 (CD:2) | 传奇商人 |
| 芬妮 | 10 | 1 | 2 | 麻烦制造者 (CD:3) | 华点发现! |
| 阿兰娜 | 9 | 1 | 1 | 铁处女 (CD:3) | 祷告 |
| 米米 | 9 | 1 | 1 | 商品补货 (CD:3) | 过期回收 |
| ... | | | | | |

完整数据见 [`wiki_bilibili/characters.json`](wiki_bilibili/characters.json)

### 卡牌系统

- **战斗卡**：攻击(中/大/特) + 防御(中/大/特) + PVE 专属
- **效果卡**：自身效果 / 指向对手 / 地块放置 / 反制
- **特殊卡**：角色专属 + 地图专属

完整数据见 [`wiki_bilibili/cards.json`](wiki_bilibili/cards.json)

### 筹码（151 个）

分类：基础系 / 星光系 / 治愈系 / 标记系 / 地图专属

完整数据见 [`wiki_bilibili/chips.json`](wiki_bilibili/chips.json)

### 地图

| 类型 | 数量 | 地图 |
|------|------|------|
| PVP | 6 | 游乐园、樱花小镇、十字路口站、魅影都市、礼物广场、丛林冒险 |
| PVE | 7 | Dreama、魂灵祭、水乡、魔法学院、龙宫游乐园、鬼巷、庭院花园 |

---

## UI 按钮分析（262 个按钮类）

### 安全性分类

| 分类 | 数量 | 说明 |
|------|------|------|
| **安全**（纯 UI 操作） | 198 | 关闭窗口、确认、显示信息 |
| **RPC 触发**（发送网络请求） | 52 | 战斗、购买、选择等 |
| **危险**（触发 Error 11018） | 12 | FightWindow、GambleWindow 等 |

### 自动化优先级

| 优先级 | 窗口 | 按钮数 | 推荐操作 |
|--------|------|--------|---------|
| Critical | FightWindow | 6 | 跳过（超时默认防御） |
| Critical | LoseCardWindow | 12 | 点击第一张 |
| High | BattleSettlementPanel | 20 | 点击确认 |
| High | LandEventWindow | 2 | 点击确认 |
| Medium | LandShopWindow | 8-10 | 购买/返回 |
| Medium | CardWindow | 4-7 | 使用/取消 |
| Low | LandLotteryWindow | 13-26 | 随机选号 |

完整数据见 [`button_analysis.json`](button_analysis.json)

---

## 文件说明

```
astral-party-vulns/
├── README.md                      # 本文件
├── gm_exploits.json               # GM/作弊接口详情
├── vulnerability_report.json      # 完整漏洞扫描报告
├── singleton_discovery.json       # 内存偏移和单例模式
├── game_addresses.json            # 网络架构和 RPC 地址
├── button_analysis.json           # 262 个 UI 按钮分类
└── wiki_bilibili/
    ├── characters.json            # 22 个角色完整数据
    ├── cards.json                 # 卡牌系统
    ├── chips.json                 # 151 个筹码
    ├── maps.json                  # 13 张地图
    ├── game_mechanics.json        # 战斗公式和闪避概率
    ├── cooperative_challenge.json # PVE 合作模式
    ├── monsters.json              # 怪物数据
    ├── activities.json            # 活动记录
    ├── balance_changes.json       # 平衡调整
    ├── gifts.json                 # 礼物系统
    └── wiki_links.json            # Wiki 页面索引
```

---

## 免责声明

本仓库仅用于**安全研究和教育目的**。所有研究均在本地 CTF 环境中进行。

- 请勿将研究成果用于破坏游戏公平性或损害其他玩家利益
- 发现的漏洞应通过正规渠道向开发者报告
- 使用本仓库内容造成的任何后果由使用者自行承担

---

*Research conducted: 2026-03-23 | Game version: 3.0.1 | Engine: Unity IL2CPP + HybridCLR*
