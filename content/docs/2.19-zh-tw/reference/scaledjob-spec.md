+++
title = "ScaledJob 規格"
weight = 4000
+++

## TL;DR

ScaledJob 是 KEDA 用來定義如何擴縮 Kubernetes Job 的 CRD。本文件詳細說明所有可用的設定參數，包括 Job 目標參考、輪詢間隔、歷史記錄限制、副本數限制、部署策略和擴縮策略。

---

## 翻譯 (Translation)

### 概述

此規格描述定義觸發器和擴縮行為的 `ScaledJob` 自訂資源定義，KEDA 使用它來擴縮 Job。`.spec.ScaleTargetRef` 區段持有 Job 的參考。

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: {scaled-job-name}
  labels:
    my-label: {my-label-value}                # 可選。ScaledJob 標籤會套用到子 Job
  annotations:
    autoscaling.keda.sh/paused: true          # 可選。用於暫停 Job 的自動擴縮
    my-annotation: {my-annotation-value}      # 可選。ScaledJob 註解會套用到子 Job
spec:
  jobTargetRef:
    parallelism: 1                            # 期望 Pod 的最大數量
    completions: 1                            # 期望成功完成的 Pod 數量
    activeDeadlineSeconds: 600                # Job 活動的最大持續時間（秒）
    backoffLimit: 6                           # 標記為失敗前的重試次數。預設：6
    template:
      # 描述 Job 模板
  pollingInterval: 30                         # 可選。預設：30 秒
  successfulJobsHistoryLimit: 5               # 可選。預設：100。保留多少完成的 Job
  failedJobsHistoryLimit: 5                   # 可選。預設：100。保留多少失敗的 Job
  envSourceContainerName: {container-name}    # 可選。預設：.spec.JobTargetRef.template.spec.containers[0]
  minReplicaCount: 10                         # 可選。預設：0
  maxReplicaCount: 100                        # 可選。預設：100
  rollout:
    strategy: gradual                         # 可選。預設：default。KEDA 使用的部署策略
    propagationPolicy: foreground             # 可選。預設：background。部署時清理現有 Job 的策略
  scalingStrategy:
    strategy: "custom"                        # 可選。預設：default。使用的擴縮策略
    customScalingQueueLengthDeduction: 1      # 可選。優化自訂擴縮策略的參數
    customScalingRunningJobPercentage: "0.5"  # 可選。優化自訂擴縮策略的參數
    pendingPodConditions:                     # 可選。根據指定的 Pod 條件計算待處理 Job 數量
      - "Ready"
      - "PodScheduled"
    multipleScalersCalculation: "max"         # 可選。預設：max。定義多個擴縮器時如何計算目標指標
  triggers:
  # {建立 Job 的觸發器列表}
```

### jobTargetRef

```yaml
  jobTargetRef:
    parallelism: 1              # 可選。期望實例的最大數量
    completions: 1              # 可選。期望成功完成的實例數量
    activeDeadlineSeconds: 600  # 可選。Job 活動的最大持續時間（秒）
    backoffLimit: 6             # 可選。標記為失敗前的重試次數。預設：6
```

`jobTargetRef` 是 batch/v1 `JobSpec` 物件。`template` 欄位為必填。

### pollingInterval

```yaml
  pollingInterval: 30  # 可選。預設：30 秒
```

這是檢查每個觸發器的間隔。預設情況下，KEDA 每 30 秒檢查每個 ScaledJob 上的每個觸發器來源一次。

### successfulJobsHistoryLimit, failedJobsHistoryLimit

```yaml
  successfulJobsHistoryLimit: 5  # 可選。預設：100。保留多少完成的 Job
  failedJobsHistoryLimit: 5      # 可選。預設：100。保留多少失敗的 Job
```

這些欄位指定應保留多少完成和失敗的 Job。預設設為 100。

### minReplicaCount

```yaml
  minReplicaCount: 10 # 可選。預設：0
```

預設建立的最小 Job 數量。這可以用於避免新 Job 的啟動時間。如果 minReplicaCount 大於 maxReplicaCount，minReplicaCount 將設為 maxReplicaCount。

### maxReplicaCount

```yaml
  maxReplicaCount: 100 # 可選。預設：100
```

單次輪詢期間建立的最大 Pod 數量。如果有執行中的 Job，將扣除執行中的 Job 數量。

| 佇列長度 | 最大副本數 | 目標平均值 | 執行中 Job 數 | 擴縮數量 |
|----------|------------|------------|---------------|----------|
| 10 | 3 | 1 | 0 | 3 |
| 10 | 3 | 2 | 0 | 3 |
| 10 | 3 | 1 | 1 | 2 |
| 10 | 100 | 1 | 0 | 10 |
| 4 | 3 | 5 | 0 | 1 |

### rollout

```yaml
  rollout:
    strategy: gradual                         # 可選。預設：default。KEDA 使用的部署策略
    propagationPolicy: foreground             # 可選。預設：background。清理現有 Job 的策略
```

- `default`：更新 ScaledJob 時，KEDA 會終止現有 Job，然後使用最新規格重新建立
- `gradual`：更新 ScaledJob 時，KEDA 不會刪除現有 Job，只有新 Job 會使用最新規格建立

### scalingStrategy

```yaml
scalingStrategy:
  strategy: "default"                 # 可選。預設：default。使用的擴縮策略
```

可能的值：`default`、`custom`、`accurate` 或 `eager`。預設值為 `default`。

**default**
```go
擴縮數量 = maxScale - runningJobCount
```

**custom**
```yaml
customScalingQueueLengthDeduction: 1      # 可選
customScalingRunningJobPercentage: "0.5"  # 可選
```

**accurate**
如果擴縮器返回的 `queueLength` 不包含鎖定訊息的數量，建議使用此策略。

**eager**
使用所有可用槽位直到 maxReplicaCount，確保等待的訊息盡快處理。

### multipleScalersCalculation

```yaml
scalingStrategy:
    multipleScalersCalculation: "max" # 可選。預設：max
```

可能的值：`max`、`min`、`avg` 或 `sum`。預設值為 `max`。

- **max** - 使用具有最大 `queueLength` 的擴縮器的指標（預設）
- **min** - 使用具有最小 `queueLength` 的擴縮器的指標
- **avg** - 加總所有活動擴縮器指標並除以活動擴縮器數量
- **sum** - 加總所有活動擴縮器指標

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `pollingInterval` | 輪詢間隔 | 30 秒 |
| `successfulJobsHistoryLimit` | 成功 Job 歷史記錄 | 100 |
| `failedJobsHistoryLimit` | 失敗 Job 歷史記錄 | 100 |
| `minReplicaCount` | 最小 Job 數 | 0 |
| `maxReplicaCount` | 最大 Job 數 | 100 |
| `rollout.strategy` | 部署策略 | default |
| `scalingStrategy.strategy` | 擴縮策略 | default |

| 擴縮策略 | 說明 |
|----------|------|
| `default` | maxScale - runningJobCount |
| `custom` | 自訂公式 |
| `accurate` | 考慮待處理 Job |
| `eager` | 使用所有可用槽位 |

| 指令 | 說明 |
|------|------|
| `kubectl get scaledjob` | 列出 ScaledJob |
| `kubectl describe scaledjob <name>` | 查看 ScaledJob 詳細資訊 |
| `kubectl get jobs` | 列出產生的 Job |
