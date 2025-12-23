
## TL;DR

PostgreSQL 擴縮器根據 PostgreSQL 查詢結果來擴縮應用程式。可用於監控資料庫中的待處理任務數量、佇列深度等指標。支援連接字串、密碼驗證和 Azure AD Workload Identity 等多種驗證方式。

---

## 翻譯 (Translation)

### 觸發器規格

此規格描述根據 PostgreSQL 查詢進行擴縮的 `postgresql` 觸發器。

PostgreSQL 擴縮器提供三種連線方式：

**完整連接字串：**
- `connectionFromEnv` - 指向包含有效連接字串的環境變數。

**個別參數：**
- `host` - PostgreSQL 服務 URL。注意應使用完整的 svc URL，因為 KEDA 需要從不同命名空間連接 PostgreSQL。
- `userName` - PostgreSQL 使用者名稱。
- `passwordFromEnv` - PostgreSQL 密碼（從環境變數讀取）。
- `port` - PostgreSQL 埠口。
- `dbName` - PostgreSQL 資料庫名稱。
- `sslmode` - 與資料庫通訊的 SSL 策略。

**查詢設定：**
- `query` - 要對 PostgreSQL 執行的查詢。查詢必須返回整數。
- `targetQueryValue` - 作為 HPA 中 `targetValue` 或 `targetAverageValue` 的閾值。（可以是浮點數）
- `activationTargetQueryValue` - 啟動擴縮器的目標值。（預設：`0`，可選，可以是浮點數）

> 注意：查詢必須返回單一整數值。如果查詢可能返回 `null`，可使用 `COALESCE` 函數設定預設值。例如：`SELECT COALESCE(column_name, 0) FROM table_name;`

```yaml
# 使用完整連接字串
triggers:
- type: postgresql
  metadata:
    connectionFromEnv: AIRFLOW_CONN_AIRFLOW_DB
    query: "SELECT ceil(COUNT(*)::decimal / 16) FROM task_instance WHERE state='running' OR state='queued';"
    targetQueryValue: "1.1"
    activationTargetQueryValue: "5"
```

```yaml
# 使用個別參數
triggers:
- type: postgresql
  metadata:
    userName: "kedaUser"
    passwordFromEnv: PG_PASSWORD
    host: postgres-svc.namespace.cluster.local
    port: "5432"
    dbName: test_db_name
    sslmode: disable
    query: "SELECT ceil(COUNT(*)::decimal / 16) FROM task_instance WHERE state='running' OR state='queued';"
    targetQueryValue: "2.2"
```

### 驗證參數

可使用密碼或連接字串進行驗證，也可使用 Azure Access Token 驗證連接 Azure Postgres Flexible Server。

**連接字串驗證：**
- `connection` - PostgreSQL 資料庫的連接字串。

**密碼驗證：**
- `host` - PostgreSQL 服務 URL。應使用完整 URL（包含命名空間）。
- `userName` - PostgreSQL 使用者名稱。
- `password` - 設定使用者登入 PostgreSQL 資料庫的密碼。
- `port` - PostgreSQL 埠口。
- `dbName` - PostgreSQL 資料庫名稱。
- `sslmode` - 與資料庫通訊的 SSL 策略。

**Azure Access Token 驗證：**

可使用 Azure AD Workload Identity 連接到 Azure Postgres Flexible Server。

前置條件：
- UAMI 需要能夠存取 Azure Postgres Flexible Server
- UAMI 需要被授予存取 KEDA 查詢的資料表的權限

---

## 說明 (Explanation)

### SSL 模式

| 模式 | 說明 |
|------|------|
| `disable` | 不使用 SSL |
| `require` | 使用 SSL（不驗證憑證）|
| `verify-ca` | 驗證 CA |
| `verify-full` | 完整驗證 |

### 連線方式比較

| 方式 | 適用場景 | 安全性 |
|------|----------|--------|
| 連接字串 | 快速設定 | 中 |
| 個別參數 | 靈活設定 | 中 |
| Azure Workload Identity | Azure 雲端 | 高 |

---

## 實作範例 (Practical Example)

