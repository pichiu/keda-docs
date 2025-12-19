+++
title = "Azure Key Vault Secret"
+++

## TL;DR

Azure Key Vault 驗證提供者允許您從 Azure Key Vault 中拉取 Secret 到 KEDA 觸發器中。支援服務主體驗證和 Azure Workload Identity。

---

## 翻譯 (Translation)

您可以使用 `azureKeyVault` 金鑰將 Azure Key Vault 中的 Secret 拉取到觸發器中。

`secrets` 清單定義 Key Vault Secret 與驗證參數之間的對應。

目前，Azure Key Vault 使用 `azureKeyVault` 內的 `podIdentity` 支援 `azure` 和 `azure-workload` Pod Identity 提供者。

也支援服務主體驗證，需要在 Azure Active Directory 中[註冊應用程式](https://docs.microsoft.com/en-us/azure/active-directory/develop/app-objects-and-service-principals)並指定其憑證。應用程式的 `clientId` 和 `tenantId` 需作為規格的一部分提供。應用程式的 `clientSecret` 預期存放在與驗證資源相同命名空間的 Kubernetes Secret 中。

請確保已在 Azure Key Vault 上授予 Azure AD 應用程式「讀取 Secret」權限。在 Azure Key Vault [文件](https://docs.microsoft.com/en-us/azure/key-vault/general/assign-access-policy?tabs=azure-portal)中了解更多。

`cloud` 參數可用於指定除了 `Azure Public Cloud` 之外的雲端環境，例如已知的 Azure 雲端如 `Azure China Cloud` 等，甚至 Azure Stack Hub 或隔離雲端。

```yaml
azureKeyVault:                                          # 可選。
  vaultUri: {key-vault-address}                         # 必填。
  podIdentity:                                          # 可選。
    provider: azure-workload                            # 必填。
    identityId: <identity-id>                           # 可選。
  credentials:                                          # 可選。
    clientId: {azure-ad-client-id}                      # 必填。
    clientSecret:                                       # 必填。
      valueFrom:                                        # 必填。
        secretKeyRef:                                   # 必填。
          name: {k8s-secret-with-azure-ad-secret}       # 必填。
          key: {key-within-the-secret}                  # 必填。
    tenantId: {azure-ad-tenant-id}                      # 必填。
  cloud:                                                # 可選。
    type: AzurePublicCloud | AzureUSGovernmentCloud | AzureChinaCloud | AzureGermanCloud | Private # 必填。
    keyVaultResourceURL: {key-vault-resource-url-for-cloud}           # Private 類型時必填。
    activeDirectoryEndpoint: {active-directory-endpoint-for-cloud}    # Private 類型時必填。
  secrets:                                              # 必填。
  - parameter: {param-name-used-for-auth}               # 必填。
    name: {key-vault-secret-name}                       # 必填。
    version: {key-vault-secret-version}                 # 可選。
```

---

## 實作範例 (Practical Example)

```yaml
# 使用服務主體的 TriggerAuthentication
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: azure-keyvault-auth
  namespace: default
spec:
  azureKeyVault:
    vaultUri: https://my-keyvault.vault.azure.net/
    credentials:
      clientId: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
      clientSecret:
        valueFrom:
          secretKeyRef:
            name: azure-sp-secret
            key: clientSecret
      tenantId: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
    secrets:
      - parameter: connectionString
        name: my-connection-string
        version: ""  # 使用最新版本

---
# 使用 Azure Workload Identity 的 TriggerAuthentication
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: azure-keyvault-workload-auth
  namespace: default
spec:
  azureKeyVault:
    vaultUri: https://my-keyvault.vault.azure.net/
    podIdentity:
      provider: azure-workload
    secrets:
      - parameter: connectionString
        name: my-connection-string
```

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `vaultUri` | Key Vault URI | 無（必填）|
| `podIdentity.provider` | Pod Identity 提供者 | 無（azure 或 azure-workload）|
| `podIdentity.identityId` | 受控識別 ID | 無 |
| `credentials.clientId` | Azure AD 用戶端 ID | 無 |
| `credentials.clientSecret` | Azure AD 用戶端密碼 | 無 |
| `credentials.tenantId` | Azure AD 租用戶 ID | 無 |
| `secrets[].parameter` | 驗證參數名稱 | 無（必填）|
| `secrets[].name` | Key Vault Secret 名稱 | 無（必填）|
| `secrets[].version` | Secret 版本 | 最新版本 |

| 雲端類型 | 說明 |
|----------|------|
| `AzurePublicCloud` | Azure 公有雲（預設）|
| `AzureUSGovernmentCloud` | Azure 美國政府雲 |
| `AzureChinaCloud` | Azure 中國雲 |
| `AzureGermanCloud` | Azure 德國雲 |
| `Private` | 私有雲（需額外設定）|

| 驗證方式 | 說明 |
|----------|------|
| Pod Identity | 使用 Azure Workload Identity |
| 服務主體 | 使用 Azure AD 應用程式憑證 |
