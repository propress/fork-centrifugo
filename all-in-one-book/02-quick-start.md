# 第2章 快速开始

> **阅读目标**：5 分钟内安装 Centrifugo，10 分钟内发出第一条实时消息

---

## 2.1 安装 Centrifugo

### 方式一：Docker（最推荐，最简单）

```bash
# 拉取并启动 Centrifugo
docker run -d --name centrifugo \
  -p 8000:8000 \
  centrifugo/centrifugo:v6 \
  centrifugo --client.insecure --admin.enabled --admin.insecure --health.enabled
```

> **说明**：
> - `--client.insecure` 允许无认证连接（仅开发用！）
> - `--admin.enabled --admin.insecure` 开启管理界面且无密码
> - `-p 8000:8000` 映射端口

启动后访问 http://localhost:8000 看到管理界面就成功了！

### 方式二：直接下载二进制

```bash
# Linux/macOS 一键安装
curl -sSLf https://centrifugal.dev/install.sh | sh

# 或者手动从 GitHub Releases 下载
# https://github.com/centrifugal/centrifugo/releases
```

### 方式三：包管理器

```bash
# Debian/Ubuntu
curl -s https://packagecloud.io/install/repositories/FZambia/centrifugo/script.deb.sh | sudo bash
sudo apt install centrifugo

# macOS (Homebrew)
brew tap centrifugal/centrifugo
brew install centrifugo
```

### 方式四：从源码编译

```bash
git clone https://github.com/centrifugal/centrifugo.git
cd centrifugo
make build
# 生成的二进制文件在当前目录
```

---

## 2.2 生成配置文件

```bash
# 生成默认配置文件
centrifugo genconfig

# 会生成 config.json，内容类似：
# {
#   "client": {
#     "token": {
#       "hmac_secret_key": "随机生成的密钥"
#     }
#   },
#   "admin": {
#     "enabled": true,
#     "password": "随机生成的密码",
#     "secret": "随机生成的密钥"
#   },
#   ...
# }
```

> **💡 新手提示**：先记住 `hmac_secret_key`，后面生成 token 要用到。

---

## 2.3 启动服务

```bash
# 使用配置文件启动
centrifugo --config config.json

# 或者用最简单的开发模式启动（不需要配置文件）
centrifugo --client.insecure --admin.enabled --admin.insecure
```

启动成功后会看到类似输出：

```
INFO  starting Centrifugo        version=6.7.1 runtime=go1.23 engine=memory
INFO  serving HTTP on :8000
```

---

## 2.4 验证安装：管理界面

打开浏览器访问 http://localhost:8000

如果用了 `--admin.insecure`，直接进入管理界面。否则需要输入配置文件中的密码。

```
管理界面功能：
┌──────────────────────────────────────────────┐
│  Centrifugo Admin                            │
│                                              │
│  📊 仪表盘                                   │
│     • 当前连接数                              │
│     • 消息吞吐量                              │
│     • 频道数                                  │
│     • 节点信息                                │
│                                              │
│  📡 频道                                      │
│     • 查看活跃频道列表                        │
│     • 查看频道的在线用户                      │
│     • 发布测试消息                            │
│                                              │
│  ⚡ 操作                                      │
│     • 发布消息到频道                          │
│     • 断开用户连接                            │
│     • 查看服务器信息                          │
└──────────────────────────────────────────────┘
```

---

## 2.5 发出第一条消息（用 curl）

### 步骤1：用 curl 发布消息到频道

```bash
# 向频道 "test" 发布一条消息
curl -X POST http://localhost:8000/api/publish \
  -H "Content-Type: application/json" \
  -d '{
    "channel": "test",
    "data": {
      "message": "Hello, Centrifugo!",
      "timestamp": "2024-01-01T00:00:00Z"
    }
  }'
```

如果没有设置 API key（开发模式），响应是：

```json
{
  "result": {}
}
```

**恭喜！你已经成功发布了一条实时消息！**

