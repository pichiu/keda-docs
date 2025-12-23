
## TL;DR

AWS SQS 擴縮器根據 SQS 佇列中的訊息數量來擴縮應用程式。預設計算公式為「可見訊息 + 處理中訊息」，可透過參數調整。支援 Pod Identity、IAM 角色和 IAM 使用者等多種驗證方式。

---

## 翻譯 (Translation)

### 觸發器規格

此規格描述根據 AWS SQS 佇列擴縮的 `aws-sqs-queue` 觸發器。

```yaml
triggers:
- type: aws-sqs-queue
  metadata:
    # 必填：queueURL 或 queueURLFromEnv。如兩者都提供，使用 queueURL
    queueURL: https://sqs.eu-west-1.amazonaws.com/account_id/QueueName
    queueURLFromEnv: QUEUE_URL # 可選。可用來替代 `queueURL` 參數
    queueLength: "5"  # 預設："5"
    # 必填：awsRegion
    awsRegion: "eu-west-1"
    # 可選：awsEndpoint
    awsEndpoint: ""
```

**參數列表：**

- `queueURL` - SQS 佇列的完整 URL。如果沒有歧義，可以使用佇列的短名稱。（可選。`queueURL` 和 `queueURLFromEnv` 只需其一。如兩者都提供，使用 `queueURL`。）
- `queueURLFromEnv` - 從擴縮目標上讀取佇列 URL 的環境變數名稱。（可選。`queueURL` 和 `queueURLFromEnv` 只需其一。）
- `queueLength` - 傳遞給擴縮器的佇列長度目標值。範例：如果一個 Pod 可以處理 10 個訊息，將佇列長度目標設為 10。如果 SQS 佇列中的實際訊息是 30，擴縮器會擴展到 3 個 Pod。（預設：5）
- `activationQueueLength` - 啟動擴縮器的目標值。（預設：`0`，可選）
- `scaleOnInFlight` - 計算 SQS 訊息數量時是否包含處理中的訊息。（預設：true，可選）
- `scaleOnDelayed` - 計算 SQS 訊息數量時是否包含延遲的訊息。（預設：false，可選）
- `awsRegion` - SQS 佇列的 AWS 區域。
- `awsEndpoint` - 覆蓋預設 AWS 端點的端點 URL。（預設：`""`，可選）

> 對於擴縮目的，預設的「實際訊息」公式等於 `ApproximateNumberOfMessages` + `ApproximateNumberOfMessagesNotVisible`，因為在 SQS 術語中 `NotVisible` 意味著訊息仍在處理中。如果您只想根據 `ApproximateNumberOfMessages` 擴縮，請將 `scaleOnInFlight` 設為 `false`。

### 驗證參數

您可以使用 `TriggerAuthentication` CRD 透過提供角色 ARN 或一組 IAM 憑證來設定驗證。

**Pod Identity 驗證：**
- `podIdentity.provider` - 需要在 `TriggerAuthentication` 中設定，且 Pod/服務帳戶必須為您的 Pod Identity 提供者正確設定。

**角色型驗證：**
- `awsRoleArn` - Amazon Resource Names (ARNs) 唯一識別 AWS 資源。

**憑證型驗證：**
- `awsAccessKeyID` - 使用者 ID。
- `awsSecretAccessKey` - 使用者的驗證存取金鑰。
- `awsSessionToken` - 工作階段令牌，僅在使用臨時憑證時需要。

使用者需要有權限從指定的 AWS SQS 佇列讀取屬性。

---

## 說明 (Explanation)

### 訊息計數公式

| 設定 | 公式 |
|------|------|
| 預設 | `ApproximateNumberOfMessages` + `ApproximateNumberOfMessagesNotVisible` |
| `scaleOnInFlight: false` | `ApproximateNumberOfMessages` |
| `scaleOnDelayed: true` | 上述 + `ApproximateNumberOfMessagesDelayed` |

### 驗證方式比較

| 方式 | 安全性 | 設定複雜度 | 適用場景 |
|------|--------|------------|----------|
| Pod Identity | 高 | 中 | EKS 生產環境 |
| IAM 角色 | 高 | 中 | 跨帳戶存取 |
| IAM 使用者 | 中 | 低 | 開發/測試 |

### 擴縮計算範例

假設設定：
- `queueLength: 10`
- 佇列中有 35 個訊息

副本數 = ceil(35 / 10) = 4 個 Pod

---

## 實作範例 (Practical Example)

