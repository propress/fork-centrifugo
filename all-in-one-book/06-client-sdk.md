# 第6章 客户端 SDK

> **阅读目标**：掌握 Centrifugo 客户端 SDK 的使用，能在浏览器和移动端接入实时消息

---

## 6.1 官方 SDK 一览

```
┌───────────────┬─────────────────┬──────────────────────────┐
│ 平台          │ SDK 名称         │ 安装方式                  │
├───────────────┼─────────────────┼──────────────────────────┤
│ 浏览器/Web    │ centrifuge-js   │ npm install centrifuge    │
│ Go            │ centrifuge-go   │ go get ...centrifuge-go   │
│ Python        │ centrifuge-py   │ pip install centrifuge    │
│ Java/Android  │ centrifuge-java │ Maven/Gradle             │
│ Swift/iOS     │ centrifuge-swift│ CocoaPods/SPM            │
│ Dart/Flutter  │ centrifuge-dart │ pub add centrifuge        │
│ gRPC (任意)   │ 标准 gRPC 客户端│ 各语言 gRPC 库            │
└───────────────┴─────────────────┴──────────────────────────┘
```

### 双向协议 vs 单向协议

```
双向协议（Bidirectional）：
  客户端 ←→ Centrifugo     使用 WebSocket
  特点：客户端可以发送命令（订阅/取消订阅/RPC）
  适用：大多数场景

单向协议（Unidirectional）：
  客户端 ← Centrifugo      使用 SSE/HTTP-Stream/WebSocket/gRPC
  特点：客户端只接收，不能发送命令
  适用：简单的数据推送场景（不需要 SDK）
```

---

## 6.2 JavaScript SDK 详解

### 安装

```bash
# npm 安装
npm install centrifuge

# 或者 CDN 引入
# <script src="https://unpkg.com/centrifuge@5/dist/centrifuge.js"></script>
```

### 基本连接

```javascript
import { Centrifuge } from 'centrifuge';

// 创建客户端实例
const centrifuge = new Centrifuge('ws://localhost:8000/connection/websocket', {
    token: 'your-jwt-token'
});

// 监听连接状态
centrifuge.on('connecting', function(ctx) {
    console.log('正在连接...', ctx.code, ctx.reason);
});

centrifuge.on('connected', function(ctx) {
    console.log('已连接!', 'transport:', ctx.transport);
});

centrifuge.on('disconnected', function(ctx) {
    console.log('已断开', ctx.code, ctx.reason);
});

centrifuge.on('error', function(ctx) {
    console.error('连接错误', ctx);
});

// 开始连接
centrifuge.connect();
```

### 自动 Token 刷新

```javascript
const centrifuge = new Centrifuge('ws://localhost:8000/connection/websocket', {
    token: initialToken,
    // SDK 会在 token 快过期时调用此函数
    getToken: async function() {
        const response = await fetch('/api/auth/centrifugo-token', {
            method: 'POST',
            headers: {
                'Authorization': 'Bearer ' + getAppToken(),
                'Content-Type': 'application/json'
            }
        });
        const data = await response.json();
        return data.token;
    }
});
```

### 订阅频道

```javascript
// 创建订阅
const sub = centrifuge.newSubscription('chat:room_42');

// 监听订阅事件
sub.on('subscribing', function(ctx) {
    console.log('正在订阅...', ctx.code, ctx.reason);
});

sub.on('subscribed', function(ctx) {
    console.log('订阅成功!');
    // ctx.data 可能包含服务端发送的初始数据
    // ctx.recoverable 表示是否支持断线恢复
});

sub.on('unsubscribed', function(ctx) {
    console.log('已取消订阅', ctx.code, ctx.reason);
});

// 监听消息
sub.on('publication', function(ctx) {
    console.log('收到消息:', ctx.data);
    // ctx.data 是 publish API 发送的 data 字段
    // ctx.offset 是消息偏移量（如果有历史记录）
    // ctx.info 是发送者信息（如果有）
});

// 监听在线状态变化
sub.on('join', function(ctx) {
    console.log('用户上线:', ctx.info.user);
});

sub.on('leave', function(ctx) {
    console.log('用户离线:', ctx.info.user);
});

// 开始订阅
sub.subscribe();
```

