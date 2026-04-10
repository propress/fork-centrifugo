# 第9章 代理机制

> **阅读目标**：理解 Centrifugo 的代理系统，能让 Centrifugo 与你的后端深度集成

---

## 9.1 什么是代理？

代理（Proxy）是 Centrifugo 的核心集成机制。它让 Centrifugo 在特定事件发生时 **把请求转发给你的后端**，由你的后端来做决策。

```
不用代理时：                            用代理时：

客户端 → Centrifugo                   客户端 → Centrifugo → 你的后端
         ↓                                        ↓          ↓
      直接处理                                转发请求    做业务决策
      (基于配置)                                  ↓          ↓
                                            ← 返回结果 ← 响应结果
```

### 代理的价值

```
没有代理：Centrifugo 只能根据静态配置来决策
         - 有合法 Token → 允许连接
         - 频道开放 → 允许订阅
         - 很有限...

有了代理：你的后端可以动态决策一切
         - 用户是否可以连接？→ 查数据库判断
         - 用户是否可以订阅某频道？→ 查权限表
         - 用户发来的消息是否要转发？→ 做内容审核
         - 客户端 RPC 调用？→ 执行业务逻辑
```

---

## 9.2 代理类型

```
┌──────────────────────────────────────────────────────────────┐
│                    Centrifugo 代理类型                        │
│                                                              │
│  ┌────────────────┐  客户端连接时触发                         │
│  │  Connect Proxy │  → 认证、分配频道、传递用户信息           │
│  └────────────────┘                                          │
│                                                              │
│  ┌────────────────┐  Token 快过期时触发                       │
│  │  Refresh Proxy │  → 决定是否延长连接                       │
│  └────────────────┘                                          │
│                                                              │
│  ┌──────────────────┐  客户端订阅频道时触发                   │
│  │  Subscribe Proxy │  → 频道级权限控制                       │
│  └──────────────────┘                                        │
│                                                              │
│  ┌────────────────┐  客户端发布消息时触发                     │
│  │  Publish Proxy │  → 消息审核、修改、拦截                   │
│  └────────────────┘                                          │
│                                                              │
│  ┌────────────────┐  客户端发送 RPC 时触发                    │
│  │    RPC Proxy   │  → 执行后端业务逻辑                       │
│  └────────────────┘                                          │
│                                                              │
│  ┌────────────────────┐  订阅 Token 快过期时触发              │
│  │  SubRefresh Proxy  │  → 决定是否延长订阅                   │
│  └────────────────────┘                                      │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 9.3 Connect Proxy — 连接代理

### 用途

替代 JWT Token 认证。客户端连接时，Centrifugo 把连接请求转发给你的后端，由后端决定是否允许。

### 配置

```yaml
client:
  proxy:
    connect:
      enabled: true
      endpoint: "http://your-backend:3000/centrifugo/connect"
      timeout: "3s"
      http_headers: ["Cookie", "Authorization"]
```

### 请求与响应

Centrifugo 发给你的后端：

```json
// POST http://your-backend:3000/centrifugo/connect
{
  "client": "abc-def-123",
  "transport": "websocket",
  "protocol": "json",
  "encoding": "json",
  "data": {}  // 客户端连接时携带的自定义数据
}
```

你的后端响应：

```json
// 允许连接
{
  "result": {
    "user": "user_1001",            // 必须：用户 ID
    "expire_at": 1704240000,        // 可选：过期时间
    "info": {"name": "张三"},       // 可选：用户信息
    "channels": ["notifications"],  // 可选：自动订阅的频道
    "meta": {"role": "admin"}       // 可选：元数据
  }
}

// 拒绝连接
{
  "error": {
    "code": 403,
    "message": "unauthorized"
  }
}

// 也可以通过 disconnect 来拒绝
{
  "disconnect": {
    "code": 4001,
    "reason": "account_banned"
  }
}
```

### 后端实现示例

```python
# Python Flask 示例
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/centrifugo/connect', methods=['POST'])
def connect_proxy():
    data = request.json

    # 从 Cookie 或 Header 中获取认证信息
    cookies = request.headers.get('Cookie', '')
    auth_header = request.headers.get('Authorization', '')

    # 验证用户身份（你的业务逻辑）
    user = authenticate(cookies, auth_header)

    if user is None:
        return jsonify({
            "error": {"code": 401, "message": "unauthorized"}
        })

    return jsonify({
        "result": {
            "user": str(user.id),
            "info": {"name": user.name, "avatar": user.avatar},
            "channels": [f"notify:user_{user.id}"]
        }
    })
```

### Connect Proxy 的优势

```
与 JWT 对比：

