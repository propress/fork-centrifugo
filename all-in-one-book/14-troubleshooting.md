# 第14章 常见问题与踩坑指南

> **阅读目标**：汇总各阶段可能遇到的坑和解决方案，节省你的排查时间

---

## 14.1 按阶段分类的常见坑

```
┌──────────────────────────────────────────────────────────────┐
│  🟢 新手阶段（刚开始用）                                     │
│  ├── 连接不上                                                │
│  ├── CORS 问题                                               │
│  ├── Token 不对                                              │
│  ├── 收不到消息                                              │
│  └── insecure 模式忘关                                       │
│                                                              │
│  🟡 熟悉阶段（接入业务）                                     │
│  ├── 频道命名空间不存在                                       │
│  ├── 历史消息不工作                                          │
│  ├── 在线状态性能差                                          │
│  ├── 代理超时                                                │
│  └── Token 刷新失败                                          │
│                                                              │
│  🔴 精通阶段（生产运维）                                     │
│  ├── Redis 连接中断                                          │
│  ├── 节点间消息不同步                                         │
│  ├── WebSocket 被负载均衡器断开                               │
│  ├── 内存持续增长                                            │
│  └── 版本升级兼容性                                          │
└──────────────────────────────────────────────────────────────┘
```

---

## 14.2 🟢 新手阶段的坑

### 坑1：WebSocket 连接被拒绝

**现象**：客户端报 `WebSocket connection failed` 或 `401`

**排查步骤**：

```bash
# 步骤1：确认 Centrifugo 正在运行
curl http://localhost:8000/health

# 步骤2：确认 WebSocket 端点可达
curl -v http://localhost:8000/connection/websocket
# 应该看到 "400 Bad Request"（因为不是 WebSocket 请求，但说明端点存在）

# 步骤3：检查 Token
centrifugo checktoken --config config.json --token "your-token"

# 步骤4：检查日志
docker logs centrifugo 2>&1 | tail -50
```

**常见原因**：
1. Token 不正确或已过期
2. HMAC 密钥不匹配
3. 没有开启 `client.insecure`（开发模式下）

### 坑2：CORS 错误

**现象**：浏览器控制台报 `Access-Control-Allow-Origin` 错误

**解决**：

```yaml
client:
  allowed_origins:
    - "http://localhost:3000"
    - "http://localhost:8080"
    - "https://yourdomain.com"
    # 开发时可以用通配符（生产不推荐！）
    # - "*"
```

> **⚠️ 注意**：`allowed_origins` 只针对浏览器客户端。curl 和后端调用不受 CORS 限制。

### 坑3：消息发布成功但客户端收不到

**排查步骤**：

```
1. 确认频道名完全一致（大小写敏感！）
   发布：channel = "Chat:Room_1"
   订阅：channel = "chat:room_1"   ← 不一致！

2. 确认客户端已经完成订阅
   - 检查 subscribed 事件是否触发

3. 确认 API key 正确（publish 响应不报错）

4. 检查浏览器 DevTools > Network > WS
   看 WebSocket 帧中是否有消息传入
```

### 坑4：Docker 中端口不通

```bash
# 确认容器端口映射
docker ps
# PORTS 列应该显示 0.0.0.0:8000->8000/tcp

# 从宿主机测试
curl http://localhost:8000/health

# 从容器内测试
docker exec centrifugo wget -qO- http://localhost:8000/health
```

### 坑5：insecure 模式上线

```
❌ 生产环境绝对不要使用：
  - client.insecure: true
  - admin.insecure: true
  - http_api.insecure: true

这些选项会导致：
  - 任何人无需认证就能连接
  - 任何人无需密码就能访问管理界面
  - 任何人无需 API key 就能调用 API
```

---

## 14.3 🟡 熟悉阶段的坑

### 坑6：命名空间不存在

**现象**：订阅 `chat:room_1` 时被拒绝

**原因**：配置文件中没有定义 `chat` 命名空间

```yaml
# ❌ 错误：没有定义 chat 命名空间
channel:
  without_namespace:
    presence: true

# ✅ 正确：添加 chat 命名空间
channel:
  namespaces:
    - name: "chat"
      presence: true
      history_size: 100
      history_ttl: "24h"
```

### 坑7：历史消息不工作

**现象**：调用 history API 返回空结果

