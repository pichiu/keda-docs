+++
title = "ScaledObject 規格"
weight = 3000
+++

## TL;DR

ScaledObject 是 KEDA 用來定義如何擴縮 Deployment、StatefulSet 和自訂資源的核心 CRD。本文件詳細說明所有可用的設定參數，包括目標資源、輪詢間隔、冷卻期、副本數限制、回退機制、進階 HPA 設定和觸發器。

---

## 翻譯 (Translation)

### 概述

此規格描述定義觸發器和擴縮行為的 `ScaledObject` 自訂資源定義，KEDA 使用它來擴縮 `Deployment`、`StatefulSet` 和 `Custom Resource` 目標資源。`.spec.ScaleTargetRef` 區段持有目標資源的參考。

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: {scaled-object-name}
  annotations:
    scaledobject.keda.sh/transfer-hpa-ownership: "true"     # 可選。用於將現有 HPA 的所有權轉移到此 ScaledObject
    validations.keda.sh/hpa-ownership: "true"               # 可選。用於停用此 ScaledObject 的 HPA 所有權驗證
    autoscaling.keda.sh/paused: "true"                      # 可選。用於明確暫停物件的自動擴縮
spec:
  scaleTargetRef:
    apiVersion:    {api-version-of-target-resource}         # 可選。預設：apps/v1
    kind:          {kind-of-target-resource}                # 可選。預設：Deployment
    name:          {name-of-target-resource}                # 必填。必須與 ScaledObject 在同一命名空間
    envSourceContainerName: {container-name}                # 可選。預設：.spec.template.spec.containers[0]
  pollingInterval:  30                                      # 可選。預設：30 秒
  cooldownPeriod:   300                                     # 可選。預設：300 秒
  initialCooldownPeriod:  0                                 # 可選。預設：0 秒
  idleReplicaCount: 0                                       # 可選。預設：忽略，必須小於 minReplicaCount
  minReplicaCount:  1                                       # 可選。預設：0
  maxReplicaCount:  100                                     # 可選。預設：100
  fallback:                                                 # 可選。指定回退選項的區段
    failureThreshold: 3                                     # 如包含 fallback 區段則為必填
    replicas: 6                                             # 如包含 fallback 區段則為必填
    behavior: {kind-of-behavior}                            # 可選。預設："static"
  advanced:                                                 # 可選。指定進階選項的區段
    restoreToOriginalReplicaCount: true/false               # 可選。預設：false
    horizontalPodAutoscalerConfig:                          # 可選。指定 HPA 相關選項的區段
      name: {name-of-hpa-resource}                          # 可選。預設：keda-hpa-{scaled-object-name}
      behavior:                                             # 可選。用於修改 HPA 的擴縮行為
        scaleDown:
          stabilizationWindowSeconds: 300
          policies:
          - type: Percent
            value: 100
            periodSeconds: 15
  triggers:
  # {啟動目標資源擴縮的觸發器列表}
```

### scaleTargetRef

```yaml
  scaleTargetRef:
    apiVersion:    {api-version-of-target-resource}  # 可選。預設：apps/v1
    kind:          {kind-of-target-resource}         # 可選。預設：Deployment
    name:          {name-of-target-resource}         # 必填。必須與 ScaledObject 在同一命名空間
    envSourceContainerName: {container-name}         # 可選。預設：.spec.template.spec.containers[0]
```

此 ScaledObject 設定的資源參考。這是 KEDA 將根據 `triggers:` 中定義的觸發器進行擴縮並設定 HPA 的資源。

要擴縮 Kubernetes Deployment，只需指定 `name`。要擴縮不同的資源（如 StatefulSet 或自訂資源），需要指定適當的 `apiVersion` 和 `kind`。

`envSourceContainerName` 是可選屬性，指定目標資源中的容器名稱，KEDA 應嘗試從中取得持有 Secret 等的環境屬性。如未定義，KEDA 將嘗試從第一個容器取得環境屬性。

### pollingInterval
```yaml
  pollingInterval: 30  # 可選。預設：30 秒
```
這是檢查每個觸發器的間隔。預設情況下，KEDA 每 30 秒檢查每個 ScaledObject 上的每個觸發器來源一次。

### cooldownPeriod
```yaml
  cooldownPeriod:  300 # 可選。預設：300 秒
```
最後一個觸發器報告活躍後，在將資源縮減回 0 之前等待的時間（秒）。預設為 300（5 分鐘）。

`cooldownPeriod` 只在觸發發生後適用；當您首次建立 `Deployment` 時，KEDA 會立即將其擴縮到 `minReplicaCount`。此外，KEDA 的 `cooldownPeriod` 只在縮減到 0 時適用；從 1 到 N 個副本的擴縮由 Kubernetes HPA 處理。

### initialCooldownPeriod
```yaml
   initialCooldownPeriod:  120 # 可選。預設：0 秒
```
在 `ScaledObject` 初始建立後，`cooldownPeriod` 開始之前的延遲（秒）。預設為 0。

### idleReplicaCount
```yaml
  idleReplicaCount: 0   # 可選。預設：忽略，必須小於 minReplicaCount
```
如設定此屬性，當觸發器沒有活動時，KEDA 會將資源縮減到此副本數。一旦目標觸發器有活動，KEDA 會立即將目標資源擴縮到 `minReplicaCount`，然後由 HPA 處理擴縮。

> 💡 **注意：** 由於 HPA 控制器的限制，此屬性唯一支援的值是 0。

### minReplicaCount
```yaml
  minReplicaCount: 1   # 可選。預設：0
