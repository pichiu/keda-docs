+++
title = "Azure Log Analytics"
availability = "v2.0+"
maintainer = "Microsoft"
category = "Data & Storage"
description = "根據 Azure Log Analytics 查詢結果擴縮應用程式"
go_file = "azure_log_analytics_scaler"
+++

## TL;DR

Azure Log Analytics 擴縮器根據 Azure Log Analytics Kusto 查詢結果自動擴縮應用程式。適用於基於日誌指標進行擴縮的場景。

---

## 翻譯 (Translation)

### 觸發器規格

此規格描述基於 Azure Log Analytics 查詢結果進行擴縮的 `azure-log-analytics` 觸發器。

```yaml
triggers:
  - type: azure-log-analytics
    metadata:
      tenantId: "AZURE_AD_TENANT_ID"
      clientId: "SERVICE_PRINCIPAL_CLIENT_ID"
      clientSecret: "SERVICE_PRINCIPAL_PASSWORD"
      workspaceId: "LOG_ANALYTICS_WORKSPACE_ID"
      query: |
        Perf
        | where CounterName == "cpuUsageNanoCores"
        | summarize MetricValue=round(avg(CounterValue))
        | project MetricValue
      threshold: "10.7"
      activationThreshold: "1.7"
      cloud: Private
      logAnalyticsResourceURL: https://api.loganalytics.airgap.io/
      activeDirectoryEndpoint: https://login.airgap.example/
```

**參數列表：**

- `tenantId` - Azure Active Directory 租用戶 ID。
- `clientId` - Azure AD 應用程式/服務主體的應用程式 ID。
- `clientSecret` - Azure AD 應用程式/服務主體的密碼。
- `workspaceId` - Log Analytics 工作區 ID。
- `query` - Log Analytics [Kusto](https://docs.microsoft.com/en-us/azure/azure-monitor/log-query/get-started-queries) 查詢，需進行 JSON 轉義。
- `threshold` - 用作閾值來計算擴縮目標的 Pod 數量的值。（此值可以是浮點數）
- `activationThreshold` - 啟動擴縮器的目標值。（預設：`0`，可選，此值可以是浮點數）
- `cloud` - Azure Log Analytics 工作區所屬的雲端環境名稱。（值：`AzurePublicCloud`、`AzureUSGovernmentCloud`、`AzureChinaCloud`、`Private`，預設：`AzurePublicCloud`，可選）
- `logAnalyticsResourceURL` - 雲端環境的 Log Analytics REST API URL。（`cloud` 設為 `Private` 時必填）
- `activeDirectoryEndpoint` - 雲端環境的 Active Directory 端點。（`cloud` 設為 `Private` 時必填）
- `unsafeSsl` - 決定 KEDA 是否驗證伺服器憑證的鏈和主機名稱。（預設：`false`，可選）

### 查詢指南

重要的是將查詢設計為傳回 1 個表格和 1 列。好的做法是在查詢結尾新增 "| limit 1"。

擴縮器將從以下位置取值：
- 第 1 個儲存格作為指標值。
- 第 2 個儲存格作為閾值（可選）。

### 驗證參數

**基於服務主體的驗證：**
- `tenantId` - Azure Active Directory 租用戶 ID。
- `clientId` - Azure AD 應用程式/服務主體的應用程式 ID。
- `clientSecret` - Azure AD 應用程式/服務主體的密碼。
- `workspaceId` - Log Analytics 工作區 ID。

**基於受控識別的驗證：**
可以使用 [Azure AD Workload Identity](https://azure.github.io/azure-workload-identity/docs/) 提供者。

---

## 實作範例 (Practical Example)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: kedaloganalytics
  namespace: kedaloganalytics
type: Opaque
data:
  tenantId: "QVpVUkVfQURfVEVOQU5UX0lE"
  clientId: "U0VSVklDRV9QUklOQ0lQQUxfQ0xJRU5UX0lE"
  clientSecret: "U0VSVklDRV9QUklOQ0lQQUxfUEFTU1dPUkQ="
  workspaceId: "TE9HX0FOQUxZVElDU19XT1JLU1BBQ0VfSUQ="
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: trigger-auth-kedaloganalytics
  namespace: kedaloganalytics
spec:
  secretTargetRef:
    - parameter: tenantId
      name: kedaloganalytics
      key: tenantId
    - parameter: clientId
      name: kedaloganalytics
      key: clientId
    - parameter: clientSecret
      name: kedaloganalytics
      key: clientSecret
    - parameter: workspaceId
      name: kedaloganalytics
      key: workspaceId
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kedaloganalytics-scaledobject
  namespace: kedaloganalytics
spec:
  scaleTargetRef:
    name: kedaloganalytics-consumer
  pollingInterval: 30
  cooldownPeriod: 30
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
  - type: azure-log-analytics
    metadata:
      query: |
        Perf
        | where CounterName == "cpuUsageNanoCores"
        | summarize MetricValue=round(avg(CounterValue))
        | project MetricValue
      threshold: "1900000000"
    authenticationRef:
      name: trigger-auth-kedaloganalytics
```

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `tenantId` | Azure AD 租用戶 ID | 無（必填）|
| `clientId` | 應用程式/服務主體 ID | 無（必填）|
| `clientSecret` | 應用程式密碼 | 無（必填）|
| `workspaceId` | Log Analytics 工作區 ID | 無（必填）|
| `query` | Kusto 查詢 | 無（必填）|
| `threshold` | 閾值 | 無（必填）|
| `activationThreshold` | 啟動閾值 | 0 |
| `cloud` | 雲端環境 | AzurePublicCloud |

| 驗證方式 | 說明 |
|----------|------|
| 服務主體 | 使用 Azure AD 應用程式憑證 |
| 受控識別 | 使用 Azure Workload Identity |
