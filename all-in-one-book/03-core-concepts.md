# 第3章 核心概念

> **阅读目标**：深入理解 Centrifugo 的 5 个核心概念——频道、发布、订阅、在线状态、历史消息

---

## 3.1 概念全景图

```
┌─────────────────────────────────────────────────────────────┐
│                    Centrifugo 核心概念                       │
│                                                             │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐              │
│  │  频道     │    │  发布     │    │  订阅     │              │
│  │ Channel  │    │ Publish  │    │Subscribe │              │
│  │          │    │          │    │          │              │
│  │ 消息的    │    │ 往频道    │    │ 监听频道  │              │
│  │ 容器/地址 │    │ 发送消息  │    │ 接收消息  │              │
│  └──────────┘    └──────────┘    └──────────┘              │
│                                                             │
│  ┌──────────┐    ┌──────────┐                              │
│  │ 在线状态  │    │ 历史消息  │                              │
│  │ Presence │    │ History  │                              │
│  │          │    │          │                              │
│  │ 谁在频道  │    │ 最近的    │                              │
│  │ 里在线    │    │ N 条消息  │                              │
│  └──────────┘    └──────────┘                              │
│                                                             │
│  额外概念：                                                  │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐              │
│  │ 命名空间  │    │ 用户频道  │    │ 断线恢复  │              │
│  │Namespace │    │User Chan │    │ Recovery │              │
│  └──────────┘    └──────────┘    └──────────┘              │
└─────────────────────────────────────────────────────────────┘
```

---

## 3.2 频道（Channel）

### 什么是频道

频道是消息流转的 **地址**。可以理解为：
- 邮件系统里的 **邮箱地址**
- 消息队列里的 **Topic**
- 聊天软件里的 **群组**

所有消息都通过频道来分发：发布者把消息发到频道，订阅者从频道接收消息。

### 频道命名规则

```
频道名称是一个字符串，最长 255 个字符（默认）

合法的频道名：
  "chat"                    ← 简单名称
  "chat:room_42"            ← 带命名空间
  "user:1001"               ← 用户相关
  "stock:AAPL"              ← 业务相关
  "game:match_789:state"    ← 多级名称

特殊前缀：
  "$chat:room_42"           ← 私有频道（$ 前缀）
  "#user_1001"              ← 用户频道（# 分隔符）
```

### 频道的分隔符约定

Centrifugo 使用特殊字符来赋予频道名称不同的含义：

```
分隔符         默认值    作用
──────────    ──────    ────────────────────────────
命名空间分隔符  :        分隔命名空间和频道名
私有频道前缀    $        标记需要订阅授权的频道
用户分隔符      #        标记个人频道
用户列表分隔符  ,        在用户频道中分隔多个用户 ID
```

**例子解析**：

```
"chat:room_42"
  │     │
  │     └─ 频道名: room_42
  └─ 命名空间: chat

"$news:breaking"
 │  │      │
 │  │      └─ 频道名: breaking
 │  └─ 命名空间: news
 └─ 私有频道标记（需要授权才能订阅）

"#user_1001"
 │     │
 │     └─ 用户 ID: 1001
 └─ 用户频道标记（只有该用户能订阅）
```

### 频道类型对比

| 类型 | 示例 | 谁能订阅 | 使用场景 |
|------|------|---------|---------|
| **普通频道** | `news` | 所有人 | 公开消息广播 |
| **命名空间频道** | `chat:room_42` | 根据命名空间配置 | 分类管理 |
| **私有频道** | `$chat:room_42` | 需要订阅 token 或代理授权 | 需要权限的频道 |
| **用户频道** | `#user_1001` | 仅该用户 | 个人通知 |

---

## 3.3 发布（Publish）

### 什么是发布

发布就是 **往频道里发送一条消息**。消息是任意的 JSON 对象或二进制数据。

### 谁来发布？

```
发布来源                      方式                         常见度
──────────────               ─────                        ──────
你的后端服务器                HTTP API / gRPC API           ⭐⭐⭐⭐⭐ 最常用
管理界面                     Admin UI 操作面板               ⭐⭐ 测试用
消息队列消费者                Kafka/PostgreSQL 消费           ⭐⭐⭐ 高级场景
客户端                       客户端 SDK（需要配置允许）       ⭐⭐ 特定场景
```

