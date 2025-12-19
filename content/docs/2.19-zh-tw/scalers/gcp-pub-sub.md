+++
title = "Google Cloud Platform Pub/Sub"
availability = "v1.0+"
maintainer = "Community"
category = "Messaging"
description = "根據 Google Cloud Platform Pub/Sub 擴縮應用程式。"
go_file = "gcp_pubsub_scaler"
+++

## TL;DR

GCP Pub/Sub 擴縮器根據 Google Cloud Pub/Sub 訂閱（Subscription）或主題（Topic）的指標來擴縮應用程式。支援多種指標模式，如訂閱大小（SubscriptionSize）和最舊未確認訊息時間（OldestUnackedMessageAge）。支援服務帳戶憑證和 GCP Workload Identity 驗證。

> 💡 **警告：** 此擴縮器已被棄用，將不會收到任何修改。

---

## 翻譯 (Translation)

### 觸發器規格

此規格描述 Google Cloud Platform Pub/Sub 的 `gcp-pubsub` 觸發器。

```yaml
triggers:
- type: gcp-pubsub
  metadata:
    mode: "SubscriptionSize"         # 可選 - 預設 SubscriptionSize。可選：SubscriptionSize 或 OldestUnackedMessageAge
    aggregation: "sum"               # 可選 - 僅對分佈值指標有意義
    value: "5.5"                     # 可選 - 預設 10
    valueIfNull: '0.0'               # 可選 - 預設 ""
    activationValue: "10.5"          # 可選 - 預設 0
    timeHorizon: "1m"                # 可選 - 預設 2m，使用 aggregation 時預設 5m
    # 訂閱或主題名稱選項（二選一）
    subscriptionName: "mysubscription"
    subscriptionNameFromEnv: "MY_SUBSCRIPTION_FROM_ENV"
    topicName: "mytopic"
    topicNameFromEnv: "MY_TOPIC_FROM_ENV"
    credentialsFromEnv: GOOGLE_APPLICATION_CREDENTIALS_JSON  # 必填
```

GCP Pub/Sub 觸發器允許您根據 Pub/Sub 訂閱或主題的任何指標進行擴縮，例如訊息數量或最舊未確認訊息時間等。

**參數列表：**

- `credentialsFromEnv` - 此屬性映射到擴縮目標（`scaleTargetRef`）中包含服務帳戶憑證（JSON）的環境變數名稱。
- `mode` - 用於擴縮工作負載的指標。它是官方指標名稱的 `PascalCase`。例如，如果要使用 `subscription/pull_request_count` 指標，填入 `PullRequestCount`。所有以 `subscription/` 和 `topic/` 開頭的指標都支援。（預設：`SubscriptionSize`，即 `NumUndeliveredMessages`）
- `aggregation` - 用於聚合指標的函數。（值：mean、median、variance、stddev、sum、count、percentileX（X：0 到 100 的整數），對於值類型為 `DISTRIBUTION` 的指標為必填）
- `activationValue` - 啟動擴縮器的目標值。（預設：`0`，可選，可以是浮點數）
- `timeHorizon` - 要檢索指標的時間範圍。（預設：`2m`，使用 aggregation 時預設：`5m`）
- `valueIfNull` - 請求未返回時間序列時返回的值。（預設：`""`，可選，可以是浮點數）
- `subscriptionName` - 定義要監控的訂閱。可使用不同格式：
  - 僅訂閱名稱，將參考當前專案或憑證檔案中指定的專案的訂閱。
  - 使用 Google 提供的完整連結，以便參考另一個專案中的訂閱，例如：`projects/myproject/subscriptions/mysubscription`。
- `topicName` - 定義要監控的主題。類似 `subscriptionName`，可使用不同格式。

### 驗證參數

您可以使用 `TriggerAuthentication` CRD 透過提供 JSON 格式的服務帳戶憑證來設定驗證。

**憑證型驗證：**

- `GoogleApplicationCredentials` - JSON 格式的服務帳戶憑證。

**身分型驗證：**

您也可以使用 `TriggerAuthentication` CRD，使用 Google Cloud 中執行機器的關聯服務帳戶來設定驗證。

---

## 說明 (Explanation)

### 指標模式

