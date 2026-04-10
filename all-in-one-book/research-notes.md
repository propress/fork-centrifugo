# 研究资料备忘

> 本文件保存了研究 Centrifugo 仓库过程中的关键发现，供后续续写文档时参考。

---

## 1. 仓库结构

```
centrifugo/
├── main.go                          # 入口文件
├── internal/
│   ├── cli/                         # CLI 命令（serve, genconfig, gentoken 等）
│   │   ├── serve.go                 # 主服务启动命令
│   │   ├── genconfig.go             # 生成配置文件
│   │   ├── gentoken.go              # 生成 JWT token
│   │   └── ...
│   ├── config/                      # 配置加载和解析
│   │   ├── config.go                # 主配置结构体 Config
│   │   ├── envconfig.go             # 环境变量映射
│   │   ├── validate.go              # 配置验证
│   │   └── testdata/                # 测试配置文件 (json/yaml/toml)
│   ├── configtypes/
│   │   └── types.go                 # 所有配置类型定义（非常重要！约1400行）
│   ├── app/
│   │   ├── run.go                   # 服务启动流程（~350行）
│   │   ├── mux.go                   # HTTP 路由设置
│   │   └── ...
│   ├── proxy/                       # 代理系统
│   │   ├── proxy.go                 # 代理工厂和类型定义
│   │   ├── connect_handler.go       # Connect 代理处理
│   │   ├── subscribe_handler.go     # Subscribe 代理处理
│   │   ├── publish_handler.go       # Publish 代理处理
│   │   ├── rpc_handler.go           # RPC 代理处理
│   │   ├── http_client.go           # HTTP 代理客户端
│   │   └── grpc_client.go           # gRPC 代理客户端
│   ├── consuming/                   # 消息队列消费者
│   │   ├── consuming.go             # 消费者创建工厂
│   │   ├── kafka.go                 # Kafka 消费者
│   │   ├── postgresql.go            # PostgreSQL Outbox 消费者
│   │   ├── nats_jetstream.go        # NATS JetStream 消费者
│   │   ├── redis_stream.go          # Redis Stream 消费者
│   │   ├── google_pubsub.go         # Google Pub/Sub 消费者
│   │   ├── aws_sqs.go              # AWS SQS 消费者
│   │   └── azure_service_bus.go     # Azure Service Bus 消费者
│   ├── apihandler/                  # HTTP API 处理
│   ├── apiexecutor/                 # API 命令执行
│   ├── middleware/                  # HTTP 中间件
│   ├── client/                      # 客户端连接处理
│   ├── jwtverify/                   # JWT 验证
│   └── ...
├── misc/                            # 杂项文件
│   └── proto/                       # Protobuf 定义
└── vendor/                          # 依赖
```

---

## 2. 关键配置路径映射

```
配置 YAML 路径                              → 代码位置
────────────                                ──────────
http_server.port                            → configtypes/types.go HTTPServer
engine.type                                 → config/config.go EngineConfig
engine.redis.address                        → configtypes/types.go Redis
client.token.hmac_secret_key                → configtypes/types.go Token
client.allowed_origins                      → configtypes/types.go ClientConfig
client.proxy.connect.endpoint               → configtypes/types.go ClientProxyContainer
channel.namespaces[].name                   → configtypes/types.go ChannelNamespace
channel.namespaces[].history_size           → configtypes/types.go ChannelOptions
channel.proxy.subscribe.endpoint            → configtypes/types.go ChannelProxyContainer
consumers[].type                            → configtypes/types.go ConsumerConfig
consumers[].kafka.brokers                   → configtypes/types.go KafkaConsumerConfig
consumers[].postgresql.dsn                  → configtypes/types.go PostgresConsumerConfig
http_api.key                                → configtypes/types.go HttpAPI
grpc_api.enabled                            → configtypes/types.go GrpcAPI
admin.enabled                               → configtypes/types.go Admin
prometheus.enabled                          → configtypes/types.go Prometheus
```

---

## 3. 服务启动流程要点

```
文件: internal/app/run.go

启动流程：
1. 加载 .env 文件
2. 加载配置（支持 JSON/YAML/TOML + 环境变量）
3. 设置日志
4. 写 PID 文件
5. 自动调整 GOMAXPROCS
6. 验证配置
7. 初始化指标系统
8. 构建代理映射
9. 创建 Centrifuge 核心节点
10. 配置引擎（Memory/Redis/NATS）
11. 创建 Token 验证器
12. 设置客户端处理器
13. 创建 API 执行器
14. 初始化消费者服务
15. 启动 Centrifuge 节点
16. 启动各种后台服务
17. 启动 gRPC 服务器（如启用）
18. 启动 HTTP 服务器
19. 信号处理：SIGHUP=重载配置, SIGINT/SIGTERM=优雅关闭
```