### 取消订阅

```javascript
// 取消订阅
sub.unsubscribe();

// 完全移除订阅（释放资源）
centrifuge.removeSubscription(sub);
```

### 获取历史消息

```javascript
// 获取最近 20 条历史消息
const result = await sub.history({ limit: 20 });
console.log('历史消息:', result.publications);
console.log('总偏移:', result.offset);
console.log('Epoch:', result.epoch);

// 从某个位置开始获取
const moreResult = await sub.history({
    limit: 20,
    since: { offset: 50, epoch: 'abc123' }
});
```

### 获取在线用户

```javascript
// 获取频道中的在线用户
const result = await sub.presence();
console.log('在线用户:', result.clients);

// 获取在线统计
const stats = await sub.presenceStats();
console.log('在线连接数:', stats.numClients);
console.log('在线用户数:', stats.numUsers);
```

### 发送 RPC 调用

RPC 允许客户端通过 Centrifugo 把请求转发到你的后端：

```javascript
try {
    const result = await centrifuge.rpc('get_user_profile', {
        user_id: 'user_1001'
    });
    console.log('RPC 响应:', result.data);
} catch (e) {
    console.error('RPC 错误:', e);
}
```

### 断开连接

```javascript
centrifuge.disconnect();
```

---

## 6.3 完整业务示例

### 示例1：实时聊天应用

```javascript
import { Centrifuge } from 'centrifuge';

class ChatClient {
    constructor(token, roomId) {
        this.centrifuge = new Centrifuge('wss://your-domain.com/connection/websocket', {
            token: token,
            getToken: () => this.refreshToken()
        });

        this.roomId = roomId;
        this.onMessage = null;
        this.onUserJoin = null;
        this.onUserLeave = null;
        this.onStatusChange = null;
    }

    connect() {
        // 连接状态
        this.centrifuge.on('connected', (ctx) => {
            this.onStatusChange?.('connected', ctx.transport);
        });
        this.centrifuge.on('disconnected', (ctx) => {
            this.onStatusChange?.('disconnected', ctx.reason);
        });

        // 订阅聊天室
        this.sub = this.centrifuge.newSubscription(`chat:${this.roomId}`);

        this.sub.on('publication', (ctx) => {
            this.onMessage?.(ctx.data, ctx.info);
        });

        this.sub.on('join', (ctx) => {
            this.onUserJoin?.(ctx.info);
        });

        this.sub.on('leave', (ctx) => {
            this.onUserLeave?.(ctx.info);
        });

        this.sub.subscribe();
        this.centrifuge.connect();
    }

    async getHistory(limit = 50) {
        const result = await this.sub.history({ limit });
        return result.publications.map(pub => ({
            data: pub.data,
            offset: pub.offset
        }));
    }

    async getOnlineUsers() {
        const result = await this.sub.presence();
        return Object.values(result.clients).map(client => ({
            user: client.user,
            info: client.connInfo
        }));
    }

    disconnect() {
        this.centrifuge.disconnect();
    }

    async refreshToken() {
        const res = await fetch('/api/auth/centrifugo-token');
        const data = await res.json();
        return data.token;
    }
}

// 使用
const chat = new ChatClient(token, 'room_42');
chat.onMessage = (data, info) => {
    console.log(`${data.user}: ${data.text}`);
};
chat.onUserJoin = (info) => {
    console.log(`${info.user} 加入了聊天室`);
};
chat.connect();
```

### 示例2：实时通知系统

