# 第5章 服务端 API

> **阅读目标**：掌握 Centrifugo 全部 HTTP API，能用 curl 完成所有服务端操作

---

## 5.1 API 概览

Centrifugo 提供 HTTP API 让你的后端服务器控制消息分发。所有 API 都是 **POST** 请求，默认路径前缀为 `/api`。

```
API 端点总览：
┌────────────────────┬──────────────────────────────────────────┐
│ 端点               │ 功能                                      │
├────────────────────┼──────────────────────────────────────────┤
│ /api/publish       │ 发布消息到频道                             │
│ /api/broadcast     │ 发布消息到多个频道                         │
│ /api/subscribe     │ 服务端强制订阅用户                         │
│ /api/unsubscribe   │ 服务端强制取消订阅                         │
│ /api/disconnect    │ 断开用户连接                               │
│ /api/presence      │ 获取频道在线用户列表                       │
│ /api/presence_stats│ 获取频道在线统计                           │
│ /api/history       │ 获取频道历史消息                           │
│ /api/history_remove│ 清除频道历史消息                           │
│ /api/channels      │ 列出所有活跃频道                           │
│ /api/info          │ 获取服务器信息                             │
│ /api/batch         │ 批量执行多个 API 命令                      │
│ /api/refresh       │ 刷新用户连接                               │
│ /api/rpc           │ 执行 RPC 调用                             │
└────────────────────┴──────────────────────────────────────────┘
```

### 通用请求格式

```bash
curl -X POST http://localhost:8000/api/<endpoint> \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '<JSON body>'
```

### 通用响应格式

```json
// 成功
{
  "result": { ... }
}

// 错误
{
  "error": {
    "code": 102,
    "message": "unknown channel"
  }
}
```

---

## 5.2 publish — 发布消息

**最常用的 API**。把消息发送到指定频道，所有订阅了该频道的客户端会立刻收到。

### 基本用法

```bash
curl -X POST http://localhost:8000/api/publish \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{
    "channel": "chat:room_42",
    "data": {
      "type": "message",
      "user": "张三",
      "text": "大家好！"
    }
  }'
```

### 响应

```json
// 频道有历史记录时
{
  "result": {
    "offset": 15,
    "epoch": "Gbcd"
  }
}

// 频道没有历史记录时
{
  "result": {}
}
```

### 业务场景示例

```bash
# 场景1：聊天消息
curl -X POST http://localhost:8000/api/publish \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{
    "channel": "chat:room_42",
    "data": {
      "type": "message",
      "sender": {"id": "u1001", "name": "张三", "avatar": "/avatars/u1001.jpg"},
      "text": "今天天气真好",
      "timestamp": 1704067200
    }
  }'

# 场景2：订单状态更新
curl -X POST http://localhost:8000/api/publish \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{
    "channel": "#user_1001",
    "data": {
      "type": "order_update",
      "order_id": "ORD-2024-001",
      "status": "shipped",
      "tracking_number": "SF1234567890"
    }
  }'

# 场景3：实时股票行情
curl -X POST http://localhost:8000/api/publish \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{
    "channel": "stock:AAPL",
    "data": {
      "symbol": "AAPL",
      "price": 150.25,
      "change": "+1.5%",
      "volume": 45000000,
      "timestamp": 1704067200
    }
  }'

# 场景4：监控告警
curl -X POST http://localhost:8000/api/publish \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{
    "channel": "alerts:ops",
    "data": {
      "level": "critical",
      "service": "payment-api",
      "message": "CPU 使用率超过 95%",
      "timestamp": 1704067200
    }
  }'
```

---

## 5.3 broadcast — 广播消息

一次性发布消息到 **多个频道**。

```bash
# 系统公告：发送到所有聊天室
curl -X POST http://localhost:8000/api/broadcast \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{
    "channels": [
      "chat:room_1",
      "chat:room_2",
      "chat:room_3",
      "chat:general"
    ],
    "data": {
      "type": "system_announcement",
      "text": "系统将于今晚 22:00 进行维护，预计持续 30 分钟"
    }
  }'
```

---

## 5.4 subscribe — 服务端订阅

强制让某个用户订阅某个频道（用户不需要在客户端主动订阅）。

