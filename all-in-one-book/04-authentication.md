# 第4章 认证与鉴权

> **阅读目标**：理解 Centrifugo 的认证机制，能在真实项目中安全地接入用户认证

---

## 4.1 认证全景图

```
┌──────────────────────────────────────────────────────────────┐
│                   Centrifugo 认证体系                        │
│                                                              │
│   客户端连接认证（谁能连上来？）                              │
│   ┌────────────────┐  ┌────────────────┐  ┌──────────────┐  │
│   │  JWT Token     │  │  代理认证       │  │  匿名连接     │  │
│   │  (最常用)      │  │  (Proxy Auth)  │  │  (开发/公开)  │  │
│   └────────────────┘  └────────────────┘  └──────────────┘  │
│                                                              │
│   频道订阅授权（谁能订阅哪个频道？）                          │
│   ┌────────────────┐  ┌────────────────┐  ┌──────────────┐  │
│   │ 订阅 Token     │  │  代理授权       │  │  公开频道     │  │
│   │(Subscription   │  │(Subscribe      │  │  (无需授权)   │  │
│   │  Token)        │  │  Proxy)        │  │              │  │
│   └────────────────┘  └────────────────┘  └──────────────┘  │
│                                                              │
│   服务端 API 认证（后端怎么调 API？）                        │
│   ┌────────────────┐  ┌────────────────┐                    │
│   │  API Key       │  │  gRPC Key      │                    │
│   └────────────────┘  └────────────────┘                    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 4.2 JWT 连接认证（最推荐的方式）

### 工作流程

```
用户                 你的后端                Centrifugo
 │                      │                      │
 │── 登录请求 ─────────▶│                      │
 │   (用户名+密码)       │                      │
 │                      │── 验证用户身份         │
 │                      │── 生成 JWT Token       │
 │◀── 返回 JWT Token ──│                      │
 │                      │                      │
 │── WebSocket 连接 ──────────────────────────▶│
 │   携带 JWT Token     │                      │── 验证 Token 签名
 │                      │                      │── 提取用户 ID
 │◀── 连接成功 ────────────────────────────────│
 │                      │                      │
```

### JWT Token 的结构

Centrifugo 的 JWT Token 是标准的 JWT，Payload 中有以下字段：

```json
{
  "sub": "user_1001",          // 必须：用户 ID
  "exp": 1704153600,           // 可选：过期时间（Unix 时间戳）
  "info": {                    // 可选：用户信息（会传给其他订阅者）
    "name": "张三",
    "avatar": "https://example.com/avatar.jpg"
  },
  "channels": [                // 可选：服务端订阅的频道列表
    "notifications",
    "chat:general"
  ],
  "meta": {                    // 可选：元数据（不会传给其他客户端）
    "role": "admin"
  }
}
```

### 配置 Centrifugo 验证 JWT

#### 方式1：HMAC 密钥（最简单，推荐新手用）

```json
{
  "client": {
    "token": {
      "hmac_secret_key": "your-256-bit-secret-key-keep-it-safe"
    }
  }
}
```

#### 方式2：RSA 公钥

```json
{
  "client": {
    "token": {
      "rsa_public_key": "-----BEGIN PUBLIC KEY-----\nMIIBIjANBg..."
    }
  }
}
```

#### 方式3：ECDSA 公钥

```json
{
  "client": {
    "token": {
      "ecdsa_public_key": "-----BEGIN PUBLIC KEY-----\nMHYwEAYHKo..."
    }
  }
}
```

#### 方式4：JWKS 端点（适合微服务/Auth0/Keycloak 等）

```json
{
  "client": {
    "token": {
      "jwks_public_endpoint": "https://your-auth-server.com/.well-known/jwks.json"
    }
  }
}
```

### 在各语言后端生成 JWT Token

#### Python

```python
import jwt
import time

secret = "your-256-bit-secret-key-keep-it-safe"

token = jwt.encode({
    "sub": "user_1001",
    "exp": int(time.time()) + 3600,  # 1 小时后过期
    "info": {"name": "张三"}
}, secret, algorithm="HS256")

print(token)
```

#### Node.js

```javascript
const jwt = require('jsonwebtoken');

const secret = 'your-256-bit-secret-key-keep-it-safe';

const token = jwt.sign({
  sub: 'user_1001',
  info: { name: '张三' }
}, secret, { expiresIn: '1h' });

console.log(token);
```

#### Go

```go
import (
    "time"
    "github.com/golang-jwt/jwt/v5"
)

secret := []byte("your-256-bit-secret-key-keep-it-safe")

token := jwt.NewWithClaims(jwt.SigningMethodHS256, jwt.MapClaims{
    "sub":  "user_1001",
    "exp":  time.Now().Add(time.Hour).Unix(),
    "info": map[string]interface{}{"name": "张三"},
})

tokenString, _ := token.SignedString(secret)
```

#### Java

```java
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;

