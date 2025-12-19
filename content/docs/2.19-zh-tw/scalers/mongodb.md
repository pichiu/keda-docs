+++
title = "MongoDB"
availability = "v2.1+"
maintainer = "Community"
category = "Data & Storage"
description = "根據 MongoDB 查詢結果擴縮應用程式。"
go_file = "mongo_scaler"
+++

## TL;DR

MongoDB 擴縮器根據 MongoDB 查詢結果來擴縮應用程式。可用於監控資料庫中符合特定條件的文件數量，實現基於資料狀態的自動擴縮。支援連接字串和個別參數兩種連線方式。

---

## 翻譯 (Translation)

### 觸發器規格

此規格描述根據 MongoDB 查詢結果進行擴縮的 `mongodb` 觸發器。

```yaml
# 使用連接字串
triggers:
  - type: mongodb
    metadata:
      # 包含有效 MongoDB 連接字串的環境變數名稱
      connectionStringFromEnv: MongoDB_CONNECTION_STRING
      # 必填：資料庫名稱
      dbName: "test"
      # 必填：集合名稱
      collection: "test_collection"
      # 必填：查詢表達式，用於過濾資料
      query: '{"region":"eu-1","state":"running","plan":"planA"}'
      # 必填：根據查詢結果數量來擴縮 TargetRef
      queryValue: "1"
      # 可選：根據查詢結果數量啟動擴縮器
      activationQueryValue: "1"
```

```yaml
# 使用個別參數
triggers:
  - type: mongodb
    metadata:
      # MongoDB 伺服器的 scheme。如使用 MongoDB Atlas，可設為 "mongodb+srv"
      scheme: "mongodb"
      # MongoDB 伺服器的主機名稱
      host: mongodb-svc.default.svc.cluster.local
      # MongoDB 伺服器的埠口
      port: "27017"
      # 連接 MongoDB 伺服器的使用者名稱
      username: test_user
      # 包含密碼的環境變數名稱
      passwordFromEnv: MongoDB_Password
      # 必填：資料庫名稱
      dbName: "test"
      # 必填：集合名稱
      collection: "test_collection"
      # 必填：查詢表達式
      query: '{"region":"eu-1","state":"running","plan":"planA"}'
      # 必填：查詢值閾值
      queryValue: "1"
```

**參數列表：**

`mongodb` 觸發器始終需要以下資訊：

- `dbName` - 資料庫名稱。
- `collection` - 集合名稱。
- `query` - 應返回單一數值的 MongoDB 查詢。
- `queryValue` - 定義何時應進行擴縮的閾值。（可以是浮點數）
- `activationQueryValue` - 啟動擴縮器的目標值。（預設：`0`，可選，可以是浮點數）

連線到 MongoDB 伺服器，可提供：

- `connectionStringFromEnv` - 包含有效 MongoDB 連接字串的環境變數名稱。

或提供更詳細的連線參數（會在執行時產生連接字串）：

- `scheme` - MongoDB 伺服器的 scheme，如使用 MongoDB Atlas，可設為 `mongodb+srv`。（預設：`mongodb`，可選）
- `host` - MongoDB 伺服器的主機名稱。
- `port` - MongoDB 伺服器的埠口號碼。
- `username` - 連接 MongoDB 資料庫的使用者名稱。
- `passwordFromEnv` - 包含連接 MongoDB 伺服器密碼的環境變數名稱。

### 連接字串格式

```
mongodb[+srv]://<username>:<password>@mongodb-svc.<namespace>.svc.cluster.local:27017/<database_name>
```

### 驗證參數

作為環境變數的替代方案，您可以使用 `TriggerAuthentication` 或 `ClusterTriggerAuthentication` 透過連接字串或密碼驗證連接 MongoDB 伺服器。

**連接字串驗證：**
- `connectionString` - MongoDB 伺服器的連接字串。

**密碼驗證：**
- `scheme` - MongoDB 伺服器的 scheme。（預設：`mongodb`，可選）
- `host` - MongoDB 伺服器的主機名稱。
- `port` - MongoDB 伺服器的埠口號碼。
- `username` - 連接 MongoDB 資料庫的使用者名稱。
- `password` - 連接 MongoDB 伺服器的密碼。
- `dbName` - 資料庫名稱。

---

## 說明 (Explanation)

### 查詢運作方式

| 設定 | 說明 |
|------|------|
| `query` | MongoDB 查詢過濾條件（JSON 格式）|
| `queryValue` | 當查詢結果 >= queryValue 時觸發擴縮 |

### 擴縮計算

假設設定 `queryValue: 5`：
- 查詢結果 = 0 → 0 個 Pod
- 查詢結果 = 15 → 3 個 Pod (15 / 5)
- 查詢結果 = 50 → 10 個 Pod (50 / 5)

