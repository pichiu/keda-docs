
## TL;DR

Metrics API 擴縮器允許您使用任何現有的 API 作為指標提供者來擴縮應用程式。支援 JSON、XML、YAML 和 Prometheus 格式的回應。

---

## 翻譯 (Translation)

### 觸發器規格

此規格描述基於 API 提供的指標值進行擴縮的 `metrics-api` 觸發器。

此擴縮器允許使用者將**任何現有的 API** 作為指標提供者。

```yaml
triggers:
  - type: metrics-api
    metadata:
      targetValue: "8.8"
      format: "json"
      activationTargetValue: "3.8"
      url: "http://api:3232/api/v1/stats"
      valueLocation: "components.worker.tasks"
```

**參數列表：**

- `url` - 要呼叫以取得指標值的 API 操作的完整 URL（例如 `http://app:1317/api/v1/stats`）。
- `format` - 以下格式之一：`json`、`xml`、`yaml`、`prometheus`。（預設：`json`，可選）
- `valueLocation` - 回應負載中指標值的位置。值是格式特定的。
  * `json` - [GJSON 路徑表示法](https://github.com/tidwall/gjson#path-syntax)來參照包含指標值的欄位。
  * `yaml`、`xml`、`prometheus` - 實作為點分隔路徑演算法進行值解析。
- `targetValue` - 要擴縮的目標值。當 API 提供的指標等於或高於此值時，KEDA 將開始擴展。（此值可以是浮點數）
- `activationTargetValue` - 啟動擴縮器的目標值。（預設：`0`，可選，此值可以是浮點數）
- `unsafeSsl` - 透過 HTTPS 連線時跳過憑證驗證。（值：`true`、`false`，預設：`false`，可選）
- `aggregateFromKubeServiceEndpoints` - 是否將 `url` 視為 Kubernetes 服務並抓取/聚合此服務所有端點的指標。（值：`true`、`false`，預設：`false`，可選）
- `aggregationType` - 當 `aggregateFromKubeServiceEndpoints` 設為 `true` 時如何聚合指標。（值：`average`、`sum`、`max`、`min`，預設：`average`，可選）

### 驗證參數

Metrics Scaler API 支援四種類型的驗證 - API Key 驗證、基本驗證、TLS 驗證和 Bearer 驗證。

**API Key 驗證：**
- `authMode` - 必須設為 `apiKey`。
- `method` - 可能的值為 `header` 和 `query`。（預設：`header`）
- `keyParamName` - 用於傳遞 apikey 的標頭鍵或查詢參數。
- `apiKey` - 驗證所需的 API Key。

**基本驗證：**
- `authMode` - 必須設為 `basic`。
- `username` - 用於基本驗證的使用者名稱。
- `password` - 用於驗證的密碼。（可選）

**TLS 驗證：**
- `authMode` - 必須設為 `tls`。
- `ca` - TLS 用戶端驗證的憑證授權單位檔案。
- `cert` - 用戶端驗證的憑證。
- `key` - 用戶端驗證的金鑰。（可選）

**Bearer 驗證：**
- `authMode` - 必須設為 `bearer`。
- `token` - 應放在 `Authorization` 標頭中的令牌。

---

## 實作範例 (Practical Example)

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: http-scaledobject
  namespace: keda
spec:
  maxReplicaCount: 12
  scaleTargetRef:
    name: dummy
  triggers:
    - type: metrics-api
      metadata:
        targetValue: "7"
        url: "http://api:3232/components/stats"
        valueLocation: 'components.worker.tasks'
```

API 端點預期回傳類似的回應：
```json
{
  "components": {
    "worker": {
      "tasks": 12
    }
  }
}
```

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `url` | API 端點 URL | 無（必填）|
| `format` | 回應格式 | json |
| `valueLocation` | 指標值位置 | 無（必填）|
| `targetValue` | 目標值 | 無（必填）|
| `activationTargetValue` | 啟動閾值 | 0 |
| `unsafeSsl` | 跳過 SSL 驗證 | false |

| 驗證模式 | 說明 |
|----------|------|
| `apiKey` | API Key 驗證（標頭或查詢參數）|
| `basic` | 基本驗證（使用者名稱/密碼）|
| `tls` | TLS 用戶端憑證驗證 |
| `bearer` | Bearer 令牌驗證 |
