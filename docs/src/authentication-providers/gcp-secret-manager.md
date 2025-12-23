
## TL;DR

GCP Secret Manager 允許您將 Google Cloud Secret Manager 中的 Secret 拉取到 KEDA 觸發器中。支援透過 GCP Workload Identity 或服務帳戶金鑰進行驗證。

---

## 翻譯 (Translation)

您可以使用 `gcpSecretManager` 金鑰將 GCP Secret Manager 中的 Secret 拉取到觸發器中。

`secrets` 清單定義 Secret 與驗證參數之間的對應。

GCP IAM 服務帳戶憑證可用於與 Secret Manager 服務進行驗證，可透過 Kubernetes Secret 提供。或者，也支援在 `gcpSecretManager` 中使用 `podIdentity` 的 `gcp` Pod Identity 提供者。

```yaml
gcpSecretManager:                                     # 可選。
  secrets:                                            # 必填。
    - parameter: {param-name-used-for-auth}           # 必填。
      id: {secret-manager-secret-name}                # 必填。
      version: {secret-manager-secret-name}           # 可選。
  podIdentity:                                        # 可選。
    provider: gcp                                     # 必填。
  credentials:                                        # 可選。
    clientSecret:                                     # 必填。
      valueFrom:                                      # 必填。
        secretKeyRef:                                 # 必填。
          name: {k8s-secret-with-gcp-iam-sa-secret}   # 必填。
          key: {key-within-the-secret}                # 必填。
```

### 建立 IAM 服務帳戶 Kubernetes Secret 的步驟

1. **建立新的 GCP IAM 服務帳戶**（如果要使用現有的服務帳戶，可以跳過此步驟）：

```shell
gcloud iam service-accounts create GSA_NAME \
  --project=GSA_PROJECT
```

替換：
- `GSA_NAME`：新 IAM 服務帳戶的名稱
- `GSA_PROJECT`：IAM 服務帳戶的 Google Cloud 專案 ID

2. **確保 IAM 服務帳戶具有足夠的[角色](https://cloud.google.com/iam/docs/understanding-roles)和[權限](https://cloud.google.com/iam/docs/permissions-reference)來擷取 Secret**，例如 [Secret Manager Secret Accessor](https://cloud.google.com/secret-manager/docs/access-control#secretmanager.secretAccessor)。您可以使用以下指令授予額外的角色：

```shell
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member "serviceAccount:GSA_NAME@GSA_PROJECT.iam.gserviceaccount.com" \
  --role "ROLE_NAME"
```

替換：
- `PROJECT_ID`：您的 Google Cloud 專案 ID
- `GSA_NAME`：IAM 服務帳戶的名稱
- `GSA_PROJECT`：IAM 服務帳戶的 Google Cloud 專案 ID
- `ROLE_NAME`：要指派給服務帳戶的 IAM 角色，如 `roles/secretmanager.secretaccessor`

3. **設定 [GCP Workload Identity](./gcp-workload-identity) 或建立用於服務帳戶驗證的 JSON 金鑰憑證**：

```shell
gcloud iam service-accounts keys create KEY_FILE \
  --iam-account=GSA_NAME@PROJECT_ID.iam.gserviceaccount.com
```

替換：
- `KEY_FILE`：本機上私鑰的新輸出檔案路徑
- `GSA_NAME`：IAM 服務帳戶的名稱
- `PROJECT_ID`：您的 Google Cloud 專案 ID

4. **在與建立 `TriggerAuthentication` 資源相同的命名空間中建立儲存 SA 金鑰檔案的 Kubernetes Secret**：

```shell
kubectl create secret generic NAME --from-file=KEY=KEY_FILE -n NAMESPACE
```

替換：
- `NAME`：Kubernetes Secret 資源的名稱
- `KEY`：SA 的 Kubernetes Secret 金鑰
- `KEY_FILE`：本機上 SA 檔案的路徑
- `NAMESPACE`：將建立 `TriggerAuthentication` 資源的命名空間

現在您可以建立參照 Secret 名稱和 SA 金鑰的 `TriggerAuthentication` 資源。

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `secrets[].parameter` | 驗證參數名稱 | 無（必填）|
| `secrets[].id` | Secret Manager 的 Secret 名稱 | 無（必填）|
| `secrets[].version` | Secret 版本 | 最新版本 |
| `podIdentity.provider` | Pod Identity 提供者 | 無（設為 `gcp`）|
| `credentials.clientSecret` | 服務帳戶金鑰 | 無 |

| 驗證方式 | 說明 |
|----------|------|
| Pod Identity | 使用 GCP Workload Identity |
| 服務帳戶金鑰 | 使用 Kubernetes Secret 中的 GCP 服務帳戶 JSON 金鑰 |

| 指令 | 說明 |
|------|------|
| `gcloud iam service-accounts create` | 建立 GCP 服務帳戶 |
| `gcloud projects add-iam-policy-binding` | 授予 IAM 角色 |
| `gcloud iam service-accounts keys create` | 建立服務帳戶金鑰 |
| `kubectl create secret generic` | 建立 Kubernetes Secret |
