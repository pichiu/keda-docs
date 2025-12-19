+++
title = "Azure Pipelines"
availability = "v2.3+"
maintainer = "Microsoft"
category = "CI/CD"
description = "根據 Azure Pipelines 的代理程式池佇列擴縮應用程式。"
go_file = "azure_pipelines_scaler"
+++

## TL;DR

Azure Pipelines 擴縮器根據給定代理程式池中待處理的管線執行數量自動擴縮應用程式。適用於擴縮自託管的 Azure DevOps 代理程式。

---

## 翻譯 (Translation)

### 觸發器規格

此規格描述 Azure Pipelines 的 `azure-pipelines` 觸發器。它根據給定代理程式池中待處理的管線執行數量進行擴縮。

```yaml
triggers:
  - type: azure-pipelines
    metadata:
      poolName: "{agentPoolName}"
      poolID: "{agentPoolId}"
      organizationURLFromEnv: "AZP_URL"
      personalAccessTokenFromEnv: "AZP_TOKEN"
      targetPipelinesQueueLength: "1"
      activationTargetPipelinesQueueLength: "5"
      parent: "{parent ADO agent name}"
      demands: "{demands}"
      requireAllDemands: false
    authenticationRef:
     name: pipeline-trigger-auth
```

**參數列表：**

- `poolName` - 池的名稱。（可選，必須設定 `poolID` 或 `poolName` 其中之一）
- `poolID` - 池的 ID。（可選，必須設定 `poolID` 或 `poolName` 其中之一）
- `organizationURLFromEnv` - 您的部署用於取得 Azure DevOps 組織 URL 的環境變數名稱。
- `personalAccessTokenFromEnv` - 提供 Azure DevOps 個人存取令牌（PAT）的環境變數名稱。
- `targetPipelinesQueueLength` - 要擴縮的佇列中待處理作業數量的目標值。（預設：`1`，可選）
- `activationTargetPipelinesQueueLength` - 啟動擴縮器的目標值。（預設：`0`，可選）
- `parent` - 輸入與 ScaledObject 匹配的 ADO 代理程式名稱。
- `demands` - 輸入提供給 ScaledObject 的需求字串。這必須是代理程式實際功能清單的子集。
- `requireAllDemands` - 如果設為 `true`，作業的需求必須與觸發器的需求完全匹配。（預設：`false`）

### 驗證參數

**個人存取令牌驗證：**
- `organizationURL` - Azure DevOps 組織的 URL。
- `personalAccessToken` - Azure DevOps 的個人存取令牌（PAT）。

**Pod Identity 驗證：**
可以使用 [Azure AD Workload Identity](https://azure.github.io/azure-workload-identity/docs/) 提供者。

### 如何確定您的池 ID

有幾種方法可以取得 `poolID`。最簡單的方式是使用 `az cli`，使用指令 `az pipelines pool list --pool-name {agentPoolName} --organization {organizationURL} --query [0].id`。

也可以透過 UI 瀏覽到組織的代理程式池（組織設定 -> 代理程式池 -> `{agentPoolName}`）並從 URL 取得池 ID。

---

## 實作範例 (Practical Example)

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: pipeline-auth
data:
  personalAccessToken: <encoded personalAccessToken>
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: pipeline-trigger-auth
  namespace: default
spec:
  secretTargetRef:
    - parameter: personalAccessToken
      name: pipeline-auth
      key: personalAccessToken
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: azure-pipelines-scaledobject
  namespace: default
spec:
  scaleTargetRef:
    name: azdevops-deployment
  minReplicaCount: 1
  maxReplicaCount: 5
  triggers:
  - type: azure-pipelines
    metadata:
      poolID: "1"
      organizationURLFromEnv: "AZP_URL"
      parent: "example-keda-template"
      demands: "maven,docker"
    authenticationRef:
     name: pipeline-trigger-auth
```

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `poolName` | 代理程式池名稱 | 無 |
| `poolID` | 代理程式池 ID | 無 |
| `targetPipelinesQueueLength` | 目標佇列長度 | 1 |
| `activationTargetPipelinesQueueLength` | 啟動閾值 | 0 |
| `demands` | 需求字串 | 無 |
| `requireAllDemands` | 要求完全匹配需求 | false |

| 驗證方式 | 說明 |
|----------|------|
| Personal Access Token | 使用 Azure DevOps PAT |
| Pod Identity | 使用 Azure AD Workload Identity |
