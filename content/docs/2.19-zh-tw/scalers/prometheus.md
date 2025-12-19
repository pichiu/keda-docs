+++
title = "Prometheus"
availability = "v1.0+"
maintainer = "Community"
category = "Metrics"
description = "根據 Prometheus 擴縮應用程式。"
go_file = "prometheus_scaler"
+++

## TL;DR

Prometheus 擴縮器根據 Prometheus 查詢結果來擴縮應用程式。支援多種驗證方式（Bearer、Basic、TLS、Custom），可與 AWS、Azure、GCP 的托管 Prometheus 服務整合。

---

## 翻譯 (Translation)

### 觸發器規格

此規格描述根據 Prometheus 擴縮的 `prometheus` 觸發器。

```yaml
triggers:
- type: prometheus
  metadata:
    # 必填欄位：
    serverAddress: http://<prometheus-host>:9090
    query: sum(rate(http_requests_total{deployment="my-deployment"}[2m])) # 注意：查詢必須返回向量/純量單一元素響應
    threshold: '100.50'
    activationThreshold: '5.5'
    # 可選欄位：
    namespace: example-namespace  # 用於命名空間查詢，例如 Thanos
    customHeaders: X-Client-Id=cid,X-Tenant-Id=tid # 可選。查詢中包含的自訂標頭。
    ignoreNullValues: "false" # 預設為 `true`，表示忽略 Prometheus 的空值列表。設為 `false` 時，當 Prometheus 目標遺失時會返回錯誤
    queryParameters: key-1=value-1,key-2=value-2
    unsafeSsl: "false" # 預設為 `false`，用於跳過自簽憑證的憑證檢查
    timeout: "1000" # 可選。此擴縮器使用的 HTTP 客戶端自訂逾時
```

**參數列表：**