```yaml
# 基本範例 - 使用連接字串
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: postgres-scaledobject
spec:
  scaleTargetRef:
    name: worker-deployment
  pollingInterval: 10
  cooldownPeriod: 30
  maxReplicaCount: 10
  triggers:
    - type: postgresql
      metadata:
        connectionFromEnv: DATABASE_URL
        query: "SELECT COUNT(*) FROM jobs WHERE status = 'pending';"
        targetQueryValue: "10"
```

```yaml
# 使用 TriggerAuthentication
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secrets
  namespace: my-project
type: Opaque
data:
  connection: cG9zdGdyZXNxbDovL3VzZXI6cGFzc0Bob3N0OjU0MzIvZGI=  # base64 編碼
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: postgres-trigger-auth
  namespace: my-project
spec:
  secretTargetRef:
    - parameter: connection
      name: postgres-secrets
      key: connection
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: postgres-scaledobject
  namespace: my-project
spec:
  scaleTargetRef:
    name: worker-deployment
  triggers:
    - type: postgresql
      metadata:
        query: "SELECT COUNT(*) FROM task_queue WHERE processed = false;"
        targetQueryValue: "5"
      authenticationRef:
        name: postgres-trigger-auth
```

```yaml
# Azure Postgres Flexible Server 搭配 Workload Identity
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: azure-pg-auth
spec:
  podIdentity:
    provider: azure-workload
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: azure-postgres-scaledobject
spec:
  scaleTargetRef:
    name: airflow-worker
  pollingInterval: 10
  cooldownPeriod: 30
  maxReplicaCount: 10
  triggers:
    - type: postgresql
      authenticationRef:
        name: azure-pg-auth
      metadata:
        host: myserver.postgres.database.azure.com
        port: "5432"
        userName: my-uami-name
        dbName: mydb
        sslmode: require
        query: "SELECT ceil(COUNT(*)::decimal / 16) FROM task_instance WHERE state='running' OR state='queued';"
        targetQueryValue: "1"
```

```bash
# 測試 PostgreSQL 查詢
psql -h postgres-host -U user -d dbname -c \
  "SELECT COUNT(*) FROM jobs WHERE status = 'pending';"

# 查看 ScaledObject 狀態
kubectl describe scaledobject postgres-scaledobject

# 查看 HPA 指標
kubectl get hpa -w
```

---

## 常見錯誤與提示 (Common Mistakes & Tips)

| 常見錯誤 | 正確做法 |
|----------|----------|
| 查詢返回多列或多欄 | 確保查詢只返回單一整數值 |
| 使用相對主機名稱 | 使用完整 FQDN，如 `postgres-svc.namespace.cluster.local` |
| 查詢可能返回 NULL | 使用 `COALESCE(count, 0)` 處理 NULL 值 |
| Azure PGBouncer 連線問題 | 使用預設埠口 5432 而非 PGBouncer 埠口 6432 |

### 提示

- 使用 `ceil()` 函數可以確保查詢結果為整數
- 對於 Airflow 等工作佇列，監控 `running` 和 `queued` 狀態的任務數量
- 設定 `pollingInterval` 不要太短，以避免資料庫負載過高
- 使用 `sslmode: require` 確保安全連線

### 查詢範例

```sql
-- 計算待處理任務（每 16 個任務 1 個 Pod）
SELECT ceil(COUNT(*)::decimal / 16)
FROM task_instance
WHERE state='running' OR state='queued';

-- 計算佇列深度
SELECT COUNT(*) FROM job_queue WHERE processed = false;

-- 使用 COALESCE 處理可能的 NULL
SELECT COALESCE(COUNT(*), 0) FROM pending_jobs;
```

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `connectionFromEnv` | 連接字串環境變數 | 無（與其他參數二選一）|
| `host` | 資料庫主機 | 無 |
| `port` | 資料庫埠口 | 5432 |
| `userName` | 使用者名稱 | 無 |
| `dbName` | 資料庫名稱 | 無 |
| `sslmode` | SSL 模式 | disable |
| `query` | 查詢語句 | 無（必填）|
| `targetQueryValue` | 目標值 | 無（必填）|
| `activationTargetQueryValue` | 啟動閾值 | 0 |

| 指令 | 說明 |
|------|------|
| `kubectl get scaledobject` | 列出 ScaledObject |
| `psql -c "SELECT..."` | 測試 PostgreSQL 查詢 |
| `kubectl describe scaledobject <name>` | 查看詳細狀態 |
