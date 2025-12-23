
## TL;DR

Datadog 擴縮器根據 Datadog 查詢結果自動擴縮應用程式。支援透過 REST API 或 Datadog Cluster Agent 作為代理來輪詢指標。使用 Cluster Agent 可以減少達到速率限制的風險。

---

## 翻譯 (Translation)

> 💡 **注意：** 在定義輪詢間隔時，請考慮 [Datadog API 端點速率限制](https://docs.datadoghq.com/api/latest/rate-limits/)。

有兩種方式可以使用 Datadog 擴縮器輪詢查詢值：使用 REST API 端點，或使用 [Datadog Cluster Agent](https://docs.datadoghq.com/containers/cluster_agent/) 作為代理。使用 Datadog Cluster Agent 作為代理可以減少達到速率限制的機會。

## 使用 Datadog Cluster Agent（實驗性）

透過此方法，Datadog 擴縮器將連線到 Datadog Cluster Agent 來擷取用於驅動 KEDA 擴縮事件的查詢值。這降低了達到 Datadog API 速率限制的風險，因為 Cluster Agent 會批次擷取指標值。

### 觸發器規格

```yaml
triggers:
- type: datadog
  metricType: Value
  metadata:
    useClusterAgentProxy: "true"
    datadogMetricName: "nginx-hits"
    datadogMetricNamespace: "default"
    targetValue: "7.75"
    activationQueryValue: "1.1"
    metricUnavailableValue: "1.5"
    timeout: "10s"
```

**參數列表：**

- `useClusterAgentProxy` - 是否使用 Cluster Agent 作為代理來取得查詢值。（值：true、false，預設：false，可選）
- `datadogMetricName` - 用於驅動擴縮事件的 `DatadogMetric` 物件名稱。
- `datadogMetricNamespace` - `DatadogMetric` 物件的命名空間。
- `targetValue` - 開始擴縮的目標值（此值可以是浮點數）。
- `activationQueryValue` - 啟動擴縮器的目標值。（預設：`0`，可選，此值可以是浮點數）
- `metricUnavailableValue` - 如果 Datadog 在指定時間視窗內找不到指標值，傳回給 HPA 的指標值。（可選，此值可以是浮點數）
- `timeout` - 此特定觸發器的逾時時間。（可選）

## 使用 Datadog REST API

### 觸發器規格

```yaml
triggers:
- type: datadog
  metricType: Value
  metadata:
    useClusterAgentProxy: "false"
    query: "sum:trace.redis.command.hits{env:none,service:redis}.as_count()"
    queryValue: "7.75"
    activationQueryValue: "1.1"
    queryAggregator: "max"
    age: "120"
    timeWindowOffset: "30"
    lastAvailablePointOffset: "1"
    metricUnavailableValue: "1.5"
    timeout: "10s"
```

**參數列表：**

- `query` - 要執行的 Datadog 查詢。
- `queryValue` - 開始擴縮的目標值（此值可以是浮點數）。
- `activationQueryValue` - 啟動擴縮器的目標值。（預設：`0`，可選，此值可以是浮點數）
- `queryAggregator` - 當 `query` 是多個查詢（以逗號分隔）時，設定如何聚合多個結果。（值：`max`、`average`，僅當 `query` 包含多個查詢時必填）
- `age` - 從 Datadog 擷取指標的時間視窗（以秒為單位）。（預設：`90`，可選）
- `timeWindowOffset` - 延遲的時間視窗偏移（以秒為單位），用於等待指標可用。（預設：`0`，可選）
- `lastAvailablePointOffset` - 擷取倒數第 X 個資料點的偏移。（預設：`0`，可選）
- `metricUnavailableValue` - 如果 Datadog 在指定時間視窗內找不到指標值，傳回給 HPA 的指標值。（可選）

### 驗證

Datadog 需要 API 金鑰和 APP 金鑰才能從您的帳戶擷取指標。

**參數列表：**

- `apiKey` - Datadog API 金鑰。
- `appKey` - Datadog APP 金鑰。
- `datadogSite` - 取得指標的 Datadog 站點。（預設：`datadoghq.com`，可選）

---

## 實作範例 (Practical Example)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: datadog-secrets
  namespace: my-project
type: Opaque
data:
  apiKey: # 必填：base64 編碼的 Datadog apiKey
  appKey: # 必填：base64 編碼的 Datadog appKey
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: keda-trigger-auth-datadog-secret
  namespace: my-project
spec:
  secretTargetRef:
  - parameter: apiKey
    name: datadog-secrets
    key: apiKey
  - parameter: appKey
    name: datadog-secrets
    key: appKey
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: datadog-scaledobject
  namespace: my-project
spec:
  scaleTargetRef:
    name: worker
  triggers:
  - type: datadog
    metricType: "Value"
    metadata:
      query: "sum:trace.redis.command.hits{env:none,service:redis}.as_count()"
      queryValue: "7"
      age: "120"
      metricUnavailableValue: "0"
    authenticationRef:
      name: keda-trigger-auth-datadog-secret
```

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `useClusterAgentProxy` | 使用 Cluster Agent 代理 | false |
| `query` | Datadog 查詢 | 無（REST API 必填）|
| `queryValue` | 目標值 | 無（必填）|
| `activationQueryValue` | 啟動閾值 | 0 |
| `age` | 時間視窗（秒）| 90 |
| `timeWindowOffset` | 時間視窗偏移（秒）| 0 |
| `lastAvailablePointOffset` | 資料點偏移 | 0 |
| `queryAggregator` | 多查詢聚合方式 | 無 |

| 驗證參數 | 說明 |
|----------|------|
| `apiKey` | Datadog API 金鑰 |
| `appKey` | Datadog APP 金鑰 |
| `datadogSite` | Datadog 站點（例如 datadoghq.com）|
