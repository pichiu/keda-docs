
## TL;DR

KEDA 提供多種驗證方式連接外部事件來源：直接在 ScaledObject 中定義 Secret 參考、使用 TriggerAuthentication（命名空間級別）或 ClusterTriggerAuthentication（叢集級別）。支援 Pod Identity、HashiCorp Vault、Azure Key Vault、AWS Secret Manager、GCP Secret Manager 等多種認證提供者。

---

## 翻譯 (Translation)

擴縮器（Scaler）通常需要驗證或密鑰和設定來檢查事件。

KEDA 提供幾種安全模式來管理驗證流程：

* 在每個 `ScaledObject` 上設定驗證
* 使用 `TriggerAuthentication` 重複使用每個命名空間的憑證或委派驗證
* 使用 `ClusterTriggerAuthentication` 重複使用全域憑證

### 在 ScaledObject 上定義 Secret 和 ConfigMap

某些 metadata 參數不允許從字面值解析，而是需要引用目標容器上定義的 Secret、ConfigMap 或環境變數。

> 💡 ***提示：*** *如果建立的 Deployment YAML 引用了 Secret，請確保在引用它的 Deployment 之前建立 Secret，並在兩者之後建立 ScaledObject，以避免無效引用。*

#### 範例

如果使用 [RabbitMQ 擴縮器](https://keda.sh/docs/2.1/scalers/rabbitmq-queue/)，`host` 參數可能包含密碼，因此需要是一個引用。您可以建立一個包含 `host` 字串值的 Secret，在 Deployment 中引用該 Secret，並將其映射到 `ScaledObject` metadata 參數，如下所示：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {secret-name}
data:
  {secret-key-name}: YW1xcDovL3VzZXI6UEFTU1dPUkRAcmFiYml0bXEuZGVmYXVsdC5zdmMuY2x1c3Rlci5sb2NhbDo1Njcy #根據 Secret 規格進行 base64 編碼
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {deployment-name}
  namespace: default
  labels:
    app: {deployment-name}
spec:
  selector:
    matchLabels:
      app: {deployment-name}
  template:
    metadata:
      labels:
        app: {deployment-name}
    spec:
      containers:
      - name: {deployment-name}
        image: {container-image}
        envFrom:
        - secretRef:
            name: {secret-name}
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: {scaled-object-name}
  namespace: default
spec:
  scaleTargetRef:
    name: {deployment-name}
  triggers:
  - type: rabbitmq
    metadata:
      queueName: hello
      host: {secret-key-name}
      queueLength  : '5'
```

如果 Deployment 中有多個容器，您需要在 `ScaledObject` 中包含具有引用的容器名稱。如果不包含 `envSourceContainerName`，將預設為第一個容器。KEDA 將嘗試從容器的 Secret、ConfigMap 和環境變數解析引用。

#### 缺點

雖然此方法適用於許多場景，但有一些缺點：

* **難以有效地跨 `ScaledObject` 共享驗證**設定
* **不支援直接引用 Secret**，只能引用容器引用的 Secret
* **不支援其他類型的驗證流程**，如 *Pod Identity*，其中可以在沒有 Secret 或連接字串的情況下獲取對來源的存取權限

基於這些和其他原因，我們還提供 `TriggerAuthentication` 資源，將驗證定義為與 `ScaledObject` 分開的資源。這允許您直接引用 Secret、設定使用 Pod Identity 或使用由不同團隊管理的驗證物件。

### 使用 TriggerAuthentication 重複使用憑證和委派驗證

`TriggerAuthentication` 允許您將驗證參數與 `ScaledObject` 和 Deployment 容器分開描述。它還啟用更進階的驗證方法，如「Pod Identity」、驗證重複使用或允許 IT 設定驗證。

```yaml
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: {trigger-authentication-name}
  namespace: default # 必須與 ScaledObject 在同一命名空間
spec:
  podIdentity:
      provider: none | azure-workload | aws | aws-eks | gcp  # 可選。預設：none
      identityId: <identity-id>                               # 可選。僅由 azure 和 azure-workload 提供者使用。
      roleArn: <role-arn>                                     # 可選。僅由 aws 提供者使用。
      identityOwner: keda|workload                            # 可選。僅由 aws 提供者使用。
  secretTargetRef:                                            # 可選。
  - parameter: {scaledObject-parameter-name}                  # 必填。
    name: {secret-name}                                       # 必填。
    key: {secret-key-name}                                    # 必填。
  env:                                                        # 可選。
  - parameter: {scaledObject-parameter-name}                  # 必填。
    name: {env-name}                                          # 必填。
    containerName: {container-name}                           # 可選。預設：ScaledObject 的 scaleTargetRef.envSourceContainerName
  hashiCorpVault:                                             # 可選。
    address: {hashicorp-vault-address}                        # 必填。
    namespace: {hashicorp-vault-namespace}                    # 可選。預設為 root 命名空間。對 Vault Enterprise 有用
    authentication: token | kubernetes                        # 必填。
    role: {hashicorp-vault-role}                              # 可選。
    mount: {hashicorp-vault-mount}                            # 可選。
    credential:                                               # 可選。
      token: {hashicorp-vault-token}                          # 可選。
      serviceAccount: {path-to-service-account-file}          # 可選。
    secrets:                                                  # 必填。
    - parameter: {scaledObject-parameter-name}                # 必填。
      key: {hashicorp-vault-secret-key-name}                  # 必填。
      path: {hashicorp-vault-secret-path}                     # 必填。
  azureKeyVault:                                              # 可選。
    vaultUri: {key-vault-address}                             # 必填。
    podIdentity:                                              # 可選。使用 Pod Identity 時必填。
      provider: azure-workload                                # 必填。
      identityId: <identity-id>                               # 可選
    credentials:                                              # 可選。不使用 Pod Identity 時必填。
      clientId: {azure-ad-client-id}                          # 必填。
      clientSecret:                                           # 必填。
        valueFrom:                                            # 必填。
          secretKeyRef:                                       # 必填。
            name: {k8s-secret-with-azure-ad-secret}           # 必填。
            key: {key-within-the-secret}                      # 必填。
      tenantId: {azure-ad-tenant-id}                          # 必填。
    cloud:                                                    # 可選。
      type: AzurePublicCloud | AzureUSGovernmentCloud | AzureChinaCloud | AzureGermanCloud | Private # 必填。
      keyVaultResourceURL: {key-vault-resource-url-for-cloud}             # type = Private 時必填。
      activeDirectoryEndpoint: {active-directory-endpoint-for-cloud}      # type = Private 時必填。
    secrets:                                                  # 必填。
    - parameter: {param-name-used-for-auth}                   # 必填。
      name: {key-vault-secret-name}                           # 必填。
      version: {key-vault-secret-version}                     # 可選。
  awsSecretManager:
    podIdentity:                                              # 可選。
      provider: aws                                           # 必填。
    credentials:                                              # 可選。
      accessKey:                                              # 必填。
        valueFrom:                                            # 必填。
          secretKeyRef:                                       # 必填。
            name: {k8s-secret-with-aws-credentials}           # 必填。
            key: AWS_ACCESS_KEY_ID                            # 必填。
      accessSecretKey:                                        # 必填。
        valueFrom:                                            # 必填。
          secretKeyRef:                                       # 必填。
            name: {k8s-secret-with-aws-credentials}           # 必填。
            key: AWS_SECRET_ACCESS_KEY                        # 必填。
    region: {aws-region}                                      # 可選。
    secrets:                                                  # 必填。
    - parameter: {param-name-used-for-auth}                   # 必填。
      name: {aws-secret-name}                                 # 必填。
      version: {aws-secret-version}                           # 可選。
      secretKey: {aws-secret-key}                             # 可選。
  gcpSecretManager:                                           # 可選。
    secrets:                                                  # 必填。
      - parameter: {param-name-used-for-auth}                 # 必填。
        id: {secret-manager-secret-name}                      # 必填。
        version: {secret-manager-secret-name}                 # 可選。
    podIdentity:                                              # 可選。
      provider: gcp                                           # 必填。
    credentials:                                              # 可選。
      clientSecret:                                           # 必填。
        valueFrom:                                            # 必填。
          secretKeyRef:                                       # 必填。
            name: {k8s-secret-with-gcp-iam-sa-secret}         # 必填。
            key: {key-within-the-secret}                      # 必填。
```

根據需求，您可以混合和匹配引用類型提供者來設定所有必需的參數。

您在 `TriggerAuthentication` 定義中定義的每個參數不需要包含在 `ScaledObject` 定義的觸發器 `metadata` 中。要從 `ScaledObject` 引用 `TriggerAuthentication`，請將 `authenticationRef` 新增到觸發器。

```yaml
# 某個 Scaled Object
# ...
  triggers:
  - type: {scaler-type}
    metadata:
      param1: {some-value}
    authenticationRef:
      name: {trigger-authentication-name} # 這可能定義了 metadata 中未定義的其他參數
```

### 驗證範圍：命名空間 vs. 叢集

每個 `TriggerAuthentication` 定義在一個命名空間中，只能被同一命名空間中的 `ScaledObject` 使用。對於想要在多個命名空間中的擴縮器之間共享單一組憑證的情況，您可以改為建立 `ClusterTriggerAuthentication`。作為全域物件，這可以從任何命名空間使用。要將觸發器設定為使用 `ClusterTriggerAuthentication`，請在驗證引用中新增 `kind` 欄位：

```yaml
    authenticationRef:
      name: {cluster-trigger-authentication-name}
      kind: ClusterTriggerAuthentication
```

預設情況下，從 `secretTargetRef` 載入的 Secret 必須與 KEDA 部署的命名空間相同（通常是 `keda`）。這可以透過為 `keda-operator` 容器設定 `KEDA_CLUSTER_OBJECT_NAMESPACE` 環境變數來覆蓋。

定義 `ClusterTriggerAuthentication` 與 `TriggerAuthentication` 幾乎相同，只是沒有 `metadata.namespace` 值：

```yaml
apiVersion: keda.sh/v1alpha1
kind: ClusterTriggerAuthentication
metadata:
  name: {cluster-trigger-authentication-name}
spec:
  # 與之前相同...
```

### 驗證參數

驗證參數可以從許多來源提取。所有這些值會合併在一起形成擴縮器的驗證資料。您可以在[這裡](./../authentication-providers/)找到所有可用的驗證。

#### 環境變數

您可以透過提供給定 `containerName` 的變數 `name` 來提取一個或多個環境變數的資訊。

```yaml
env:                              # 可選。
  - parameter: region             # 必填 - 由擴縮觸發器定義
    name: my-env-var              # 必填。
    containerName: my-container   # 可選。預設：ScaledObject 的 scaleTargetRef.envSourceContainerName
```

**假設：** `containerName` 與 ScaledObject 中 `scaleTargetRef.name` 引用的資源相同，除非另有指定。

#### Secret

您可以透過定義 Kubernetes Secret 的 `name` 和要使用的 `key` 來將一個或多個 Secret 提取到觸發器中。

```yaml
secretTargetRef:                          # 可選。
  - parameter: connectionString           # 必填 - 由擴縮觸發器定義
    name: my-keda-secret-entity           # 必填。
    key: azure-storage-connectionstring   # 必填。
```

**假設：** `namespace` 與 ScaledObject 中 `scaleTargetRef.name` 引用的資源相同，除非另有指定。

#### 綁定服務帳戶令牌

您可以透過定義 Kubernetes 服務帳戶的 `serviceAccountName` 來將一個或多個服務帳戶令牌提取到觸發器中。

```yaml
boundServiceAccountToken:                   # 可選。
  - parameter: connectionString             # 必填 - 由擴縮觸發器定義
    serviceAccountName: my-service-account  # 必填。
```

**假設：** `namespace` 與 ScaledObject 中 `scaleTargetRef.name` 引用的資源相同，除非另有指定。

#### HashiCorp Vault Secret

您可以透過定義驗證 metadata（如 Vault `address` 和 `authentication` 方法（token | kubernetes））將一個或多個 HashiCorp Vault Secret 提取到觸發器中。如果選擇 kubernetes 驗證方法，您還應該提供 `role` 和 `mount`。
`credential` 根據驗證方法定義 HashiCorp Vault 憑證，對於 kubernetes，您應該提供服務帳戶令牌的路徑（預設值為 `/var/run/secrets/kubernetes.io/serviceaccount/token`），對於 token 驗證方法，請提供令牌。
`secrets` 列表定義了 Vault 中 Secret 的路徑和鍵到參數的映射。
`namespace` 可用於指向給定的 Vault Enterprise 命名空間。

```yaml
hashiCorpVault:                                     # 可選。
  address: {hashicorp-vault-address}                # 必填。
  namespace: {hashicorp-vault-namespace}            # 可選。預設為 root 命名空間。對 Vault Enterprise 有用
  authentication: token | kubernetes                # 必填。
  role: {hashicorp-vault-role}                      # 可選。
  mount: {hashicorp-vault-mount}                    # 可選。
  credential:                                       # 可選。
    token: {hashicorp-vault-token}                  # 可選。
    serviceAccount: {path-to-service-account-file}  # 可選。預設為 /var/run/secrets/kubernetes.io/serviceaccount/token
  secrets:                                          # 必填。
  - parameter: {scaledObject-parameter-name}        # 必填。
    key: {hashicorp-vault-secret-key-name}          # 必填。
    path: {hashicorp-vault-secret-path}             # 必填。
```

#### Azure Key Vault Secret

您可以使用 `azureKeyVault` 鍵將 Azure Key Vault 的 Secret 提取到觸發器中。

`secrets` 列表定義了 Key Vault Secret 和驗證參數之間的映射。

您可以透過在 `TriggerAuthentication` / `ClusterTriggerAuthentication` 定義中指定 `azure` 或 `azure-workload` Pod Identity 提供者來使用 Pod Identity 驗證到 Key Vault。Pod Identity 綁定需要在 keda 命名空間中套用。

如果您不想使用 Pod Identity 提供者，您需要向 Azure Active Directory 註冊一個[應用程式](https://docs.microsoft.com/en-us/azure/active-directory/develop/app-objects-and-service-principals)並指定其憑證。應用程式的 `clientId` 和 `tenantId` 應作為規格的一部分提供。應用程式的 `clientSecret` 應該在與驗證資源相同命名空間的 Kubernetes Secret 中。

確保已在 Azure Key Vault 上授予受管身分識別 / Azure AD 應用程式「讀取 Secret」權限。在 Azure Key Vault [文件](https://docs.microsoft.com/en-us/azure/key-vault/general/assign-access-policy?tabs=azure-portal)中了解更多。

`cloud` 參數可用於指定除「Azure 公用雲」之外的雲端環境，如已知的 Azure 雲端（如「Azure 中國雲」等），甚至 Azure Stack Hub 或隔離雲端。

#### GCP Secret Manager Secret

您可以使用 `gcpSecretManager` 鍵將 GCP Secret Manager 的 Secret 提取到觸發器中。

`secrets` 列表定義了 Secret 和驗證參數之間的映射。

GCP IAM 服務帳戶憑證可用於與 Secret Manager 服務進行驗證，可以使用 Kubernetes Secret 提供。或者，在 `gcpSecretManager` 中使用 `podIdentity` 也支援 GCP Secret Manager 的 `gcp` Pod Identity 提供者。

#### Pod 驗證提供者

多個服務提供者允許您為 Pod 指派身分。透過使用該身分，您可以將驗證委派給 Pod 和服務提供者，而不是設定 Secret。

目前我們支援以下提供者：

```yaml
podIdentity:
  provider: none | azure-workload | aws | aws-eks | gcp       # 可選。預設：none
  identityId: <identity-id>                                   # 可選。僅由 azure 和 azure-workload 提供者使用。
  roleArn: <role-arn>                                         # 可選。僅由 aws 提供者使用。
  identityOwner: keda|workload                                # 可選。僅由 aws 提供者使用。
```

##### Azure Workload Identity

[**Azure AD Workload Identity**](https://github.com/Azure/azure-workload-identity) 是 [**Azure AD Pod Identity**](https://github.com/Azure/aad-pod-identity) 的較新版本。它讓您的 Kubernetes 工作負載可以使用 [**Azure AD 應用程式**](https://docs.microsoft.com/en-us/azure/active-directory/develop/app-objects-and-service-principals) 存取 Azure 資源而無需指定 Secret，使用[聯合身分憑證](https://azure.github.io/azure-workload-identity/docs/topics/federated-identity-credential.html) - *不用管理 Secret，讓 Azure AD 來處理困難的工作*。

您可以透過 `podIdentity.provider` 告訴 KEDA 使用 Azure AD Workload Identity。

```yaml
podIdentity:
  provider: azure-workload  # 可選。預設：none
  identityId: <identity-id> # 可選。預設：從服務帳戶 annotation 取得的 ClientId。
```

Azure AD Workload Identity 將授予具有適當標籤和 annotation 的服務帳戶的 Pod 存取權限。請參閱這些[文件](https://azure.github.io/azure-workload-identity/docs/topics/service-account-labels-and-annotations.html)以取得更多資訊。您可以在 KEDA Operator 服務帳戶上設定這些標籤和 annotation。這可以在使用 Helm 部署時透過以下旗標為您完成：

1. `--set podIdentity.azureWorkload.enabled=true`
2. `--set podIdentity.azureWorkload.clientId={azure-ad-client-id}`
3. `--set podIdentity.azureWorkload.tenantId={azure-ad-tenant-id}`

將 `podIdentity.azureWorkload.enabled` 設定為 `true` 是 Workload Identity 驗證正常運作所必需的。為了讓 KEDA 存取提供的用戶端 ID，必須在目標受管身分識別 / Azure AD 應用程式上設定聯合憑證。請參閱這些[文件](https://azure.github.io/azure-workload-identity/docs/topics/federated-identity-credential.html)。聯合憑證應使用此主體（如果 KEDA 安裝在 `keda` 命名空間中）：`system:serviceaccount:keda:keda-operator`。

您可以透過在 `podIdentity` 欄位下指定 `identityId` 參數來覆蓋安裝期間指派給 KEDA 的身分。這允許終端使用者使用不同的身分來存取各種資源，這比使用單一身分存取多個資源更安全。在覆蓋的情況下，應為每個使用的身分設定聯合憑證。

##### AWS Secret Manager

您可以透過在 KEDA 擴縮規格中設定 `awsSecretManager` 鍵來將 AWS Secret Manager Secret 整合到您的觸發器中。

`podIdentity` 區段設定 AWS Pod Identity 的使用，提供者設定為 AWS。

`credentials` 區段指定 AWS 憑證，包括 `accessKey` 和 `secretAccessKey`。

`region` 參數是可選的，表示 Secret 所在的 AWS 區域，如果未指定則預設為預設區域。

`awsSecretManager` 中的 `secrets` 列表定義了 AWS Secret Manager Secret 和應用程式中使用的驗證參數之間的映射，包括參數名稱、AWS Secret Manager Secret 名稱、可選的 Secret 鍵參數和可選的版本參數，如果未指定則預設為最新版本。

##### AWS Pod Identity Webhook for AWS

[**AWS IAM Roles for Service Accounts (IRSA) Pod Identity Webhook**](https://github.com/aws/amazon-eks-pod-identity-webhook)（[文件](https://aws.amazon.com/blogs/opensource/introducing-fine-grained-iam-roles-service-accounts/)）允許您使用與 Pod 關聯的服務帳戶上的 annotation 來提供角色名稱。

您可以透過 `podIdentity.provider` 告訴 KEDA 使用 EKS Pod Identity Webhook。

```yaml
podIdentity:
  provider: aws      # 可選。預設：none
  roleArn: <role-arn> # 可選。
  identityOwner: keda|workload # 可選。
```

##### AWS EKS Pod Identity Webhook

> [已棄用：這將在 KEDA v3 中移除](https://github.com/kedacore/keda/discussions/5343)

[**EKS Pod Identity Webhook**](https://github.com/aws/amazon-eks-pod-identity-webhook)，在[這裡](https://aws.amazon.com/blogs/opensource/introducing-fine-grained-iam-roles-service-accounts/)有更深入的描述，允許您使用與 Pod 關聯的服務帳戶上的 annotation 來提供角色名稱。

您可以透過 `podIdentity.provider` 告訴 KEDA 使用 EKS Pod Identity Webhook。

```yaml
podIdentity:
  provider: aws-eks # 可選。預設：none
```

---

## 說明 (Explanation)

### 驗證方式比較

| 方式 | 範圍 | 適用場景 |
|------|------|----------|
| 在 ScaledObject 中定義 | 單一 ScaledObject | 簡單場景，不需共享 |
| TriggerAuthentication | 單一命名空間 | 命名空間內多個 ScaledObject 共享 |
| ClusterTriggerAuthentication | 整個叢集 | 跨命名空間共享 |

### 認證提供者類型

| 提供者 | 說明 |
|--------|------|
| `secretTargetRef` | 直接引用 Kubernetes Secret |
| `env` | 從容器環境變數取得 |
| `podIdentity` | 使用雲端提供者的 Pod Identity |
| `hashiCorpVault` | 從 HashiCorp Vault 取得 |
| `azureKeyVault` | 從 Azure Key Vault 取得 |
| `awsSecretManager` | 從 AWS Secret Manager 取得 |
| `gcpSecretManager` | 從 GCP Secret Manager 取得 |

---

## 實作範例 (Practical Example)

```yaml
# 建立 Kubernetes Secret
apiVersion: v1
kind: Secret
metadata:
  name: rabbitmq-credentials
  namespace: default
type: Opaque
data:
  host: YW1xcDovL3VzZXI6cGFzc3dvcmRAcmFiYml0bXE6NTY3Mg==  # base64 編碼

---
# 建立 TriggerAuthentication
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: rabbitmq-auth
  namespace: default
spec:
  secretTargetRef:
    - parameter: host           # 擴縮器的參數名稱
      name: rabbitmq-credentials # Kubernetes Secret 名稱
      key: host                  # Secret 中的鍵名

---
# 在 ScaledObject 中引用
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: rabbitmq-scaledobject
  namespace: default
spec:
  scaleTargetRef:
    name: my-deployment
  triggers:
    - type: rabbitmq
      metadata:
        queueName: my-queue
        queueLength: "10"
      authenticationRef:
        name: rabbitmq-auth      # 引用上面建立的 TriggerAuthentication
```

```bash
# 查看 TriggerAuthentication
kubectl get triggerauthentication             # 列出命名空間級別的 TriggerAuth
kubectl get clustertriggerauthentication      # 列出叢集級別的 TriggerAuth

# 建立 Secret（從字串）
kubectl create secret generic my-secret \
  --from-literal=password=mysecretpassword

# 建立 Secret（從檔案）
kubectl create secret generic my-secret \
  --from-file=credentials.json

# 查看 Secret（解碼）
kubectl get secret my-secret -o jsonpath='{.data.password}' | base64 -d

# 驗證 TriggerAuthentication 是否正確設定
kubectl describe triggerauthentication rabbitmq-auth
```

---

## 常見錯誤與提示 (Common Mistakes & Tips)

| 常見錯誤 | 正確做法 |
|----------|----------|
| TriggerAuthentication 和 ScaledObject 在不同命名空間 | 確保兩者在同一命名空間，或使用 ClusterTriggerAuthentication |
| Secret 的值未進行 base64 編碼 | Kubernetes Secret 的 data 欄位需要 base64 編碼 |
| 忘記在觸發器中新增 authenticationRef | 每個需要認證的觸發器都要引用 TriggerAuthentication |
| Pod Identity 設定不完整 | 確保服務帳戶有正確的 annotation 和標籤 |

### 提示

- 使用 `kubectl create secret generic` 可以自動進行 base64 編碼
- ClusterTriggerAuthentication 的 Secret 預設需要在 `keda` 命名空間
- 可以在一個 TriggerAuthentication 中混合使用多種認證提供者
- 對於生產環境，建議使用 Pod Identity 或 Secret Manager 而非直接存儲憑證

---

## 快速參考 (Quick Reference)

| 資源類型 | 說明 | 範圍 |
|----------|------|------|
| `TriggerAuthentication` | 命名空間級別的認證資源 | 單一命名空間 |
| `ClusterTriggerAuthentication` | 叢集級別的認證資源 | 整個叢集 |

| Pod Identity 提供者 | 說明 |
|---------------------|------|
| `none` | 不使用 Pod Identity（預設）|
| `azure-workload` | Azure Workload Identity |
| `aws` | AWS IRSA Pod Identity |
| `aws-eks` | AWS EKS Pod Identity（已棄用）|
| `gcp` | GCP Workload Identity |

| 指令 | 說明 |
|------|------|
| `kubectl get triggerauthentication` | 列出 TriggerAuthentication |
| `kubectl get clustertriggerauthentication` | 列出 ClusterTriggerAuthentication |
| `kubectl describe triggerauthentication <name>` | 查看詳細資訊 |
| `kubectl create secret generic <name> --from-literal=key=value` | 建立 Secret |
