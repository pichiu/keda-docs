
## TL;DR

Azure Storage Queue 擴縮器根據 Azure 儲存體佇列中的訊息數量來擴縮應用程式。支援連接字串和 Pod Identity（Azure AD Workload Identity）兩種驗證方式。可選擇計算所有訊息或僅可見訊息。

---

## 翻譯 (Translation)

### 觸發器規格

此規格描述 Azure Storage Queue 的 `azure-queue` 觸發器。

```yaml
triggers:
- type: azure-queue
  metadata:
    queueName: orders
    queueLength: '5'
    queueLengthStrategy: all|visibleonly
    activationQueueLength: '50'
    connectionFromEnv: STORAGE_CONNECTIONSTRING_ENV_NAME
    accountName: storage-account-name
    cloud: AzureUSGovernmentCloud
```

**參數列表：**

- `queueName` - 佇列名稱。
- `queueLength` - 傳遞給擴縮器的佇列長度目標值。範例：如果一個 Pod 可以處理 10 個訊息，將佇列長度目標設為 10。如果佇列中的實際訊息是 30，擴縮器會擴展到 3 個 Pod。（預設：`5`，可選）
- `queueLengthStrategy` - `all` 會計算可見和不可見訊息，而 `visibleonly` 使用 Peek 只計算可見訊息。在 `visibleonly` 模式下，如果訊息數量為 32 或更高，會回退到預設的 `all` 策略。（預設：`all`，可選）
- `activationQueueLength` - 啟動擴縮器的目標值。（預設：`0`，可選）
- `connectionFromEnv` - 部署中用於取得連接字串的環境變數名稱。
- `accountName` - 佇列所屬的儲存體帳戶名稱。
- `cloud` - 佇列所屬的雲端環境名稱。（有效值：`AzurePublicCloud`、`AzureUSGovernmentCloud`、`AzureChinaCloud`、`AzureGermanCloud`、`Private`；預設：`AzurePublicCloud`）

當 `cloud` 設為 `Private` 時，需要 `endpointSuffix` 參數。否則，它會根據雲端環境自動產生。`endpointSuffix` 代表佇列所屬雲端環境的儲存體佇列端點後綴，例如 `AzurePublicCloud` 的 `queue.core.windows.net`。

### 驗證參數

可使用 Pod Identity 或連接字串進行驗證。

**連接字串驗證：**

- `connection` - Azure 儲存體帳戶的連接字串。

**Pod Identity 驗證：**

可使用 [Azure AD Workload Identity](https://azure.github.io/azure-workload-identity/docs/) 提供者。

---

## 說明 (Explanation)

### 佇列長度策略比較

| 策略 | 說明 | 使用場景 |
|------|------|----------|
| `all` | 計算可見和不可見訊息 | 需要準確總數（預設）|
| `visibleonly` | 只計算可見訊息 | 排除處理中的訊息 |

> 注意：`visibleonly` 策略在訊息數量 >= 32 時會回退到 `all` 策略。

### 雲端環境

| 環境 | 說明 |
|------|------|
| `AzurePublicCloud` | Azure 公用雲端（預設）|
| `AzureUSGovernmentCloud` | Azure 美國政府雲端 |
| `AzureChinaCloud` | Azure 中國雲端 |
| `Private` | Azure Stack Hub 或隔離雲端 |

---

## 實作範例 (Practical Example)

```yaml
# 使用 Pod Identity（Azure AD Workload Identity）
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: azure-queue-auth
spec:
  podIdentity:
    provider: azure-workload
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: azure-queue-scaledobject
spec:
  scaleTargetRef:
    name: my-processor-deployment
  minReplicaCount: 0
  maxReplicaCount: 20
  triggers:
    - type: azure-queue
      metadata:
        queueName: my-queue
        accountName: mystorageaccount
        queueLength: "5"
        queueLengthStrategy: "all"
      authenticationRef:
        name: azure-queue-auth
```

```yaml
# 使用連接字串
apiVersion: v1
kind: Secret
metadata:
  name: storage-secret
data:
  connection-string: <base64-encoded-connection-string>
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: azure-queue-auth
spec:
  secretTargetRef:
    - parameter: connection
      name: storage-secret
      key: connection-string
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: azure-queue-scaledobject
spec:
  scaleTargetRef:
    name: my-processor-deployment
  triggers:
    - type: azure-queue
      metadata:
        queueName: orders
        queueLength: "10"
      authenticationRef:
        name: azure-queue-auth
```

```yaml
# 使用 Azure Stack Hub（Private Cloud）
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: azure-queue-private
spec:
  scaleTargetRef:
    name: my-processor-deployment
  triggers:
    - type: azure-queue
      metadata:
        queueName: my-queue
        queueLength: "5"
        cloud: Private
        endpointSuffix: queue.local.azurestack.external
        accountName: mystorageaccount
      authenticationRef:
        name: azure-queue-auth
```

```bash
# 使用 Azure CLI 查看佇列訊息數量
az storage queue show \
  --name my-queue \
  --account-name mystorageaccount \
  --query "approximateMessageCount"

# 新增訊息到佇列
az storage message put \
  --queue-name my-queue \
  --account-name mystorageaccount \
  --content "test message"

# 查看佇列中的訊息
az storage message peek \
  --queue-name my-queue \
  --account-name mystorageaccount

# 查看 ScaledObject 狀態
kubectl describe scaledobject azure-queue-scaledobject
```

---

## 常見錯誤與提示 (Common Mistakes & Tips)

| 常見錯誤 | 正確做法 |
|----------|----------|
| 連接字串格式錯誤 | 確認使用完整的 Azure 儲存體連接字串 |
| 使用 Pod Identity 但未設定 accountName | Pod Identity 驗證需要 `accountName` 參數 |
| visibleonly 策略結果不準確 | 當訊息 >= 32 時會回退到 all 策略 |
| 佇列不存在 | 確認佇列名稱和儲存體帳戶正確 |

### 提示

- 生產環境建議使用 Azure AD Workload Identity 而非連接字串
- 使用 `queueLengthStrategy: visibleonly` 可以排除正在處理中的訊息
- 設定 `activationQueueLength` 可避免少量訊息就觸發擴縮
- 對於 Azure Stack Hub，記得設定正確的 `endpointSuffix`

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `queueName` | 佇列名稱 | 無（必填）|
| `queueLength` | 佇列長度閾值 | 5 |
| `queueLengthStrategy` | 計算策略 | all |
| `activationQueueLength` | 啟動閾值 | 0 |
| `accountName` | 儲存體帳戶名稱 | 無（Pod Identity 時必填）|
| `cloud` | 雲端環境 | AzurePublicCloud |

| 指令 | 說明 |
|------|------|
| `kubectl get scaledobject` | 列出 ScaledObject |
| `az storage queue show` | 查看佇列資訊 |
| `az storage message put` | 新增訊息 |
| `az storage message peek` | 查看訊息 |
