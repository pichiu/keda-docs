+++
title = "New Relic"
availability = "2.6+"
maintainer = "Community"
category = "Metrics"
description = "根據 New Relic NRQL 擴縮應用程式"
go_file = "newrelic_scaler"
+++

## TL;DR

New Relic 擴縮器根據 New Relic NRQL 查詢結果自動擴縮應用程式。適用於基於應用程式效能指標（如交易時長、錯誤率等）進行擴縮的場景。

---

## 翻譯 (Translation)

### 觸發器規格

此規格描述基於 New Relic 指標進行擴縮的 `new-relic` 觸發器。

```yaml
triggers:
  - type: new-relic
    metadata:
      # 必填：帳戶 - 執行查詢的子帳戶
      account: '1234567'
      # 必填：QueryKey - 連線到 New Relic 的 API 金鑰
      queryKey: "NRAK-xxxxxxxxxxxxxxxxxxxxxxxxxxx"
      # 可選：nrRegion - 查詢資料的區域。預設值為 US。
      region: "US"
      # 可選：noDataError - 如果查詢傳回無資料是否應視為錯誤。預設值為 false。
      noDataError: "true"
      # 必填：nrql
      nrql: "SELECT average(duration) from Transaction where appName='SITE'"
      # 必填：threshold
      threshold: "50.50"
      # 可選：activationThreshold - 啟動擴縮器的目標值。
      activationThreshold: "20.1"
```

**參數列表：**

- `account` - New Relic 中請求應針對的帳戶。
- `queryKey` - 用於連線到 New Relic 並發出請求的 API 金鑰。[官方文件](https://docs.newrelic.com/docs/apis/intro-apis/new-relic-api-keys/)
- `region` - 連線到 New Relic API 的區域。（值：`LOCAL`、`EU`、`STAGING`、`US`，預設：`US`，可選）
- `noDataError` - 傳回無資料的查詢是否應視為錯誤，如果設為 false 且查詢傳回無資料，結果將為 `0`。（值：`true`、`false`，預設：`false`，可選）
- `nrql` - 將執行以取得請求資料的 New Relic 查詢。

  注意：New Relic 查詢的預設時間範圍是過去 30 分鐘，這可能會產生意外的回應。若要模擬 TIMESERIES 查詢的行為，您需要將範圍縮減到 1 分鐘，可以透過在查詢中新增 `SINCE 1 MINUTE AGO` 來實現。

- `threshold` - 在 HPA 設定中用作 `targetValue` 或 `targetAverageValue` 的閾值（取決於觸發器指標類型）。（此值可以是浮點數）
- `activationThreshold` - 啟動擴縮器的目標值。（預設：`0`，可選，此值可以是浮點數）

### 驗證參數

您可以使用 `TriggerAuthentication` CRD 透過 `queryKey` 設定驗證。

- `queryKey` - 用於連線到 New Relic 並發出請求的 API 金鑰。
- `account` - New Relic 中請求應針對的帳戶。
- `region` - 連線到 New Relic API 的區域。

---

## 實作範例 (Practical Example)

基於交易時長平均值指標的自動擴縮觸發器：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: new-relic-secret
  namespace: my-project
type: Opaque
data:
  apiKey: TlJBSy0xMjM0NTY3ODkwMTIzNDU2Nwo= # NRAK-12345678901234567 的 base64 編碼
  account: MTIzNDU2 # 123456 的 base64 編碼
  region: VVM= # US 的 base64 編碼
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: keda-trigger-auth-new-relic
  namespace: my-project
spec:
  secretTargetRef:
  - parameter: queryKey
    name: new-relic-secret
    key: apiKey
  - parameter: account
    name: new-relic-secret
    key: account
  - parameter: region
    name: new-relic-secret
    key: region
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: newrelic-scaledobject
  namespace: keda
spec:
  maxReplicaCount: 12
  scaleTargetRef:
    name: dummy
  triggers:
    - type: new-relic
      metadata:
        nrql: "SELECT average(duration) from Transaction where appName='SITE'"
        noDataError: "true"
        threshold: '1000'
      authenticationRef:
        name: keda-trigger-auth-new-relic
```

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `account` | New Relic 帳戶 ID | 無（必填）|
| `queryKey` | New Relic API 金鑰 | 無（必填）|
| `nrql` | NRQL 查詢 | 無（必填）|
| `threshold` | 閾值 | 無（必填）|
| `region` | New Relic 區域 | US |
| `noDataError` | 無資料時視為錯誤 | false |
| `activationThreshold` | 啟動閾值 | 0 |

| 區域 | 說明 |
|------|------|
| `US` | 美國區域（預設）|
| `EU` | 歐洲區域 |
| `STAGING` | 測試區域 |
| `LOCAL` | 本機區域 |