---

## 實作範例 (Practical Example)

```yaml
# 使用連接字串的 ScaledJob 範例
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-secret
type: Opaque
data:
  # base64 編碼的連接字串：mongodb://test_user:test_password@mongodb-svc:27017/test
  connect: bW9uZ29kYjovL3Rlc3RfdXNlcjp0ZXN0X3Bhc3N3b3JkQG1vbmdvZGItc3ZjOjI3MDE3L3Rlc3Q=
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: mongodb-trigger-auth
spec:
  secretTargetRef:
    - parameter: connectionString
      name: mongodb-secret
      key: connect
---
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: mongodb-job
spec:
  jobTargetRef:
    template:
      spec:
        containers:
          - name: mongodb-processor
            image: my-processor:latest
            env:
              - name: DB_NAME
                value: "test"
        restartPolicy: Never
    backoffLimit: 1
  pollingInterval: 30
  maxReplicaCount: 30
  successfulJobsHistoryLimit: 0
  failedJobsHistoryLimit: 10
  triggers:
    - type: mongodb
      metadata:
        dbName: "test"
        collection: "task_collection"
        query: '{"status":"pending"}'
        queryValue: "1"
      authenticationRef:
        name: mongodb-trigger-auth
```

```yaml
# ScaledObject 範例
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: mongodb-scaledobject
spec:
  scaleTargetRef:
    name: my-worker-deployment
  pollingInterval: 15
  cooldownPeriod: 120
  minReplicaCount: 0
  maxReplicaCount: 20
  triggers:
    - type: mongodb
      metadata:
        connectionStringFromEnv: MONGODB_URI
        dbName: "production"
        collection: "jobs"
        query: '{"state":"queued","priority":{"$gte":5}}'
        queryValue: "10"
        activationQueryValue: "5"
```

```yaml
# 使用個別參數
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: mongodb-individual-params
spec:
  scaleTargetRef:
    name: my-worker-deployment
  triggers:
    - type: mongodb
      metadata:
        scheme: "mongodb"
        host: mongodb-svc.database.svc.cluster.local
        port: "27017"
        username: keda_user
        passwordFromEnv: MONGODB_PASSWORD
        dbName: "myapp"
        collection: "orders"
        query: '{"status":"processing"}'
        queryValue: "5"
```

```bash
# 測試 MongoDB 查詢
mongosh "mongodb://localhost:27017/test" --eval \
  'db.task_collection.countDocuments({"status":"pending"})'

# 插入測試文件
mongosh "mongodb://localhost:27017/test" --eval \
  'db.task_collection.insertOne({"status":"pending","region":"eu-1"})'

# 查看 ScaledObject 狀態
kubectl describe scaledobject mongodb-scaledobject

# 查看 ScaledJob 狀態
kubectl describe scaledjob mongodb-job
```

---

## 常見錯誤與提示 (Common Mistakes & Tips)

| 常見錯誤 | 正確做法 |
|----------|----------|
| 查詢語法錯誤 | 確保查詢是有效的 JSON 格式 |
| 連接字串格式不正確 | 使用 `mongodb://` 或 `mongodb+srv://` 前綴 |
| 無法連接到 MongoDB | 確認網路連通性和防火牆設定 |
| 使用者權限不足 | 確保使用者有讀取集合的權限 |

### 提示

- 使用 MongoDB Atlas 時，設定 `scheme: "mongodb+srv"`
- 查詢應該是 MongoDB 的 find 查詢語法
- 對於 Kubernetes 內部的 MongoDB，使用完整的 Service FQDN
- 使用 `activationQueryValue` 可以設定最小觸發閾值

### 查詢範例

```javascript
// 計算待處理任務
{"status": "pending"}

// 多條件查詢
{"region": "eu-1", "state": "running", "plan": "planA"}

// 使用比較運算子
{"priority": {"$gte": 5}, "status": "queued"}

// 使用 $or 運算子
{"$or": [{"status": "pending"}, {"status": "retry"}]}
```

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `connectionStringFromEnv` | 連接字串環境變數 | 無（與其他參數二選一）|
| `scheme` | 連線 scheme | mongodb |
| `host` | 資料庫主機 | 無 |
| `port` | 資料庫埠口 | 27017 |
| `dbName` | 資料庫名稱 | 無（必填）|
| `collection` | 集合名稱 | 無（必填）|
| `query` | 查詢表達式 | 無（必填）|
| `queryValue` | 目標值 | 無（必填）|
| `activationQueryValue` | 啟動閾值 | 0 |

| 指令 | 說明 |
|------|------|
| `kubectl get scaledobject` | 列出 ScaledObject |
| `kubectl get scaledjob` | 列出 ScaledJob |
| `mongosh -eval 'db.collection.countDocuments(...)'` | 測試查詢 |