```javascript
import { Centrifuge } from 'centrifuge';

class NotificationClient {
    constructor(token) {
        this.centrifuge = new Centrifuge('wss://your-domain.com/connection/websocket', {
            token: token,
            getToken: () => this.refreshToken()
        });
        this.handlers = {};
    }

    connect() {
        this.centrifuge.on('connected', () => {
            console.log('通知系统已连接');
        });

        // 个人通知频道（通过 JWT 中的 channels 自动订阅）
        // 或者客户端主动订阅
        this.centrifuge.on('publication', (ctx) => {
            // 服务端订阅的频道，消息通过此事件接收
            this.handleNotification(ctx.channel, ctx.data);
        });

        this.centrifuge.connect();
    }

    // 也可以订阅特定主题的频道
    subscribeTopic(topic) {
        const sub = this.centrifuge.newSubscription(`notify:${topic}`);
        sub.on('publication', (ctx) => {
            this.handleNotification(topic, ctx.data);
        });
        sub.subscribe();
        return sub;
    }

    handleNotification(channel, data) {
        const handler = this.handlers[data.type] || this.handlers['default'];
        if (handler) {
            handler(data);
        }
    }

    on(type, handler) {
        this.handlers[type] = handler;
    }

    disconnect() {
        this.centrifuge.disconnect();
    }

    async refreshToken() {
        const res = await fetch('/api/auth/centrifugo-token');
        return (await res.json()).token;
    }
}

// 使用
const notify = new NotificationClient(token);
notify.on('order_update', (data) => {
    showToast(`订单 ${data.order_id} 状态更新：${data.status}`);
});
notify.on('message', (data) => {
    showToast(`${data.from} 给你发了一条消息`);
});
notify.on('default', (data) => {
    console.log('未处理的通知:', data);
});
notify.connect();
```

### 示例3：实时数据仪表盘

```javascript
import { Centrifuge } from 'centrifuge';

class DashboardClient {
    constructor(token) {
        this.centrifuge = new Centrifuge('wss://your-domain.com/connection/websocket', {
            token: token
        });
        this.subscriptions = {};
    }

    connect() {
        this.centrifuge.connect();
    }

    watchMetric(metricName, callback) {
        const channel = `metrics:${metricName}`;
        const sub = this.centrifuge.newSubscription(channel);
        sub.on('publication', (ctx) => {
            callback(ctx.data);
        });
        sub.subscribe();
        this.subscriptions[metricName] = sub;
    }

    unwatchMetric(metricName) {
        const sub = this.subscriptions[metricName];
        if (sub) {
            sub.unsubscribe();
            this.centrifuge.removeSubscription(sub);
            delete this.subscriptions[metricName];
        }
    }

    disconnect() {
        this.centrifuge.disconnect();
    }
}

// 使用
const dashboard = new DashboardClient(token);
dashboard.connect();

dashboard.watchMetric('cpu', (data) => {
    updateChart('cpu', data.value, data.timestamp);
});

dashboard.watchMetric('memory', (data) => {
    updateChart('memory', data.value, data.timestamp);
});

dashboard.watchMetric('requests', (data) => {
    updateChart('requests', data.rps, data.timestamp);
});
```

---

## 6.4 单向传输（无需 SDK）

对于简单的数据推送场景，可以使用标准的 SSE（Server-Sent Events），不需要任何 SDK：

### 使用 EventSource（原生浏览器 API）

```javascript
// 直接使用浏览器原生 SSE API
const eventSource = new EventSource(
    'http://localhost:8000/connection/uni_sse?' +
    'cf_connect=' + encodeURIComponent(JSON.stringify({
        token: 'your-jwt-token',
        subs: {
            'chat:general': {}
        }
    }))
);

eventSource.onmessage = function(event) {
    const data = JSON.parse(event.data);
    console.log('收到:', data);
};

eventSource.onerror = function(event) {
    console.error('SSE 错误:', event);
};
```

### 使用 curl 测试单向 SSE

