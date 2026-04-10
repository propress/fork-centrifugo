# 第12章 生产部署

> **阅读目标**：掌握 Centrifugo 生产环境的完整部署方案

---

## 12.1 生产部署检查清单

```
上线前必须确认的事项：

安全
  □ 关闭 client.insecure
  □ 关闭 admin.insecure
  □ 关闭 http_api.insecure
  □ 设置强 HMAC 密钥（≥32 字节）
  □ 设置 API Key
  □ 设置 Admin 密码和 Secret
  □ 配置 TLS（wss://）
  □ 配置 allowed_origins

可用性
  □ 使用 Redis 引擎（不要用 Memory）
  □ 部署至少 2 个节点
  □ 配置负载均衡
  □ 配置健康检查
  □ 设置文件描述符限制（ulimits）

监控
  □ 启用 Prometheus
  □ 配置 Grafana 仪表盘
  □ 设置告警规则
  □ 配置日志收集

运维
  □ 配置优雅关闭超时
  □ 准备配置热更新流程
  □ 准备版本升级方案
```

---

## 12.2 TLS 配置

### 方式1：Centrifugo 直接终止 TLS

```yaml
http_server:
  port: 443
  tls:
    enabled: true
    cert_pem_file: "/path/to/fullchain.pem"
    key_pem_file: "/path/to/privkey.pem"
```

### 方式2：Let's Encrypt 自动证书

```yaml
http_server:
  port: 443
  tls_autocert:
    enabled: true
    host_whitelist: ["realtime.yourdomain.com"]
    cache_dir: "/var/lib/centrifugo/autocert"
    email: "admin@yourdomain.com"
    http: true         # 允许 HTTP-01 验证
    http_addr: ":80"   # HTTP 验证端口
```

### 方式3：在负载均衡器终止 TLS（推荐）

```
客户端 ──(wss://)──▶ Nginx/ALB ──(ws://)──▶ Centrifugo
                     终止 TLS                 无需 TLS 配置
```

Nginx 配置：

```nginx
server {
    listen 443 ssl http2;
    server_name realtime.yourdomain.com;

    ssl_certificate /path/to/fullchain.pem;
    ssl_certificate_key /path/to/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    location /connection/ {
        proxy_pass http://centrifugo_upstream;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }

    location /api/ {
        proxy_pass http://centrifugo_upstream;
        proxy_set_header Host $host;
    }

    location / {
        proxy_pass http://centrifugo_upstream;
        proxy_set_header Host $host;
    }
}
```

---

## 12.3 Docker 生产部署

### Dockerfile 优化

```yaml
# docker-compose.prod.yml
version: "3"

services:
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes --maxmemory 2gb --maxmemory-policy allkeys-lru
    volumes:
      - redis-data:/data
    restart: always
    deploy:
      resources:
        limits:
          memory: 3G

  centrifugo:
    image: centrifugo/centrifugo:v6
    command: centrifugo --config /centrifugo/config.yaml
    volumes:
      - ./config.yaml:/centrifugo/config.yaml:ro
    depends_on:
      - redis
    restart: always
    ulimits:
      nofile:
        soft: 65535
        hard: 65535
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: "2"
          memory: 1G
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:8000/health"]
      interval: 30s
      timeout: 5s
      retries: 3

  nginx:
    image: nginx:alpine
    ports:
      - "443:443"
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certs:/etc/nginx/certs:ro
    depends_on:
      - centrifugo
    restart: always

volumes:
  redis-data:
```

---

## 12.4 Kubernetes 部署

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: centrifugo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: centrifugo
  template:
    metadata:
      labels:
        app: centrifugo
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8000"
        prometheus.io/path: "/metrics"
    spec:
      containers:
        - name: centrifugo
          image: centrifugo/centrifugo:v6
          args: ["centrifugo", "--config", "/centrifugo/config.yaml"]
          ports:
            - containerPort: 8000
              name: http
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "2000m"
              memory: "1Gi"
          livenessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 5
            periodSeconds: 30
          readinessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 3
            periodSeconds: 10
          volumeMounts:
            - name: config
              mountPath: /centrifugo
              readOnly: true
      volumes:
        - name: config
          configMap:
            name: centrifugo-config
```

### Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: centrifugo
spec:
  type: ClusterIP
  ports:
    - port: 8000
      targetPort: 8000
      protocol: TCP
  selector:
    app: centrifugo
```