JWT 方式：
  你的后端 → 生成 JWT → 传给前端 → 前端带着 JWT 连 Centrifugo
  ✅ 简单、高性能
  ❌ 不适合基于 Cookie 的认证
  ❌ Token 过期需要额外处理

Connect Proxy 方式：
  前端 → 连 Centrifugo → Centrifugo 转发给你的后端 → 后端验证
  ✅ 直接复用现有的 Cookie/Session 认证
  ✅ 可以动态分配频道
  ✅ 每次连接都是最新的权限检查
  ❌ 每次连接都要调后端（稍微慢一点）
```

---

## 9.4 Subscribe Proxy — 订阅代理

### 用途

客户端订阅频道时，由你的后端决定是否允许。

### 配置

```yaml
channel:
  proxy:
    subscribe:
      enabled: true
      endpoint: "http://your-backend:3000/centrifugo/subscribe"
      timeout: "3s"
```

也可以只对特定命名空间启用：

```yaml
channel:
  namespaces:
    - name: "private"
      proxy:
        subscribe:
          enabled: true
          endpoint: "http://your-backend:3000/centrifugo/subscribe"
```

### 请求与响应

```json
// Centrifugo → 你的后端
// POST http://your-backend:3000/centrifugo/subscribe
{
  "client": "abc-def-123",
  "user": "user_1001",
  "channel": "private:room_42",
  "data": {}
}

// 允许订阅
{
  "result": {
    "info": {"role": "member"},    // 可选：频道级别的用户信息
    "data": {"welcome": "欢迎！"}  // 可选：订阅时返回的数据
  }
}

// 拒绝订阅
{
  "error": {
    "code": 403,
    "message": "你没有权限访问这个频道"
  }
}
```

### 后端实现

```python
@app.route('/centrifugo/subscribe', methods=['POST'])
def subscribe_proxy():
    data = request.json
    user_id = data['user']
    channel = data['channel']

    # 检查用户是否有权限订阅该频道
    if not has_channel_permission(user_id, channel):
        return jsonify({
            "error": {"code": 403, "message": "no permission"}
        })

    return jsonify({
        "result": {
            "info": {"role": get_user_role(user_id, channel)}
        }
    })
```

---

## 9.5 Publish Proxy — 发布代理

### 用途

客户端通过 SDK 发布消息时（需要先开启 `allow_publish_for_subscriber`），由后端决定是否允许并可以修改消息。

### 配置

```yaml
channel:
  namespaces:
    - name: "chat"
      allow_publish_for_subscriber: true
      proxy:
        publish:
          enabled: true
          endpoint: "http://your-backend:3000/centrifugo/publish"
```

### 后端实现（消息审核）

```python
@app.route('/centrifugo/publish', methods=['POST'])
def publish_proxy():
    data = request.json
    user_id = data['user']
    channel = data['channel']
    message_data = data['data']

    # 内容审核
    if contains_sensitive_content(message_data.get('text', '')):
        return jsonify({
            "error": {"code": 400, "message": "消息包含敏感内容"}
        })

    # 保存到数据库
    save_message(user_id, channel, message_data)

    # 可以修改消息再转发
    message_data['server_timestamp'] = int(time.time())
    message_data['verified'] = True

    return jsonify({
        "result": {
            "data": message_data  # 修改后的消息
        }
    })
```

---

## 9.6 RPC Proxy — 远程过程调用

### 用途

客户端通过 Centrifugo 调用后端的业务逻辑，不需要单独建立 HTTP 连接。

### 配置

```yaml
rpc:
  proxy:
    enabled: true
    endpoint: "http://your-backend:3000/centrifugo/rpc"
    timeout: "5s"
  # 可以为不同的 RPC 方法设置不同的代理
  namespaces:
    - name: "chat"
      proxy:
        enabled: true
        endpoint: "http://your-backend:3000/centrifugo/rpc/chat"
```

### 客户端调用

```javascript
// 客户端发起 RPC
const result = await centrifuge.rpc('get_user_profile', {
    user_id: 'user_1002'
});
console.log(result.data);
// { name: "李四", avatar: "...", online: true }
```

### 后端实现

```python
@app.route('/centrifugo/rpc', methods=['POST'])
def rpc_proxy():
    data = request.json
    user_id = data['user']
    method = data['method']
    rpc_data = data.get('data', {})

    if method == 'get_user_profile':
        profile = get_user_profile(rpc_data['user_id'])
        return jsonify({
            "result": {
                "data": {
                    "name": profile.name,
                    "avatar": profile.avatar,
                    "online": is_online(profile.id)
                }
            }
        })

    elif method == 'send_friend_request':
        send_friend_request(user_id, rpc_data['target_user_id'])
        return jsonify({
            "result": {"data": {"success": True}}
        })

    else:
        return jsonify({
            "error": {"code": 404, "message": f"unknown method: {method}"}
        })