```bash
# 让用户 user_1001 订阅 chat:room_42
curl -X POST http://localhost:8000/api/subscribe \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{
    "user": "user_1001",
    "channel": "chat:room_42"
  }'

# 场景：后台管理员将用户加入某个群组
curl -X POST http://localhost:8000/api/subscribe \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{
    "user": "user_1001",
    "channel": "team:engineering",
    "data": {
      "info": {"role": "member"}
    }
  }'
```

---

## 5.5 unsubscribe — 取消订阅

强制取消用户对某个频道的订阅。

```bash
# 将用户从频道中移除
curl -X POST http://localhost:8000/api/unsubscribe \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{
    "user": "user_1001",
    "channel": "chat:room_42"
  }'
```

---

## 5.6 disconnect — 断开连接

强制断开用户的所有连接。

```bash
# 踢出用户
curl -X POST http://localhost:8000/api/disconnect \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{
    "user": "user_1001"
  }'

# 带断开原因（客户端可以看到）
curl -X POST http://localhost:8000/api/disconnect \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{
    "user": "user_1001",
    "disconnect": {
      "code": 4000,
      "reason": "account_banned"
    }
  }'

# 场景：用户修改密码后，强制其他设备重新登录
curl -X POST http://localhost:8000/api/disconnect \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{
    "user": "user_1001",
    "disconnect": {
      "code": 4001,
      "reason": "password_changed"
    }
  }'
```

---

## 5.7 presence — 在线用户列表

获取频道中当前在线的用户详情。

> ⚠️ 需要频道开启了 `presence: true`

```bash
curl -X POST http://localhost:8000/api/presence \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{
    "channel": "chat:room_42"
  }'
```

响应：

```json
{
  "result": {
    "presence": {
      "d6e5eae0-58e4-4f35-8612-3a4a8e2c1a7f": {
        "user": "user_1001",
        "client": "d6e5eae0-58e4-4f35-8612-3a4a8e2c1a7f",
        "conn_info": {"name": "张三"},
        "chan_info": {}
      },
      "a1b2c3d4-e5f6-7890-abcd-ef1234567890": {
        "user": "user_1002",
        "client": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
        "conn_info": {"name": "李四"},
        "chan_info": {}
      }
    }
  }
}
```

---

## 5.8 presence_stats — 在线统计

获取频道的在线人数统计（不返回完整列表，更轻量）。

```bash
curl -X POST http://localhost:8000/api/presence_stats \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{
    "channel": "chat:room_42"
  }'
```

响应：

```json
{
  "result": {
    "num_clients": 156,
    "num_users": 142
  }
}
```

> **💡 提示**：`num_clients > num_users` 说明有用户使用了多个设备（多个连接）。

---

## 5.9 history — 历史消息

获取频道的历史消息。

> ⚠️ 需要频道配置了 `history_size` 和 `history_ttl`

```bash
# 获取最近 10 条消息
curl -X POST http://localhost:8000/api/history \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{
    "channel": "chat:room_42",
    "limit": 10
  }'

# 从某个位置开始获取（分页）
curl -X POST http://localhost:8000/api/history \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{
    "channel": "chat:room_42",
    "limit": 10,
    "since": {
      "offset": 100,
      "epoch": "Gbcd"
    },
    "reverse": true
  }'
```

响应：

```json
{
  "result": {
    "publications": [
      {
        "data": {"text": "消息内容", "user": "张三"},
        "offset": 101
      },
      {
        "data": {"text": "另一条消息", "user": "李四"},
        "offset": 102
      }
    ],
    "offset": 102,
    "epoch": "Gbcd"
  }
}
```

---

## 5.10 history_remove — 清除历史

清除频道的所有历史消息。

```bash
curl -X POST http://localhost:8000/api/history_remove \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{
    "channel": "chat:room_42"
  }'
```

---

## 5.11 channels — 列出频道

获取当前所有有订阅者的活跃频道。

```bash
curl -X POST http://localhost:8000/api/channels \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{}'

# 按模式过滤
curl -X POST http://localhost:8000/api/channels \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{
    "pattern": "chat:*"
  }'
```

响应：

```json
{
  "result": {
    "channels": {
      "chat:room_1": {"num_clients": 5},
      "chat:room_2": {"num_clients": 12},
      "chat:general": {"num_clients": 45}
    }
  }
}
```

> **⚠️ 注意**：在大量频道的场景下（>10000），此 API 可能会比较慢。

---

## 5.12 info — 服务器信息

获取 Centrifugo 集群的节点信息。

```bash
curl -X POST http://localhost:8000/api/info \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{}'
```

