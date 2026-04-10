# 第15章 进阶模式与最佳实践

> **阅读目标**：掌握 Centrifugo 的高级使用模式，设计可靠的实时消息架构

---

## 15.1 架构模式

### 模式1：标准 PUB/SUB 模式

最基本的使用方式，后端推送、客户端接收。

```
适用场景：通知推送、数据广播、监控面板

你的后端                 Centrifugo               客户端
   │                        │                       │
   │── POST /api/publish ──▶│                       │
   │   channel: "news"      │── WebSocket push ───▶│
   │   data: {...}          │                       │
```

配置要点：
```yaml
# 简单，不需要代理和消费者
client:
  token:
    hmac_secret_key: "secret"
http_api:
  key: "api-key"
```

### 模式2：请求-响应 模式（RPC）

客户端通过 Centrifugo 调用后端逻辑，替代部分 REST API。

```
适用场景：实时查询、游戏操作、轻量级 API

客户端                 Centrifugo               你的后端
   │                      │                        │
   │── rpc("method") ────▶│── POST /rpc ──────────▶│
   │                      │                        │── 处理请求
   │                      │◀── response ───────────│
   │◀── result ───────────│                        │
```

配置要点：
```yaml
rpc:
  proxy:
    enabled: true
    endpoint: "http://backend:3000/centrifugo/rpc"
```

优势：
```
1. 复用 WebSocket 连接，减少 HTTP 开销
2. 适合高频的轻量级查询
3. 客户端不需要知道后端 API 地址
4. 自动携带用户身份信息
```

### 模式3：事件驱动模式

通过消息队列解耦后端和 Centrifugo。

```
适用场景：微服务架构、事件溯源、高吞吐场景

微服务A ──▶ ┌───────┐
微服务B ──▶ │ Kafka │ ──▶ Centrifugo ──▶ 客户端
微服务C ──▶ └───────┘
```

配置要点：
```yaml
consumers:
  - name: "events"
    type: "kafka"
    kafka:
      brokers: ["kafka:9092"]
      topics: ["centrifugo-events"]
      consumer_group: "centrifugo"

engine:
  type: "redis"
  redis:
    address: "redis:6379"
```

### 模式4：事务性 Outbox 模式

保证消息和数据库操作的一致性。

```
适用场景：订单系统、支付系统、需要事务保证的场景

你的后端 ──(事务)──▶ PostgreSQL ──(消费者)──▶ Centrifugo ──▶ 客户端
                     [业务表]
                     [outbox 表]
```

配置要点：
```yaml
consumers:
  - name: "outbox"
    type: "postgresql"
    postgresql:
      dsn: "postgres://..."
      outbox_table_name: "centrifugo_outbox"
```

### 模式5：全代理模式

所有决策都由后端做，Centrifugo 只负责传输。

```
适用场景：复杂权限系统、已有完善后端架构

客户端 ──▶ Centrifugo ──(每个动作都代理)──▶ 你的后端
                                              │
                                         连接验证
                                         订阅授权
                                         发布审核
                                         RPC 处理
```

---

## 15.2 消息设计最佳实践

### 消息格式规范

建议所有消息统一使用以下格式：

```json
{
  "type": "message_type",    // 消息类型标识
  "data": {                  // 实际数据
    "key": "value"
  },
  "meta": {                  // 元数据（可选）
    "timestamp": 1704067200,
    "version": 1
  }
}
```

### 消息类型设计

```json
// 聊天消息
{
  "type": "chat.message",
  "data": {
    "id": "msg_001",
    "sender": {"id": "u1", "name": "张三"},
    "text": "Hello!",
    "reply_to": null
  },
  "meta": {"timestamp": 1704067200}
}

// 状态更新
{
  "type": "chat.typing",
  "data": {
    "user_id": "u1",
    "typing": true
  }
}

// 系统通知
{
  "type": "system.notification",
  "data": {
    "level": "info",
    "title": "系统公告",
    "body": "服务器将在 10 分钟后维护"
  }
}
```

