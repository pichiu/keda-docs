
## TL;DR

Loki 擴縮器根據 Loki LogQL 查詢結果自動擴縮應用程式。適用於基於日誌指標進行擴縮的場景。

---

## 翻譯 (Translation)

### 觸發器規格

此規格描述基於 Loki 查詢結果進行擴縮的 `loki` 觸發器。

```yaml
triggers:
- type: loki
  metadata:
    # 必填欄位：
    serverAddress: http://<loki-host>:3100
    query: sum(rate({filename="/var/log/syslog"}[1m])) # 查詢必須傳回向量/純量單一元素回應
    threshold: '0.7'
    # 可選欄位：
    activationThreshold: '2.50'
    tenantName: Tenant1
    ignoreNullValues: "false"
    unsafeSsl: "false"
```

**參數列表：**

- `serverAddress` - Loki 伺服器的 URL。
- `query` - 要執行的 LogQL 查詢。查詢必須傳回向量/純量單一元素回應。
- `threshold` - 開始擴縮的值。（此值可以是浮點數）
- `activationThreshold` - 啟動擴縮器的目標值。（預設：`0`，可選，此值可以是浮點數）
- `tenantName` - 用於在多租戶設定中指定租戶名稱的 `X-Scope-OrgID` 標頭。（可選）
- `ignoreNullValues` - 當 Loki 目標遺失時報告錯誤的值。（值：`true`、`false`，預設：`true`，可選）
- `unsafeSsl` - 用於跳過憑證檢查，例如使用自簽憑證。（值：`true`、`false`，預設：`false`，可選）
- `authModes` - 要使用的驗證模式。（值：`bearer`、`basic`，可選）

### 驗證參數

Loki 本身不提供任何開箱即用的驗證。然而，Loki 最常與 Basic Auth 或 Bearer Auth 一起設定，這是這裡唯一有效的驗證選項。

**Bearer 驗證：**
- `authModes`：如果使用 Bearer 驗證，必須包含 `bearer`。在觸發器設定中指定。
- `bearerToken`：驗證所需的令牌。

**基本驗證：**
- `authModes`：如果使用基本驗證，必須包含 `basic`。在觸發器設定中指定。
- `username` - 提供用於基本驗證的使用者名稱。
- `password` - 提供用於驗證的密碼。（可選）

---

## 實作範例 (Practical Example)

基本範例：

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: loki-scaledobject
  namespace: default
spec:
  maxReplicaCount: 12
  scaleTargetRef:
    name: nginx
  triggers:
    - type: loki
      metadata:
        serverAddress: http://<loki-host>:3100
        threshold: '0.7'
        query: sum(rate({filename="/var/log/syslog"}[1m]))
```

使用 Bearer 驗證的範例：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: keda-loki-secret
  namespace: default
data:
  bearerToken: "BEARER_TOKEN"
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: keda-loki-creds
  namespace: default
spec:
  secretTargetRef:
    - parameter: bearerToken
      name: keda-loki-secret
      key: bearerToken
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: loki-scaledobject
  namespace: default
spec:
  maxReplicaCount: 12
  scaleTargetRef:
    name: nginx
  triggers:
    - type: loki
      metadata:
        serverAddress: http://<loki-host>:3100
        threshold: '0.7'
        query: sum(rate({filename="/var/log/syslog"}[1m]))
        authModes: "bearer"
      authenticationRef:
        name: keda-loki-creds
```

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `serverAddress` | Loki 伺服器 URL | 無（必填）|
| `query` | LogQL 查詢 | 無（必填）|
| `threshold` | 閾值 | 無（必填）|
| `activationThreshold` | 啟動閾值 | 0 |
| `tenantName` | 租戶名稱（X-Scope-OrgID）| 無 |
| `ignoreNullValues` | 忽略空值 | true |
| `unsafeSsl` | 跳過 SSL 驗證 | false |

| 驗證模式 | 說明 |
|----------|------|
| `bearer` | Bearer 令牌驗證 |
| `basic` | 基本驗證（使用者名稱/密碼）|