### 发布消息的数据格式

```json
{
  "channel": "chat:room_42",
  "data": {
    "type": "message",
    "user": "张三",
    "text": "大家好！",
    "timestamp": 1704067200
  }
}
```

**重要**：Centrifugo **不关心** `data` 里面是什么，它只负责把 data 原样传递给订阅者。消息的业务含义完全由你定义。

### 发布 API 的 curl 示例

```bash
# 基本发布
curl -X POST http://localhost:8000/api/publish \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{
    "channel": "chat:room_42",
    "data": {"text": "Hello!", "user": "张三"}
  }'

# 广播到多个频道
curl -X POST http://localhost:8000/api/broadcast \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{
    "channels": ["chat:room_1", "chat:room_2", "chat:room_3"],
    "data": {"text": "系统公告：服务器将在 10 分钟后维护"}
  }'
```

### 发布响应

```json
// 成功
{
  "result": {
    "offset": 5,     // 消息在历史中的偏移量（如果频道开启了历史）
    "epoch": "abc123" // 历史的时代标识
  }
}

// 频道未开启历史时
{
  "result": {}
}
```

---

## 3.4 订阅（Subscribe）

### 什么是订阅

订阅是客户端 **告诉 Centrifugo「我要接收这个频道的消息」** 的动作。

### 订阅方式

Centrifugo 支持两种订阅方式：

```
方式1：客户端主动订阅（Client-side Subscribe）
──────────────────────────────────────────────

客户端代码中显式调用订阅：

  const sub = centrifuge.newSubscription('chat:room_42');
  sub.subscribe();

适用场景：客户端需要动态选择订阅哪些频道
例如：用户进入某个聊天室时订阅该聊天室

──────────────────────────────────────────────

方式2：服务端订阅（Server-side Subscribe）
──────────────────────────────────────────────

在 JWT token 中指定用户应该订阅哪些频道：

  {
    "sub": "user123",
    "channels": ["notifications", "chat:general"]
  }

或者通过 API 强制订阅：

  curl -X POST http://localhost:8000/api/subscribe \
    -d '{"user": "user123", "channel": "chat:room_42"}'

适用场景：服务端决定用户应该订阅什么
例如：用户登录后自动订阅个人通知频道

──────────────────────────────────────────────
```

### 订阅生命周期

```
客户端                      Centrifugo
  │                            │
  │─── subscribe ─────────────▶│  ← 发起订阅
  │                            │
  │                            │── 检查权限（token/proxy）
  │                            │
  │◀── subscribed ─────────────│  ← 订阅成功
  │    (可能包含恢复的历史消息)  │
  │                            │
  │◀── publication ────────────│  ← 收到消息
  │◀── publication ────────────│  ← 收到消息
  │◀── publication ────────────│  ← 收到消息
  │                            │
  │─── unsubscribe ───────────▶│  ← 取消订阅
  │                            │
  │◀── unsubscribed ───────────│  ← 取消成功
  │                            │
```

---

## 3.5 在线状态（Presence）

### 什么是在线状态

在线状态功能让你知道 **某个频道里当前有哪些用户在线**。

> ⚠️ 在线状态需要在频道配置中 **显式开启**，默认是关闭的。

### 启用在线状态

```json
{
  "channel": {
    "namespaces": [
      {
        "name": "chat",
        "presence": true
      }
    ]
  }
}
```

### 查询在线状态 API

```bash
# 查看频道中的在线用户
curl -X POST http://localhost:8000/api/presence \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{"channel": "chat:room_42"}'
```

响应：

```json
{
  "result": {
    "presence": {
      "conn_id_1": {
        "user": "user_1001",
        "client": "abc-def-123"
      },
      "conn_id_2": {
        "user": "user_1002",
        "client": "xyz-789-456"
      }
    }
  }
}
```

### 在线状态统计

```bash
# 只获取统计数字（不获取完整用户列表，性能更好）
curl -X POST http://localhost:8000/api/presence_stats \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{"channel": "chat:room_42"}'
```

响应：