---

## 4. HTTP 路由映射

```
文件: internal/app/mux.go

外部端点：
  /connection/websocket        → WebSocket 双向连接
  /connection/sse              → SSE 双向连接（模拟）
  /connection/http_stream      → HTTP 流双向连接（模拟）
  /connection/webtransport     → WebTransport (HTTP/3)
  /connection/uni_websocket    → 单向 WebSocket
  /connection/uni_sse          → 单向 SSE
  /connection/uni_http_stream  → 单向 HTTP 流
  /connection/init             → 连接初始化
  /emulation                   → 双向模拟层

内部端点（可配置到独立端口）：
  /api/*                       → HTTP Server API
  /health                      → 健康检查
  /metrics                     → Prometheus 指标
  /debug/pprof/*               → Go pprof 调试
  /swagger                     → Swagger UI
  /dev                         → 开发工具页
  /                            → Admin Web UI
```

---

## 5. PRO 版功能（基于代码和官网研究）

```
PRO 专有功能列表：
  1. 推送通知 API (FCM/APNs/HMS)
  2. JWT 撤销/失效机制
  3. 操作速率限制
  4. 用户封禁 API
  5. 频道权限 (Capabilities)
  6. CEL 表达式自定义权限
  7. 频道模式 (Channel Patterns)
  8. 频道状态事件 (Occupied/Vacated)
  9. 缓存空事件通知
  10. 按命名空间选择不同引擎
  11. Admin SSO (OpenID Connect)
  12. 实时频道/用户追踪
  13. ClickHouse 实时分析
  14. 连接查询 API
  15. 用户状态 API
  16. 增强的集群状态洞察
  17. 性能优化（CPU/内存/带宽）
```

---

## 6. 待续写的内容

```
以下内容可以在后续续写：

1. 更详细的业务场景案例
   - 完整的聊天系统实现（前后端代码）
   - 完整的通知系统实现
   - 完整的实时监控面板实现
   - 在线协作文档方案

2. 各语言后端集成指南
   - Python (Django/Flask/FastAPI) 完整集成
   - Node.js (Express/Koa/NestJS) 完整集成
   - Go 完整集成
   - Java (Spring Boot) 完整集成
   - PHP (Laravel) 完整集成

3. 框架集成指南
   - React + Centrifugo
   - Vue + Centrifugo
   - Flutter + Centrifugo
   - React Native + Centrifugo

4. 深入源码分析
   - Centrifuge 核心引擎工作原理
   - Redis Broker 的 PUB/SUB 实现
   - 断线恢复的精确机制
   - Delta 压缩的算法细节

5. 运维手册
   - Kubernetes Helm Chart 部署
   - 灰度发布方案
   - 容灾和备份
   - 性能基准测试数据

6. 对照表和速查卡
   - 完整配置选项速查表
   - API 速查卡
   - 错误码速查表
   - 协议版本兼容性矩阵
```

---

## 7. 关键源码参考

```
配置类型定义：     internal/configtypes/types.go
配置加载逻辑：     internal/config/config.go
配置验证逻辑：     internal/config/validate.go
环境变量映射：     internal/config/envconfig.go
服务启动流程：     internal/app/run.go
HTTP 路由设置：    internal/app/mux.go
代理系统入口：     internal/proxy/proxy.go
消费者工厂：       internal/consuming/consuming.go
JWT 验证：        internal/jwtverify/
API 处理：        internal/apihandler/
API 执行：        internal/apiexecutor/
客户端处理：       internal/client/
中间件：          internal/middleware/
CLI 命令：        internal/cli/
```

---

## 8. 外部资源

```
官方文档：          https://centrifugal.dev
GitHub 仓库：       https://github.com/centrifugal/centrifugo
JS SDK：           https://github.com/centrifugal/centrifuge-js
Go SDK：           https://github.com/centrifugal/centrifuge-go
Python SDK：       https://github.com/centrifugal/centrifuge-python
Telegram 社区：    https://t.me/joinchat/ABFVWBE0AhkyyhREoaboXQ
Discord 社区：     https://discord.gg/tYgADKx
PRO 版信息：       https://centrifugal.dev/docs/pro/overview
```
