
## TL;DR

InfluxDB 擴縮器根據 InfluxDB 查詢結果自動擴縮應用程式。支援 InfluxDB v2.x（Flux 查詢）和 v3.x（InfluxQL/FlightSQL）。

---

## 翻譯 (Translation)

### 觸發器規格

此規格描述基於 InfluxDB 查詢結果進行擴縮的 `influxdb` 觸發器。

```yaml
triggers:
  - type: influxdb
    metadata:
      serverURL: http://influxdb:8086
      organizationName: influx-org # v2.x 必填，v3.x 可選
      thresholdValue: '4.4'
      activationThresholdValue: '6.2'
      query: |
        from(bucket: "bucket-of-interest")
        |> range(start: -12h)
        |> filter(fn: (r) => r._measurement == "stat")
      metricKey: 'mymetric' # v3.x 必填，v2.x 忽略
      queryType: 'InfluxQL' # v3.x 必填，v2.x 忽略
      influxVersion: '2' # 可選，預設為 2
      database: 'some-influx-db' # v3.x 必填
      authToken: some-auth-token
```

**參數列表：**

- `authToken` - InfluxDB 用戶端與關聯伺服器通訊所需的驗證令牌。
- `authTokenFromEnv` - 定義授權令牌，類似於 `authToken`，但從擴縮目標上的環境變數讀取。
- `organizationName` - 用戶端定位該[組織](https://docs.influxdata.com/influxdb/v2.0/organizations/)中包含的所有資訊（如 buckets、tasks 等）所需的組織名稱（可選，如果 `influxVersion: '2'` 則必填）。
- `serverURL` - InfluxDB 伺服器的 URL 值。
- `thresholdValue` - 由使用者提供。此值可能因使用案例而異，取決於感興趣的資料，用於根據查詢傳回的值觸發擴縮。（此值可以是浮點數）
- `activationThresholdValue` - 啟動擴縮器的目標值。（預設：`0`，可選，此值可以是浮點數）
- `influxVersion` - 指定正在使用的 InfluxDB 版本。（值：`2`、`3`，預設：`2`，可選）
- `database` - 要查詢的 InfluxDB 資料庫名稱。v3.x 必填。（v2.x 可選）
- `metricKey` - 用於擴縮決策的指標名稱。v3.x 必填，v2.x 忽略。
- `queryType` - 指定正在使用的查詢類型。對於 InfluxDB v3.x，可以是 `InfluxQL` 或 `FlightSQL`。（值：`InfluxQL`、`FlightSQL`，預設：`InfluxQL`，可選）
- `query` - 將產生擴縮器用於與 `thresholdValue` 比較的值的 Flux 查詢。
- `unsafeSsl` - 透過 HTTPS 連線時跳過憑證驗證。（值：`true`、`false`，預設：`false`，可選）

### 驗證參數

您可以使用授權令牌進行驗證。

- `authToken` - InfluxDB 伺服器的授權令牌。

---

## 實作範例 (Practical Example)

InfluxDB v2 範例：

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: influxdb-scaledobject
  namespace: my-project
spec:
  scaleTargetRef:
    name: nginx-worker
  triggers:
    - type: influxdb
      metadata:
        serverURL: http://influxdb:8086
        organizationNameFromEnv: INFLUXDB_ORG_NAME
        thresholdValue: '4'
        activationThresholdValue: '6'
        query: |
          from(bucket: "bucket-of-interest")
          |> range(start: -12h)
          |> filter(fn: (r) => r._measurement == "stat")
        authTokenFromEnv: INFLUXDB_AUTH_TOKEN
```

InfluxDB v3 範例：

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: influxdb-scaledobject
  namespace: my-project
spec:
  scaleTargetRef:
    name: nginx-worker
  triggers:
    - type: influxdb
      metadata:
        serverURL: http://influxdb:8086
        database: 'my-metrics-db'
        influxVersion: '3'
        queryType: 'InfluxQL'
        metricKey: 'mean'
        thresholdValue: '2'
        activationThresholdValue: '10'
        query: |
          SELECT mean("water_level") FROM "h2o_feet"
          GROUP BY time(5m)
          ORDER BY time DESC LIMIT 1;
        authTokenFromEnv: INFLUXDB_AUTH_TOKEN
```

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `serverURL` | InfluxDB 伺服器 URL | 無（必填）|
| `influxVersion` | InfluxDB 版本 | 2 |
| `organizationName` | 組織名稱（v2.x）| 無（v2.x 必填）|
| `database` | 資料庫名稱（v3.x）| 無（v3.x 必填）|
| `query` | Flux 或 InfluxQL 查詢 | 無（必填）|
| `thresholdValue` | 閾值 | 無（必填）|
| `activationThresholdValue` | 啟動閾值 | 0 |
| `queryType` | 查詢類型（v3.x）| InfluxQL |
| `metricKey` | 指標金鑰（v3.x）| 無（v3.x 必填）|

| 版本 | 查詢語言 |
|------|----------|
| v2.x | Flux |
| v3.x | InfluxQL 或 FlightSQL |
