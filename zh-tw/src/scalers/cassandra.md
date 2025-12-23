
## TL;DR

Cassandra 擴縮器根據 Cassandra 查詢輸出結果自動擴縮應用程式。適用於需要根據 Cassandra 資料庫中的資料量進行擴縮的場景。

---

## 翻譯 (Translation)

### 觸發器規格

此規格描述基於 Cassandra 查詢輸出進行擴縮的 `cassandra` 觸發器。

```yaml
triggers:
  - type: cassandra
    metadata:
      username: "cassandra"
      port: "9042"
      clusterIPAddress: "cassandra.default"
      consistency: "Quorum"
      protocolVersion: "4"
      keyspace: "test_keyspace"
      query: "SELECT COUNT(*) FROM test_keyspace.test_table;"
      targetQueryValue: "1"
      activationTargetQueryValue: "10"
```

**參數列表：**

- `username` - 連線到 Cassandra 實例的使用者名稱憑證。
- `port` - Cassandra 實例的埠號。（可選，可以在這裡或在 `clusterIPAddress` 中設定）
- `clusterIPAddress` - Cassandra 實例的 IP 位址或主機名稱。
- `consistency` - 會話或個別讀取操作的設定。（值：`LOCAL_ONE`、`LOCAL_QUORUM`、`EACH_QUORUM`、`LOCAL_SERIAL`、`ONE`、`TWO`、`THREE`、`QUORUM`、`SERIAL`、`ALL`，預設：`ONE`，可選）
- `protocolVersion` - CQL 二進位協定。（預設：`4`，可選）
- `keyspace` - Cassandra 中使用的 keyspace 名稱。
- `query` - 應傳回單一數值的 Cassandra 查詢。
- `targetQueryValue` - 由使用者提供的閾值，在 Horizontal Pod Autoscaler（HPA）中用作 `targetValue` 或 `targetAverageValue`（取決於觸發器指標類型）。
- `activationTargetQueryValue` - 啟動擴縮器的目標值。（預設：`0`，可選）

### 驗證參數

您可以透過 `TriggerAuthentication` 設定使用密碼進行驗證。

**密碼驗證：**
- `password` - 登入 Cassandra 實例的設定使用者密碼。
- `tls` - 要為 Cassandra 會話啟用 SSL 驗證，請將此設為 enable。（值：enable、disable，預設：disable，可選）
- `cert` - 用戶端驗證的憑證路徑。如果啟用 TLS 則為必填。（可選）
- `key` - 用戶端驗證的金鑰路徑。如果啟用 TLS 則為必填。（可選）

---

## 實作範例 (Practical Example)

無 TLS 驗證的範例：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cassandra-secrets
type: Opaque
data:
  cassandra_password: CASSANDRA_PASSWORD
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: keda-trigger-auth-cassandra-secret
spec:
  secretTargetRef:
  - parameter: password
    name: cassandra-secrets
    key: cassandra_password
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: cassandra-scaledobject
spec:
  scaleTargetRef:
    name: nginx-deployment
  triggers:
  - type: cassandra
    metadata:
      username: "cassandra"
      port: "9042"
      clusterIPAddress: "cassandra.default"
      consistency: "Quorum"
      protocolVersion: "4"
      query: "SELECT COUNT(*) FROM test_keyspace.test_table;"
      targetQueryValue: "1"
    authenticationRef:
      name: keda-trigger-auth-cassandra-secret
```

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `username` | Cassandra 使用者名稱 | 無（必填）|
| `clusterIPAddress` | Cassandra 主機位址 | 無（必填）|
| `port` | Cassandra 埠號 | 9042 |
| `keyspace` | Keyspace 名稱 | 無（必填）|
| `query` | Cassandra 查詢 | 無（必填）|
| `targetQueryValue` | 目標查詢值 | 無（必填）|
| `consistency` | 一致性層級 | ONE |
| `protocolVersion` | CQL 協定版本 | 4 |

| 一致性層級 | 說明 |
|------------|------|
| `ONE` | 單一副本確認 |
| `QUORUM` | 法定數副本確認 |
| `LOCAL_QUORUM` | 本地資料中心法定數 |
| `ALL` | 所有副本確認 |
