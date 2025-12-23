
## TL;DR

Apache Pulsar 擴縮器根據 Pulsar 主題訂閱中的訊息積壓（backlog）自動擴縮應用程式。支援 Bearer、Basic、TLS 和 OAuth 驗證方式。

---

## 翻譯 (Translation)

### 觸發器規格

此規格描述 Apache Pulsar 主題的 `pulsar` 觸發器。

```yaml
triggers:
- type: pulsar
  metadata:
    adminURL: http://localhost:80
    topic: persistent://public/default/my-topic
    isPartitionedTopic: false
    subscription: sub1
    msgBacklogThreshold: '5'
    activationMsgBacklogThreshold: '2'
    authModes: ""
```

**參數列表：**

- `adminURL` - 主題 admin API 的統計 URL。
- `topic` - Pulsar 主題。格式為 `persistent://{tenant}/{namespace}/{topicName}`
- `isPartitionedTopic` - 主題是否為分割區主題。當為 `true` 時，`msgBacklogThreshold` 將是跨分割區的累積訂閱積壓。（預設：`false`，可選）
- `subscription` - 主題訂閱的名稱
- `msgBacklogThreshold` - 觸發擴縮動作的平均目標值。（預設：10）
- `activationMsgBacklogThreshold` - 啟動擴縮器的目標值。（預設：`0`，可選）
- `authModes` - 要使用的驗證模式，以逗號分隔的清單。（值：`bearer`、`tls`、`basic`、`oauth`，預設：`""`，可選）

### 驗證參數

驗證定義在 `authModes` 中。相關設定參數在 `TriggerAuthentication` 規格中指定。

**Bearer 驗證**
- `bearerToken` - 此令牌將以 `Authorization: Bearer <token>` 標頭形式發送。

**Basic 驗證**
- `username` - 使用者名稱
- `password` - 密碼（可選）

**TLS 驗證**
- `ca` - 用於驗證伺服器憑證的受信任根憑證授權單位。
- `cert` - 用戶端驗證的憑證。
- `key` - 用戶端驗證的金鑰。

**OAuth 2 驗證**
- `oauthTokenURI` - OAuth 提供者的 OAuth 存取令牌 URI。（可選）
- `scope` - OAuth 範圍，以逗號分隔的清單。（可選）
- `clientID` - OAuth 提供者的用戶端 ID。（可選）
- `clientSecret` - OAuth 提供者的用戶端密碼。（可選）
- `endpointParams` - OAuth 提供者令牌端點請求的額外參數，為 URL 編碼的查詢字串。（可選）

---

## 實作範例 (Practical Example)

無驗證和無 TLS：

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: pulsar-scaledobject
  namespace: default
spec:
  scaleTargetRef:
    name: pulsar-consumer
  pollingInterval: 30
  triggers:
  - type: pulsar
    metadata:
      adminURL: http://localhost:80
      topic: persistent://public/default/my-topic
      isPartitionedTopic: false
      subscription: sub1
      msgBacklogThreshold: '5'
```

使用 Bearer 令牌和自訂 CA 憑證的 TLS：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: keda-pulsar-secrets
  namespace: default
data:
  ca: <your self-signed root CA>
  token: <your token>
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: keda-trigger-auth-pulsar-credential
  namespace: default
spec:
  secretTargetRef:
  - parameter: ca
    name: keda-pulsar-secrets
    key: ca
  - parameter: bearerToken
    name: keda-pulsar-secrets
    key: token
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: pulsar-scaledobject
  namespace: default
spec:
  scaleTargetRef:
    name: pulsar-consumer
  pollingInterval: 30
  triggers:
  - type: pulsar
    metadata:
      authModes: "bearer"
      adminURL: https://localhost:8443
      topic: persistent://public/default/my-topic
      subscription: sub1
      msgBacklogThreshold: '5'
    authenticationRef:
      name: keda-trigger-auth-pulsar-credential
```

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `adminURL` | Pulsar Admin API URL | 無（必填）|
| `topic` | Pulsar 主題 | 無（必填）|
| `subscription` | 訂閱名稱 | 無（必填）|
| `isPartitionedTopic` | 是否為分割區主題 | false |
| `msgBacklogThreshold` | 訊息積壓閾值 | 10 |
| `activationMsgBacklogThreshold` | 啟動閾值 | 0 |
| `authModes` | 驗證模式 | "" |

| 驗證模式 | 說明 |
|----------|------|
| `bearer` | Bearer 令牌驗證 |
| `tls` | 雙向 TLS 驗證 |
| `basic` | 基本驗證 |
| `oauth` | OAuth 2.0 驗證 |
