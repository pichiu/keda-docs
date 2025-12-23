
## TL;DR

Azure Event Hubs 擴縮器根據未處理的事件數量自動擴縮應用程式。支援連線字串和 Pod Identity 驗證，需要 Azure Storage 來儲存檢查點。

---

## 翻譯 (Translation)

### 觸發器規格

此規格描述 Azure Event Hubs 的 `azure-eventhub` 觸發器。

```yaml
triggers:
- type: azure-eventhub
  metadata:
    connectionFromEnv: EVENTHUB_CONNECTIONSTRING_ENV_NAME
    storageConnectionFromEnv: STORAGE_CONNECTIONSTRING_ENV_NAME
    consumerGroup: $Default
    unprocessedEventThreshold: '64'
    activationUnprocessedEventThreshold: '10'
    blobContainer: 'name_of_container'
    # 可選（預設：AzurePublicCloud）
    cloud: Private
    # cloud = Private 時必填
    endpointSuffix: servicebus.airgap.example
    # cloud = Private 時必填
    storageEndpointSuffix: airgap.example
    # 使用 Pod Identity 驗證 Blob Storage 時必填
    storageAccountName: 'name_of_account'
```

**參數列表：**

- `connectionFromEnv` - 部署用來取得連線字串的環境變數名稱，需附加 `EntityPath=<event_hub_name>`。如果連線字串不以 `EntityPath=<event_hub_name>` 結尾，則必須使用 `eventHubName` / `eventHubNameFromEnv` 參數來提供 Event Hub 名稱。
- `storageConnectionFromEnv` - 提供 Azure 儲存體帳戶連線字串的環境變數名稱，用於儲存檢查點。目前 Event Hub 擴縮器只從 Azure Blob Storage 讀取。（不使用 Pod Identity 時必填）
- `consumerGroup` - Azure Event Hub 消費者的消費者群組。（預設：`$default`，可選）
- `unprocessedEventThreshold` - 觸發擴縮動作的平均目標值。（預設：`64`，可選）
- `activationUnprocessedEventThreshold` - 啟動擴縮器的目標值。在[這裡](./../concepts/scaling-deployments.md#activating-and-scaling-thresholds)了解更多。（預設：`0`，可選）
- `blobContainer` - 儲存檢查點的容器名稱。除了 `AzureFunction` 之外，每個 `checkpointStrategy` 都需要此參數。使用 Azure Functions 時，`blobContainer` 是自動產生的，無法覆寫。
- `eventHubNamespace` - 包含 Event Hub 的 Event Hub 命名空間名稱。（可選）
- `eventHubName` - 包含訊息的 Event Hub 名稱。（可選）
- `storageAccountName` - 用於檢查點的 Blob 儲存體帳戶名稱。（未指定 `storageConnectionFromEnv` 時必填）
- `checkpointStrategy` - 設定不同 Event Hub SDK 的檢查點行為。（值：`azureFunction`、`blobMetadata`、`goSdk`、`dapr`，預設：`""`，可選）
- `cloud` - Event Hub 所屬的雲端環境名稱。（值：`AzurePublicCloud`、`AzureUSGovernmentCloud`、`AzureChinaCloud`、`AzureGermanCloud`、`Private`，預設：`AzurePublicCloud`，可選）

### 驗證參數

您可以使用 Pod Identity 或連線字串驗證進行驗證。

**連線字串驗證：**

- `connection` - Azure Event Hubs 命名空間的連線字串。

**基於 Pod Identity 的驗證：**

可以使用 [Azure AD Workload Identity](https://azure.github.io/azure-workload-identity/docs/) 提供者。

```yaml
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: nameOfTriggerAuth
  namespace: default
spec:
  podIdentity:
    provider: azure-workload
```

### 檢查點行為

- **舊版行為：** 較舊的實作基於 `EventProcessorHost` 用戶端，將檢查點資訊儲存為 Storage Blob 的內容。
- **目前行為：** 較新的實作基於 `EventProcessorClient`，將檢查點資訊儲存為 Storage Blob 的中繼資料。

---

## 實作範例 (Practical Example)

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: azure-eventhub-scaledobject
  namespace: default
spec:
  scaleTargetRef:
    name: azureeventhub-function
  triggers:
  - type: azure-eventhub
    metadata:
      storageConnectionFromEnv: AzureWebJobsStorage
      connectionFromEnv: EventHub
      eventHubNamespace: AzureEventHubNameSpace
      eventHubName: NameOfTheEventHub
      consumerGroup: $Default
      unprocessedEventThreshold: '64'
      blobContainer: ehcontainer
```

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `connectionFromEnv` | Event Hub 連線字串環境變數 | 無 |
| `storageConnectionFromEnv` | Storage 連線字串環境變數 | 無 |
| `consumerGroup` | 消費者群組 | $Default |
| `unprocessedEventThreshold` | 未處理事件閾值 | 64 |
| `blobContainer` | Blob 容器名稱 | 無 |
| `checkpointStrategy` | 檢查點策略 | "" |

| 檢查點策略 | 說明 |
|------------|------|
| `azureFunction` | Azure Functions 和 WebJobs SDK |
| `blobMetadata` | 較新的 C#、Python、Java、JavaScript SDK |
| `goSdk` | Golang SDK |
| `dapr` | 較舊的 Dapr 實作 |