```json
{
  "result": {
    "num_clients": 42,
    "num_users": 38
  }
}
```

> **💡 注意**：`num_clients` 是连接数（一个用户可能多设备连接），`num_users` 是去重后的用户数。

### Join/Leave 通知

可以配置当用户加入/离开频道时自动通知其他订阅者：

```json
{
  "channel": {
    "namespaces": [
      {
        "name": "chat",
        "presence": true,
        "join_leave": true
      }
    ]
  }
}
```

```
客户端收到的 join 事件示例：

{
  "type": "join",
  "info": {
    "user": "user_1003",
    "client": "new-conn-id"
  }
}
```

> **⚠️ 性能提示**：在大型频道（>100 人）中开启 `join_leave` 会产生大量通知消息。
> 建议只在小型频道（如私聊、小群）中使用。大型频道用 `presence_stats` 轮询代替。

---

## 3.6 历史消息（History）

### 什么是历史消息

Centrifugo 可以在内存（或 Redis）中 **缓存频道的最近 N 条消息**。新订阅的客户端可以拿到历史消息。

> ⚠️ 这不是持久化存储！历史消息是临时缓存，有 TTL（过期时间）。
> 如果需要永久保存消息，请在你的后端/数据库中存储。

### 启用历史消息

```json
{
  "channel": {
    "namespaces": [
      {
        "name": "chat",
        "history_size": 100,
        "history_ttl": "24h"
      }
    ]
  }
}
```

| 参数 | 说明 | 建议值 |
|------|------|--------|
| `history_size` | 保留最近 N 条消息 | 聊天：100-1000，通知：10-50 |
| `history_ttl` | 消息过期时间 | 聊天：24h，通知：1h |

### 查询历史消息 API

```bash
# 获取频道的历史消息
curl -X POST http://localhost:8000/api/history \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{
    "channel": "chat:room_42",
    "limit": 10
  }'
```

响应：

```json
{
  "result": {
    "publications": [
      {
        "offset": 1,
        "data": {"text": "第一条消息", "user": "user_1"},
        "info": {"user": "user_1", "client": "conn_1"}
      },
      {
        "offset": 2,
        "data": {"text": "第二条消息", "user": "user_2"},
        "info": {"user": "user_2", "client": "conn_2"}
      }
    ],
    "epoch": "abc123",
    "offset": 2
  }
}
```

### 分页获取历史

```bash
# 使用 since 参数分页
curl -X POST http://localhost:8000/api/history \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{
    "channel": "chat:room_42",
    "limit": 10,
    "since": {
      "offset": 5,
      "epoch": "abc123"
    }
  }'
```

---

## 3.7 断线恢复（Recovery）

### 什么是断线恢复

当客户端因为网络不稳定而断线后，重新连接时可以 **自动获取断线期间遗漏的消息**。

```
正常场景：                              断线恢复场景：

消息1 ──▶ 客户端收到 ✅                 消息1 ──▶ 客户端收到 ✅
消息2 ──▶ 客户端收到 ✅                 消息2 ──▶ 客户端收到 ✅
消息3 ──▶ 客户端收到 ✅                 ─── 断线 ───
消息4 ──▶ 客户端收到 ✅                 消息3 ──▶ ❌ 丢失
                                        消息4 ──▶ ❌ 丢失
没有断线恢复：                           ─── 重连 ───
  4 条消息全部遗漏                       消息3 ──▶ 自动补发 ✅
                                        消息4 ──▶ 自动补发 ✅
有断线恢复：                             消息5 ──▶ 正常接收 ✅
  自动补发遗漏的消息
```

### 启用断线恢复

断线恢复依赖历史消息功能，所以 **必须同时开启历史消息**：

```json
{
  "channel": {
    "namespaces": [
      {
        "name": "chat",
        "history_size": 100,
        "history_ttl": "5m",
        "force_recovery": true
      }
    ]
  }
}
```

客户端 SDK 也需要开启恢复模式（大部分 SDK 默认支持）。

### 恢复的工作原理

