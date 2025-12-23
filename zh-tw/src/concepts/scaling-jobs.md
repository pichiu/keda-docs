
## TL;DR

ScaledJob 用於擴縮 Kubernetes Job，適合長時間執行的批次任務。每個事件觸發一個獨立的 Job，處理完成後自動終止。相比 ScaledObject，ScaledJob 可以確保每個訊息被完整處理，不會因 HPA 縮減而被中斷。

---

## 翻譯 (Translation)

本頁描述 KEDA 的 Job 擴縮行為。詳細的設定方式請參閱 [ScaledJob 規格](../reference/scaledjob-spec.md)。

### 概述

作為[將事件驅動程式碼擴縮為 Deployment](./scaling-deployments) 的替代方案，您也可以將程式碼作為 Kubernetes Job 執行和擴縮。考慮此選項的主要原因是處理長時間執行的作業。與在 Deployment 中處理多個事件不同，對於每個偵測到的事件，會排程一個單獨的 Kubernetes Job。該 Job 將初始化、從訊息來源提取單一事件、處理完成並終止。

例如，如果您想使用 KEDA 為落在 RabbitMQ 佇列上的每個訊息執行一個 Job，流程可能如下：

1. 當沒有訊息等待處理時，不會建立任何 Job。
2. 當訊息到達佇列時，KEDA 建立一個 Job。
3. 當 Job 開始執行時，它提取*單一*訊息並處理完成。
4. 隨著額外訊息到達，會建立額外的 Job。每個 Job 處理單一訊息直到完成。
5. 定期透過 `SuccessfulJobsHistoryLimit` 和 `FailedJobsHistoryLimit` 移除已完成/失敗的 Job。

### 暫停自動擴縮

指示 KEDA 暫停物件的自動擴縮可能很有用，例如進行叢集維護或透過移除非關鍵任務工作負載來避免資源耗盡。

這比刪除資源更好，因為它可以從操作中移除正在執行的實例而不觸碰應用程式本身。準備好後，您可以重新啟用擴縮。

您可以透過在 `ScaledJob` 定義中新增此 annotation 來暫停自動擴縮：

```yaml
metadata:
  annotations:
    autoscaling.keda.sh/paused: true
```

要重新啟用自動擴縮，請從 `ScaledJob` 定義中移除 annotation 或將值設定為 `false`。

```yaml
metadata:
  annotations:
    autoscaling.keda.sh/paused: false
```

### 範例

以下是使用 RabbitMQ 擴縮器自動擴縮 Job 的範例設定。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: rabbitmq-consumer
data:
  RabbitMqHost: <omitted>
---
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: rabbitmq-consumer
  namespace: default
spec:
  jobTargetRef:
    template:
      spec:
        containers:
        - name: demo-rabbitmq-client
          image: demo-rabbitmq-client:1
          imagePullPolicy: Always
          command: ["receive",  "amqp://user:PASSWORD@rabbitmq.default.svc.cluster.local:5672"]
          envFrom:
            - secretRef:
                name: rabbitmq-consumer-secrets
        restartPolicy: Never
    backoffLimit: 4
  pollingInterval: 10             # 可選。預設：30 秒
  maxReplicaCount: 30             # 可選。預設：100
  successfulJobsHistoryLimit: 3   # 可選。預設：100。保留多少個已完成的 Job。
  failedJobsHistoryLimit: 2       # 可選。預設：100。保留多少個失敗的 Job。
  scalingStrategy:
    strategy: "custom"                        # 可選。預設：default。使用哪種擴縮策略。
    customScalingQueueLengthDeduction: 1      # 可選。用於優化自訂擴縮策略的參數。
    customScalingRunningJobPercentage: "0.5"  # 可選。用於優化自訂擴縮策略的參數。
  triggers:
  - type: rabbitmq
    metadata:
      queueName: hello
      host: RabbitMqHost
      queueLength  : '5'
```

### 排除標籤不傳播到 Job

您可以使用 `scaledjob.keda.sh/job-excluded-labels` annotation 排除特定標籤不傳播到產生的 Job 物件。此 annotation 接受應排除的標籤鍵的逗號分隔列表。

```yaml
metadata:
  annotations:
    scaledjob.keda.sh/job-excluded-labels: "foo.bar/environment,foo.bar/version"
  labels:
    team: backend
    foo.bar/environment: bf5011472247b67cce3ee7b24c9a08c5
    foo.bar/version: "1"