**常见原因**：
1. 只设了 `history_size` 没设 `history_ttl`（或反过来）
2. 消息已过期（超过 `history_ttl`）
3. 使用 Memory 引擎且 Centrifugo 重启过

```yaml
# 两个参数必须同时设置
channel:
  namespaces:
    - name: "chat"
      history_size: 100    # ✅ 保留 100 条
      history_ttl: "24h"   # ✅ 保留 24 小时
```

### 坑8：在线状态查询很慢

**现象**：`/api/presence` 响应时间长

**原因**：频道中在线用户太多（>1000）

**解决**：
```
1. 用 presence_stats 代替 presence（只返回数字，不返回列表）
2. 不要在大频道中开启 join_leave
3. 在应用层自己维护在线列表
```

### 坑9：代理请求超时

**现象**：Connect/Subscribe Proxy 经常超时

**解决**：

```yaml
client:
  proxy:
    connect:
      timeout: "5s"    # 增加超时时间

# 同时优化你的后端：
# 1. 确保代理端点响应快（< 1s）
# 2. 使用连接池
# 3. 避免在代理中做重查询
```

### 坑10：Token 刷新死循环

**现象**：Token 过期后客户端不断尝试刷新但一直失败

**常见原因**：
1. `getToken` 回调中用的认证 Token 也过期了
2. `getToken` 请求的后端地址不对
3. 后端返回的新 Token 已经过期

```javascript
// 确保 getToken 中的认证是独立的
getToken: async function() {
    try {
        const response = await fetch('/api/auth/centrifugo-token', {
            credentials: 'include'  // 使用 Cookie 认证
        });
        if (!response.ok) {
            // 如果刷新失败，可能需要让用户重新登录
            window.location.href = '/login';
            return '';
        }
        const data = await response.json();
        return data.token;
    } catch (e) {
        console.error('Token 刷新失败:', e);
        return '';
    }
}
```

---

## 14.4 🔴 精通阶段的坑

### 坑11：Redis 连接断开

**现象**：日志中出现 Redis 连接错误，消息无法投递

**排查**：

```bash
# 检查 Redis 是否正常
redis-cli ping
# 应该返回 PONG

# 检查 Redis 连接数
redis-cli info clients
# 查看 connected_clients

# 检查网络
docker exec centrifugo ping redis
```

**解决**：
```
1. 确保 Redis 高可用（Sentinel 或 Cluster）
2. 监控 Redis 的 CPU 和内存
3. Centrifugo 会自动重连 Redis
4. 检查 Redis 的 maxclients 配置
```

### 坑12：节点间消息不同步

**现象**：连接到节点1 的客户端收到消息，节点2 的客户端收不到

**排查**：

```bash
# 确认所有节点使用同一个 Redis
curl -X POST http://node1:8000/api/info -H "X-API-Key: key" -d '{}'
curl -X POST http://node2:8000/api/info -H "X-API-Key: key" -d '{}'
# 两个响应中应该能看到对方的节点信息
```

**常见原因**：
1. 节点连接了不同的 Redis
2. Redis key 前缀不同
3. 节点还在启动中

### 坑13：WebSocket 被负载均衡器断开

**现象**：连接在 60 秒后被断开

**原因**：大多数负载均衡器默认的空闲超时是 60 秒

**解决**：

```
Nginx:
  proxy_read_timeout 3600s;
  proxy_send_timeout 3600s;

AWS ALB:
  Idle timeout: 3600 秒

Azure Application Gateway:
  Connection draining timeout: 3600 秒

GCP Load Balancer:
  Backend service timeout: 3600 秒
```

### 坑14：内存持续增长

**可能原因**：

```
1. 历史消息占用过多内存
   → 减少 history_size 或 history_ttl

2. 大量空闲连接未清理
   → 检查 ping/pong 超时设置

3. 消息队列堆积（客户端消费慢）
   → 检查 client.queue_max_size

4. Goroutine 泄漏
   → 检查 /debug/pprof/goroutine

# 排查工具
curl http://localhost:9000/debug/pprof/heap > heap.prof
go tool pprof heap.prof
```

### 坑15：版本升级后不兼容

```
升级检查清单：
1. 阅读 CHANGELOG / Release Notes
2. 关注 Breaking Changes
3. 测试环境先升级验证
4. 客户端 SDK 是否需要同步升级
5. Redis 中的历史数据格式是否兼容
6. 配置文件格式是否有变化

安全升级流程：
  测试环境验证 → 金丝雀发布(1个节点) → 观察 → 全量升级
```