响应：

```json
{
  "result": {
    "nodes": [
      {
        "uid": "f5a6b7c8-d9e0-1234-5678-abcdef123456",
        "name": "centrifugo-node-1_8000",
        "version": "6.7.1",
        "num_clients": 1250,
        "num_users": 1100,
        "num_subs": 3500,
        "num_channels": 450,
        "uptime": 86400,
        "metrics": {
          "interval": 60,
          "items": {}
        }
      }
    ]
  }
}
```

---

## 5.13 batch — 批量操作

一次请求中执行多个 API 命令，减少网络往返。

```bash
curl -X POST http://localhost:8000/api/batch \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{
    "commands": [
      {
        "publish": {
          "channel": "chat:room_1",
          "data": {"text": "消息1"}
        }
      },
      {
        "publish": {
          "channel": "chat:room_2",
          "data": {"text": "消息2"}
        }
      },
      {
        "presence_stats": {
          "channel": "chat:room_1"
        }
      }
    ]
  }'
```

响应：

```json
{
  "replies": [
    {"publish": {"result": {}}},
    {"publish": {"result": {}}},
    {"presence_stats": {"result": {"num_clients": 5, "num_users": 4}}}
  ]
}
```

> **💡 性能提示**：如果需要同时发布到多个频道，用 `broadcast` 比 `batch` + 多个 `publish` 更高效。

---

## 5.14 refresh — 刷新连接

刷新用户的连接（延长过期时间）。

```bash
curl -X POST http://localhost:8000/api/refresh \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{
    "user": "user_1001",
    "expire_at": 1704240000
  }'
```

---

## 5.15 错误码参考

| 错误码 | 含义 | 常见原因 |
|--------|------|---------|
| 100 | Internal error | 服务器内部错误 |
| 101 | Unauthorized | API key 不正确或缺失 |
| 102 | Unknown channel | 频道名无效 |
| 103 | Bad request | 请求格式错误 |
| 104 | Not available | 功能未启用（如 presence 未开启） |
| 105 | Token expired | Token 已过期 |
| 107 | Limit exceeded | 超出限制 |
| 108 | Method not found | 方法不存在 |
| 109 | Permission denied | 权限不足 |

---

## 5.16 最佳实践

### 使用连接池

你的后端应该复用 HTTP 连接来调用 Centrifugo API：

```python
# Python 示例：使用 requests.Session 复用连接
import requests

session = requests.Session()
session.headers.update({
    'Content-Type': 'application/json',
    'X-API-Key': 'your-api-key'
})

def publish(channel, data):
    response = session.post(
        'http://centrifugo:8000/api/publish',
        json={'channel': channel, 'data': data}
    )
    return response.json()
```

### 批量发布

当需要发布很多消息时，使用 batch API 或 broadcast 减少网络往返：

```bash
# 用 broadcast 同时通知多个用户
curl -X POST http://localhost:8000/api/broadcast \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{
    "channels": ["#user_1", "#user_2", "#user_3", "#user_4", "#user_5"],
    "data": {"type": "group_invite", "group_id": "team_42"}
  }'
```

### gRPC API 更高性能

对于高吞吐场景，gRPC API 比 HTTP API 更高效（持久连接、二进制协议、HTTP/2 多路复用）：

```json
{
  "grpc_api": {
    "enabled": true,
    "port": 10000,
    "key": "your-grpc-key"
  }
}
```

---

## 5.17 本章小结

| API | 使用频率 | 一句话描述 |
|-----|---------|-----------|
| `publish` | ⭐⭐⭐⭐⭐ | 发消息到频道，最常用 |
| `broadcast` | ⭐⭐⭐⭐ | 一次发到多个频道 |
| `info` | ⭐⭐⭐ | 查看服务器状态 |
| `presence` | ⭐⭐⭐ | 查看谁在线 |
| `history` | ⭐⭐⭐ | 获取历史消息 |
| `subscribe` | ⭐⭐ | 服务端强制订阅 |
| `unsubscribe` | ⭐⭐ | 服务端取消订阅 |
| `disconnect` | ⭐⭐ | 踢人下线 |
| `batch` | ⭐⭐ | 批量操作 |
| `channels` | ⭐ | 列出频道 |
| `refresh` | ⭐ | 刷新连接 |

**下一章**：学习客户端 SDK 的使用 → [第6章 客户端 SDK](06-client-sdk.md)