### Ingress（WebSocket 支持）

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: centrifugo
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
spec:
  tls:
    - hosts: ["realtime.yourdomain.com"]
      secretName: centrifugo-tls
  rules:
    - host: realtime.yourdomain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: centrifugo
                port:
                  number: 8000
```

---

## 12.5 生产配置模板

```yaml
# config.yaml - 生产环境配置模板
http_server:
  port: 8000
  # 内部端口（Admin、Prometheus、Debug）
  internal_port: "9000"

log:
  level: "info"

engine:
  type: "redis"
  redis:
    address: "redis:6379"
    password: "${REDIS_PASSWORD}"

client:
  token:
    hmac_secret_key: "${CENTRIFUGO_TOKEN_SECRET}"
  allowed_origins:
    - "https://yourdomain.com"
    - "https://app.yourdomain.com"
  connection_limit: 100000
  channel_limit: 128
  queue_max_size: 1048576

channel:
  namespaces:
    - name: "chat"
      presence: true
      history_size: 200
      history_ttl: "24h"
      force_recovery: true

    - name: "notify"
      history_size: 50
      history_ttl: "7d"

http_api:
  key: "${CENTRIFUGO_API_KEY}"

admin:
  enabled: true
  password: "${ADMIN_PASSWORD}"
  secret: "${ADMIN_SECRET}"
  external: false  # 只在内部端口暴露

health:
  enabled: true

prometheus:
  enabled: true

shutdown:
  timeout: "30s"

node:
  name: "${HOSTNAME}"
```

---

## 12.6 优雅关闭

```yaml
shutdown:
  timeout: "30s"    # 给连接 30 秒的时间优雅关闭
```

关闭流程：

```
收到 SIGTERM
     │
     ▼
停止接受新连接
     │
     ▼
通知所有客户端服务器即将关闭
     │
     ▼
等待 timeout 秒让客户端自行重连到其他节点
     │
     ▼
强制关闭剩余连接
     │
     ▼
进程退出
```

---

## 12.7 性能调优

### 操作系统级别

```bash
# 增加文件描述符限制
# /etc/security/limits.conf
centrifugo soft nofile 65535
centrifugo hard nofile 65535

# 或者 systemd service 中
[Service]
LimitNOFILE=65535

# 增加 TCP 连接队列
sysctl -w net.core.somaxconn=65535
sysctl -w net.ipv4.tcp_max_syn_backlog=65535
```

### Centrifugo 级别

```yaml
client:
  # 每个连接的消息队列大小（字节）
  queue_max_size: 1048576   # 1MB
  # 并发操作限制
  concurrency: 0            # 0 = 无限制
  # Ping 间隔
  ping_interval: "25s"

# WebSocket 优化
websocket:
  compression: false        # 压缩会增加 CPU 开销
  write_buffer_size: 0      # 0 = 默认
  read_buffer_size: 0       # 0 = 默认
  use_write_buffer_pool: true  # 复用 buffer 减少 GC
```

### 容量规划

```
单个 Centrifugo 节点（4 核 8GB）大约能承受：

  50,000 - 100,000 并发 WebSocket 连接
  10,000 - 50,000 消息/秒 的吞吐量

影响因素：
  - 消息大小
  - 每个连接的订阅数
  - 是否启用 presence、history
  - 是否启用压缩
  - Redis 网络延迟
```

---

## 12.8 版本升级策略

### 滚动升级

```bash
# Kubernetes 中
kubectl set image deployment/centrifugo centrifugo=centrifugo/centrifugo:v6.x.y

# Docker Compose 中
docker compose pull centrifugo
docker compose up -d --no-deps centrifugo
```

### 升级注意事项

```
1. 阅读 CHANGELOG 了解 breaking changes
2. 先在测试环境验证
3. 使用滚动升级，不要同时停止所有节点
4. 客户端 SDK 通常向后兼容
5. Redis 引擎的数据格式在大版本间可能变化
```

---

## 12.9 本章小结

| 部署方式 | 适用场景 | 复杂度 |
|---------|---------|--------|
| 单节点 Docker | 开发/小规模 | ⭐ |
| Docker Compose + Redis | 中小规模生产 | ⭐⭐ |
| Kubernetes | 大规模生产 | ⭐⭐⭐ |

**下一章**：PRO 版 vs 开源版对比 → [第13章 PRO 版对比](13-pro-vs-oss.md)
