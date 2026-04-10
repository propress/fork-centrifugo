# 第7章 频道与命名空间配置

> **阅读目标**：掌握频道命名空间的配置方法，能为不同业务场景设计合适的频道结构

---

## 7.1 为什么需要命名空间？

一个实际项目中会有很多不同类型的频道，它们有不同的需求：

```
聊天频道：需要历史消息、在线状态、断线恢复
通知频道：需要历史消息、不需要在线状态
行情频道：不需要历史、不需要在线状态、要 Delta 压缩
游戏频道：需要在线状态、不需要历史
```

命名空间让你对每种类型的频道设置不同的行为。

---

## 7.2 命名空间配置详解

### 配置结构

```yaml
channel:
  # 没有命名空间的频道（如 "general"）使用这里的配置
  without_namespace:
    presence: false
    join_leave: false
    history_size: 0
    history_ttl: "0s"
    force_recovery: false

  # 命名空间分隔符（默认是 ":"）
  namespace_boundary: ":"

  # 私有频道前缀（默认是 "$"）
  private_prefix: "$"

  # 频道名最大长度
  max_length: 255

  # 命名空间列表
  namespaces:
    - name: "chat"
      presence: true
      join_leave: true
      history_size: 200
      history_ttl: "24h"
      force_recovery: true

    - name: "notify"
      history_size: 50
      history_ttl: "7d"

    - name: "live"
      # 纯实时推送，不需要任何额外功能
      presence: false
      history_size: 0
```

### 频道名 → 命名空间匹配规则

```
频道名              命名空间        匹配规则
────────           ────────       ──────────
"chat:room_42"     chat           以 "chat:" 开头
"chat:room_99"     chat           以 "chat:" 开头
"notify:user_1"    notify         以 "notify:" 开头
"live:stream_1"    live           以 "live:" 开头
"general"          (无)           没有冒号，使用 without_namespace
"my_channel"       (无)           没有冒号，使用 without_namespace
```

---

## 7.3 所有频道选项

以下是每个命名空间可以配置的所有选项：

### 消息相关

| 选项 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `history_size` | int | 0 | 保留的历史消息数量（0=不保留） |
| `history_ttl` | duration | "0s" | 历史消息过期时间 |
| `force_recovery` | bool | false | 强制开启断线恢复 |
| `allow_publish_for_subscriber` | bool | false | 允许订阅者发布消息 |
| `allow_publish_for_client` | bool | false | 允许已连接客户端发布消息 |

### 在线状态

| 选项 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `presence` | bool | false | 开启在线状态跟踪 |
| `join_leave` | bool | false | 开启上下线通知 |
| `force_presence` | bool | false | 强制所有订阅者的在线状态 |

### 权限控制

| 选项 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `allow_subscribe_for_anonymous` | bool | false | 允许匿名用户订阅 |
| `allow_subscribe_for_client` | bool | false | 允许客户端订阅（无需 token） |
| `allow_history_for_subscriber` | bool | false | 允许订阅者获取历史 |
| `allow_history_for_client` | bool | false | 允许客户端获取历史 |
| `allow_presence_for_subscriber` | bool | false | 允许订阅者获取在线状态 |
| `allow_presence_for_client` | bool | false | 允许客户端获取在线状态 |

### 高级选项

| 选项 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `delta_publish` | bool | false | 开启 Delta 压缩 |
| `force_push_join_leave` | bool | false | 强制推送上下线通知 |
| `allow_user_limited_channels` | bool | false | 允许用户限定频道 |

---

## 7.4 业务场景配置示例

### 场景1：完整聊天系统

```yaml
channel:
  namespaces:
    # 群聊
    - name: "group"
      presence: true
      join_leave: true
      history_size: 500
      history_ttl: "72h"
      force_recovery: true
      allow_history_for_subscriber: true
      allow_presence_for_subscriber: true

    # 私聊
    - name: "dm"
      history_size: 200
      history_ttl: "168h"    # 7 天
      force_recovery: true
      allow_history_for_subscriber: true
      # 不需要 presence（只有两个人）

    # 客服聊天
    - name: "support"
      presence: true
      history_size: 1000
      history_ttl: "720h"   # 30 天
      force_recovery: true
      allow_history_for_subscriber: true
      allow_presence_for_subscriber: true
```

### 场景2：电商平台

```yaml
channel:
  namespaces:
    # 订单状态更新（推送给个人）
    - name: "order"
      history_size: 20
      history_ttl: "24h"
      force_recovery: true

    # 商品价格变动（公开数据）
    - name: "price"
      allow_subscribe_for_client: true
      delta_publish: true
      # 不需要历史

    # 秒杀活动倒计时
    - name: "flash"
      allow_subscribe_for_anonymous: true
      allow_subscribe_for_client: true
      # 纯推送

    # 客服在线沟通
    - name: "cs"
      presence: true
      history_size: 100
      history_ttl: "48h"
      force_recovery: true
```

### 场景3：实时监控系统

```yaml
channel:
  namespaces:
    # 系统指标（高频更新）
    - name: "metrics"
      delta_publish: true
      allow_subscribe_for_client: true
      # 不需要历史和 presence

    # 告警通知
    - name: "alerts"
      history_size: 100
      history_ttl: "24h"
      force_recovery: true

    # 日志流
    - name: "logs"
      # 纯推送，不保留历史
      allow_subscribe_for_client: true
```

### 场景4：多人在线协作

