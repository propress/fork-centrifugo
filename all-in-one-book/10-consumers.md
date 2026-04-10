# 第10章 消息队列消费者

> **阅读目标**：理解 Centrifugo 的消费者模式，掌握 Kafka、PostgreSQL 等消息源的接入方式

---

## 10.1 为什么需要消费者？

在前面的章节中，发布消息的方式是后端直接调用 Centrifugo 的 HTTP API。但在某些场景下，这种方式有局限：

```
问题场景：事务性消息保证

假设你要在数据库事务中发送消息：

BEGIN TRANSACTION;
  INSERT INTO orders ...;          ← 创建订单
  -- 调用 Centrifugo API 推送通知   ← 如果这里失败了？
COMMIT;

问题：
1. API 调用失败 → 订单创建了但通知没发出
2. API 调用成功但事务回滚 → 通知发了但订单不存在
3. 网络超时 → 不知道通知是否发出
```

### 解决方案：Transactional Outbox 模式

```
┌──────────────────────────────────────────────────────────┐
│                  Outbox 模式                              │
│                                                          │
│  BEGIN TRANSACTION;                                      │
│    INSERT INTO orders (...);                             │
│    INSERT INTO outbox (channel, data, ...);  ← 写入同库  │
│  COMMIT;                                                 │
│                                                          │
│  事务保证了订单和消息同时成功或失败！                     │
│                                                          │
│  Centrifugo 消费者自动从 outbox 表读取并推送消息          │
└──────────────────────────────────────────────────────────┘
```

```
完整流程：

你的后端          数据库 outbox 表         Centrifugo          客户端
   │                    │                     │                  │
   │── 写入订单+消息 ──▶│                     │                  │
   │   (同一事务)       │                     │                  │
   │                    │                     │                  │
   │                    │── 消费者自动读取 ──▶│                  │
   │                    │                     │── 实时推送 ────▶│
   │                    │                     │                  │
```

---

## 10.2 支持的消费者类型

```
┌──────────────────────────────────────────────────────────────┐
│                    Centrifugo 消费者                          │
│                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────────┐    │
│  │ PostgreSQL  │  │   Kafka     │  │ NATS JetStream   │    │
│  │ Outbox 表   │  │  Topics     │  │  Streams         │    │
│  └─────────────┘  └─────────────┘  └──────────────────┘    │
│                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────────┐    │
│  │ Redis Stream│  │ Google      │  │ Azure Service    │    │
│  │             │  │ Pub/Sub     │  │ Bus              │    │
│  └─────────────┘  └─────────────┘  └──────────────────┘    │
│                                                              │
│  ┌─────────────┐                                            │
│  │  AWS SQS    │                                            │
│  └─────────────┘                                            │
└──────────────────────────────────────────────────────────────┘
```

---

## 10.3 PostgreSQL 消费者（Outbox 模式）

### Outbox 表结构

在你的数据库中创建 outbox 表：

```sql
CREATE TABLE IF NOT EXISTS centrifugo_outbox (
    id BIGSERIAL PRIMARY KEY,
    method VARCHAR(255) NOT NULL,       -- "publish", "broadcast" 等
    payload JSONB NOT NULL,              -- 消息内容
    partition INTEGER NOT NULL DEFAULT 0, -- 分区号
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 创建索引
CREATE INDEX idx_centrifugo_outbox_partition_id
  ON centrifugo_outbox (partition, id);
```

### 写入 Outbox

```sql
-- 在你的业务事务中写入
BEGIN;

INSERT INTO orders (user_id, product_id, amount)
VALUES (1001, 'PROD-001', 99.99);

INSERT INTO centrifugo_outbox (method, payload, partition)
VALUES (
  'publish',
  '{"channel": "#user_1001", "data": {"type": "order_created", "order_id": "ORD-001"}}',
  0
);

COMMIT;
```

### Centrifugo 配置

```yaml
consumers:
  - name: "pg_outbox"
    enabled: true
    type: "postgresql"
    postgresql:
      dsn: "postgres://user:password@localhost:5432/mydb?sslmode=disable"
      outbox_table_name: "centrifugo_outbox"
      num_partitions: 1
      partition_select_limit: 100
      partition_poll_interval: "300ms"
      # 可选：使用 LISTEN/NOTIFY 实现低延迟通知
      partition_notification_channel: "centrifugo_outbox_notify"
```

### 使用 LISTEN/NOTIFY 降低延迟

