# 第11章 管理与监控

> **阅读目标**：掌握 Centrifugo 的管理界面和监控方案

---

## 11.1 Admin UI

Centrifugo 内置了一个 Web 管理界面，可以查看服务器状态、管理频道和连接。

### 启用 Admin UI

```yaml
admin:
  enabled: true
  password: "your-admin-password"
  secret: "your-admin-jwt-secret"
```

访问 `http://localhost:8000` 即可看到登录页面。

### Admin UI 功能

```
┌──────────────────────────────────────────────────────────┐
│  Centrifugo Admin Dashboard                              │
│                                                          │
│  📊 概览                                                 │
│  ├── 当前连接数、用户数                                   │
│  ├── 频道数、消息吞吐率                                   │
│  ├── 节点列表和各节点状态                                 │
│  └── 内存使用和运行时间                                   │
│                                                          │
│  📡 操作                                                  │
│  ├── 发布消息到指定频道                                   │
│  ├── 查看频道的在线用户                                   │
│  ├── 查看频道的历史消息                                   │
│  ├── 断开指定用户的连接                                   │
│  └── 取消用户的订阅                                       │
│                                                          │
│  🔧 调试                                                  │
│  ├── 查看服务器信息（/api/info）                          │
│  └── 执行 API 命令                                       │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### 安全建议

```
生产环境 Admin UI 安全配置：

1. 使用强密码
2. 只在内部网络暴露 Admin UI
3. 或者将 Admin 放在内部端口：

admin:
  enabled: true
  password: "strong-password"
  secret: "strong-secret"
  external: false  # 只在内部端口暴露

http_server:
  internal_port: "9000"  # Admin 只在 9000 端口可访问
```

---

## 11.2 Prometheus 监控

### 启用 Prometheus

```yaml
prometheus:
  enabled: true
  handler_prefix: "/metrics"
```

访问 `http://localhost:8000/metrics` 查看指标。

### 关键指标

```
# 连接相关
centrifugo_transport_connections_total        # 当前连接总数
centrifugo_transport_messages_sent_total      # 发送的消息总数
centrifugo_transport_messages_received_total  # 收到的消息总数

# API 相关
centrifugo_api_command_duration_seconds       # API 命令耗时
centrifugo_api_command_total                  # API 命令总数

# 频道相关
centrifugo_node_num_channels                  # 活跃频道数
centrifugo_node_num_clients                   # 客户端连接数
centrifugo_node_num_users                     # 唯一用户数
centrifugo_node_num_subscriptions             # 订阅总数

# 消息相关
centrifugo_node_messages_sent_total           # 节点发送的消息数
centrifugo_node_messages_received_total       # 节点收到的消息数

# Redis Broker 相关（如果使用 Redis）
centrifugo_redis_*                            # Redis 操作指标
```

### Prometheus 配置

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'centrifugo'
    scrape_interval: 15s
    static_configs:
      - targets:
        - 'centrifugo1:8000'
        - 'centrifugo2:8000'
        - 'centrifugo3:8000'
    metrics_path: '/metrics'
```

---

## 11.3 Grafana 仪表盘

Centrifugo 提供了官方的 Grafana 仪表盘。

### 导入方式

1. 在 Grafana 中选择 "Import Dashboard"
2. 搜索 Centrifugo 或使用官方仪表盘 ID
3. 选择 Prometheus 数据源

### 仪表盘包含的面板

```
┌────────────────────────────────────────────────────────┐
│  Centrifugo Grafana Dashboard                          │
│                                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐│
│  │   连接数      │  │   消息速率    │  │   频道数      ││
│  │   1,250      │  │   5.2K/s     │  │   450        ││
│  └──────────────┘  └──────────────┘  └──────────────┘│
│                                                        │
│  ┌────────────────────────────────────────────────────┐│
│  │  连接数趋势图                                      ││
│  │  ▁▂▃▅▇██▇▅▃▂▁▂▃▅▇██▇▅▃▂                          ││
│  └────────────────────────────────────────────────────┘│
│                                                        │
│  ┌────────────────────────────────────────────────────┐│
│  │  消息发送/接收速率                                  ││
│  │  ▂▃▅▇██▇▅▃▂▂▃▅▇██▇▅▃                              ││
│  └────────────────────────────────────────────────────┘│
│                                                        │
│  ┌────────────────────────────────────────────────────┐│
│  │  API 延迟分布                                      ││
│  │  p50: 1ms  p95: 5ms  p99: 15ms                    ││
│  └────────────────────────────────────────────────────┘│
│                                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐│
│  │  CPU 使用率   │  │  内存使用     │  │  Goroutine   ││
│  │   25%        │  │   512MB      │  │   1,500      ││
│  └──────────────┘  └──────────────┘  └──────────────┘│
└────────────────────────────────────────────────────────┘
```

---

## 11.4 Health Check

```yaml
health:
  enabled: true
  handler_prefix: "/health"
