
## TL;DR

MSSQL 擴縮器根據 Microsoft SQL Server 查詢結果自動擴縮應用程式。支援本機 MSSQL 容器和雲端 SQL Server 端點，如 Azure SQL Database。

---

## 翻譯 (Translation)

### 觸發器規格

此規格描述基於 [Microsoft SQL Server](https://www.microsoft.com/sql-server/)（MSSQL）查詢結果進行擴縮的 `mssql` 觸發器。此觸發器支援本機 [MSSQL 容器](https://hub.docker.com/_/microsoft-mssql-server)以及雲端託管的 SQL Server 端點，如 [Azure SQL Database](https://azure.microsoft.com/services/sql-database/)。

```yaml
triggers:
- type: mssql
  metadata:
    connectionStringFromEnv: MSSQL_CONNECTION_STRING
    query: "SELECT COUNT(*) FROM backlog WHERE state='running' OR state='queued'"
    targetValue: "5.5"
    activationTargetValue: '5'
```

> 💡 **注意：** 此擴縮器支援的連線字串格式與其他平台（如 .NET）支援的連線字串格式有一些不相容之處。例如，MSSQL 實例的埠號必須分離到自己的 `Port` 屬性中，而不是加到 `Server` 屬性中。

或者，您可以明確設定連線參數而不是提供連線字串：

```yaml
triggers:
- type: mssql
  metadata:
    username: "kedaUser"
    passwordFromEnv: MSSQL_PASSWORD
    host: mssqlinst.namespace.svc.cluster.local
    port: "1433" # 可選
    database: test_db_name
    query: "SELECT COUNT(*) FROM backlog WHERE state='running' OR state='queued'"
    targetValue: 1
```

`mssql` 觸發器始終需要以下資訊：

- `query` - 傳回單一數值的 [T-SQL](https://docs.microsoft.com/sql/t-sql/language-reference) 查詢。這可以是一般查詢或預存程序的名稱。
- `targetValue` - 在 Horizontal Pod Autoscaler（HPA）中用作 `targetValue` 或 `targetAverageValue` 的閾值（取決於觸發器指標類型）。（此值可以是浮點數）
- `activationTargetValue` - 啟動擴縮器的目標值。在[這裡](./../concepts/scaling-deployments.md#activating-and-scaling-thresholds)了解更多。（預設：`0`，可選，此值可以是浮點數）

### 驗證參數

**連線字串驗證：**
- `connectionString` - MSSQL 實例的連線字串。

**密碼驗證：**
- `host` - MSSQL 實例端點的主機名稱。
- `port` - MSSQL 實例端點的埠號。（預設：1433）
- `database` - 要查詢的資料庫名稱。
- `username` - 連線到 MSSQL 實例的使用者名稱憑證。
- `password` - 連線到 MSSQL 實例的密碼憑證。

---

## 實作範例 (Practical Example)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mssql-secrets
type: Opaque
data:
  mssql-connection-string: <base64 編碼的連線字串>
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: keda-trigger-auth-mssql-secret
spec:
  secretTargetRef:
  - parameter: connectionString
    name: mssql-secrets
    key: mssql-connection-string
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: mssql-scaledobject
spec:
  scaleTargetRef:
    name: consumer
  triggers:
  - type: mssql
    metadata:
      targetValue: "1"
      query: "SELECT COUNT(*) FROM backlog WHERE state='running' OR state='queued'"
    authenticationRef:
      name: keda-trigger-auth-mssql-secret
```

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `connectionStringFromEnv` | 連線字串環境變數 | 無 |
| `query` | T-SQL 查詢 | 無（必填）|
| `targetValue` | 目標值 | 無（必填）|
| `activationTargetValue` | 啟動閾值 | 0 |
| `host` | MSSQL 主機名稱 | 無 |
| `port` | MSSQL 埠號 | 1433 |
| `database` | 資料庫名稱 | 無 |

| 連線字串格式 | 範例 |
|-------------|------|
| URL 格式 | `sqlserver://user:Pass@host:1433?database=DB` |
| ADO 格式 | `Server=host;Port=1433;Database=DB;User ID=user;Password=Pass;` |