```bash
# 使用 curl 测试 SSE 流
curl -N "http://localhost:8000/connection/uni_sse" \
  -H "Content-Type: application/json" \
  -d '{
    "token": "your-jwt-token",
    "subs": {
      "chat:general": {}
    }
  }'
```

---

## 6.5 连接参数详解

```javascript
const centrifuge = new Centrifuge(url, {
    // 认证
    token: 'jwt-token',              // 初始 JWT token
    getToken: async () => '...',      // Token 刷新回调

    // 连接选项
    minReconnectDelay: 500,           // 最小重连延迟（毫秒）
    maxReconnectDelay: 20000,         // 最大重连延迟（毫秒）
    timeout: 5000,                    // 操作超时（毫秒）
    maxServerPingDelay: 10000,        // 服务端 ping 最大延迟

    // 数据格式
    data: { device: 'web' },          // 连接时携带的自定义数据

    // 调试
    debug: true                       // 开启调试日志
});
```

---

## 6.6 SDK 事件速查表

### 连接级事件

| 事件 | 触发时机 | 回调参数 |
|------|---------|---------|
| `connecting` | 正在连接（含重连） | `{code, reason}` |
| `connected` | 连接成功 | `{transport, client}` |
| `disconnected` | 连接断开 | `{code, reason}` |
| `error` | 连接错误 | `{type, error}` |
| `publication` | 收到服务端订阅的消息 | `{channel, data}` |

### 订阅级事件

| 事件 | 触发时机 | 回调参数 |
|------|---------|---------|
| `subscribing` | 正在订阅 | `{code, reason}` |
| `subscribed` | 订阅成功 | `{channel, data}` |
| `unsubscribed` | 取消订阅 | `{code, reason}` |
| `publication` | 收到频道消息 | `{data, offset, info}` |
| `join` | 用户加入频道 | `{info: {user, client}}` |
| `leave` | 用户离开频道 | `{info: {user, client}}` |
| `error` | 订阅错误 | `{type, error}` |

---

## 6.7 常见坑

### 坑1：忘记调用 connect()

```javascript
// ❌ 常见错误：创建了客户端但忘记连接
const centrifuge = new Centrifuge(url, { token });
const sub = centrifuge.newSubscription('chat:room');
sub.subscribe();
// 忘记了 centrifuge.connect() !

// ✅ 正确
centrifuge.connect();
```

### 坑2：重复创建订阅

```javascript
// ❌ 错误：每次进入页面都创建新订阅
function enterRoom(roomId) {
    const sub = centrifuge.newSubscription(`chat:${roomId}`);
    sub.subscribe(); // 如果已经有同名订阅，会报错！
}

// ✅ 正确：先检查是否已存在
function enterRoom(roomId) {
    const channel = `chat:${roomId}`;
    let sub = centrifuge.getSubscription(channel);
    if (!sub) {
        sub = centrifuge.newSubscription(channel);
        sub.on('publication', handleMessage);
    }
    sub.subscribe();
}
```

### 坑3：生产环境用 ws 而不是 wss

```javascript
// ❌ 生产环境不安全
new Centrifuge('ws://your-domain.com/connection/websocket');

// ✅ 生产环境必须用 wss
new Centrifuge('wss://your-domain.com/connection/websocket');
```

---

## 6.8 本章小结

| 概念 | 关键点 |
|------|-------|
| SDK 安装 | `npm install centrifuge` |
| 连接 | `new Centrifuge(url, {token}).connect()` |
| 订阅 | `centrifuge.newSubscription(channel).subscribe()` |
| 收消息 | `sub.on('publication', callback)` |
| Token 刷新 | 配置 `getToken` 回调 |
| 断线恢复 | SDK 自动处理 |
| 无 SDK | 单向 SSE/HTTP-Stream |

**下一章**：深入频道配置和命名空间 → [第7章 频道与命名空间](07-channel-config.md)
