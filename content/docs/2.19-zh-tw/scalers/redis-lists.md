+++
title = "Redis Lists"
availability = "v1.0+"
maintainer = "Community"
category = "Data & Storage"
description = "根據 Redis 列表擴縮應用程式。"
go_file = "redis_scaler"
+++

## TL;DR

Redis Lists 擴縮器根據 Redis 列表（List）的長度來擴縮應用程式。當列表中有大量待處理項目時自動擴展，處理完成後縮減。支援 TLS 加密連線和多種驗證方式。

---

## 翻譯 (Translation)

### 觸發器規格

此規格描述根據 Redis 列表長度進行擴縮的 `redis` 觸發器。

```yaml
triggers:
- type: redis
  metadata:
    address: localhost:6379       # 格式必須是 host:port
    usernameFromEnv: REDIS_USERNAME  # 可選
    passwordFromEnv: REDIS_PASSWORD
    listName: mylist              # 必填
    listLength: "5"               # 必填
    activationListLength: "5"     # 可選
    enableTLS: "false"            # 可選
    unsafeSsl: "false"            # 可選
    databaseIndex: "0"            # 可選
    # 也可使用環境變數：
    addressFromEnv: REDIS_HOST    # 可選。可替代 `address` 參數
```

**參數列表：**

- `address` - Redis 伺服器的主機和埠口。
- `host` - Redis 伺服器的主機。`address` 的替代方案，需搭配 `port` 使用。
- `port` - Redis 伺服器的埠口。`address` 的替代方案，需搭配 `host` 使用。
- `usernameFromEnv` - 從環境變數讀取驗證使用者名稱。
- `passwordFromEnv` - 從環境變數讀取驗證密碼。
- `listName` - 要監控的 Redis 列表名稱。
- `listLength` - 觸發擴縮動作的平均目標值。
- `activationListLength` - 啟動擴縮器的目標值。（預設：`0`，可選）
- `enableTLS` - 啟用 TLS 連線到 Redis 佇列。（值：`true`、`false`，預設：`false`，可選）
- `unsafeSsl` - 跳過憑證檢查，例如使用自簽憑證。（值：`true`、`false`，預設：`false`，可選，需要 `enableTLS: true`）
- `databaseIndex` - 使用的 Redis 資料庫索引。（預設：0）

**環境變數參數：**

- `addressFromEnv` - 從環境變數讀取 Redis 伺服器位址。
- `hostFromEnv` - 從環境變數讀取 Redis 主機。
- `portFromEnv` - 從環境變數讀取 Redis 埠口。

### 驗證參數

可使用使用者名稱（可選）和密碼進行驗證。

**連線驗證：**

- `address` - Redis 伺服器的主機名稱和埠口（host:port 格式）。
- `host` - Redis 伺服器的主機名稱。如指定，也應指定 `port`。
- `port` - Redis 伺服器的埠口。如指定，也應指定 `host`。

**TLS 驗證：**

- `tls` - 設為 `enable` 啟用 Redis 的 SSL 驗證。（值：`enable`、`disable`，預設：`disable`，可選）
- `ca` - TLS 客戶端驗證的憑證授權單位檔案。（可選）
- `cert` - 客戶端驗證的憑證。（可選）
- `key` - 客戶端驗證的金鑰。（可選）
- `keyPassword` - 如設定，用於解密提供的 `key`。（可選）

**驗證：**

- `username` - Redis 使用者名稱。
- `password` - Redis 密碼。

---

## 說明 (Explanation)

### 擴縮計算

| 設定 | 說明 |
|------|------|
| `listLength: 10` | 每個 Pod 處理 10 個列表項目 |
| 列表長度 50 | 會擴展到 5 個 Pod (50 / 10) |

### TLS 設定選項

| 方式 | 說明 |
|------|------|
| ScaledObject 中 `enableTLS` | 簡單啟用 TLS |
| TriggerAuthentication 中 `tls` | 完整 TLS 憑證設定 |

> 注意：這兩種方式不能同時使用。

---

