# 第8章 水平扩展

> **阅读目标**：理解 Centrifugo 的扩展机制，能部署多节点集群

---

## 8.1 为什么需要水平扩展？

```
单节点 Centrifugo 的限制：

┌─────────────────────────────────────────────────────┐
│  单节点                                              │
│                                                      │
│  ✅ 处理 ~50,000 并发连接（取决于硬件）               │
│  ✅ 简单部署                                         │
│  ❌ 单点故障                                         │
│  ❌ 无法横向扩展                                      │
│  ❌ 内存引擎，重启后数据丢失                          │
│                                                      │
│  适用：开发环境、小规模应用（<5万连接）                │
└─────────────────────────────────────────────────────┘

多节点 Centrifugo（通过 Redis/NATS）：

┌─────────────────────────────────────────────────────┐
│  多节点集群                                          │
│                                                      │
│  ✅ 水平扩展到百万级连接                             │
│  ✅ 高可用（节点宕机不影响整体）                      │
│  ✅ 负载均衡                                         │
│  ✅ 历史消息持久化（Redis）                           │
│                                                      │
│  适用：生产环境、大规模应用                           │
└─────────────────────────────────────────────────────┘
```

---

## 8.2 扩展架构概览

```
                    ┌──────────────┐
                    │  负载均衡器   │
                    │ (Nginx/ALB)  │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
        ┌─────┴────┐ ┌────┴─────┐ ┌────┴─────┐
        │Centrifugo│ │Centrifugo│ │Centrifugo│
        │  Node 1  │ │  Node 2  │ │  Node 3  │
        └─────┬────┘ └────┬─────┘ └────┬─────┘
              │            │            │
              └────────────┼────────────┘
                           │
                    ┌──────┴───────┐
                    │    Redis     │
                    │  (消息同步)   │
                    └──────────────┘

关键：所有 Centrifugo 节点通过 Redis 同步消息。
     用户连到任意一个节点都能收到所有消息。
```

---

## 8.3 引擎类型

Centrifugo 支持三种引擎用于节点间消息同步：

### 引擎对比

| 特性 | Memory | Redis | NATS |
|------|--------|-------|------|
| 多节点 | ❌ | ✅ | ✅ |
| 消息历史 | ⚡ 快但不持久 | ✅ 持久 | ❌ 不支持 |
| 在线状态 | ⚡ 快但不持久 | ✅ 持久 | ❌ 不支持 |
| 断线恢复 | ⚠️ 仅同节点 | ✅ 跨节点 | ❌ 不支持 |
| 部署复杂度 | ⭐ | ⭐⭐ | ⭐⭐ |
| 适用场景 | 开发/单节点 | 大多数生产 | 只需要PUB/SUB |

### 推荐

```
开发环境        → Memory（默认，无需额外依赖）
一般生产环境    → Redis（最常用，功能最全）
只需消息转发    → NATS（轻量级，高性能）
混合使用        → Redis + NATS（实验性功能）
```

---

## 8.4 Redis 引擎配置

### 基本配置

```yaml
engine:
  type: "redis"
  redis:
    address: "redis:6379"
    # 可选
    password: "your-redis-password"
    db: 0
    prefix: "centrifugo"    # Redis key 前缀
    presence_ttl: "30s"     # 在线状态 TTL
```

### Redis Sentinel（高可用）

```yaml
engine:
  type: "redis"
  redis:
    sentinel:
      master_name: "mymaster"
      addresses:
        - "sentinel1:26379"
        - "sentinel2:26379"
        - "sentinel3:26379"
      password: "sentinel-password"
    password: "redis-password"
```

```
Redis Sentinel 架构：

┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  Sentinel 1 │  │  Sentinel 2 │  │  Sentinel 3 │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │
       └────────────────┼────────────────┘
                        │
                 ┌──────┴──────┐
                 │ Redis Master │ ◀── 写入
                 └──────┬──────┘
                        │
              ┌─────────┴─────────┐
              │                   │
        ┌─────┴─────┐      ┌─────┴─────┐
        │Redis Slave │      │Redis Slave │ ◀── 只读（可选）
        └───────────┘      └───────────┘
```

### Redis Cluster

```yaml
engine:
  type: "redis"
  redis:
    cluster:
      addresses:
        - "redis-node1:6379"
        - "redis-node2:6379"
        - "redis-node3:6379"
        - "redis-node4:6379"
        - "redis-node5:6379"
        - "redis-node6:6379"
```

### Redis TLS

```yaml
engine:
  type: "redis"
  redis:
    address: "redis:6380"
    tls:
      enabled: true
      cert_pem: "/path/to/cert.pem"
      key_pem: "/path/to/key.pem"
      root_ca_pem: "/path/to/ca.pem"
      insecure_skip_verify: false
```

### 兼容的 Redis 替代品

Centrifugo 支持以下 Redis 兼容存储：

```
✅ Redis（官方）
✅ AWS ElastiCache
✅ Valkey
✅ KeyDB
✅ DragonflyDB
```

---

## 8.5 NATS Broker 配置

NATS 作为一个轻量级的消息 Broker，适合只需要消息转发的场景。

### 基本配置

```yaml
broker:
  type: "nats"
  nats:
    url: "nats://nats:4222"
    prefix: "centrifugo"
```

> **⚠️ 注意**：NATS broker **不支持** 消息历史和在线状态。如果需要这些功能，必须用 Redis。

### NATS + Redis 分离使用

可以用 NATS 做消息 Broker，用 Redis 做 Presence Manager：