| 模式 | 說明 |
|------|------|
| `SubscriptionSize` | 未傳遞的訊息數量（預設）|
| `OldestUnackedMessageAge` | 最舊未確認訊息的時間 |
| `PullRequestCount` | 拉取請求數量 |
| 其他 | 參考 GCP Pub/Sub 指標文件 |

### 驗證方式比較

| 方式 | 說明 | 使用場景 |
|------|------|----------|
| credentialsFromEnv | 服務帳戶 JSON | 非 GKE 環境 |
| GCP Workload Identity | Pod Identity | GKE 生產環境 |

---

## 實作範例 (Practical Example)

```yaml
# 基本範例 - 使用環境變數中的憑證
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: pubsub-scaledobject
spec:
  scaleTargetRef:
    name: my-consumer-deployment
  minReplicaCount: 0
  maxReplicaCount: 20
  triggers:
    - type: gcp-pubsub
      metadata:
        mode: "SubscriptionSize"
        value: "5"
        subscriptionName: "mysubscription"
        credentialsFromEnv: GOOGLE_APPLICATION_CREDENTIALS_JSON
```

```yaml
# 使用 TriggerAuthentication
apiVersion: v1
kind: Secret
metadata:
  name: pubsub-secret
type: Opaque
data:
  GOOGLE_APPLICATION_CREDENTIALS_JSON: <base64-encoded-service-account-json>
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: gcp-pubsub-auth
spec:
  secretTargetRef:
    - parameter: GoogleApplicationCredentials
      name: pubsub-secret
      key: GOOGLE_APPLICATION_CREDENTIALS_JSON
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: pubsub-scaledobject
spec:
  scaleTargetRef:
    name: my-consumer-deployment
  triggers:
    - type: gcp-pubsub
      authenticationRef:
        name: gcp-pubsub-auth
      metadata:
        subscriptionName: "input"
```

```yaml
# 使用 GCP Workload Identity
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: gcp-identity-auth
spec:
  podIdentity:
    provider: gcp
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: pubsub-identity-scaledobject
spec:
  scaleTargetRef:
    name: my-consumer-deployment
  triggers:
    - type: gcp-pubsub
      authenticationRef:
        name: gcp-identity-auth
      metadata:
        subscriptionName: "input"
```

```bash
# 使用 gcloud 查看訂閱狀態
gcloud pubsub subscriptions describe mysubscription

# 查看未傳遞的訊息數量
gcloud pubsub subscriptions seek mysubscription --snapshot=snap1

# 發送測試訊息到主題
gcloud pubsub topics publish mytopic --message="test message"

# 查看 ScaledObject 狀態
kubectl describe scaledobject pubsub-scaledobject
```

---

## 常見錯誤與提示 (Common Mistakes & Tips)

| 常見錯誤 | 正確做法 |
|----------|----------|
| 服務帳戶權限不足 | 確保有 Pub/Sub Subscriber 權限 |
| 憑證 JSON 格式錯誤 | 確認 JSON 正確且已 base64 編碼 |
| 使用已棄用的 subscriptionSize 參數 | 改用 mode 和 value 參數 |
| 訂閱不存在 | 確認訂閱名稱和專案 ID 正確 |

### 提示

- 此擴縮器已被棄用，建議考慮使用其他擴縮方案
- GKE 環境建議使用 GCP Workload Identity
- 對於分佈值指標，必須設定 `aggregation` 參數
- 使用 `valueIfNull` 處理無資料時的情況

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `subscriptionName` | 訂閱名稱 | 無（與 topicName 二選一）|
| `topicName` | 主題名稱 | 無（與 subscriptionName 二選一）|
| `mode` | 指標模式 | SubscriptionSize |
| `value` | 目標值 | 10 |
| `activationValue` | 啟動閾值 | 0 |
| `timeHorizon` | 時間範圍 | 2m |
| `aggregation` | 聚合函數 | 無（分佈值指標必填）|

| 指令 | 說明 |
|------|------|
| `kubectl get scaledobject` | 列出 ScaledObject |
| `gcloud pubsub subscriptions describe <name>` | 查看訂閱詳細資訊 |
| `gcloud pubsub topics publish <topic> --message="..."` | 發送訊息 |