### 消息大小建议

```
推荐：< 1KB （大多数场景）
可接受：< 64KB （包含小型附件信息）
不推荐：> 64KB （考虑分片或 URL 引用）

对于大型数据（如图片、文件）：
  消息中只包含 URL 和元数据
  客户端自行从 CDN/OSS 下载
```

---

## 15.3 频道设计模式

### 模式1：分层频道

```
组织/租户级别：    org:company_a
                     │
团队级别：       team:engineering
                     │
项目级别：     project:web_app
                     │
频道级别：   chat:room_42
```

### 模式2：事件分离

```
一个业务实体，多个事件频道：

order:12345:status       ← 状态变更
order:12345:messages     ← 沟通消息
order:12345:logistics    ← 物流更新
```

### 模式3：扇出模式

```
全局频道 → 个人频道

全局事件：
  POST /api/publish → channel: "events:new_feature"

精确推送：
  POST /api/broadcast → channels: ["#user_1", "#user_2", "#user_3"]
```

---

## 15.4 性能优化指南

### 减少消息数量

```
❌ 每次击键都发消息
{type: "typing", text: "H"}
{type: "typing", text: "He"}
{type: "typing", text: "Hel"}
{type: "typing", text: "Hell"}
{type: "typing", text: "Hello"}

✅ 节流（throttle）后再发
{type: "typing", user: "u1", active: true}
// 300ms 后没有新按键
{type: "typing", user: "u1", active: false}
```

### 使用 Delta 压缩

对于频繁更新的结构化数据：

```yaml
channel:
  namespaces:
    - name: "realtime"
      delta_publish: true
```

### 合理设置历史

```
不需要历史的频道（实时行情、状态）：
  history_size: 0

需要少量历史（通知）：
  history_size: 20
  history_ttl: "1h"

需要较多历史（聊天）：
  history_size: 200
  history_ttl: "24h"

不要设太大，会占用大量 Redis 内存
```

### 批量操作

```bash
# 不好：循环调用 publish
for user in users:
    publish(f"#user_{user.id}", data)

# 好：使用 broadcast
broadcast(
    channels=[f"#user_{u.id}" for u in users],
    data=data
)

# 好：使用 batch
batch(commands=[
    {"publish": {"channel": ch, "data": d}}
    for ch, d in messages
])
```

---

## 15.5 可靠性设计

### 消息去重

```javascript
// 客户端侧消息去重
const processedMessages = new Set();

sub.on('publication', function(ctx) {
    const msgId = ctx.data.id;

    // 跳过已处理的消息
    if (processedMessages.has(msgId)) return;
    processedMessages.add(msgId);

    // 定期清理（避免 Set 无限增长）
    if (processedMessages.size > 10000) {
        const arr = Array.from(processedMessages);
        arr.splice(0, 5000);
        processedMessages.clear();
        arr.forEach(id => processedMessages.add(id));
    }

    handleMessage(ctx.data);
});
```

### 消息排序

```
Centrifugo 保证：
  ✅ 同一频道内的消息顺序
  ❌ 不保证跨频道的消息顺序

如果需要跨频道排序：
  在消息中包含时间戳，客户端侧排序
```

### 断线处理策略

```javascript
centrifuge.on('disconnected', function(ctx) {
    switch (ctx.code) {
        case 3001: // 正常断开（服务器关闭）
            // 自动重连会处理
            break;
        case 3500: // 被踢出
            showMessage("您已被管理员断开连接");
            break;
        case 3000: // Token 过期
            // getToken 回调会处理
            break;
        default:
            console.log('断开连接', ctx.code, ctx.reason);
    }
});
```

---

## 15.6 安全加固清单