## 實作範例 (Practical Example)

```yaml
# 建立 Secret
apiVersion: v1
kind: Secret
metadata:
  name: redis-secret
  namespace: my-project
type: Opaque
data:
  redis_username: YWRtaW4=     # base64: admin
  redis_password: cGFzc3dvcmQ=  # base64: password
---
# 建立 TriggerAuthentication
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: redis-trigger-auth
  namespace: my-project
spec:
  secretTargetRef:
    - parameter: username
      name: redis-secret
      key: redis_username
    - parameter: password
      name: redis-secret
      key: redis_password
---
# 建立 ScaledObject
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: redis-scaledobject
  namespace: my-project
spec:
  scaleTargetRef:
    name: my-worker-deployment
  pollingInterval: 15
  cooldownPeriod: 300
  minReplicaCount: 0
  maxReplicaCount: 20
  triggers:
    - type: redis
      metadata:
        address: redis:6379
        listName: task-queue
        listLength: "10"
      authenticationRef:
        name: redis-trigger-auth
```

```yaml
# 使用環境變數（不使用 TriggerAuthentication）
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: redis-scaledobject
spec:
  scaleTargetRef:
    name: my-worker-deployment
  triggers:
    - type: redis
      metadata:
        addressFromEnv: REDIS_HOST
        usernameFromEnv: REDIS_USERNAME
        passwordFromEnv: REDIS_PASSWORD
        listName: job-queue
        listLength: "5"
```

```yaml
# 啟用 TLS
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: redis-tls-scaledobject
spec:
  scaleTargetRef:
    name: my-worker-deployment
  triggers:
    - type: redis
      metadata:
        address: redis:6379
        listName: secure-queue
        listLength: "10"
        enableTLS: "true"
        unsafeSsl: "true"    # 自簽憑證時使用
```

```bash
# 查看 Redis 列表長度
redis-cli -h redis LLEN task-queue

# 新增測試項目到列表
redis-cli -h redis LPUSH task-queue "task-1" "task-2" "task-3"

# 查看 ScaledObject 狀態
kubectl describe scaledobject redis-scaledobject

# 查看 HPA 狀態
kubectl get hpa
```

---

## 常見錯誤與提示 (Common Mistakes & Tips)

| 常見錯誤 | 正確做法 |
|----------|----------|
| address 格式錯誤 | 使用 `host:port` 格式，如 `redis:6379` |
| 未設定密碼但 Redis 需要驗證 | 使用 `passwordFromEnv` 或 TriggerAuthentication |
| TLS 設定衝突 | 不要同時使用 ScaledObject 的 enableTLS 和 TriggerAuthentication 的 tls |
| 連線到錯誤的資料庫 | 使用 `databaseIndex` 指定正確的資料庫 |

### 提示

- Redis Cluster 模式使用 `redis-cluster` 擴縮器而非 `redis`
- 對於 Redis Sentinel，使用 `redis-sentinel` 擴縮器
- 使用 `activationListLength` 可以避免在列表只有少量項目時就啟動擴縮
- 生產環境建議使用 TriggerAuthentication 管理憑證

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `address` | Redis 伺服器位址 | 無（必填或使用 addressFromEnv）|
| `listName` | 列表名稱 | 無（必填）|
| `listLength` | 列表長度閾值 | 無（必填）|
| `activationListLength` | 啟動閾值 | 0 |
| `enableTLS` | 啟用 TLS | false |
| `unsafeSsl` | 跳過憑證檢查 | false |
| `databaseIndex` | 資料庫索引 | 0 |

| Redis 指令 | 說明 |
|------------|------|
| `LLEN <list>` | 查看列表長度 |
| `LPUSH <list> <value>` | 從左邊新增項目 |
| `RPOP <list>` | 從右邊取出項目 |
| `LRANGE <list> 0 -1` | 查看列表所有項目 |

| 指令 | 說明 |
|------|------|
| `kubectl get scaledobject` | 列出 ScaledObject |
| `kubectl describe scaledobject <name>` | 查看詳細狀態 |
