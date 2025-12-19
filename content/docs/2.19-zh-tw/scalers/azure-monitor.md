+++
title = "Azure Monitor"
availability = "v1.3+"
maintainer = "Microsoft"
category = "Metrics"
description = "根據 Azure Monitor 指標擴縮應用程式。"
go_file = "azure_monitor_scaler"
+++

## TL;DR

Azure Monitor 擴縮器根據 Azure Monitor 指標自動擴縮應用程式。支援 Azure 平台指標和自訂指標，可用於監控任何 Azure 資源。

---

## 翻譯 (Translation)

### 觸發器規格

此規格描述基於 Azure Monitor 指標進行擴縮的 `azure-monitor` 觸發器。

```yaml
triggers:
- type: azure-monitor
  metadata:
    resourceURI: Microsoft.ContainerService/managedClusters/azureMonitorCluster
    tenantId: xxx-xxx-xxx-xxx-xxx
    subscriptionId: yyy-yyy-yyy-yyy-yyy
    resourceGroupName: azureMonitor
    metricName: kube_pod_status_ready
    metricFilter: namespace eq 'default'
    metricAggregationInterval: "0:1:0"
    targetValue: "0.5"
    activationTargetValue: "3.5"
    activeDirectoryClientId: <client id value>
    cloud: Private
    azureResourceManagerEndpoint: https://management.azure.airgap.com/
    activeDirectoryEndpoint: https://login.airgap.example/
```

**參數列表：**

- `resourceURI` - Azure 資源的縮短 URI，格式為 `"<resourceProviderNamespace>/<resourceType>/<resourceName>"`。
- `tenantId` - 包含 Azure 資源的租用戶 ID。用於驗證。
- `subscriptionId` - 包含 Azure 資源的 Azure 訂閱 ID。用於確定完整的資源 URI。
- `resourceGroupName` - Azure 資源的資源群組名稱。
- `metricName` - 要查詢的指標名稱。
- `metricNamespace` - 指標命名空間的名稱。當 `metricName` 是自訂指標時必填。
- `targetValue` - 觸發擴縮動作的目標值。（此值可以是浮點數）
- `activationTargetValue` - 啟動擴縮器的目標值。（預設：`0`，可選，此值可以是浮點數）
- `metricAggregationType` - Azure Monitor 指標的聚合方法。選項包括 `Average`、`Total`、`Maximum`。
- `metricFilter` - 使用維度進行更具體篩選的過濾器名稱。（可選）
- `metricAggregationInterval` - 指標的收集時間，格式為 `"hh:mm:ss"`（預設：`"0:5:0"`，可選）
- `cloud` - Azure 資源所屬的雲端環境名稱。（值：`AzurePublicCloud`、`AzureUSGovernmentCloud`、`AzureGermanCloud`、`AzureChinaCloud`、`Private`，預設：`AzurePublicCloud`，可選）

### 驗證參數

您可以使用 `TriggerAuthentication` CRD 透過提供一組 Azure Active Directory 憑證或使用 Pod Identity 來設定驗證。

**基於憑證的驗證：**
- `activeDirectoryClientId` - Active Directory 應用程式的 ID，需要至少 `Monitoring Reader` 權限。
- `activeDirectoryClientPassword` - Active Directory 應用程式的密碼。

**基於 Pod Identity 的驗證：**
可以使用 [Azure AD Workload Identity](https://azure.github.io/azure-workload-identity/docs/) 提供者。

---

## 實作範例 (Practical Example)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: azure-monitor-secrets
data:
  activeDirectoryClientId: <clientId>
  activeDirectoryClientPassword: <clientPassword>
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: azure-monitor-trigger-auth
spec:
  secretTargetRef:
    - parameter: activeDirectoryClientId
      name: azure-monitor-secrets
      key: activeDirectoryClientId
    - parameter: activeDirectoryClientPassword
      name: azure-monitor-secrets
      key: activeDirectoryClientPassword
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: azure-monitor-scaler
spec:
  scaleTargetRef:
    name: azure-monitor-example
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
  - type: azure-monitor
    metadata:
      resourceURI: Microsoft.ContainerService/managedClusters/azureMonitorCluster
      tenantId: xxx-xxx-xxx-xxx-xxx
      subscriptionId: yyy-yyy-yyy-yyy-yyy
      resourceGroupName: azureMonitor
      metricName: pod_custom_metric
      metricNamespace: pod_custom_metrics_namespace
      metricFilter: namespace eq 'default'
      metricAggregationInterval: "0:1:0"
      metricAggregationType: Average
      targetValue: "1"
    authenticationRef:
      name: azure-monitor-trigger-auth
```

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `resourceURI` | Azure 資源 URI | 無（必填）|
| `tenantId` | 租用戶 ID | 無（必填）|
| `subscriptionId` | 訂閱 ID | 無（必填）|
| `resourceGroupName` | 資源群組名稱 | 無（必填）|
| `metricName` | 指標名稱 | 無（必填）|
| `targetValue` | 目標值 | 無（必填）|
| `metricAggregationType` | 聚合類型 | 無 |
| `metricAggregationInterval` | 聚合間隔 | 0:5:0 |

| 雲端類型 | 說明 |
|----------|------|
| `AzurePublicCloud` | Azure 公有雲（預設）|
| `AzureUSGovernmentCloud` | Azure 美國政府雲 |
| `AzureChinaCloud` | Azure 中國雲 |
| `Private` | 私有雲 |