```
□ 使用 TLS（wss://）
□ 设置强 HMAC 密钥（≥32 字节随机字符串）
□ 设置 Token 过期时间（建议 1-24 小时）
□ 配置 allowed_origins
□ API Key 不暴露给前端
□ Admin UI 只在内部网络暴露
□ 私有频道使用 $ 前缀 + 订阅 Token
□ 代理端点做好鉴权
□ 定期轮换密钥
□ 监控异常连接模式
□ 关闭所有 insecure 选项
□ Debug 端点只在内部端口
```

---

## 15.7 测试策略

### 单元测试后端集成

```python
# 测试 Token 生成
def test_generate_centrifugo_token():
    token = generate_token(user_id="u1")
    decoded = jwt.decode(token, SECRET, algorithms=["HS256"])
    assert decoded["sub"] == "u1"

# 测试代理端点
def test_connect_proxy():
    response = client.post('/centrifugo/connect', json={
        "client": "test-client",
        "transport": "websocket"
    })
    assert response.status_code == 200
    assert response.json()["result"]["user"] == "test_user"
```

### 集成测试

```python
# 使用 Centrifugo 的 API 测试完整流程
def test_publish_and_receive():
    # 1. 连接客户端（使用测试 Token）
    # 2. 订阅频道
    # 3. 通过 API 发布消息
    # 4. 验证客户端收到消息

    # 发布消息
    response = requests.post(
        'http://centrifugo:8000/api/publish',
        headers={'X-API-Key': API_KEY},
        json={'channel': 'test', 'data': {'text': 'hello'}}
    )
    assert response.json().get('error') is None
```

### 压力测试

```bash
# 使用 websocket-bench 或类似工具
# 测试连接数、消息吞吐量、延迟

# 关注指标：
# - 连接建立时间
# - 消息端到端延迟
# - 内存使用趋势
# - CPU 使用率
```

---

## 15.8 配置管理建议

### 环境分离

```
开发环境 (config.dev.yaml):
  - client.insecure: true
  - admin.insecure: true
  - engine: memory
  - log.level: debug

测试环境 (config.test.yaml):
  - 正式认证
  - engine: redis
  - log.level: info

生产环境 (config.prod.yaml):
  - 所有安全选项启用
  - engine: redis (cluster/sentinel)
  - log.level: info
  - prometheus: true
```

### 使用环境变量

```yaml
# config.yaml 中使用环境变量
client:
  token:
    hmac_secret_key: "${CENTRIFUGO_SECRET}"

engine:
  type: "redis"
  redis:
    address: "${REDIS_ADDRESS}"
    password: "${REDIS_PASSWORD}"
```

---

## 15.9 本章小结

| 主题 | 关键要点 |
|------|---------|
| 架构模式 | 选择适合业务的模式（PUB/SUB、RPC、事件驱动、Outbox） |
| 消息设计 | 统一格式、控制大小、使用类型标识 |
| 频道设计 | 分层、事件分离、合理粒度 |
| 性能优化 | 批量操作、Delta 压缩、合理历史配置 |
| 可靠性 | 消息去重、断线处理、顺序保证 |
| 安全 | TLS、强密钥、最小权限、监控 |
| 测试 | 单元测试、集成测试、压力测试 |
| 配置管理 | 环境分离、环境变量、版本控制 |

---

恭喜你读完了全部内容！🎉

现在你应该能够：
- ✅ 从零部署 Centrifugo
- ✅ 设计适合业务的频道和消息结构
- ✅ 实现安全的认证和授权
- ✅ 部署多节点生产集群
- ✅ 监控和排查问题
- ✅ 做出 OSS vs PRO 的选型决策

如果有任何问题，可以参考：
- [Centrifugo 官方文档](https://centrifugal.dev)
- [GitHub Issues](https://github.com/centrifugal/centrifugo/issues)
- [Telegram 社区](https://t.me/joinchat/ABFVWBE0AhkyyhREoaboXQ)
- [Discord 社区](https://discord.gg/tYgADKx)