```
客户端                     Centrifugo                   Redis/Memory
  │                           │                            │
  │── 首次订阅 ──────────────▶│                            │
  │◀── offset=5, epoch=abc ──│                            │
  │                           │                            │
  │◀── msg(offset=6) ────────│                            │
  │◀── msg(offset=7) ────────│                            │
  │                           │                            │
  ╳ 断线                      │                            │
  │                           │── 缓存消息 ──────────────▶│
  │                           │   offset=8                 │
  │                           │   offset=9                 │
  │                           │                            │
  │── 重连，带上              │                            │
  │   offset=7, epoch=abc ──▶│                            │
  │                           │── 查询 offset>7 ─────────▶│
  │                           │◀── 返回 offset 8,9 ───────│
  │◀── 补发 msg(offset=8) ──│                            │
  │◀── 补发 msg(offset=9) ──│                            │
  │                           │                            │
  │── 恢复完成，继续正常接收  │                            │
```

> **⚠️ 恢复的限制**：
> 1. 只能恢复 `history_size` 范围内的消息
> 2. 只能恢复 `history_ttl` 时间内的消息
> 3. 如果断线太久或消息太多，可能无法完全恢复

---

## 3.8 命名空间（Namespace）

### 什么是命名空间

命名空间让你对 **不同类型的频道设置不同的配置**。

```
没有命名空间时：
  所有频道共用一套配置（历史大小、在线状态等）

有命名空间后：
  chat:*     → 开启历史(100条)、在线状态、断线恢复
  notify:*   → 开启历史(10条)、不要在线状态
  live:*     → 不需要历史、不需要在线状态（纯推送）
```

### 配置命名空间

```json
{
  "channel": {
    "without_namespace": {
      "presence": false,
      "history_size": 0,
      "history_ttl": "0s"
    },
    "namespaces": [
      {
        "name": "chat",
        "presence": true,
        "join_leave": true,
        "history_size": 100,
        "history_ttl": "24h",
        "force_recovery": true
      },
      {
        "name": "notify",
        "history_size": 10,
        "history_ttl": "1h"
      },
      {
        "name": "live",
        "presence": false,
        "history_size": 0
      }
    ]
  }
}
```

### 命名空间如何匹配

```
频道名                命名空间        使用的配置
─────                ────────        ─────────
"chat:room_42"       chat            chat 命名空间的配置
"chat:room_99"       chat            chat 命名空间的配置
"notify:user_1001"   notify          notify 命名空间的配置
"live:stream_1"      live            live 命名空间的配置
"random"             (无)            without_namespace 的配置
```

---

## 3.9 用户频道（User Channel）

### 自动个人频道

Centrifugo 支持让用户 **自动订阅** 一个以自己 ID 命名的个人频道：

```json
{
  "client": {
    "subscribe_to_user_personal_channel": {
      "enabled": true
    }
  }
}
```

当用户 `user_1001` 连接时，会自动订阅频道 `#user_1001`。

然后你的后端可以精确推送消息给某个用户：

```bash
# 推送通知给用户 1001
curl -X POST http://localhost:8000/api/publish \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{
    "channel": "#user_1001",
    "data": {"type": "notification", "text": "你有一条新订单"}
  }'
```

---

## 3.10 服务端订阅 vs 客户端订阅

这是 Centrifugo 中一个重要的设计选择：

```
┌──────────────────────────────────────────────────────────────┐
│                   客户端订阅                                  │
│  Client-side Subscribe                                       │
│                                                              │
│  客户端代码中主动订阅：                                       │
│    const sub = centrifuge.newSubscription('chat:room_42');    │
│    sub.subscribe();                                          │
│                                                              │
│  ✅ 灵活：客户端决定订阅什么                                  │
│  ✅ 动态：可以随时订阅/取消                                   │
│  ⚠️ 需要处理频道权限                                         │
│                                                              │
│  适用：聊天室（用户选择进入哪个房间）                          │
├──────────────────────────────────────────────────────────────┤
│                   服务端订阅                                  │
│  Server-side Subscribe                                       │
│                                                              │
│  方式1：在 JWT 中指定频道                                     │
│    token payload: { "channels": ["notify", "chat:general"] } │
│                                                              │
│  方式2：通过 API 强制订阅                                     │
│    POST /api/subscribe                                       │
│    { "user": "user123", "channel": "notify" }                │
│                                                              │
│  ✅ 简单：客户端不需要知道频道名                              │
│  ✅ 安全：服务端完全控制                                      │
│  ⚠️ 不够灵活：连接时就确定了                                 │
│                                                              │
│  适用：通知频道（用户登录就自动订阅）                          │
└──────────────────────────────────────────────────────────────┘
```

