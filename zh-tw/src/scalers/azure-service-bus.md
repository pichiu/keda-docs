
## TL;DR

Azure Service Bus 擴縮器根據 Azure Service Bus 佇列（Queue）或主題（Topic）中的訊息數量來擴縮應用程式。支援連接字串和 Pod Identity（Azure AD Workload Identity）兩種驗證方式。可使用正則表達式批量監控多個佇列或訂閱。

---

## 翻譯 (Translation)

> ⚠️ **警告：** KEDA 不負責管理實體。如果佇列、主題或訂閱不存在，它不會自動建立。

### 觸發器規格

此規格描述 Azure Service Bus 佇列或主題的 `azure-servicebus` 觸發器。

```yaml
triggers:
- type: azure-servicebus
  metadata:
    # 必填：queueName 或 topicName 搭配 subscriptionName
    queueName: functions-sbqueue
    # 或
    topicName: functions-sbtopic
    subscriptionName: sbtopic-sub1
    # 可選，使用 Pod Identity 時為必填
    namespace: service-bus-namespace
    # 可選，也可使用 TriggerAuthentication
    connectionFromEnv: SERVICEBUS_CONNECTIONSTRING_ENV_NAME
    # 可選
    messageCount: "5"            # 觸發擴縮的訊息數量閾值。預設：5
    activationMessageCount: "2"  # 啟動閾值
    cloud: Private               # 可選。預設：AzurePublicCloud
    endpointSuffix: servicebus.airgap.example  # cloud=Private 時必填
```

**參數列表：**

- `messageCount` - Azure Service Bus 佇列或主題中觸發擴縮的活動訊息數量。
- `activationMessageCount` - 啟動擴縮器的目標值。（預設：`0`，可選）
- `queueName` - 要擴縮的 Azure Service Bus 佇列名稱。（可選）
- `topicName` - 要擴縮的 Azure Service Bus 主題名稱。（可選）
- `subscriptionName` - 要擴縮的訂閱名稱。（指定 `topicName` 時為必填）
- `namespace` - 包含佇列或主題的 Azure Service Bus 命名空間名稱。（使用 Pod Identity 時為必填）
- `connectionFromEnv` - 部署中用於取得連接字串的環境變數名稱。（可選）
- `useRegex` - 指示是否在 `queueName` 或 `subscriptionName` 參數中使用正則表達式。（值：`true`、`false`，預設：`false`，可選）
- `operation` - 當 `useRegex` 設為 `true` 時，定義如何計算訊息數量。（值：`sum`、`max` 或 `avg`，預設：`sum`，可選）
- `cloud` - Service Bus 所屬的雲端環境名稱。（有效值：`AzurePublicCloud`、`AzureUSGovernmentCloud`、`AzureChinaCloud`、`AzureGermanCloud`、`Private`；預設：`AzurePublicCloud`）

當 `cloud` 設為 `Private` 時，需要 `endpointSuffix` 參數，例如 `servicebus.usgovcloudapi.net`。

> 💡 **注意：** Service Bus 共用存取原則需要是 `Manage` 類型。KEDA 需要 Manage 權限才能從 Service Bus 取得指標。

### 驗證參數

可使用 Pod Identity 或連接字串進行驗證。

**連接字串驗證：**

- `connection` - Azure Service Bus 命名空間的連接字串。

  支援以下格式：

  - **SharedAccessKey** -
    `Endpoint=sb://<sb>.servicebus.windows.net/;SharedAccessKeyName=<key name>;SharedAccessKey=<key value>`
  - **SharedAccessSignature** -
    `Endpoint=sb://<sb>.servicebus.windows.net/;SharedAccessSignature=SharedAccessSignature sig=<signature-string>&se=<expiry>&skn=<keyName>&sr=<URL-encoded-resourceURI>`

**Pod Identity 驗證：**

