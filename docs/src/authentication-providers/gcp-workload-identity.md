
## TL;DR

GCP Workload Identity 允許 GKE 叢集中的工作負載模擬 IAM 服務帳戶以存取 Google Cloud 服務。使用此功能需要在 GKE 叢集上啟用 Workload Identity，並正確設定服務帳戶之間的繫結。

---

## 翻譯 (Translation)

[**GCP Workload Identity**](https://cloud.google.com/kubernetes-engine/docs/concepts/workload-identity) 允許 GKE 叢集中的工作負載模擬 Identity and Access Management (IAM) 服務帳戶以存取 Google Cloud 服務。

您可以透過 `podIdentity.provider` 告訴 KEDA 使用 GCP Workload Identity。

```yaml
podIdentity:
  provider: gcp # 可選。預設：none
```

### 設定 Workload Identity 的步驟

如果您使用 `gcp` 作為 podIdentity 提供者，需要按照以下步驟設定 Workload Identity，且您的 GKE 叢集必須啟用 Workload Identity。

1. **建立 GCP IAM 服務帳戶**（具有適當權限以檢索特定擴縮器的指標）：

```shell
gcloud iam service-accounts create GSA_NAME \
  --project=GSA_PROJECT
```

替換：
- `GSA_NAME`：新 IAM 服務帳戶的名稱
- `GSA_PROJECT`：IAM 服務帳戶的 Google Cloud 專案 ID

2. **確保 IAM 服務帳戶具有所需的角色**：

```shell
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member "serviceAccount:GSA_NAME@GSA_PROJECT.iam.gserviceaccount.com" \
  --role "ROLE_NAME"
```

替換：
- `PROJECT_ID`：您的 Google Cloud 專案 ID
- `GSA_NAME`：IAM 服務帳戶的名稱
- `GSA_PROJECT`：IAM 服務帳戶的 Google Cloud 專案 ID
- `ROLE_NAME`：要指派給服務帳戶的 IAM 角色，如 `roles/monitoring.viewer`

3. **允許 Kubernetes 服務帳戶模擬 IAM 服務帳戶**：

```shell
gcloud iam service-accounts add-iam-policy-binding GSA_NAME@GSA_PROJECT.iam.gserviceaccount.com \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:PROJECT_ID.svc.id.goog[NAMESPACE/KSA_NAME]"
```

替換：
- `PROJECT_ID`：您的 Google Cloud 專案 ID
- `GSA_NAME`：IAM 服務帳戶的名稱
- `GSA_PROJECT`：IAM 服務帳戶的 Google Cloud 專案 ID
- `NAMESPACE`：KEDA Operator 安裝的命名空間；預設為 `keda`
- `KSA_NAME`：KEDA 的 Kubernetes 服務帳戶名稱；預設為 `keda-operator`

4. **使用 IAM 服務帳戶的電子郵件地址註解 Kubernetes 服務帳戶**：

```shell
kubectl annotate serviceaccount keda-operator \
  --namespace keda \
  iam.gke.io/gcp-service-account=GSA_NAME@GSA_PROJECT.iam.gserviceaccount.com
```

替換：
- `GSA_NAME`：IAM 服務帳戶的名稱
- `GSA_PROJECT`：IAM 服務帳戶的 Google Cloud 專案 ID

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `provider` | Pod Identity 提供者 | none |

| 設定步驟 | 說明 |
|----------|------|
| 1. 建立 IAM 服務帳戶 | `gcloud iam service-accounts create` |
| 2. 授予角色 | `gcloud projects add-iam-policy-binding` |
| 3. 設定 Workload Identity 繫結 | `gcloud iam service-accounts add-iam-policy-binding` |
| 4. 註解 K8s 服務帳戶 | `kubectl annotate serviceaccount` |