- `serverAddress` - Prometheus 伺服器位址。如使用 VictoriaMetrics 叢集版本，設定完整的 Prometheus 查詢 API URL，例如 `http://<vmselect>:8481/select/0/prometheus`
- `query` - 要執行的查詢。
- `threshold` - 開始擴縮的值（此值可以是浮點數）
- `activationThreshold` - 啟動擴縮器的目標值（預設：`0`，可選，此值可以是浮點數）
- `namespace` - 用於命名空間查詢的命名空間。某些高可用 Prometheus 設定（如 [Thanos](https://thanos.io)）需要這些。（可選）
- `ignoreNullValues` - 當 Prometheus 目標遺失時是否報告錯誤（值：`true`、`false`，預設：`true`，可選）
- `unsafeSsl` - 用於跳過憑證檢查，例如使用自簽憑證（值：`true`、`false`，預設：`false`，可選）

### 驗證參數

Prometheus 擴縮器支援多種驗證類型以幫助您與 Prometheus 整合。

您可以使用 `TriggerAuthentication` CRD 設定驗證。可以指定多種驗證類型，例如 `authModes: "tls,basic"`

**Bearer 驗證：**
- `authModes`: 必須包含 `bearer`。
- `bearerToken`: 驗證所需的令牌。必填欄位。

**Basic 驗證：**
- `authModes`: 必須包含 `basic`。
- `username` - 使用者名稱。必填欄位。
- `password` - 密碼。可選。

**TLS 驗證：**
- `authModes`: 必須包含 `tls`。
- `ca` - TLS 客戶端驗證的憑證授權單位檔案。
- `cert` - 客戶端驗證的憑證。必填欄位。
- `key` - 客戶端驗證的金鑰。必填欄位。

**自訂驗證：**
- `authModes`: 必須包含 `custom`。
- `customAuthHeader`: 自訂授權標頭名稱。必填欄位。
- `customAuthValue`: 自訂授權標頭值。必填欄位。

### 整合雲端服務

#### Amazon Managed Service for Prometheus

AWS 提供 [Prometheus 托管服務](https://aws.amazon.com/prometheus/)。可使用 EKS Pod Identity 進行驗證。

#### Azure Monitor Managed Service for Prometheus

Azure 提供 [Prometheus 托管服務](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/prometheus-metrics-overview)。可使用 Azure AD Workload Identity 進行驗證。

#### Google Managed Service for Prometheus

GCP 提供 [Prometheus 托管服務](https://cloud.google.com/stackdriver/docs/managed-prometheus)。可使用 GCP Workload Identity 進行驗證。

---

## 說明 (Explanation)

### 查詢注意事項

| 要點 | 說明 |
|------|------|
| 返回類型 | 查詢必須返回向量或純量的單一元素 |
| 空值處理 | `ignoreNullValues: true`（預設）會忽略空值 |
| 指標類型 | 預設使用 `AverageValue`，會除以副本數 |

### 驗證方式比較

| 驗證方式 | 使用場景 | 設定複雜度 |
|----------|----------|------------|
| Bearer | API Token 驗證 | 低 |
| Basic | 使用者名稱/密碼 | 低 |
| TLS | 憑證驗證 | 中 |
| Custom | 自訂標頭 | 中 |
| Pod Identity | 雲端原生 | 高（但更安全）|

---

## 實作範例 (Practical Example)

```yaml
# 基本範例 - 無驗證
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: prometheus-scaledobject
spec:
  scaleTargetRef:
    name: my-deployment
  minReplicaCount: 1
  maxReplicaCount: 20
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus:9090
        query: sum(rate(http_requests_total{app="my-app"}[2m]))
        threshold: "100"
```

```yaml
# 使用 Basic 驗證
apiVersion: v1
kind: Secret
metadata:
  name: prometheus-secret
data:
  username: dXNlcm5hbWU=  # base64
  password: cGFzc3dvcmQ=  # base64
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: prometheus-auth
spec:
  secretTargetRef:
    - parameter: username
      name: prometheus-secret
      key: username
    - parameter: password
      name: prometheus-secret
      key: password
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: prometheus-scaledobject
spec:
  scaleTargetRef:
    name: my-deployment
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus:9090
        query: sum(rate(http_requests_total{app="my-app"}[2m]))
        threshold: "100"
        authModes: "basic"
      authenticationRef:
        name: prometheus-auth
```

```yaml
# Azure 托管 Prometheus 範例
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: azure-prometheus-auth
spec:
  podIdentity:
    provider: azure-workload
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: azure-prometheus-scaler
spec:
  scaleTargetRef:
    name: my-deployment
  triggers:
    - type: prometheus
      metadata:
        serverAddress: https://my-workspace.eastus.prometheus.monitor.azure.com
        query: sum(rate(http_requests_total[2m]))
        threshold: "100"
      authenticationRef:
        name: azure-prometheus-auth
```

```bash
# 測試 Prometheus 查詢
curl "http://prometheus:9090/api/v1/query?query=sum(rate(http_requests_total[2m]))"

# 查看 ScaledObject 狀態
kubectl describe scaledobject prometheus-scaledobject

# 查看 HPA 指標
kubectl get hpa -w
```

---

## 常見錯誤與提示 (Common Mistakes & Tips)

| 常見錯誤 | 正確做法 |
|----------|----------|
| 查詢返回多個元素 | 確保查詢返回單一值（使用 sum、avg 等聚合函數）|
| Prometheus 無法存取 | 確認網路連通性和防火牆設定 |
| 指標為 0 但不縮減 | 檢查 `activationThreshold` 和 `cooldownPeriod` |
| 使用 Pod Identity 時驗證失敗 | 確認角色綁定和服務帳戶設定 |

### 提示

- 使用 `ignoreNullValues: false` 可以在指標來源失效時得到明確的錯誤
- 對於自簽憑證，設定 `unsafeSsl: true`
- 結合 `customHeaders` 可以新增租戶 ID 等標頭
- 雲端托管 Prometheus 建議使用 Pod Identity 而非 API 金鑰

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `serverAddress` | Prometheus 伺服器位址 | 無（必填）|
| `query` | PromQL 查詢 | 無（必填）|
| `threshold` | 擴縮閾值 | 無（必填）|
| `activationThreshold` | 啟動閾值 | 0 |
| `ignoreNullValues` | 忽略空值 | true |
| `unsafeSsl` | 跳過 SSL 驗證 | false |
| `namespace` | Thanos 命名空間 | 無 |

| authModes | 說明 | 必要參數 |
|-----------|------|----------|
| `bearer` | Bearer Token | bearerToken |
| `basic` | 使用者名稱/密碼 | username, password |
| `tls` | TLS 憑證 | cert, key |
| `custom` | 自訂標頭 | customAuthHeader, customAuthValue |

| 指令 | 說明 |
|------|------|
| `kubectl get scaledobject` | 列出 ScaledObject |
| `curl "http://prometheus:9090/api/v1/query?query=..."` | 測試查詢 |
