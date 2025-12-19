+++
title = "MySQL"
availability = "v1.2+"
maintainer = "Community"
category = "Data & Storage"
description = "根據 MySQL 查詢結果擴縮應用程式。"
go_file = "mysql_scaler"
+++

## TL;DR

MySQL 擴縮器根據 MySQL 查詢結果來擴縮應用程式。可用於監控資料庫中的待處理任務數量、佇列深度等指標，實現基於資料庫狀態的自動擴縮。

---

## 翻譯 (Translation)

### 觸發器規格

此規格描述根據 MySQL 查詢結果進行擴縮的 `mysql` 觸發器。

觸發器始終需要以下資訊：

- `query` - 應返回單一數值的 MySQL 查詢。
- `queryValue` - 作為 HPA 中 `targetValue` 或 `targetAverageValue` 的閾值。（可以是浮點數）
- `activationQueryValue` - 啟動擴縮器的目標值。（預設：`0`，可選，可以是浮點數）

> 注意：查詢必須返回單一整數值。如果查詢可能返回 `null`，可使用 `COALESCE` 函數設定預設值。例如：`SELECT COALESCE(column_name, 0) FROM table_name;`

提供連線資訊的方式：

**連接字串：**
- `connectionStringFromEnv` - 指向包含有效連接字串的環境變數。

**個別參數：**
- `host` - MySQL 伺服器主機。
- `port` - MySQL 伺服器埠口。
- `dbName` - 資料庫名稱。
- `username` - 連接 MySQL 資料庫的使用者名稱。
- `passwordFromEnv` - 使用者密碼，應為空白（無密碼）或指向包含密碼的環境變數。

**環境變數參數：**
- `hostFromEnv` - 從環境變數讀取 MySQL 伺服器主機。
- `portFromEnv` - 從環境變數讀取 MySQL 伺服器埠口。
- `usernameFromEnv` - 從環境變數讀取使用者名稱。

### 驗證參數

可使用連接字串或密碼驗證。

**連接字串驗證：**
- `connectionString` - MySQL 資料庫的連接字串。

**密碼驗證：**
- `host` - MySQL 伺服器主機。
- `port` - MySQL 伺服器埠口。
- `dbName` - 資料庫名稱。
- `username` - 連接 MySQL 資料庫的使用者名稱。
- `password` - 設定使用者登入 MySQL 資料庫的密碼。

---

## 說明 (Explanation)

### 連接字串格式

```
user:password@tcp(host:port)/database
```

範例：
```
admin:password123@tcp(mysql:3306)/mydb
```

### 擴縮計算

| 設定 | 說明 |
|------|------|
| `queryValue: 10` | 當查詢結果達到 10 時需要 1 個 Pod |
| 查詢結果 50 | 需要 5 個 Pod (50 / 10) |

---

## 實作範例 (Practical Example)

```yaml
# 建立 Secret
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secrets
  namespace: my-project
type: Opaque
data:
  # base64 編碼的連接字串：user:password@tcp(mysql:3306)/stats_db
  mysql_conn_str: dXNlcjpwYXNzd29yZEB0Y3AobXlzcWw6MzMwNikvc3RhdHNfZGI=
---
# 建立 TriggerAuthentication
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: mysql-trigger-auth
  namespace: my-project
spec:
  secretTargetRef:
    - parameter: connectionString
      name: mysql-secrets
      key: mysql_conn_str
---
# 建立 ScaledObject
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: mysql-scaledobject
  namespace: my-project
spec:
  scaleTargetRef:
    name: worker-deployment
  pollingInterval: 15
  cooldownPeriod: 60
  minReplicaCount: 0
  maxReplicaCount: 20
  triggers:
    - type: mysql
      metadata:
        queryValue: "4"
        activationQueryValue: "2"
        query: "SELECT CEIL(COUNT(*) / 6) FROM task_instance WHERE state='running' OR state='queued'"
      authenticationRef:
        name: mysql-trigger-auth
```

```yaml
# 使用個別參數
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: mysql-scaledobject
spec:
  scaleTargetRef:
    name: worker-deployment
  triggers:
    - type: mysql
      metadata:
        host: mysql.default.svc.cluster.local
        port: "3306"
        username: keda_user
        passwordFromEnv: MYSQL_PASSWORD
        dbName: app_db
        query: "SELECT COUNT(*) FROM pending_jobs"
        queryValue: "10"
```

```yaml
# 使用環境變數連接字串
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: mysql-scaledobject
spec:
  scaleTargetRef:
    name: worker-deployment
  triggers:
    - type: mysql
      metadata:
        connectionStringFromEnv: DATABASE_URL
        query: "SELECT COUNT(*) FROM orders WHERE status = 'pending'"
        queryValue: "5"
```

```bash
# 測試 MySQL 查詢
mysql -h mysql-host -u user -p -e \
  "SELECT COUNT(*) FROM jobs WHERE status = 'pending';"

# 建立 MySQL Secret
kubectl create secret generic mysql-secrets \
  --from-literal=mysql_conn_str='user:password@tcp(mysql:3306)/mydb'

# 查看 ScaledObject 狀態
kubectl describe scaledobject mysql-scaledobject

# 查看 HPA 狀態
kubectl get hpa -w
```

---

## 常見錯誤與提示 (Common Mistakes & Tips)

| 常見錯誤 | 正確做法 |
|----------|----------|
| 連接字串格式錯誤 | 使用 `user:password@tcp(host:port)/db` 格式 |
| 查詢返回多列 | 確保查詢只返回單一數值 |
| 查詢返回 NULL | 使用 `COALESCE(COUNT(*), 0)` 處理 NULL |
| 無法連接到 MySQL | 確認網路連通性和防火牆設定 |

### 提示

- 使用 `CEIL()` 函數可以確保查詢結果向上取整
- 設定 `activationQueryValue` 可避免少量待處理項目就觸發擴縮
- 對於 Airflow 等工作佇列，監控 `running` 和 `queued` 狀態的任務
- 使用 TriggerAuthentication 管理敏感的連線資訊

### 查詢範例

```sql
-- 計算待處理任務（每 6 個任務 1 個 Pod）
SELECT CEIL(COUNT(*) / 6)
FROM task_instance
WHERE state='running' OR state='queued';

-- 計算佇列深度
SELECT COUNT(*) FROM job_queue WHERE processed = 0;

-- 使用 COALESCE 處理可能的 NULL
SELECT COALESCE(COUNT(*), 0) FROM pending_orders;

-- 計算特定時間範圍內的待處理訂單
SELECT COUNT(*) FROM orders
WHERE status = 'pending'
AND created_at > DATE_SUB(NOW(), INTERVAL 1 HOUR);
```

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `connectionStringFromEnv` | 連接字串環境變數 | 無（與其他參數二選一）|
| `host` | 資料庫主機 | 無 |
| `port` | 資料庫埠口 | 3306 |
| `username` | 使用者名稱 | 無 |
| `dbName` | 資料庫名稱 | 無 |
| `query` | 查詢語句 | 無（必填）|
| `queryValue` | 目標值 | 無（必填）|
| `activationQueryValue` | 啟動閾值 | 0 |

| 指令 | 說明 |
|------|------|
| `kubectl get scaledobject` | 列出 ScaledObject |
| `mysql -e "SELECT..."` | 測試 MySQL 查詢 |
| `kubectl describe scaledobject <name>` | 查看詳細狀態 |