---

## 14.5 项目生命周期中的变化

### 早期阶段

```
项目刚启动时：
  - 用 Memory 引擎 + 单节点即可
  - 不需要复杂的认证（insecure 开发）
  - 频道结构简单
  - 不需要消费者

可能遇到的问题：
  - CORS 配置
  - Token 生成不正确
  - 频道命名不规范
```

### 中期阶段

```
项目上线运行后：
  - 切换到 Redis 引擎
  - 实现正式的 JWT 认证
  - 增加命名空间配置
  - 可能需要代理（订阅授权）
  - 开启监控

可能遇到的问题：
  - Redis 连接不稳定
  - Token 刷新逻辑有 Bug
  - 频道权限设计不合理
  - 消息格式需要调整（客户端也要改）
```

### 后期阶段

```
项目大规模运行后：
  - 多节点部署 + 负载均衡
  - 可能引入消费者（Kafka/PostgreSQL）
  - 性能优化
  - 可能考虑 PRO 版

可能遇到的问题：
  - 水平扩展的运维复杂度
  - Redis 容量规划
  - 消息堆积和延迟
  - 版本升级的兼容性
  - 大量频道的管理
```

---

## 14.6 FAQ

### Q：Centrifugo 重启后历史消息会丢失吗？

```
Memory 引擎：会丢失。所有数据都在内存中。
Redis 引擎：不会丢失。数据保存在 Redis 中（直到 TTL 过期）。
```

### Q：一个频道最多能有多少订阅者？

```
没有硬限制。取决于服务器资源。
单个热门频道可以有数十万订阅者。
但要注意：
  - 大量订阅者 + join_leave 会产生大量通知
  - presence 查询在大频道中会变慢
```

### Q：消息最大能有多大？

```
WebSocket 默认限制：65536 字节 (64KB)
可通过配置调整：
  websocket:
    message_size_limit: 131072  # 128KB

建议：保持消息小。大数据用 URL 引用。
```

### Q：Centrifugo 支持消息持久化吗？

```
Centrifugo 的历史消息是 临时缓存，不是持久存储。
如果需要永久保存消息，在你的后端/数据库中存储。
Centrifugo 的历史消息用于：
  - 断线恢复
  - 新订阅者获取最近几条消息
  - 不是用于消息归档
```

### Q：可以不用任何 SDK 吗？

```
可以！使用单向传输：
  - SSE (Server-Sent Events)：浏览器原生支持
  - HTTP-Stream：任何 HTTP 客户端
  - gRPC：标准 gRPC 客户端

但单向传输的限制：
  - 客户端不能发送命令（订阅/RPC）
  - 连接时指定所有要订阅的频道
```

### Q：Centrifugo 和我的后端之间是什么协议？

```
后端 → Centrifugo：HTTP API (POST) 或 gRPC API
Centrifugo → 后端：HTTP 代理 (POST) 或 gRPC 代理

简单理解：都是标准的 HTTP/gRPC 调用
```

---

## 14.7 调试工具速查

```bash
# 查看服务器状态
curl -X POST http://localhost:8000/api/info \
  -H "X-API-Key: key" -d '{}'

# 查看活跃频道
curl -X POST http://localhost:8000/api/channels \
  -H "X-API-Key: key" -d '{}'

# 测试发布消息
curl -X POST http://localhost:8000/api/publish \
  -H "X-API-Key: key" \
  -d '{"channel":"test","data":{"msg":"debug"}}'

# 验证 Token
centrifugo checktoken --config config.json --token "eyJ..."

# 验证配置
centrifugo checkconfig --config config.json

# 查看默认配置
centrifugo defaultconfig

# 查看日志（Docker）
docker logs -f centrifugo

# 查看 WebSocket 帧（浏览器）
# DevTools → Network → WS → 点击连接 → Messages
```

---

## 14.8 本章小结

```
新手阶段重点：确保连接通、消息通、Token 对
熟悉阶段重点：频道配置正确、代理工作正常、刷新机制完善
精通阶段重点：Redis 稳定、多节点同步、性能监控、升级流程
```

**下一章**：进阶模式与最佳实践 → [第15章 进阶模式](15-advanced.md)