String token = Jwts.builder()
    .setSubject("user_1001")
    .claim("info", Map.of("name", "张三"))
    .setExpiration(new Date(System.currentTimeMillis() + 3600000))
    .signWith(SignatureAlgorithm.HS256, "your-256-bit-secret-key-keep-it-safe".getBytes())
    .compact();
```

#### PHP

```php
use Firebase\JWT\JWT;

$secret = 'your-256-bit-secret-key-keep-it-safe';

$token = JWT::encode([
    'sub' => 'user_1001',
    'exp' => time() + 3600,
    'info' => ['name' => '张三']
], $secret, 'HS256');
```

### 用 CLI 快速生成测试 Token

```bash
# Centrifugo 自带了 token 生成工具
centrifugo gentoken --config config.json --user "user_1001"

# 输出类似：
# HMAC SHA-256 JWT for user "user_1001" with no expiration:
# eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### 客户端使用 Token

```javascript
// JavaScript 客户端
const centrifuge = new Centrifuge('ws://localhost:8000/connection/websocket', {
    token: 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...'
});
centrifuge.connect();
```

---

## 4.3 Token 过期与刷新

### 问题：Token 过期了怎么办？

JWT Token 通常有过期时间。过期后客户端需要获取新 Token，否则会被断开连接。

### 解决方案：Token 刷新机制

```javascript
const centrifuge = new Centrifuge('ws://localhost:8000/connection/websocket', {
    token: initialToken,
    // 当 token 快过期时，SDK 会调用这个函数
    getToken: function() {
        return fetch('/api/centrifugo-token', {
            method: 'POST',
            headers: { 'Authorization': 'Bearer ' + yourAppToken }
        })
        .then(res => res.json())
        .then(data => data.token);
    }
});
centrifuge.connect();
```

```
Token 刷新流程：

客户端                  你的后端                 Centrifugo
  │                       │                        │
  │── getToken() ────────▶│                        │
  │   (你的应用 token)     │                        │
  │                       │── 验证用户身份           │
  │                       │── 生成新的 Centrifugo JWT │
  │◀── 新 JWT Token ─────│                        │
  │                        │                        │
  │── 用新 Token 刷新连接 ──────────────────────────▶│
  │                        │                        │── 验证新 Token
  │◀── 刷新成功 ──────────────────────────────────│
  │                        │                        │
```

### 密钥轮换

当需要更换 HMAC 密钥时，可以配置旧密钥的过渡期：

```json
{
  "client": {
    "token": {
      "hmac_secret_key": "new-secret-key",
      "hmac_previous_secret_key": "old-secret-key",
      "hmac_previous_secret_key_valid_until": 1704240000
    }
  }
}
```

---

## 4.4 频道订阅授权

### 为什么需要订阅授权？

连接认证解决了 **「谁能连上来」** 的问题，但没有解决 **「谁能订阅哪个频道」** 的问题。

```
场景：
  用户A 连接成功后，尝试订阅 "chat:vip_room"
  但用户A 不是 VIP，不应该能进入 VIP 聊天室

  → 需要订阅级别的授权
```

### 方式1：私有频道 + 订阅 Token

以 `$` 开头的频道是私有频道，订阅时需要额外的 Token：

```json
{
  "client": {
    "subscription_token": {
      "hmac_secret_key": "subscription-secret-key"
    }
  }
}
```

订阅 Token 的 Payload：

```json
{
  "sub": "user_1001",
  "channel": "$chat:vip_room",
  "exp": 1704153600
}
```

客户端代码：

```javascript
const sub = centrifuge.newSubscription('$chat:vip_room', {
    token: subscriptionToken,
    // 或者动态获取
    getToken: function() {
        return fetch('/api/centrifugo-sub-token?channel=$chat:vip_room')
            .then(res => res.json())
            .then(data => data.token);
    }
});
sub.subscribe();
```

### 方式2：代理授权（Subscribe Proxy）

让 Centrifugo 把订阅请求转发给你的后端来决定是否允许：

```json
{
  "channel": {
    "proxy": {
      "subscribe": {
        "enabled": true,
        "endpoint": "http://your-backend:3000/centrifugo/subscribe"
      }
    }
  }
}
```

```
客户端                  Centrifugo              你的后端
  │                        │                       │
  │── subscribe ──────────▶│                       │
  │   channel: chat:vip    │                       │
  │                        │── POST /subscribe ───▶│
  │                        │   {user, channel}     │
  │                        │                       │── 检查权限
  │                        │◀── 200 OK ────────────│
  │                        │   {result: {}}        │
  │◀── subscribed ─────────│                       │
  │                        │                       │
```

你的后端处理逻辑（伪代码）：

```python
@app.post("/centrifugo/subscribe")
def handle_subscribe(request):
    user = request.json["user"]
    channel = request.json["channel"]

    if can_access_channel(user, channel):
        return {"result": {}}
    else:
        return {"error": {"code": 403, "message": "forbidden"}}
```

---

## 4.5 匿名连接

### 什么时候用匿名连接？