```

---

## 說明 (Explanation)

### ScaledObject vs ScaledJob

| 特性 | ScaledObject | ScaledJob |
|------|--------------|-----------|
| 目標資源 | Deployment, StatefulSet | Job |
| 處理模式 | 多個事件由持續運行的 Pod 處理 | 每個事件一個獨立的 Job |
| 長時間任務 | 可能被 HPA 中斷 | 保證完成 |
| 資源使用 | 持續佔用（除非縮到 0）| 按需建立，完成後釋放 |
| 適用場景 | Web 服務、持續處理 | 批次處理、長時間任務 |

### 擴縮策略（Scaling Strategy）

| 策略 | 說明 |
|------|------|
| `default` | 標準擴縮，根據佇列長度建立 Job |
| `custom` | 自訂擴縮，考慮正在執行的 Job 數量 |
| `accurate` | 精確擴縮，更準確地計算需要的 Job 數量 |

### Job 歷史記錄管理

- `successfulJobsHistoryLimit`：保留多少個成功的 Job（預設 100）
- `failedJobsHistoryLimit`：保留多少個失敗的 Job（預設 100）

建議設定較小的值以避免大量歷史 Job 佔用 etcd 儲存空間。

---

## 實作範例 (Practical Example)

```yaml
# 完整的 ScaledJob 範例 - 處理 AWS SQS 訊息
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: sqs-processor
  namespace: default
spec:
  jobTargetRef:
    parallelism: 1                           # 每個 Job 的平行度
    completions: 1                           # 需要完成的次數
    backoffLimit: 3                          # 失敗重試次數
    template:
      spec:
        containers:
        - name: processor
          image: my-processor:latest
          env:
            - name: QUEUE_URL
              value: "https://sqs.region.amazonaws.com/account/queue"
        restartPolicy: Never
  pollingInterval: 15                        # 每 15 秒檢查一次佇列
  minReplicaCount: 0                         # 沒有訊息時不建立 Job
  maxReplicaCount: 50                        # 最多同時執行 50 個 Job
  successfulJobsHistoryLimit: 5              # 保留 5 個成功的 Job
  failedJobsHistoryLimit: 3                  # 保留 3 個失敗的 Job
  scalingStrategy:
    strategy: "accurate"                     # 使用精確擴縮策略
  triggers:
    - type: aws-sqs-queue
      metadata:
        queueURL: https://sqs.region.amazonaws.com/account/queue
        queueLength: "5"                     # 每 5 個訊息建立一個 Job
        awsRegion: "ap-northeast-1"
      authenticationRef:
        name: aws-credentials
```

```bash
# 查看 ScaledJob 狀態
kubectl get scaledjob                        # 列出所有 ScaledJob

# 查看由 ScaledJob 建立的 Job
kubectl get jobs -l scaledjob.keda.sh/name=sqs-processor

# 查看正在執行的 Job Pod
kubectl get pods -l job-name -w

# 暫停 ScaledJob
kubectl annotate scaledjob sqs-processor \
  autoscaling.keda.sh/paused="true"

# 恢復 ScaledJob
kubectl annotate scaledjob sqs-processor \
  autoscaling.keda.sh/paused-

# 手動清理舊的 Job
kubectl delete jobs -l scaledjob.keda.sh/name=sqs-processor \
  --field-selector status.successful=1
```

---

## 常見錯誤與提示 (Common Mistakes & Tips)

| 常見錯誤 | 正確做法 |
|----------|----------|
| 不設定 `successfulJobsHistoryLimit` | 設定較小的值（如 5-10）避免堆積 |
| Job 失敗後無限重試 | 設定適當的 `backoffLimit` |
| `restartPolicy` 設為 `Always` | ScaledJob 的 Job 應使用 `Never` 或 `OnFailure` |
| 不了解 `scalingStrategy` 的差異 | 根據使用場景選擇適當的策略 |

### 提示

- 每個 Job 應該是**冪等的**（Idempotent），即多次執行結果相同
- 使用 `accurate` 策略可以更精確地控制 Job 數量
- 設定適當的 `pollingInterval` 平衡響應速度和資源消耗
- 考慮使用 `parallelism` 讓單個 Job 處理多個訊息（提高效率）

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `pollingInterval` | 檢查事件來源的間隔（秒） | 30 |
| `maxReplicaCount` | 最大同時 Job 數量 | 100 |
| `successfulJobsHistoryLimit` | 保留的成功 Job 數量 | 100 |
| `failedJobsHistoryLimit` | 保留的失敗 Job 數量 | 100 |
| `scalingStrategy.strategy` | 擴縮策略（default/custom/accurate） | default |
| `jobTargetRef.backoffLimit` | 失敗重試次數 | 6 |

| 指令 | 說明 |
|------|------|
| `kubectl get scaledjob` | 列出所有 ScaledJob |
| `kubectl describe scaledjob <name>` | 查看 ScaledJob 詳細資訊 |
| `kubectl get jobs -l scaledjob.keda.sh/name=<name>` | 列出由 ScaledJob 建立的 Job |
| `kubectl annotate scaledjob <name> autoscaling.keda.sh/paused="true"` | 暫停 ScaledJob |