> **⚠️ 坑提示**：如果你看到 `"error": {"code": 101, "message": "unauthorized"}`，
> 说明需要在请求头加上 API key：
> ```bash
> curl -X POST http://localhost:8000/api/publish \
>   -H "Content-Type: application/json" \
>   -H "X-API-Key: YOUR_API_KEY" \
>   -d '{"channel": "test", "data": {"message": "Hello!"}}'
> ```
> API key 在配置文件的 `http_api.key` 字段设置。

### 步骤2：查看服务器信息

```bash
curl -X POST http://localhost:8000/api/info \
  -H "Content-Type: application/json" \
  -d '{}'
```

响应示例：

```json
{
  "result": {
    "nodes": [
      {
        "uid": "xxx-xxx-xxx",
        "name": "hostname_8000",
        "version": "6.7.1",
        "num_clients": 0,
        "num_channels": 0,
        "uptime": 120
      }
    ]
  }
}
```

---

## 2.6 完整端到端示例：实时聊天

现在让我们做一个完整的端到端示例，体验消息的实时推送。

### 步骤1：启动 Centrifugo（开发模式）

```bash
centrifugo --client.insecure \
           --admin.enabled \
           --admin.insecure \
           --health.enabled
```

### 步骤2：创建一个简单的 HTML 页面

创建文件 `chat.html`：

```html
<!DOCTYPE html>
<html lang="zh">
<head>
    <meta charset="UTF-8">
    <title>Centrifugo 实时聊天 Demo</title>
    <style>
        body { font-family: sans-serif; max-width: 600px; margin: 50px auto; }
        #messages { border: 1px solid #ccc; height: 300px; overflow-y: scroll; padding: 10px; margin-bottom: 10px; }
        .msg { padding: 5px 0; border-bottom: 1px solid #eee; }
        .msg .time { color: #999; font-size: 12px; }
        .status { padding: 5px; background: #f0f0f0; margin-bottom: 10px; border-radius: 4px; }
        .connected { background: #d4edda; }
        .disconnected { background: #f8d7da; }
    </style>
</head>
<body>
    <h2>🟢 Centrifugo 实时聊天 Demo</h2>
    <div id="status" class="status">正在连接...</div>
    <div id="messages"></div>
    <input type="text" id="input" placeholder="输入消息后按回车发送" style="width: 100%; padding: 8px; box-sizing: border-box;">

    <!-- 引入 Centrifugo JavaScript 客户端 SDK -->
    <script src="https://unpkg.com/centrifuge@5/dist/centrifuge.js"></script>
    <script>
        const messagesDiv = document.getElementById('messages');
        const statusDiv = document.getElementById('status');
        const input = document.getElementById('input');

        // 创建 Centrifugo 客户端连接
        // 注意：生产环境必须提供 token！
        const centrifuge = new Centrifuge('ws://localhost:8000/connection/websocket');

        // 连接状态监听
        centrifuge.on('connected', function(ctx) {
            statusDiv.textContent = '✅ 已连接 (transport: ' + ctx.transport + ')';
            statusDiv.className = 'status connected';
        });

        centrifuge.on('disconnected', function(ctx) {
            statusDiv.textContent = '❌ 已断开: ' + ctx.reason;
            statusDiv.className = 'status disconnected';
        });

        // 订阅频道
        const sub = centrifuge.newSubscription('chat:demo');

        // 收到消息时显示
        sub.on('publication', function(ctx) {
            const msg = document.createElement('div');
            msg.className = 'msg';
            msg.innerHTML = '<span class="time">' + new Date().toLocaleTimeString() + '</span> ' + ctx.data.text;
            messagesDiv.appendChild(msg);
            messagesDiv.scrollTop = messagesDiv.scrollHeight;
        });

        sub.on('subscribed', function(ctx) {
            addSystemMessage('已订阅频道 chat:demo');
        });

        sub.subscribe();
        centrifuge.connect();

        // 发送消息（通过后端 API）
        input.addEventListener('keypress', function(e) {
            if (e.key === 'Enter' && input.value.trim()) {
                // 实际项目中，这里应该调你的后端 API
                // 后端收到后调 Centrifugo API 发布消息
                // 这里为了演示，直接调 Centrifugo API
                fetch('http://localhost:8000/api/publish', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({
                        channel: 'chat:demo',
                        data: { text: input.value }
                    })
                });
                input.value = '';
            }
        });

        function addSystemMessage(text) {
            const msg = document.createElement('div');
            msg.className = 'msg';
            msg.innerHTML = '<span class="time">' + new Date().toLocaleTimeString() + '</span> <em>' + text + '</em>';
            messagesDiv.appendChild(msg);
        }
    </script>
</body>
</html>
```