```

```bash
# 健康检查
curl http://localhost:8000/health

# 用于 Kubernetes liveness probe
# 或 Docker healthcheck
```

Docker Compose 中配置健康检查：

```yaml
services:
  centrifugo:
    image: centrifugo/centrifugo:v6
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
```

---

## 11.5 日志

### 日志级别

```yaml
log:
  level: "info"     # trace, debug, info, warn, error
  file: ""          # 可选：输出到文件
```

### 日志级别使用建议

```
level: "trace"   → 开发调试时使用，输出非常详细
level: "debug"   → 开发环境
level: "info"    → 生产环境（默认）
level: "warn"    → 只关注警告和错误
level: "error"   → 只关注错误
```

### 日志格式

Centrifugo 使用结构化 JSON 日志（zerolog），便于 ELK/Loki 等日志系统收集：

```json
{"level":"info","time":"2024-01-01T00:00:00Z","message":"starting Centrifugo","version":"6.7.1"}
{"level":"info","time":"2024-01-01T00:00:01Z","message":"serving HTTP on :8000"}
```

---

## 11.6 Debug / Profiling

```yaml
debug:
  enabled: true
  handler_prefix: "/debug/pprof"
```

用于性能分析（Go pprof）：

```bash
# CPU 分析
go tool pprof http://localhost:8000/debug/pprof/profile?seconds=30

# 内存分析
go tool pprof http://localhost:8000/debug/pprof/heap

# Goroutine 分析
curl http://localhost:8000/debug/pprof/goroutine?debug=2
```

> **⚠️ 生产环境建议**：Debug 端点只在内部端口暴露，不要对外开放。

---

## 11.7 OpenTelemetry

```yaml
open_telemetry:
  enabled: true
  api: true         # 跟踪 API 请求
  consuming: true   # 跟踪消费者操作
```

支持将 trace 数据导出到 Jaeger、Zipkin 等。

---

## 11.8 Swagger UI

```yaml
swagger:
  enabled: true
  handler_prefix: "/swagger"
```

访问 `http://localhost:8000/swagger` 查看交互式 API 文档。

---

## 11.9 监控告警建议

| 指标 | 阈值 | 告警级别 |
|------|------|---------|
| 连接数突然下降 >50% | 与前5分钟均值对比 | 🔴 Critical |
| API 延迟 p99 > 1s | 持续 5 分钟 | 🟡 Warning |
| Redis 连接失败 | 任何失败 | 🔴 Critical |
| 内存使用 > 80% | 持续增长 | 🟡 Warning |
| 消息堆积 | 消费者 lag 增长 | 🟡 Warning |
| 节点数减少 | 与预期节点数对比 | 🔴 Critical |

---

## 11.10 本章小结

| 功能 | 配置 | 用途 |
|------|------|------|
| Admin UI | `admin.enabled: true` | Web 管理界面 |
| Prometheus | `prometheus.enabled: true` | 指标收集 |
| Health | `health.enabled: true` | 健康检查 |
| Debug | `debug.enabled: true` | 性能分析 |
| Swagger | `swagger.enabled: true` | API 文档 |
| OpenTelemetry | `open_telemetry.enabled: true` | 分布式追踪 |

**下一章**：生产环境部署 → [第12章 生产部署](12-production.md)
