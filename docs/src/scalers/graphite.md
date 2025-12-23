
## TL;DR

Graphite 擴縮器根據 Graphite 查詢結果自動擴縮應用程式。適用於使用 Graphite 作為指標儲存的監控系統。

---

## 翻譯 (Translation)

### 觸發器規格

此規格描述基於 Graphite 指標進行擴縮的 `graphite` 觸發器。

```yaml
triggers:
- type: graphite
  metadata:
    # 必填
    serverAddress: http://<graphite-host>:81
    query: stats.counters.http.hello-world.request.count.count
    threshold: '10.5'
    activationThreshold: '5'
    queryTime: '-10Minutes'
```

**參數列表：**

- `serverAddress` - Graphite 的位址
- `query` - 要執行的查詢。查詢必須傳回向量/純量單一元素回應。
- `threshold` - 觸發擴縮動作的目標值。（預設：100，可選，此值可以是浮點數）
- `activationThreshold` - 啟動擴縮器的目標值。（預設：`0`，可選，此值可以是浮點數）
- `queryTime` - 執行查詢的相對時間範圍。請參閱 [Graphite API 文件](https://graphite-api.readthedocs.io/en/latest/api.html#from-until)以取得更多資訊。

### 驗證參數

Graphite 擴縮器支援一種驗證類型 - 基本驗證

**基本驗證：**
- `authMode`：如果使用基本驗證，必須包含 `basic`。在觸發器設定中指定。
- `username` - 必填欄位。提供用於基本驗證的使用者名稱。
- `password` - 提供用於驗證的密碼。為方便起見，這被標記為可選，因為許多應用程式將基本驗證實作為使用者名稱作為 apikey 而密碼為空。

---

## 實作範例 (Practical Example)

基本範例：

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: graphite-scaledobject
  namespace: default
spec:
  scaleTargetRef:
    name: my-deployment
  triggers:
  - type: graphite
    metadata:
      serverAddress: http://<graphite-host>:81
      threshold: '100'
      query: maxSeries(keepLastValue(reportd.*.gauge.detect.latest_max_time.value, 1))
      queryTime: '-1Minutes'
```

使用基本驗證的範例：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: keda-graphite-secret
  namespace: default
data:
  username: "dXNlcm5hbWUK"
  password: "cGFzc3dvcmQK"
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: keda-graphite-creds
  namespace: default
spec:
  secretTargetRef:
    - parameter: username
      name: keda-graphite-secret
      key: username
    - parameter: password
      name: keda-graphite-secret
      key: password
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: graphite-scaledobject
  namespace: default
spec:
  scaleTargetRef:
    name: php-apache-graphite
  triggers:
  - type: graphite
    metadata:
      authMode: "basic"
      query: https_metric
      queryTime: -1Hours
      serverAddress: http://<graphite server>:81
      threshold: "100"
    authenticationRef:
      name: keda-graphite-creds
```

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `serverAddress` | Graphite 伺服器位址 | 無（必填）|
| `query` | Graphite 查詢 | 無（必填）|
| `threshold` | 閾值 | 100 |
| `activationThreshold` | 啟動閾值 | 0 |
| `queryTime` | 查詢時間範圍 | 無（必填）|

| 時間格式範例 | 說明 |
|-------------|------|
| `-10Seconds` | 過去 10 秒 |
| `-10Minutes` | 過去 10 分鐘 |
| `-1Hours` | 過去 1 小時 |