```yaml
channel:
  namespaces:
    # 文档协同编辑
    - name: "doc"
      presence: true
      join_leave: true
      history_size: 1000
      history_ttl: "1h"
      force_recovery: true
      allow_presence_for_subscriber: true
      allow_history_for_subscriber: true

    # 画布/白板
    - name: "canvas"
      presence: true
      join_leave: true
      delta_publish: true
      allow_presence_for_subscriber: true

    # 评论
    - name: "comments"
      history_size: 200
      history_ttl: "72h"
      force_recovery: true
```

---

## 7.5 频道命名设计建议

### 命名规范

```
推荐的命名模式：

{namespace}:{resource_type}_{resource_id}

示例：
  chat:room_42           ← 聊天室 42
  chat:dm_u1001_u1002    ← 用户 1001 和 1002 的私聊
  order:user_1001        ← 用户 1001 的订单更新
  notify:user_1001       ← 用户 1001 的通知
  game:match_789         ← 游戏对局 789
  metrics:cpu_node1      ← 节点 1 的 CPU 指标
```

### 设计原则

```
1. 频道粒度要合适
   ✅ "chat:room_42"        ← 每个聊天室一个频道
   ❌ "chat:all"            ← 所有聊天消息放一个频道（太粗）
   ❌ "chat:room_42:msg_1"  ← 每条消息一个频道（太细）

2. 命名空间按业务功能划分
   ✅ chat / notify / order / metrics
   ❌ channel1 / channel2 / channel3

3. 频道名中包含足够的标识信息
   ✅ "chat:room_42"        ← 知道是哪个聊天室
   ❌ "chat:42"             ← 42 是什么？

4. 避免频道名中包含敏感信息
   ✅ "chat:dm_u1001_u1002" ← 用 ID
   ❌ "chat:dm_张三_李四"     ← 用真实姓名不安全
```

---

## 7.6 用户频道高级配置

### 自动个人频道

```yaml
client:
  subscribe_to_user_personal_channel:
    enabled: true
    # 个人频道的命名空间（如果设置，频道会变成 "personal:#user_1001"）
    personal_channel_namespace: "personal"
```

配置后：
```
用户 "user_1001" 连接时自动订阅频道：

没有 personal_channel_namespace:
  → "#user_1001"

有 personal_channel_namespace: "personal":
  → "personal:#user_1001"
```

### 用户限定频道

```yaml
channel:
  namespaces:
    - name: "personal"
      allow_user_limited_channels: true
```

用户限定频道的格式是 `频道名#user_id`，只有对应用户才能订阅。

---

## 7.7 频道权限矩阵

```
场景                    需要的权限配置
──────                 ──────────────

公开频道（任何人可订阅）
  → allow_subscribe_for_client: true
  → allow_subscribe_for_anonymous: true

登录用户可订阅
  → allow_subscribe_for_client: true

需要订阅 Token
  → 使用 $ 前缀：$chat:vip_room
  → 配置 subscription_token

需要代理授权
  → channel.proxy.subscribe.enabled: true

订阅者可以发布消息
  → allow_publish_for_subscriber: true

订阅者可以查看历史
  → allow_history_for_subscriber: true
  → 同时需要 history_size > 0

订阅者可以查看在线状态
  → allow_presence_for_subscriber: true
  → 同时需要 presence: true
```

---

## 7.8 配置热更新

修改配置文件后，**不需要重启** Centrifugo。向进程发送 SIGHUP 信号即可重新加载频道配置：

```bash
# 方法1：发送 SIGHUP 信号
kill -HUP $(pgrep centrifugo)

# 方法2：Docker 中
docker kill --signal=HUP centrifugo

# 方法3：Docker Compose 中
docker compose kill --signal=HUP centrifugo
```

> **⚠️ 注意**：不是所有配置都支持热更新。频道命名空间选项和 Token 密钥支持，但服务器端口、引擎类型等不支持。

---

## 7.9 常见坑

### 坑1：命名空间不存在

```
❌ 错误：客户端订阅 "chat:room_1" 时被拒绝

原因：配置中没有定义 "chat" 命名空间
      Centrifugo 不会自动创建命名空间

解决：在配置中添加命名空间
```

### 坑2：history_size 设了但没设 history_ttl

```
❌ 错误：历史消息不工作

原因：必须同时设置 history_size 和 history_ttl

✅ 正确：
  history_size: 100
  history_ttl: "24h"
```

### 坑3：在线状态在大量用户时性能差

```
⚠️ 如果一个频道有 10000+ 在线用户：
  - 不要开启 join_leave（会产生大量消息）
  - 用 presence_stats 代替 presence 来查询人数
  - 考虑在你的后端维护在线状态
```

### 坑4：without_namespace 的 history 默认是 0

```
❌ 如果你用了不带命名空间的频道（如 "general"）
   历史消息、在线状态等默认全部关闭

✅ 要么给所有频道加命名空间
   要么在 without_namespace 中设置需要的选项
```

---

## 7.10 本章小结

| 概念 | 要点 |
|------|------|
| 命名空间 | 用 `:` 分隔，不同命名空间可有不同配置 |
| 默认频道 | 没有命名空间的频道用 `without_namespace` 配置 |
| 历史消息 | 需要同时设置 `history_size` 和 `history_ttl` |
| 在线状态 | 需要显式开启 `presence: true` |
| 权限 | 通过 `allow_*` 系列选项控制 |
| 热更新 | 发送 SIGHUP 信号重新加载配置 |

**下一章**：学习如何水平扩展 → [第8章 水平扩展](08-scaling.md)