```yaml
broker:
  type: "nats"
  nats:
    url: "nats://nats:4222"

presence_manager:
  type: "redis"
  redis:
    address: "redis:6379"
```

---

## 8.6 多节点部署步骤

### 步骤1：启动 Redis

```bash
docker run -d --name redis \
  -p 6379:6379 \
  redis:7-alpine
```

### 步骤2：配置文件

```yaml
# config.yaml（所有节点共用）
http_server:
  port: 8000

engine:
  type: "redis"
  redis:
    address: "redis:6379"

client:
  token:
    hmac_secret_key: "your-secret-key"

admin:
  enabled: true
  password: "admin123"
  secret: "admin-secret"

http_api:
  key: "your-api-key"

health:
  enabled: true

prometheus:
  enabled: true
```

### 步骤3：启动多个节点

```bash
# 节点1
docker run -d --name centrifugo1 \
  -p 8001:8000 \
  -v $(pwd)/config.yaml:/centrifugo/config.yaml:ro \
  centrifugo/centrifugo:v6 \
  centrifugo --config /centrifugo/config.yaml

# 节点2
docker run -d --name centrifugo2 \
  -p 8002:8000 \
  -v $(pwd)/config.yaml:/centrifugo/config.yaml:ro \
  centrifugo/centrifugo:v6 \
  centrifugo --config /centrifugo/config.yaml

# 节点3
docker run -d --name centrifugo3 \
  -p 8003:8000 \
  -v $(pwd)/config.yaml:/centrifugo/config.yaml:ro \
  centrifugo/centrifugo:v6 \
  centrifugo --config /centrifugo/config.yaml
```

### 步骤4：配置负载均衡

Nginx 配置示例：

```nginx
upstream centrifugo {
    # 使用 ip_hash 确保同一客户端连接到同一节点
    # （对 WebSocket 不是必须的，但对 HTTP-Stream/SSE 是）
    ip_hash;

    server centrifugo1:8000;
    server centrifugo2:8000;
    server centrifugo3:8000;
}

server {
    listen 80;
    server_name realtime.yourdomain.com;

    location /connection/ {
        proxy_pass http://centrifugo;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket 超时
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }

    location /api/ {
        proxy_pass http://centrifugo;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location / {
        proxy_pass http://centrifugo;
        proxy_set_header Host $host;
    }
}
```

### 步骤5：验证集群

```bash
# 查看所有节点信息
curl -X POST http://localhost:8001/api/info \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{}'
```

响应中应该能看到所有节点：

```json
{
  "result": {
    "nodes": [
      {"name": "centrifugo1_8000", "num_clients": 100, ...},
      {"name": "centrifugo2_8000", "num_clients": 150, ...},
      {"name": "centrifugo3_8000", "num_clients": 120, ...}
    ]
  }
}
```

---

## 8.7 Docker Compose 完整集群

```yaml
version: "3"

services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data

  centrifugo1:
    image: centrifugo/centrifugo:v6
    command: centrifugo --config /centrifugo/config.yaml
    volumes:
      - ./config.yaml:/centrifugo/config.yaml:ro
    depends_on:
      - redis
    ulimits:
      nofile:
        soft: 65535
        hard: 65535

  centrifugo2:
    image: centrifugo/centrifugo:v6
    command: centrifugo --config /centrifugo/config.yaml
    volumes:
      - ./config.yaml:/centrifugo/config.yaml:ro
    depends_on:
      - redis
    ulimits:
      nofile:
        soft: 65535
        hard: 65535

  nginx:
    image: nginx:alpine
    ports:
      - "8000:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - centrifugo1
      - centrifugo2

volumes:
  redis-data:
```

---

## 8.8 性能优化建议

### Redis 优化

```
1. 使用 Redis Cluster 分散负载
2. 确保 Redis 和 Centrifugo 在同一个网络/数据中心
3. Redis 内存要足够（历史消息会占用内存）
4. 监控 Redis 的 CPU 和内存使用率
5. 考虑使用 KeyDB 或 DragonflyDB 获得更好性能
```

### Centrifugo 节点优化

```
1. 增加文件描述符限制（ulimits nofile）
2. 调整 GOMAXPROCS（默认自动设置）
3. 合理设置 client.queue_max_size
4. 监控每个节点的连接数和内存
5. 使用 Prometheus + Grafana 监控
```

---

## 8.9 常见坑

### 坑1：Redis 连接不上

```
❌ 错误：cannot connect to Redis

常见原因：
1. Redis 地址配置错误（Docker 中用服务名而不是 localhost）
2. Redis 需要密码但没配置
3. 防火墙阻止了 6379 端口
```

### 坑2：节点间消息不同步

```
❌ 现象：客户端A发消息，连接到另一个节点的客户端B收不到

检查：
1. 所有节点是否连接到同一个 Redis
2. Redis 是否正常运行
3. 用 /api/info 确认所有节点都可见
```

### 坑3：WebSocket 被负载均衡器超时断开

```
❌ 现象：连接 60 秒后自动断开

解决：增加 Nginx 的 proxy_read_timeout：
  proxy_read_timeout 3600s;
  proxy_send_timeout 3600s;
```

---

## 8.10 本章小结

| 场景 | 推荐方案 |
|------|---------|
| 开发环境 | Memory 引擎（默认） |
| 生产环境（< 5万连接） | 单节点 + Redis |
| 生产环境（> 5万连接） | 多节点 + Redis + Nginx |
| 超大规模 | 多节点 + Redis Cluster |
| 只需消息转发 | NATS Broker |

**下一章**：学习代理机制 → [第9章 代理机制](09-proxy.md)
