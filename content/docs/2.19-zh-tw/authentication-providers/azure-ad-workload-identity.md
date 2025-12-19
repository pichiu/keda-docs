+++
title = "Azure AD Workload Identity"
+++

## TL;DR

Azure AD Workload Identity 是 Azure AD Pod Identity 的新版本。它讓 Kubernetes 工作負載使用 Azure AD 應用程式或 Azure 受控識別（Managed Identity）存取 Azure 資源，無需指定 Secret，使用聯合身分憑證。

---

## 翻譯 (Translation)

[**Azure AD Workload Identity**](https://github.com/Azure/azure-workload-identity) 是 [**Azure AD Pod Identity**](https://github.com/Azure/aad-pod-identity) 的新版本。它讓您的 Kubernetes 工作負載使用 [**Azure AD 應用程式**](https://docs.microsoft.com/en-us/azure/active-directory/develop/app-objects-and-service-principals)或 [**Azure 受控識別**](https://docs.microsoft.com/azure/active-directory/managed-identities-azure-resources/overview)存取 Azure 資源，無需指定 Secret，使用[聯合身分憑證](https://azure.github.io/azure-workload-identity/docs/topics/federated-identity-credential.html) - *讓 Azure AD 處理複雜的工作，不用管理 Secret*。

您可以透過 `podIdentity.provider` 告訴 KEDA 使用 Azure AD Workload Identity。

```yaml
podIdentity:
  provider: azure-workload                # 可選。預設：none
  identityId: <identity-id>               # 可選。預設：服務帳戶註解中的 ClientId
  identityTenantId: <tenant-id>           # 可選。預設：服務帳戶註解中的 TenantId
  identityAuthorityHost: <authority-host> # 可選。預設：azure-wi-webhook-controller-manager 注入的 AZURE_AUTHORITY_HOST 環境變數
```

Azure AD Workload Identity 會授予具有適當標籤和註解的服務帳戶的 Pod 存取權限。您可以在 KEDA Operator 服務帳戶上設定這些標籤和註解。使用 Helm 部署時可以透過以下參數設定：

1. `--set podIdentity.azureWorkload.enabled=true`
2. `--set podIdentity.azureWorkload.clientId={azure-ad-client-id}`
3. `--set podIdentity.azureWorkload.tenantId={azure-ad-tenant-id}`

您可以透過在 `podIdentity` 欄位下指定 `identityId` 參數來覆蓋安裝期間指派給 KEDA 的身分識別。這允許使用者使用不同的身分識別存取各種資源，比使用單一身分識別存取多個資源更安全。

此外，可能需要 Azure Workload Identity 跨租戶和/或雲端進行驗證（例如 AzureCloud、AzureChinaCloud、AzureUSGovernment、AzureGermanCloud）。若要對同一雲端中的不同租戶進行驗證，可以在 `podIdentity` 欄位下指定 `identityTenantId` 參數。若要對不同雲端中的租戶進行驗證，必須同時指定 `identityTenantId` 和 `identityAuthorityHost` 參數。

---

## 說明 (Explanation)

### 聯合和覆蓋的考量

#### 案例 1

假設您有一個 KEDA 身分識別可以存取 ServiceBus A、B 和 C。另外，您有各種工作負載的個別身分識別：

- KEDA 的身分識別可存取 ServiceBus A、B 和 C（安裝時設定且未覆蓋）
- 工作負載 A 的身分識別可存取 Service Bus A
- 工作負載 B 的身分識別可存取 Service Bus B
- 工作負載 C 的身分識別可存取 Service Bus C

在此情況下，KEDA 的受控服務身分識別只需要與 KEDA 的服務帳戶聯合。

#### 案例 2

為避免授予 KEDA 身分識別過多權限，您有一個無法存取任何 Service Bus 的 KEDA 身分識別（可能是不相關的，如 Key Vault）：

- KEDA 的身分識別無法存取任何 Service Bus
- 工作負載 A 的身分識別可存取 Service Bus A
- 工作負載 B 的身分識別可存取 Service Bus B
- 工作負載 C 的身分識別可存取 Service Bus C

在此情況下，您透過「TriggerAuthentication」選項（`.spec.podIdentity.identityId`）覆蓋安裝期間設定的預設身分識別。每個「ScaledObject」現在使用自己的「TriggerAuthentication」，每個都指定覆蓋。因此，您不需要在 KEDA 的身分識別上堆疊過多權限。但是，在此情況下，KEDA 的服務帳戶必須與它可能嘗試承擔的所有身分識別聯合。

#### 案例 3

類似前一個情況，但您有不同租戶中工作負載的個別身分識別：

- KEDA 的身分識別無法存取任何 Service Bus
- 工作負載 A 的身分識別可存取租戶 A 中的 Service Bus A
- 工作負載 B 的身分識別可存取租戶 B 中的 Service Bus B

在此情況下，您透過「TriggerAuthentication」選項覆蓋安裝期間設定的預設身分識別和租戶（`.spec.podIdentity.identityId` 和 `.spec.podIdentity.identityTenantId`）。重要的是，在此情況下，KEDA 的服務帳戶必須與每個租戶中的所有身分識別聯合。

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `provider` | Pod Identity 提供者 | none |
| `identityId` | Azure AD Client ID | 服務帳戶註解 |
| `identityTenantId` | Azure AD Tenant ID | 服務帳戶註解 |
| `identityAuthorityHost` | Authority Host | 環境變數 |

| Helm 參數 | 說明 |
|-----------|------|
| `podIdentity.azureWorkload.enabled` | 啟用 Azure Workload Identity |
| `podIdentity.azureWorkload.clientId` | Azure AD Client ID |
| `podIdentity.azureWorkload.tenantId` | Azure AD Tenant ID |
