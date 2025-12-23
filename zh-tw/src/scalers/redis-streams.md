
## TL;DR

Redis Streams 擴縮器根據 Redis 5.0+ 引入的 Streams 資料結構來擴縮應用程式。支援三種觸發模式：Pending Entries List（待處理列表）、Stream Length（串流長度）和 Consumer Group Lag（消費者群組延遲）。Lag 模式需要 Redis 7+ 且是唯一支援縮減到 0 的模式。

---

## 翻譯 (Translation)

Redis 5.0 引入了 [Redis Streams](https://redis.io/topics/streams-intro)，這是一種僅附加（append-only）的日誌資料結構。

其功能之一包括 [`Consumer Groups`](https://redis.io/topics/streams-intro#consumer-groups)（消費者群組），允許一組客戶端協作消費同一串流訊息的不同部分。

有三種方式設定 `redis-streams` 觸發器：
1. 基於 Redis Stream 特定 Consumer Group 的 *Pending Entries List*（待處理項目列表）（參見 [`XPENDING`](https://redis.io/commands/xpending)）
2. 基於 *Stream Length*（串流長度）（參見 [`XLEN`](https://redis.io/commands/xlen)）
3. 基於 *Consumer Group Lag*（消費者群組延遲）（參見 [`XINFO GROUPS`](https://redis.io/commands/xinfo-groups/)）。這是唯一支援縮減到 0 的設定。**重要：此功能需要 Redis 7+**

### 觸發器規格

```yaml
triggers:
- type: redis-streams
  metadata:
    address: localhost:6379          # 必填（如未提供 host 和 port）。格式：host:port
    host: localhost                  # 必填（如未提供 address）
    port: "6379"                     # 必填（如提供 host 則需要）
    usernameFromEnv: REDIS_USERNAME  # 可選（也可使用 authenticationRef）
    passwordFromEnv: REDIS_PASSWORD  # 可選（也可使用 authenticationRef）
    stream: my-stream                # 必填 - Redis Stream 名稱
    consumerGroup: my-consumer-group # 可選 - 與 Redis Stream 關聯的消費者群組名稱
    pendingEntriesCount: "10"        # 可選 - 待處理項目列表中的項目數量
    streamLength: "50"               # 可選 - Redis 串流長度（pendingEntriesCount 的替代方案）
    lagCount: "5"                    # 可選 - 消費者群組中的延遲項目數量
    activationLagCount: "3"          # 如提供 lagCount 則必填 - 觸發擴縮的延遲數量
    enableTLS: "false"               # 可選
    unsafeSsl: "false"               # 可選
    databaseIndex: "0"               # 可選
    addressFromEnv: REDIS_ADDRESS    # 可選。可替代 `address` 參數
```

**參數列表：**

- `address` - Redis 伺服器的主機和埠口，格式為 `host:port`，例如 `my-redis:6379`。
- `host` - Redis 伺服器的主機。如未提供 `address` 則為必填。
- `port` - Redis 伺服器的埠口。需與 `host` 一起使用。
- `usernameFromEnv` - 包含 Redis 使用者名稱的環境變數名稱。（可選）
- `passwordFromEnv` - 包含 Redis 密碼的環境變數名稱。（可選）
- `stream` - Redis Stream 的名稱。
- `consumerGroup` - 與 Redis Stream 關聯的消費者群組名稱。
  > 設定 `consumerGroup` 會使擴縮器基於 `pendingEntriesCount` 運作。缺少 `consumerGroup` 會使擴縮器基於 `streamLength`
- `pendingEntriesCount` - `Pending Entries List` 的閾值。這是擴縮工作負載的平均目標值。（預設：`5`，可選）
- `streamLength` - 串流長度的閾值，擴縮工作負載的替代平均目標值。（預設：`5`，可選）
- `lagCount` - 消費者群組延遲數量的閾值，擴縮工作負載的替代平均目標值。（預設：`5`，可選）
- `activationLagCount` - 開始擴縮的延遲數量閾值。任何低於此值的平均延遲數量都不會觸發擴縮器。（預設：`0`，可選）
- `enableTLS` - 啟用 TLS 連線到 Redis。（值：`true`、`false`，預設：`false`，可選）
- `unsafeSsl` - 跳過憑證檢查，例如使用自簽憑證。（值：`true`、`false`，預設：`false`，可選，需要 `enableTLS: true`）
- `databaseIndex` - 使用的 Redis 資料庫索引。如未指定，預設值為 0。

### 驗證參數

擴縮器支援兩種驗證模式：

**使用者名稱/密碼驗證：**
在 `metadata` 中使用 `usernameFromEnv` 和 `passwordFromEnv` 欄位。

**TLS 驗證：**
- `tls` - 設為 `enable` 啟用 Redis 的 SSL 驗證。（值：`enable`、`disable`，預設：`disable`，可選）
- `ca` - TLS 客戶端驗證的憑證授權單位檔案。（可選）
- `cert` - 客戶端驗證的憑證。（可選）
- `key` - 客戶端驗證的金鑰。（可選）
- `keyPassword` - 如設定，用於解密提供的 `key`。（可選）

---

## 說明 (Explanation)

### 三種觸發模式比較

| 模式 | 設定方式 | 適用場景 | 可縮減到 0 |
|------|----------|----------|------------|
| Pending Entries | 設定 `consumerGroup` | 監控待處理訊息 | 否 |
| Stream Length | 不設定 `consumerGroup` | 監控串流總長度 | 否 |
| Consumer Group Lag | 設定 `lagCount` | 監控消費者延遲 | 是（需 Redis 7+）|

### Redis 指令對應

| 模式 | Redis 指令 |
|------|------------|
| Pending Entries | `XPENDING` |
| Stream Length | `XLEN` |
| Consumer Group Lag | `XINFO GROUPS` |

---

## 實作範例 (Practical Example)

```yaml
# 使用 Pending Entries（待處理項目）
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: redis-streams-pending
spec:
  scaleTargetRef:
    name: redis-streams-consumer
  pollingInterval: 15
  cooldownPeriod: 200
  maxReplicaCount: 25
  minReplicaCount: 1
  triggers:
    - type: redis-streams
      metadata:
        addressFromEnv: REDIS_HOST
        usernameFromEnv: REDIS_USERNAME
        passwordFromEnv: REDIS_PASSWORD
        stream: my-stream
        consumerGroup: consumer-group-1
        pendingEntriesCount: "10"
```

```yaml
# 使用 Stream Length（串流長度）- 不設定 consumerGroup
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: redis-streams-length
spec:
  scaleTargetRef:
    name: redis-streams-consumer
  pollingInterval: 20
  cooldownPeriod: 200
  maxReplicaCount: 10
  minReplicaCount: 1
  triggers:
    - type: redis-streams
      metadata:
        addressFromEnv: REDIS_HOST
        usernameFromEnv: REDIS_USERNAME
        passwordFromEnv: REDIS_PASSWORD
        stream: my-stream
        streamLength: "50"       # 不設定 consumerGroup
```

```yaml
# 使用 Consumer Group Lag（支援縮減到 0，需 Redis 7+）
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: redis-streams-lag
spec:
  scaleTargetRef:
    name: redis-streams-consumer
  pollingInterval: 15
  cooldownPeriod: 200
  maxReplicaCount: 25
  minReplicaCount: 0              # 可縮減到 0
  triggers:
    - type: redis-streams
      metadata:
        addressFromEnv: REDIS_HOST
        usernameFromEnv: REDIS_USERNAME
        passwordFromEnv: REDIS_PASSWORD
        stream: my-stream
        consumerGroup: consumer-group-1
        lagCount: "10"
        activationLagCount: "3"   # lagCount 時必填
```

```yaml
# 完整範例：使用 TriggerAuthentication
apiVersion: v1
kind: Secret
metadata:
  name: redis-streams-auth
type: Opaque
data:
  redis_username: <base64-encoded-username>
  redis_password: <base64-encoded-password>
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: redis-stream-triggerauth
spec:
  secretTargetRef:
    - parameter: username
      name: redis-streams-auth
      key: redis_username
    - parameter: password
      name: redis-streams-auth
      key: redis_password
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: redis-streams-scaledobject
spec:
  scaleTargetRef:
    name: redis-streams-consumer
  triggers:
    - type: redis-streams
      metadata:
        address: redis:6379
        stream: my-stream
        consumerGroup: consumer-group-1
        pendingEntriesCount: "10"
      authenticationRef:
        name: redis-stream-triggerauth
```

```bash
# 查看 Redis Stream 長度
redis-cli XLEN my-stream

# 查看 Consumer Group 資訊
redis-cli XINFO GROUPS my-stream

# 查看待處理項目
redis-cli XPENDING my-stream consumer-group-1

# 新增訊息到 Stream
redis-cli XADD my-stream '*' field1 value1 field2 value2

# 查看 ScaledObject 狀態
kubectl describe scaledobject redis-streams-scaledobject
```

---

## 常見錯誤與提示 (Common Mistakes & Tips)

| 常見錯誤 | 正確做法 |
|----------|----------|
| 使用 lagCount 但 Redis 版本 < 7 | 升級到 Redis 7+ 或使用其他模式 |
| 使用 lagCount 但未設定 activationLagCount | 必須同時設定 activationLagCount |
| 想縮減到 0 但使用 pendingEntriesCount | 改用 lagCount 模式 |
| 同時設定多種模式 | 選擇一種模式使用 |

### 提示

- 只有 `lagCount` 模式支援縮減到 0，且需要 Redis 7+
- 設定 `consumerGroup` 會使用 `pendingEntriesCount` 模式
- 不設定 `consumerGroup` 會使用 `streamLength` 模式
- 使用 TriggerAuthentication 管理敏感的連線資訊

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `address` | Redis 伺服器位址 | 無（必填或使用 host + port）|
| `stream` | Stream 名稱 | 無（必填）|
| `consumerGroup` | 消費者群組名稱 | 無（可選）|
| `pendingEntriesCount` | 待處理項目閾值 | 5 |
| `streamLength` | 串流長度閾值 | 5 |
| `lagCount` | 延遲數量閾值 | 5 |
| `activationLagCount` | 啟動延遲閾值 | 0 |
| `enableTLS` | 啟用 TLS | false |
| `databaseIndex` | 資料庫索引 | 0 |

| Redis 指令 | 說明 |
|------------|------|
| `XLEN <stream>` | 查看串流長度 |
| `XINFO GROUPS <stream>` | 查看消費者群組資訊 |
| `XPENDING <stream> <group>` | 查看待處理項目 |
| `XADD <stream> * <field> <value>` | 新增訊息 |

| 指令 | 說明 |
|------|------|
| `kubectl get scaledobject` | 列出 ScaledObject |
| `kubectl describe scaledobject <name>` | 查看詳細狀態 |