可使用 [Azure AD Workload Identity](https://azure.github.io/azure-workload-identity/docs/) 提供者。

---

## 說明 (Explanation)

### 佇列 vs 主題/訂閱

| 類型 | 使用場景 | 設定參數 |
|------|----------|----------|
| 佇列 | 點對點通訊 | `queueName` |
| 主題/訂閱 | 發布/訂閱模式 | `topicName` + `subscriptionName` |

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
  name: azure-servicebus-auth
spec:
  podIdentity:
    provider: azure-workload
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: azure-servicebus-scaledobject
spec:
  scaleTargetRef:
    name: my-processor-deployment
  minReplicaCount: 0
  maxReplicaCount: 20
  triggers:
    - type: azure-servicebus
      metadata:
        queueName: my-queue
        namespace: my-servicebus-namespace
        messageCount: "5"
      authenticationRef:
        name: azure-servicebus-auth
```

```yaml
# 使用連接字串
apiVersion: v1
kind: Secret
metadata:
  name: servicebus-secret
data:
  connection-string: <base64-encoded-connection-string>
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: azure-servicebus-auth
spec:
  secretTargetRef:
    - parameter: connection
      name: servicebus-secret
      key: connection-string
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: azure-servicebus-scaledobject
spec:
  scaleTargetRef:
    name: my-processor-deployment
  triggers:
    - type: azure-servicebus
      metadata:
        topicName: my-topic
        subscriptionName: my-subscription
        messageCount: "10"
      authenticationRef:
        name: azure-servicebus-auth
```

```bash
# 使用 Azure CLI 查看佇列訊息數量
az servicebus queue show \
  --resource-group my-rg \
  --namespace-name my-namespace \
  --name my-queue \
  --query "countDetails.activeMessageCount"

# 查看 ScaledObject 狀態
kubectl describe scaledobject azure-servicebus-scaledobject
```

---

## 常見錯誤與提示 (Common Mistakes & Tips)

| 常見錯誤 | 正確做法 |
|----------|----------|
| 共用存取原則權限不足 | 確保使用 `Manage` 權限的原則 |
| 使用 Pod Identity 但未設定 namespace | Pod Identity 驗證需要 `namespace` 參數 |
| 佇列不存在導致錯誤 | KEDA 不會自動建立佇列，需先手動建立 |
| 節流導致無法取得指標 | 升級 SKU 或增加 pollingInterval |

### 提示

- 生產環境建議使用 Azure AD Workload Identity 而非連接字串
- 如遇到 `no CountDetails element` 錯誤，可能是 Azure Service Bus 節流，考慮升級 SKU
- 使用 `useRegex: true` 可以監控多個符合模式的佇列
- 設定 `activationMessageCount` 可避免少量訊息就觸發擴縮

### 疑難排解

如果 KEDA 日誌顯示 `invalid queue runtime properties: no CountDetails element` 錯誤，通常是 Azure Service Bus 節流造成的。考慮：
- 將 Azure Service Bus 命名空間升級到更高 SKU 或使用 Premium
- 增加 ScaledObject/ScaledJob 的 pollingInterval
- 使用[指標快取](../concepts/scaling-deployments/#caching-metrics)

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `queueName` | 佇列名稱 | 無（與 topicName 二選一）|
| `topicName` | 主題名稱 | 無（與 queueName 二選一）|
| `subscriptionName` | 訂閱名稱 | 無（topicName 時必填）|
| `namespace` | Service Bus 命名空間 | 無（Pod Identity 時必填）|
| `messageCount` | 訊息數量閾值 | 5 |
| `activationMessageCount` | 啟動閾值 | 0 |
| `cloud` | 雲端環境 | AzurePublicCloud |

| 指令 | 說明 |
|------|------|
| `kubectl get scaledobject` | 列出 ScaledObject |
| `az servicebus queue show --query countDetails` | 查看佇列訊息統計 |
| `az servicebus topic subscription show` | 查看訂閱資訊 |