```yaml
# 使用 Pod Identity 驗證
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: aws-sqs-auth
spec:
  podIdentity:
    provider: aws
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: aws-sqs-scaledobject
spec:
  scaleTargetRef:
    name: my-processor-deployment
  minReplicaCount: 0
  maxReplicaCount: 20
  triggers:
    - type: aws-sqs-queue
      authenticationRef:
        name: aws-sqs-auth
      metadata:
        queueURL: https://sqs.ap-northeast-1.amazonaws.com/123456789/my-queue
        queueLength: "10"
        awsRegion: "ap-northeast-1"
```

```yaml
# 使用 IAM 使用者憑證
apiVersion: v1
kind: Secret
metadata:
  name: aws-secrets
data:
  AWS_ACCESS_KEY_ID: <base64-encoded-access-key>
  AWS_SECRET_ACCESS_KEY: <base64-encoded-secret-key>
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: aws-sqs-auth
spec:
  secretTargetRef:
    - parameter: awsAccessKeyID
      name: aws-secrets
      key: AWS_ACCESS_KEY_ID
    - parameter: awsSecretAccessKey
      name: aws-secrets
      key: AWS_SECRET_ACCESS_KEY
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: aws-sqs-scaledobject
spec:
  scaleTargetRef:
    name: my-processor-deployment
  minReplicaCount: 0
  triggers:
    - type: aws-sqs-queue
      authenticationRef:
        name: aws-sqs-auth
      metadata:
        queueURL: my-queue             # 可使用短名稱
        queueLength: "5"
        awsRegion: "ap-northeast-1"
```

```yaml
# 只計算可見訊息（排除處理中訊息）
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: aws-sqs-visible-only
spec:
  scaleTargetRef:
    name: my-processor-deployment
  triggers:
    - type: aws-sqs-queue
      authenticationRef:
        name: aws-sqs-auth
      metadata:
        queueURL: my-queue
        queueLength: "5"
        awsRegion: "ap-northeast-1"
        scaleOnInFlight: "false"       # 只計算可見訊息
```

```bash
# 查看 SQS 佇列狀態
aws sqs get-queue-attributes \
  --queue-url https://sqs.ap-northeast-1.amazonaws.com/123456789/my-queue \
  --attribute-names ApproximateNumberOfMessages ApproximateNumberOfMessagesNotVisible

# 發送測試訊息
aws sqs send-message \
  --queue-url https://sqs.ap-northeast-1.amazonaws.com/123456789/my-queue \
  --message-body "test message"

# 查看 ScaledObject 狀態
kubectl describe scaledobject aws-sqs-scaledobject
```

---

## 常見錯誤與提示 (Common Mistakes & Tips)

| 常見錯誤 | 正確做法 |
|----------|----------|
| 未設定 AWS 區域 | `awsRegion` 是必填參數 |
| IAM 權限不足 | 確保有 `sqs:GetQueueAttributes` 權限 |
| 處理中訊息導致過度擴縮 | 設定 `scaleOnInFlight: false` |
| 使用完整 URL 但拼錯 | 使用短名稱更簡潔且不易出錯 |

### 提示

- 生產環境建議使用 Pod Identity 而非 IAM 使用者憑證
- 考慮設定 `activationQueueLength` 避免少量訊息就觸發擴縮
- 使用 `scaleOnInFlight: false` 可避免因長時間處理訊息導致的過度擴縮
- 可使用 `awsEndpoint` 覆蓋端點以連接 LocalStack 等本地服務

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `queueURL` | SQS 佇列 URL | 必填（或使用 queueURLFromEnv）|
| `queueLength` | 每個 Pod 處理的目標訊息數 | 5 |
| `activationQueueLength` | 啟動閾值 | 0 |
| `awsRegion` | AWS 區域 | 必填 |
| `scaleOnInFlight` | 計算處理中訊息 | true |
| `scaleOnDelayed` | 計算延遲訊息 | false |

| AWS CLI 指令 | 說明 |
|--------------|------|
| `aws sqs get-queue-attributes --queue-url <url> --attribute-names All` | 查看佇列屬性 |
| `aws sqs send-message --queue-url <url> --message-body "<msg>"` | 發送訊息 |
| `aws sqs receive-message --queue-url <url>` | 接收訊息 |
| `aws sqs purge-queue --queue-url <url>` | 清空佇列 |

| 指令 | 說明 |
|------|------|
| `kubectl get scaledobject` | 列出 ScaledObject |
| `kubectl describe scaledobject <name>` | 查看詳細狀態 |