```
KEDA 將資源縮減到的最小副本數。預設為 0（縮減到零），但您也可以使用其他值。

### maxReplicaCount
```yaml
  maxReplicaCount: 100 # 可選。預設：100
```
此設定傳遞給 KEDA 為給定資源建立的 HPA 定義，持有目標資源的最大副本數。

### fallback
```yaml
  fallback:                                          # 可選。指定回退選項的區段
    failureThreshold: 3                              # 如包含 fallback 區段則為必填
    replicas: 6                                      # 如包含 fallback 區段則為必填
    behavior: "static"                               # 可選。預設："static"
```
`fallback` 區段是可選的。它定義如果擴縮器處於錯誤狀態時回退到的副本數。

KEDA 會追蹤每個擴縮器從其來源取得指標失敗的連續次數。一旦該值超過 `failureThreshold`，擴縮器將返回正規化的指標，而不是不傳播指標到 HPA。

#### fallback.behavior

- `static`（預設）：使用 `fallback.replicas` 指定的副本數
- `currentReplicas`：使用當前副本數
- `currentReplicasIfHigher`：使用當前副本數（如果較高）
- `currentReplicasIfLower`：使用當前副本數（如果較低）

### advanced

#### restoreToOriginalReplicaCount
```yaml
advanced:
  restoreToOriginalReplicaCount: true/false        # 可選。預設：false
```
指定在刪除 `ScaledObject` 後，目標資源是否應縮減回原始副本數。

#### horizontalPodAutoscalerConfig
```yaml
advanced:
  horizontalPodAutoscalerConfig:                   # 可選。指定 HPA 相關選項的區段
    name: {name-of-hpa-resource}                   # 可選。預設：keda-hpa-{scaled-object-name}
    behavior:                                      # 可選。用於修改 HPA 的擴縮行為
      scaleDown:
        stabilizationWindowSeconds: 300
        policies:
        - type: Percent
          value: 100
          periodSeconds: 15
```

#### scalingModifiers
```yaml
advanced:
  scalingModifiers:                                       # 可選。指定擴縮修飾符的區段
    target: {target-value-to-scale-on}                    # 必填。組合指標的新目標
    activationTarget: {activation-target-value}           # 可選。組合指標的新啟動目標
    metricType:  {metric-type-for-the-modifier}           # 可選。用於修飾符的指標類型
    formula: {formula-for-fetched-metrics}                # 必填。計算公式
```

### triggers
```yaml
  triggers:
  # {啟動目標資源擴縮的觸發器列表}
```

觸發器欄位：
- **type**：要使用的觸發器類型。（必填）
- **metadata**：觸發器需要的設定參數。（必填）
- **name**：此觸發器的名稱。（可選）
- **useCachedMetrics**：在輪詢間隔期間啟用指標值快取。（值：`false`、`true`，預設：`false`，可選）
- **authenticationRef**：指向用於驗證的 `TriggerAuthentication` 或 `ClusterTriggerAuthentication` 物件的參考。（可選）
- **metricType**：應使用的指標類型。（值：`AverageValue`、`Value`、`Utilization`，預設：`AverageValue`，可選）

---

## 說明 (Explanation)

### 參數總覽

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `pollingInterval` | 檢查觸發器的間隔 | 30 秒 |
| `cooldownPeriod` | 縮減到 0 前的等待時間 | 300 秒 |
| `initialCooldownPeriod` | 建立後的初始冷卻延遲 | 0 秒 |
| `minReplicaCount` | 最小副本數 | 0 |
| `maxReplicaCount` | 最大副本數 | 100 |
| `idleReplicaCount` | 閒置時的副本數 | 忽略 |

### 指標類型說明

| 類型 | 說明 | 計算方式 |
|------|------|----------|
| `AverageValue` | 每個 Pod 的平均值 | 總值 / 副本數 |
| `Value` | 絕對值 | 總值 × 副本數 / 目標 |
| `Utilization` | 使用率百分比 | 僅 CPU/Memory |

---

## 實作範例 (Practical Example)

```yaml
# 完整的 ScaledObject 範例
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: comprehensive-scaledobject
  namespace: default
spec:
  scaleTargetRef:
    name: my-deployment
  pollingInterval: 15                   # 每 15 秒檢查
  cooldownPeriod: 120                   # 2 分鐘冷卻
  minReplicaCount: 1                    # 至少 1 個副本
  maxReplicaCount: 50                   # 最多 50 個副本
  fallback:
    failureThreshold: 3                 # 失敗 3 次後回退
    replicas: 5                         # 回退到 5 個副本
  advanced:
    horizontalPodAutoscalerConfig:
      behavior:
        scaleDown:
          stabilizationWindowSeconds: 60
          policies:
            - type: Percent
              value: 50
              periodSeconds: 30
        scaleUp:
          policies:
            - type: Percent
              value: 100
              periodSeconds: 15
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus:9090
        query: sum(rate(http_requests_total[1m]))
        threshold: "100"
    - type: cpu
      metricType: Utilization
      metadata:
        value: "70"
```

---

## 快速參考 (Quick Reference)

| 區段 | 說明 |
|------|------|
| `scaleTargetRef` | 目標資源參考 |
| `pollingInterval` | 輪詢間隔 |
| `cooldownPeriod` | 冷卻期 |
| `minReplicaCount` / `maxReplicaCount` | 副本數範圍 |
| `fallback` | 回退機制 |
| `advanced` | 進階 HPA 設定 |
| `triggers` | 觸發器列表 |