有些场景不需要知道用户身份，比如：
- 公开的实时数据展示（股票行情、监控面板）
- 直播页面的弹幕
- 公告广播

### 配置匿名连接

```json
{
  "client": {
    "allow_anonymous_connect_without_token": true
  }
}
```

匿名用户的 user ID 为空字符串 `""`。

> **⚠️ 注意**：匿名连接无法使用基于用户的功能（如用户频道、在线状态中的用户信息）。

---

## 4.6 服务端 API 认证

### HTTP API Key

```json
{
  "http_api": {
    "key": "your-api-key-keep-secret"
  }
}
```

调用 API 时需要在请求头中携带：

```bash
curl -X POST http://localhost:8000/api/publish \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key-keep-secret" \
  -d '{"channel": "test", "data": {"msg": "hello"}}'
```

### gRPC API Key

```json
{
  "grpc_api": {
    "enabled": true,
    "key": "your-grpc-api-key"
  }
}
```

---

## 4.7 Admin UI 认证

```json
{
  "admin": {
    "enabled": true,
    "password": "admin-login-password",
    "secret": "admin-jwt-secret-for-session"
  }
}
```

- `password` 用于 Web 界面登录
- `secret` 用于生成管理会话 Token

开发时可以用 `"insecure": true` 跳过认证：

```json
{
  "admin": {
    "enabled": true,
    "insecure": true
  }
}
```

---

## 4.8 认证方案选型指南

```
你的项目情况                          推荐方案
──────────                          ────────

刚开始开发，想快速试验              → client.insecure = true
                                     （记得上线前关掉！）

单体应用，后端能直接生成 Token     → JWT + HMAC
                                     简单高效

微服务架构，有统一认证中心          → JWT + JWKS 端点
                                     自动获取公钥

需要细粒度的频道权限控制            → Subscribe Proxy
                                     后端动态决定

用 Auth0/Keycloak 等                → JWKS 端点
                                     直接对接

公开数据展示，不需要认证            → 匿名连接
                                     allow_anonymous_connect_without_token
```

---

## 4.9 安全最佳实践

### ❌ 不要这样做

```
1. 不要在前端代码中硬编码 HMAC 密钥
   → 密钥只在后端使用

2. 不要在生产环境使用 insecure 模式
   → 任何人都能连接并订阅所有频道

3. 不要使用太短的密钥
   → HMAC 密钥至少 32 字节

4. 不要给 Token 设置太长的过期时间
   → 建议 1-24 小时，配合刷新机制

5. 不要把 API Key 暴露给前端
   → API Key 只在后端服务器间使用
```

### ✅ 应该这样做

```
1. 后端生成 JWT Token，前端只负责传递
2. 使用 getToken 回调实现自动刷新
3. 不同环境（开发/测试/生产）使用不同密钥
4. 定期轮换密钥（利用 previous_secret_key 实现无缝切换）
5. 私有频道用 $ 前缀 + 订阅 Token
6. 生产环境启用 TLS（wss://）
```

---

## 4.10 新手常见坑

### 坑1：Token 格式错误

```
❌ 错误：connection refused with code 101
```

检查：
- Token 是否是标准 JWT 格式（三段 base64 用 `.` 分隔）
- 签名算法是否匹配（HS256/RS256/ES256）
- 密钥是否正确

```bash
# 用 Centrifugo 内置工具验证 Token
centrifugo checktoken --config config.json --token "eyJhbGci..."
```

### 坑2：过期时间用错单位

```
❌ 错误：exp 用了毫秒而不是秒

# 错误
{"sub": "user1", "exp": 1704153600000}  ← 毫秒（会被认为是很遥远的将来）

# 正确
{"sub": "user1", "exp": 1704153600}     ← 秒（Unix timestamp）
```

### 坑3：订阅 Token 的 channel 字段不匹配

```
❌ 错误：订阅被拒绝

Token 中：  "channel": "chat:room_42"
订阅时：    centrifuge.newSubscription("$chat:room_42")

注意：如果频道有 $ 前缀，Token 中的 channel 也必须包含 $
```

### 坑4：CORS + 认证冲突

```
❌ 现象：浏览器报 401 或 CORS 错误

确保 allowed_origins 配置正确：
{
  "client": {
    "allowed_origins": ["http://localhost:3000"]
  }
}
```

---

## 4.11 本章小结

| 认证场景 | 方案 | 复杂度 |
|---------|------|--------|
| 客户端连接 | JWT Token (HMAC) | ⭐ 简单 |
| 客户端连接 | JWT Token (JWKS) | ⭐⭐ 中等 |
| 客户端连接 | Connect Proxy | ⭐⭐⭐ 复杂 |
| 频道订阅 | 订阅 Token | ⭐⭐ 中等 |
| 频道订阅 | Subscribe Proxy | ⭐⭐⭐ 复杂 |
| 后端 API | API Key | ⭐ 简单 |
| 管理界面 | 密码 + Secret | ⭐ 简单 |

**下一章**：全面掌握服务端 API → [第5章 服务端 API](05-server-api.md)