```sql
-- 创建触发器，在写入 outbox 后自动通知
CREATE OR REPLACE FUNCTION notify_centrifugo_outbox()
RETURNS TRIGGER AS $$
BEGIN
  PERFORM pg_notify('centrifugo_outbox_notify', NEW.partition::text);
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER centrifugo_outbox_trigger
  AFTER INSERT ON centrifugo_outbox
  FOR EACH ROW
  EXECUTE FUNCTION notify_centrifugo_outbox();
```

---

## 10.4 Kafka 消费者

### 配置

```yaml
consumers:
  - name: "kafka_events"
    enabled: true
    type: "kafka"
    kafka:
      brokers:
        - "kafka1:9092"
        - "kafka2:9092"
      topics:
        - "centrifugo-events"
      consumer_group: "centrifugo"
      max_poll_records: 100
```

### 消息格式

Kafka 消息需要包含 `centrifugo-method` 头来指定操作类型：

```
Kafka 消息头:
  centrifugo-method: publish

Kafka 消息体:
{
  "channel": "chat:room_42",
  "data": {"text": "来自 Kafka 的消息"}
}
```

### 生产者示例（Python）

```python
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers=['kafka:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

# 发布消息
producer.send(
    'centrifugo-events',
    value={
        "channel": "chat:room_42",
        "data": {"text": "Hello from Kafka!", "user": "system"}
    },
    headers=[
        ('centrifugo-method', b'publish')
    ]
)

# 广播消息
producer.send(
    'centrifugo-events',
    value={
        "channels": ["chat:room_1", "chat:room_2"],
        "data": {"text": "广播消息"}
    },
    headers=[
        ('centrifugo-method', b'broadcast')
    ]
)
```

### Kafka TLS 和 SASL 认证

```yaml
consumers:
  - name: "kafka_secure"
    enabled: true
    type: "kafka"
    kafka:
      brokers:
        - "kafka:9093"
      topics:
        - "events"
      consumer_group: "centrifugo"
      tls:
        enabled: true
        cert_pem_file: "/path/to/cert.pem"
        key_pem_file: "/path/to/key.pem"
        root_ca_pem_file: "/path/to/ca.pem"
      sasl_mechanism: "scram-sha-256"
      sasl_user: "centrifugo"
      sasl_password: "secret"
```

### AWS MSK IAM 认证

```yaml
consumers:
  - name: "msk_events"
    enabled: true
    type: "kafka"
    kafka:
      brokers:
        - "b-1.msk-cluster.kafka.us-east-1.amazonaws.com:9098"
      topics:
        - "events"
      consumer_group: "centrifugo"
      tls:
        enabled: true
      sasl_mechanism: "aws-msk-iam"
      # 可选：使用 AssumeRole
      assume_role_arn: "arn:aws:iam::123456789:role/MskConsumerRole"
```

---

## 10.5 NATS JetStream 消费者

```yaml
consumers:
  - name: "nats_events"
    enabled: true
    type: "nats_jetstream"
    nats_jetstream:
      url: "nats://nats:4222"
      stream_name: "centrifugo"
      subjects:
        - "centrifugo.>"
      durable_consumer_name: "centrifugo-consumer"
      deliver_policy: "new"
      max_ack_pending: 100
```

---

## 10.6 Redis Stream 消费者

```yaml
consumers:
  - name: "redis_events"
    enabled: true
    type: "redis_stream"
    redis_stream:
      redis:
        address: "redis:6379"
      streams:
        - "centrifugo:events"
      consumer_group: "centrifugo"
      num_workers: 2
```

### 写入 Redis Stream

```bash
# 使用 redis-cli 写入
redis-cli XADD centrifugo:events "*" \
  method publish \
  payload '{"channel":"chat:room_42","data":{"text":"来自 Redis Stream"}}'
```

---

## 10.7 Google Pub/Sub 消费者

```yaml
consumers:
  - name: "gcp_events"
    enabled: true
    type: "google_pub_sub"
    google_pub_sub:
      project_id: "your-gcp-project"
      subscriptions:
        - "centrifugo-events-sub"
      max_outstanding_messages: 100
      method_attribute: "centrifugo-method"
```

---

## 10.8 AWS SQS 消费者

```yaml
consumers:
  - name: "sqs_events"
    enabled: true
    type: "aws_sqs"
    aws_sqs:
      queues:
        - "https://sqs.us-east-1.amazonaws.com/123456789/centrifugo-events"
      region: "us-east-1"
      max_number_of_messages: 10
      poll_wait_time: "20s"
```