---

## 3.11 消息格式

### JSON 格式（默认）

```json
{
  "channel": "chat:room_42",
  "data": {
    "type": "message",
    "user": "张三",
    "text": "大家好！",
    "attachments": [
      {"type": "image", "url": "https://example.com/photo.jpg"}
    ]
  }
}
```

### 二进制格式（Protobuf）

对于高性能场景，Centrifugo 也支持 Protobuf 二进制消息。需要在配置和客户端 SDK 中开启。

---

## 3.12 Delta 压缩

Centrifugo 支持消息的 **增量压缩**（Delta Compression）。对于频繁更新的数据（如股票行情、游戏状态），每次只传输变化的部分，可以显著减少带宽。

```json
{
  "channel": {
    "namespaces": [
      {
        "name": "stock",
        "delta_publish": true
      }
    ]
  }
}
```

```
没有 Delta 压缩：
  消息1: {"AAPL": 150.00, "GOOGL": 2800.00, "MSFT": 300.00}  ← 完整数据
  消息2: {"AAPL": 150.05, "GOOGL": 2800.00, "MSFT": 300.00}  ← 完整数据
  消息3: {"AAPL": 150.10, "GOOGL": 2801.00, "MSFT": 300.00}  ← 完整数据

有 Delta 压缩：
  消息1: {"AAPL": 150.00, "GOOGL": 2800.00, "MSFT": 300.00}  ← 完整数据
  消息2: +{"AAPL": 150.05}                                     ← 只有差异
  消息3: +{"AAPL": 150.10, "GOOGL": 2801.00}                  ← 只有差异
```

---

## 3.13 核心概念速查表

| 概念 | 一句话解释 | 默认开启？ | 需要配置？ |
|------|-----------|-----------|-----------|
| **频道** | 消息流转的地址 | ✅ | 仅需命名 |
| **发布** | 往频道发消息 | ✅ | 需要 API key |
| **订阅** | 监听频道接收消息 | ✅ | 可能需要 token |
| **在线状态** | 查看频道内在线用户 | ❌ | `presence: true` |
| **Join/Leave** | 上下线通知 | ❌ | `join_leave: true` |
| **历史消息** | 缓存最近 N 条消息 | ❌ | `history_size` + `history_ttl` |
| **断线恢复** | 重连后补发遗漏消息 | ❌ | `force_recovery: true` + 历史 |
| **命名空间** | 不同频道类型不同配置 | ❌ | `namespaces: [...]` |
| **用户频道** | 个人专属频道 | ❌ | `subscribe_to_user_personal_channel` |
| **Delta 压缩** | 只传输数据差异 | ❌ | `delta_publish: true` |

---

## 3.14 业务场景对照表

| 业务场景 | 需要的功能组合 |
|---------|--------------|
| **聊天室** | 频道 + 订阅 + 历史 + 在线状态 + 断线恢复 |
| **私聊** | 私有频道 + 订阅 token + 历史 + 断线恢复 |
| **系统通知** | 用户频道 + 服务端订阅 + 历史 |
| **实时行情** | 频道 + 订阅 + Delta 压缩 |
| **监控面板** | 频道 + 订阅（不需要历史）|
| **直播弹幕** | 频道 + 订阅（不需要历史和在线状态）|
| **协同编辑** | 频道 + 订阅 + 在线状态 + 断线恢复 |

---

## 3.15 本章小结

```
记住这个核心流转：

  后端 ──[Publish]──▶ 频道 ──[Subscribe]──▶ 客户端
                       │
                  ┌────┴────┐
                  │ 可选功能 │
                  ├─────────┤
                  │ 在线状态 │
                  │ 历史消息 │
                  │ 断线恢复 │
                  │ Delta   │
                  └─────────┘
```

**下一章**：学习如何做用户认证 → [第4章 认证与鉴权](04-authentication.md)