```

---

## 9.7 gRPC 代理

代理不仅支持 HTTP，还支持 gRPC。对于高性能场景，gRPC 代理更高效。

### 配置

```yaml
client:
  proxy:
    connect:
      enabled: true
      endpoint: "grpc://your-backend:50051"   # gRPC 地址
      timeout: "3s"
```

> **💡 判断规则**：endpoint 以 `http://` 或 `https://` 开头就用 HTTP 代理，否则用 gRPC 代理。

---

## 9.8 命名代理（Named Proxies）

可以定义多个命名代理，在不同的频道命名空间中使用不同的代理：

```yaml
proxies:
  - name: "auth_service"
    endpoint: "http://auth-service:3000/centrifugo"
    timeout: "3s"

  - name: "chat_service"
    endpoint: "http://chat-service:3001/centrifugo"
    timeout: "5s"

  - name: "game_service"
    endpoint: "grpc://game-service:50051"
    timeout: "2s"

channel:
  namespaces:
    - name: "chat"
      proxy:
        subscribe:
          enabled: true
          proxy_name: "chat_service"
        publish:
          enabled: true
          proxy_name: "chat_service"

    - name: "game"
      proxy:
        subscribe:
          enabled: true
          proxy_name: "game_service"
```

---

## 9.9 代理请求中的额外信息

### 转发 HTTP 头

```yaml
client:
  proxy:
    connect:
      enabled: true
      endpoint: "http://your-backend:3000/centrifugo/connect"
      http_headers: ["Cookie", "Authorization", "X-Custom-Header"]
```

### 添加静态头

```yaml
client:
  proxy:
    connect:
      enabled: true
      endpoint: "http://your-backend:3000/centrifugo/connect"
      http:
        static_headers:
          X-Internal-Service: "centrifugo"
          X-Service-Version: "6.7.1"
```

---

## 9.10 代理设计模式

### 模式1：纯代理模式（无 JWT）

适合已有 Session/Cookie 认证的应用：

```yaml
client:
  proxy:
    connect:
      enabled: true
      endpoint: "http://your-backend:3000/centrifugo/connect"
      http_headers: ["Cookie"]
```

### 模式2：混合模式

JWT 做连接认证，代理做订阅授权：

```yaml
client:
  token:
    hmac_secret_key: "your-secret"

channel:
  namespaces:
    - name: "private"
      proxy:
        subscribe:
          enabled: true
          endpoint: "http://your-backend:3000/centrifugo/subscribe"
```

### 模式3：全代理模式

所有决策都由后端做：

```yaml
client:
  proxy:
    connect:
      enabled: true
      endpoint: "http://your-backend:3000/centrifugo/connect"

channel:
  proxy:
    subscribe:
      enabled: true
      endpoint: "http://your-backend:3000/centrifugo/subscribe"
    publish:
      enabled: true
      endpoint: "http://your-backend:3000/centrifugo/publish"

rpc:
  proxy:
    enabled: true
    endpoint: "http://your-backend:3000/centrifugo/rpc"
```

---

## 9.11 常见坑

### 坑1：代理超时

```
❌ 现象：客户端连接/订阅很慢，然后超时

原因：后端处理代理请求太慢

解决：
1. 优化后端响应时间
2. 增加 timeout 配置
3. 确保后端有足够的并发处理能力
```

### 坑2：代理端点返回非 JSON

```
❌ 现象：代理失败，日志报解析错误

确保你的后端：
1. 返回 Content-Type: application/json
2. 返回有效的 JSON
3. 即使拒绝，也要返回标准的错误格式
```

### 坑3：代理端点不可达

```
❌ 现象：所有客户端都无法连接

如果 Connect Proxy 的后端挂了，所有新连接都会失败。

建议：
1. 后端做好高可用
2. 合理设置超时
3. 监控代理请求的成功率
```

---

## 9.12 本章小结

| 代理类型 | 触发时机 | 典型用途 |
|---------|---------|---------|
| Connect | 客户端连接 | 认证、分配频道 |
| Subscribe | 客户端订阅 | 频道权限控制 |
| Publish | 客户端发布 | 消息审核、修改 |
| RPC | 客户端 RPC | 执行后端逻辑 |
| Refresh | Token 过期 | 延长连接 |
| SubRefresh | 订阅 Token 过期 | 延长订阅 |

**下一章**：学习消息队列消费者 → [第10章 消息队列消费者](10-consumers.md)