---

## 10.9 Azure Service Bus 消费者

```yaml
consumers:
  - name: "azure_events"
    enabled: true
    type: "azure_service_bus"
    azure_service_bus:
      connection_string: "Endpoint=sb://your-namespace.servicebus.windows.net/..."
      queues:
        - "centrifugo-events"
      max_concurrent_calls: 5
```

---

## 10.10 消费者消息格式

所有消费者都支持相同的消息格式：

```json
// publish - 发布消息
{
  "channel": "chat:room_42",
  "data": {"text": "hello"}
}

// broadcast - 广播
{
  "channels": ["chat:room_1", "chat:room_2"],
  "data": {"text": "broadcast message"}
}
```

method（操作类型）通过消息头/属性传递：

| 消费者类型 | method 指定方式 |
|-----------|----------------|
| Kafka | 消息头 `centrifugo-method` |
| NATS | 消息头 `centrifugo-method` |
| Redis Stream | 字段 `method` |
| Google Pub/Sub | 属性 `centrifugo-method` |
| AWS SQS | 属性 `centrifugo-method` |
| Azure Service Bus | 属性 `centrifugo-method` |
| PostgreSQL | 列 `method` |

---

## 10.11 业务架构示例

### 微服务 + Kafka 架构

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│  订单服务 │  │  支付服务 │  │  通知服务 │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │              │              │
     └──────────────┼──────────────┘
                    │
            ┌───────┴───────┐
            │    Kafka      │
            │  events topic │
            └───────┬───────┘
                    │
            ┌───────┴───────┐
            │  Centrifugo   │  ← Kafka 消费者
            │  (消费并推送)  │
            └───────┬───────┘
                    │
        ┌───────────┼───────────┐
        │           │           │
   ┌────┴───┐  ┌───┴────┐  ┌──┴─────┐
   │ Web 端 │  │ App 端  │  │ 小程序  │
   └────────┘  └────────┘  └────────┘
```

### 单体应用 + PostgreSQL Outbox

```
┌─────────────────────────────────────────┐
│              你的后端应用                 │
│                                         │
│  处理业务逻辑                            │
│    │                                    │
│    └── 在同一事务中写入：                 │
│        1. 业务数据表                     │
│        2. centrifugo_outbox 表           │
│                                         │
└─────────────────────┬───────────────────┘
                      │
               ┌──────┴──────┐
               │ PostgreSQL  │
               │ outbox 表   │
               └──────┬──────┘
                      │
               ┌──────┴──────┐
               │ Centrifugo  │ ← PostgreSQL 消费者
               │ (消费并推送) │
               └──────┬──────┘
                      │
                ┌─────┴─────┐
                │  客户端    │
                └───────────┘
```

---

## 10.12 常见坑

### 坑1：消息重复

```
⚠️ 消费者提供的是 at-least-once 语义

可能出现消息重复推送的情况（如消费者重启时）
解决：客户端做好消息去重（基于唯一 ID）
```

### 坑2：PostgreSQL Outbox 表无限增长

```
⚠️ Centrifugo 消费完消息后会自动删除记录
但如果消费者停止工作，outbox 表会持续增长

建议：
1. 监控 outbox 表的大小
2. 设置告警
3. 定期清理过期记录
```

### 坑3：Kafka 消费者组偏移量

```
⚠️ 使用相同的 consumer_group 时，多个 Centrifugo 实例会分摊消费

这通常是期望的行为。但要注意：
1. 每个 Centrifugo 节点都会收到部分消息
2. Redis 引擎确保消息被转发到所有节点
3. 必须使用 Redis 引擎（不能用 Memory）
```

---

## 10.13 本章小结

| 消费者类型 | 最佳场景 | 复杂度 |
|-----------|---------|--------|
| PostgreSQL | 单体应用，事务一致性 | ⭐⭐ |
| Kafka | 微服务架构，高吞吐 | ⭐⭐⭐ |
| NATS JetStream | 已经使用 NATS | ⭐⭐ |
| Redis Stream | 已经使用 Redis | ⭐⭐ |
| Google Pub/Sub | GCP 云原生 | ⭐⭐ |
| AWS SQS | AWS 云原生 | ⭐⭐ |
| Azure Service Bus | Azure 云原生 | ⭐⭐ |

**下一章**：管理界面和监控 → [第11章 管理与监控](11-admin-monitoring.md)