### 步骤3：打开 HTML 文件

直接在浏览器中打开 `chat.html`（或者用任意 HTTP 服务器 serve）。

### 步骤4：多开几个标签页

打开 2-3 个浏览器标签页，都打开同一个 `chat.html`。

### 步骤5：发消息体验实时推送

在任意一个标签页输入消息按回车，**所有标签页会立刻收到消息**！

你也可以用 curl 发消息，所有浏览器标签页也会收到：

```bash
curl -X POST http://localhost:8000/api/publish \
  -H "Content-Type: application/json" \
  -d '{
    "channel": "chat:demo",
    "data": {"text": "这是来自 curl 的消息！"}
  }'
```

---

## 2.7 理解刚才发生了什么

```
时间线：

浏览器标签页1          Centrifugo             浏览器标签页2/3
    │                     │                       │
    │── WebSocket 连接 ──▶│                       │
    │                     │◀── WebSocket 连接 ────│
    │── 订阅 chat:demo ──▶│                       │
    │                     │◀── 订阅 chat:demo ────│
    │                     │                       │
    │── HTTP POST ──────▶│                       │
    │   /api/publish      │                       │
    │   channel:chat:demo │                       │
    │   data: "Hello!"    │                       │
    │                     │── WebSocket 推送 ────▶│
    │◀── WebSocket 推送 ──│   data: "Hello!"      │
    │    data: "Hello!"   │                       │
    │                     │                       │
```

**关键理解**：
1. 客户端通过 WebSocket **持久连接** 到 Centrifugo
2. 客户端 **订阅** 一个频道（channel）
3. 通过 HTTP API **发布** 消息到频道
4. Centrifugo **即时推送** 给所有订阅了该频道的客户端

---

## 2.8 配置文件详解

上面我们用命令行参数启动，实际项目中应该用配置文件。以下是一个最小的生产可用配置：

### JSON 格式（config.json）

```json
{
  "client": {
    "token": {
      "hmac_secret_key": "my-super-secret-key-change-me"
    }
  },
  "admin": {
    "enabled": true,
    "password": "admin123",
    "secret": "admin-secret-key-change-me"
  },
  "http_api": {
    "key": "my-api-key"
  },
  "health": {
    "enabled": true
  }
}
```

### YAML 格式（config.yaml）

```yaml
client:
  token:
    hmac_secret_key: "my-super-secret-key-change-me"

admin:
  enabled: true
  password: "admin123"
  secret: "admin-secret-key-change-me"

http_api:
  key: "my-api-key"

health:
  enabled: true
```

### TOML 格式（config.toml）

```toml
[client.token]
hmac_secret_key = "my-super-secret-key-change-me"

[admin]
enabled = true
password = "admin123"
secret = "admin-secret-key-change-me"

[http_api]
key = "my-api-key"

[health]
enabled = true
```

> **💡 三种格式都支持**，选你喜欢的。Centrifugo 会根据文件扩展名自动识别。

### 用配置文件启动

```bash
centrifugo --config config.json
# 或
centrifugo --config config.yaml
# 或
centrifugo --config config.toml
```

### 验证配置文件

```bash
# 检查配置文件是否有语法错误
centrifugo checkconfig --config config.json
```

---

## 2.9 环境变量配置

所有配置项都可以通过环境变量设置，规则是：

```
配置路径 → 大写 + 下划线 + 前缀 CENTRIFUGO_

例如：
client.token.hmac_secret_key → CENTRIFUGO_CLIENT_TOKEN_HMAC_SECRET_KEY
admin.enabled                → CENTRIFUGO_ADMIN_ENABLED
http_api.key                 → CENTRIFUGO_HTTP_API_KEY
```

Docker 中常用环境变量方式：

```bash
docker run -d --name centrifugo \
  -p 8000:8000 \
  -e CENTRIFUGO_CLIENT_TOKEN_HMAC_SECRET_KEY="my-secret" \
  -e CENTRIFUGO_ADMIN_ENABLED=true \
  -e CENTRIFUGO_ADMIN_PASSWORD="admin123" \
  -e CENTRIFUGO_ADMIN_SECRET="admin-secret" \
  -e CENTRIFUGO_HTTP_API_KEY="my-api-key" \
  centrifugo/centrifugo:v6 centrifugo
```

---

## 2.10 Docker Compose 开发环境

创建 `docker-compose.yml`：

```yaml
version: "3"
services:
  centrifugo:
    image: centrifugo/centrifugo:v6
    command: centrifugo --config /centrifugo/config.json
    ports:
      - "8000:8000"
    volumes:
      - ./config.json:/centrifugo/config.json:ro
    restart: unless-stopped
    ulimits:
      nofile:
        soft: 65535
        hard: 65535
```

```bash
# 启动
docker compose up -d

# 查看日志
docker compose logs -f centrifugo

# 停止
docker compose down
```

---

## 2.11 常用 CLI 命令速查

```bash
# 生成配置文件
centrifugo genconfig

# 检查配置文件
centrifugo checkconfig --config config.json

# 启动服务
centrifugo --config config.json

# 查看版本
centrifugo version

# 生成连接 JWT token（开发/测试用）
centrifugo gentoken --config config.json --user "user123"

# 生成订阅 JWT token
centrifugo gensubtoken --config config.json --user "user123" --channel "chat:room1"

# 验证 token
centrifugo checktoken --config config.json --token "eyJhbGci..."

# 查看默认配置
centrifugo defaultconfig

# 查看配置文档
centrifugo configdoc
```

---

## 2.12 新手常见坑

### 坑1：CORS 问题

```
❌ 错误：浏览器控制台报 CORS 错误
```

**原因**：浏览器安全策略阻止了跨域请求。

**解决**：在配置中设置允许的来源：

```json
{
  "client": {
    "allowed_origins": [
      "http://localhost:3000",
      "http://localhost:8080",
      "https://yourdomain.com"
    ]
  }
}
```

### 坑2：连接成功但收不到消息

```
❌ 现象：WebSocket 连接成功，但 publish 后收不到消息
```

**可能原因**：
1. 订阅的频道名和发布的频道名不一致（大小写敏感！）
2. API key 不对导致 publish 失败
3. 还没有完成订阅就开始发布了

### 坑3：Docker 中访问不了

```
❌ 错误：localhost:8000 无法访问
```

**解决**：确保端口映射正确且 Centrifugo 监听在 `0.0.0.0`（Docker 中默认就是）。

### 坑4：「insecure」模式上生产

```
❌ 千万不要在生产环境使用 --client.insecure！
```

这个选项会让任何人无需认证就能连接和订阅任意频道。开发用完后一定要关闭。

---

## 2.13 本章小结

| 你学到了 | 关键命令/操作 |
|---------|--------------|
| 安装 | `docker run` 或 `curl install.sh` |
| 生成配置 | `centrifugo genconfig` |
| 启动 | `centrifugo --config config.json` |
| 发消息 | `curl POST /api/publish` |
| 管理界面 | 浏览器访问 `http://localhost:8000` |
| 配置格式 | JSON / YAML / TOML 三选一 |

**下一章**：深入理解频道、订阅、发布等核心概念 → [第3章 核心概念](03-core-concepts.md)
