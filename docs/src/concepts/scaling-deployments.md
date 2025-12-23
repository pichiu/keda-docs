
## TL;DR

KEDA 透過 ScaledObject 擴縮 Deployment 和 StatefulSet，支援從 0 到 N 的動態擴縮。可透過 annotation 暫停擴縮、使用 Scaling Modifiers 組合多個指標、設定啟動閾值和擴縮閾值來精細控制擴縮行為。

---

## 翻譯 (Translation)

本頁描述 KEDA 的部署擴縮行為。

### 規格

詳細的設定方式請參閱 [ScaledObject 規格](../reference/scaledobject-spec.md)。

### 擴縮物件

#### 擴縮 Deployment 和 StatefulSet

Deployment 和 StatefulSet 是使用 KEDA 擴縮工作負載最常見的方式。

它允許您定義希望 KEDA 根據擴縮觸發器（Scale Trigger）來擴縮的 Kubernetes Deployment 或 StatefulSet。KEDA 會監控該服務，並根據發生的事件自動相應地擴展或縮減您的資源。

在幕後，KEDA 負責監控事件來源，並將該資料饋送到 Kubernetes 和 HPA（水平 Pod 自動擴縮器）以驅動資源的快速擴縮。資源的每個副本（Replica）都會主動從事件來源提取項目。使用 KEDA 擴縮 Deployment/StatefulSet，您可以根據事件進行擴縮，同時保留與事件來源的豐富連接和處理語義（例如：按序處理、重試、死信佇列、檢查點）。

例如，如果您想使用 KEDA 搭配 Apache Kafka 主題（Topic）作為事件來源，資訊流程如下：

* 當沒有訊息等待處理時，KEDA 可以將 Deployment 縮減到零。
* 當訊息到達時，KEDA 偵測到此事件並啟動 Deployment。
* 當 Deployment 開始執行時，其中一個容器連接到 Kafka 並開始提取訊息。
* 隨著更多訊息到達 Kafka 主題，KEDA 可以將這些資料饋送到 HPA 以驅動擴展。
* Deployment 的每個副本都在主動處理訊息。很可能每個副本都以分散式方式處理一批訊息。

#### 擴縮自訂資源

使用 KEDA，您可以擴縮任何定義為「自訂資源」（Custom Resource）的工作負載（例如 `ArgoRollout` [資源](https://argoproj.github.io/argo-rollouts/)）。擴縮行為與擴縮任意 Kubernetes `Deployment` 或 `StatefulSet` 相同。

唯一的限制是目標「自訂資源」必須定義 `/scale` [子資源](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#scale-subresource)。

### 功能

#### 快取指標

此功能在輪詢間隔（如 `.spec.pollingInterval` 中指定）期間啟用指標值的快取。Kubernetes（HPA 控制器）每隔幾秒（由 `--horizontal-pod-autoscaler-sync-period` 定義，通常為 15 秒）請求一次指標，然後這個請求被路由到 KEDA Metrics Server，預設情況下會查詢擴縮器並讀取指標值。啟用此功能會改變這個行為，使 KEDA Metrics Server 首先嘗試從快取讀取指標。此快取在輪詢間隔期間定期更新。

啟用 [`useCachedMetrics`](../reference/scaledobject-spec/#triggers) 可以顯著減少對擴縮器服務的負載。

此功能不支援 `cpu`、`memory` 或 `cron` 擴縮器。

#### 暫停自動擴縮

指示 KEDA 暫停物件的自動擴縮可能很有用，例如進行叢集維護或透過移除非關鍵任務工作負載來避免資源耗盡。

這比刪除資源更好，因為它可以從操作中移除正在執行的實例而不觸碰應用程式本身。準備好後，您可以重新啟用擴縮。

您可以透過在 `ScaledObject` 定義中新增此 annotation 來暫停自動擴縮：

```yaml
metadata:
  annotations:
    autoscaling.keda.sh/paused-replicas: "0"
    autoscaling.keda.sh/paused: "true"
```

無論提供多少副本數，這些 annotation 的存在都會暫停自動擴縮。

`autoscaling.keda.sh/paused` annotation 會立即暫停擴縮並使用當前實例數，而 `autoscaling.keda.sh/paused-replicas: "<number>"` annotation 會將您目前的工作負載擴縮到指定的副本數並暫停自動擴縮。您可以將要暫停的物件的副本數設定為任意數字。

通常，兩者之一會被使用，因為它們服務於不同的目的/場景。但是，如果同時設定了 `paused` 和 `paused-replicas`，KEDA 會將您目前的工作負載擴縮到 `paused-replicas` 中指定的數量，然後暫停自動擴縮。

要取消暫停（重新啟用）自動擴縮，請從 `ScaledObject` 定義中移除所有暫停 annotation。如果您使用 `autoscaling.keda.sh/paused` 暫停，可以透過將 annotation 設定為 `false` 來取消暫停。

此外，我們提供暫時暫停擴縮目標的縮減功能：

```yaml
metadata:
  annotations:
    autoscaling.keda.sh/paused-scale-in: "true"
```

當設定此 annotation 時，KEDA 會更新產生的 HPA 以停用縮減（透過將 HPA 的 Scale Down Select Policy 設定為 Disabled），如果服務設定了縮減到零，也會阻止縮減到零。當取消設定 annotation 時，HPA 上的縮減行為將恢復到其原始設定，如果有設定，縮減到零也會解除阻止。

相反地，我們提供暫時暫停擴縮目標的擴展功能：

```yaml
metadata:
  annotations:
    autoscaling.keda.sh/paused-scale-out: "true"
```

當設定此 annotation 時，KEDA 會更新產生的 HPA 以停用擴展（透過將 HPA 的 Scale Up Select Policy 設定為 Disabled），如果服務設定了從零擴縮，也會阻止從零擴縮。當取消設定 annotation 時，HPA 上的擴展行為將恢復到其原始設定，如果有設定，從零擴展也會解除阻止。

如果您想在兩個方向都停用擴縮，我們建議您使用 `autoscaling.keda.sh/paused`，因為這會停止擴縮迴圈並暫停對 ScaledObject 設定的擴縮器的查詢。

#### 擴縮修飾符

**範例：組合平均值**

```yaml
advanced:
  scalingModifiers:
    formula: "(trig_one + trig_two)/2"
    target: "2"
    activationTarget: "2"
    metricType: "AverageValue"
...
triggers:
  - type: kubernetes-workload
    name: trig_one
    metadata:
      podSelector: 'pod=workload-test'
  - type: metrics-api
    name: trig_two
    metadata:
      url: "https://mockbin.org/bin/336a8d99-9e09-4f1f-979d-851a6d1b1423"
      valueLocation: "tasks"
```

公式將來自 2 個觸發器 `kubernetes-workload`（命名為 `trig_one`）和 `metrics-api`（命名為 `trig_two`）的 2 個指標組合為平均值，並返回一個用於做出自動擴縮決策的最終指標。

**範例：activationTarget**

```yaml
advanced:
  scalingModifiers:
    activationTarget: "2"
```

如果計算值 <=2，ScaledObject 不是「Active」，如果允許的話它會縮減到 0。

**範例：三元運算子**

```yaml
advanced:
  scalingModifiers:
    formula: "trig_one > 2 ? trig_one + trig_two : 1"
```

如果觸發器 `trig_one` 的指標值大於 2，則返回 `trig_one` + `trig_two`，否則返回 1。

**範例：count 函數**

```yaml
advanced:
  scalingModifiers:
    formula: "count([trig_one,trig_two,trig_three],{#>1}) > 1 ? 5 : 0"
```

如果至少有 2 個指標（從列表 `trig_one`、`trig_two`、`trig_three` 中）的值大於 1，則返回 5，否則返回 0。

**範例：巢狀條件和運算子**

```yaml
advanced:
  scalingModifiers:
    formula: "trig_one < 2 ? trig_one+trig_two >= 2 ? 5 : 10 : 0"
```

條件也可以在另一個條件中使用。
如果 `trig_one` 的值小於 2 且 `trig_one`+`trig_two` 至少為 2，則返回 5；如果只有第一個條件為真，則返回 10；如果第一個條件為假，則返回 0。

`expr` 套件的完整語言定義可以在[這裡](https://expr.medv.io/docs/Language-Definition)找到。公式必須返回單一值（不是布林值）。所有公式在內部都使用 float 轉換包裝。

#### 啟動和擴縮閾值

KEDA 在自動擴縮過程中有 2 個不同的階段。

- **啟動階段：** 啟動（或停用）階段是 KEDA（Operator）必須決定工作負載是否應該從/到零擴縮的時刻。KEDA 根據擴縮器 `IsActive` 函數的結果負責此動作，僅適用於 0<->1 擴縮。有些使用案例中啟動值（0-1 和 1-0）與 0 完全不同，例如使用 Prometheus 擴縮器擴縮的工作負載，其值從 -X 到 X。
- **擴縮階段：** 擴縮階段是 KEDA 已決定擴展到 1 個實例，現在由 HPA 控制器根據產生的 HPA（來自 ScaledObject 資料）中定義的設定和 KEDA 公開的指標（指標伺服器）做出擴縮決策的時刻。此階段適用於 1<->N 擴縮。

KEDA 允許您為每個場景指定不同的值：

- **啟動：** 定義擴縮器何時啟動或不啟動，並據此從/到 0 擴縮。
- **擴縮：** 定義將工作負載從 1 擴縮到 _n_ 個實例（反之亦然）的目標值。為實現這一點，KEDA 將目標值傳遞給水平 Pod 自動擴縮器（HPA），內建的 HPA 控制器將處理所有自動擴縮。

> ⚠️ **注意：** 如果最小副本數 >= 1，擴縮器始終處於啟動狀態，啟動值將被忽略。

每個擴縮器為其使用案例定義參數，但啟動始終與擴縮值相同，前綴為 `activation`（即：用於擴縮的 `threshold` 和用於啟動的 `activationThreshold`）。

有一些重要的主題需要考慮：

- 與擴縮值相反，啟動值始終是可選的，預設值為 0。
- 啟動僅在此值大於設定值時發生；不是大於或等於。
  - 即，在預設情況下：`activationThreshold: 0` 只有在指標值為 1 或更多時才會啟動
- 在不同決策的情況下，啟動值比擴縮值有更高的優先級。例如：`threshold: 10` 和 `activationThreshold: 50`，在 40 個訊息的情況下，擴縮器不活躍，即使 HPA 需要 4 個實例，它也會被縮減到零。

> ⚠️ **注意：** 如果擴縮器沒有定義「activation」參數（以 `activation` 前綴開頭的屬性），則此特定擴縮器不支援可設定的啟動值，啟動值始終為 0。

#### 強制啟動

我們提供暫時強制啟動擴縮目標的功能：

```yaml
metadata:
  annotations:
    autoscaling.keda.sh/force-activation: "true"
```

當設定此 annotation 時，KEDA 會將所有設定的擴縮器視為活躍狀態。如果擴縮器之前不活躍，KEDA 會將服務從 0 擴展。

當隨後取消設定 annotation 時，擴縮器啟動的狀態將恢復為從擴縮器指標的狀態計算。

#### 轉移現有 HPA 的所有權

如果您的環境已經使用 Kubernetes HPA 運作，您可以將此資源的所有權轉移到新的 ScaledObject：

```yaml
metadata:
  annotations:
    scaledobject.keda.sh/transfer-hpa-ownership: "true"
spec:
   advanced:
      horizontalPodAutoscalerConfig:
        name: {name-of-hpa-resource}
```

> ⚠️ **注意：** 您需要在 ScaledObject 中指定一個自訂 HPA 名稱，與您希望它管理的現有 HPA 名稱相符。

#### 停用現有 HPA 上的驗證

您可以使用以下程式碼片段停用 Admission Webhook 驗證。它給您更大的靈活性，但也帶來漏洞。請**自行承擔風險**。

```yaml
metadata:
  annotations:
    validations.keda.sh/hpa-ownership: "true"
```

#### 長時間執行的作業

需要考慮的一個重要事項是這種模式如何與長時間執行的作業一起運作。想像一個 Deployment 在 RabbitMQ 佇列訊息上觸發。每個訊息需要 3 小時處理。如果許多佇列訊息到達，KEDA 可能會幫助驅動擴展到許多副本——假設是 4 個。現在 HPA 做出從 4 個副本縮減到 2 個的決定。無法控制哪 2 個副本被終止以縮減。這意味著 HPA 可能會嘗試終止一個在處理 3 小時佇列訊息中已經進行了 2.9 小時的副本。

有兩種主要方法來處理這種場景。

##### 利用容器生命週期

Kubernetes 提供了一些[生命週期掛鉤](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/)，可以用來延遲終止。想像一個副本被安排終止，並且在處理 3 小時訊息中已經進行了 2.9 小時。Kubernetes 會發送 [`SIGTERM`](https://www.gnu.org/software/libc/manual/html_node/Termination-Signals.html) 來信號終止意圖。Deployment 可以延遲終止直到處理當前批次訊息完成，而不是立即終止。Kubernetes 會等待 `SIGTERM` 回應或 `terminationGracePeriodSeconds` 後才終止副本。

> 💡 **注意：** 還有其他方法可以延遲終止，包括 [`preStop` 掛鉤](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/#container-hooks)。

使用此方法可以保留副本並啟用長時間執行的作業。然而，這種方法的一個缺點是在延遲終止期間，Pod 階段將保持在 `Terminating` 狀態。這意味著延遲終止很長時間的 Pod 在整個延遲期間可能會顯示 `Terminating`。

##### 作為 Job 執行

處理長時間執行作業的另一種替代方法是在 Kubernetes Job 中執行事件驅動程式碼，而不是 Deployment 或自訂資源。這種方法在[下一節](./scaling-jobs)中討論。

#### 排除標籤不傳播到 HPA

您可以使用 `scaledobject.keda.sh/hpa-excluded-labels` annotation 排除特定標籤不傳播到產生的 HPA 物件。此 annotation 接受應排除的標籤鍵的逗號分隔列表。

```yaml
metadata:
  annotations:
    scaledobject.keda.sh/hpa-excluded-labels: "foo.bar/environment,foo.bar/version"
  labels:
    team: backend
    foo.bar/environment: bf5011472247b67cce3ee7b24c9a08c5
    foo.bar/version: "1"
```

---

## 說明 (Explanation)

### ScaledObject 擴縮流程

```
外部事件來源 → Scaler 取得指標 → KEDA Operator 評估
                                        ↓
                              是否需要從 0 → 1？
                                   ↓        ↓
                                  是        否
                                   ↓        ↓
                          KEDA 啟動 Pod   HPA 處理 1 → N 擴縮
```

### 暫停模式比較

| Annotation | 行為 |
|------------|------|
| `autoscaling.keda.sh/paused: "true"` | 使用當前副本數暫停 |
| `autoscaling.keda.sh/paused-replicas: "N"` | 擴縮到 N 個副本後暫停 |
| `autoscaling.keda.sh/paused-scale-in: "true"` | 只暫停縮減（仍可擴展）|
| `autoscaling.keda.sh/paused-scale-out: "true"` | 只暫停擴展（仍可縮減）|

---

## 實作範例 (Practical Example)

```yaml
# 完整的 ScaledObject 範例
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: my-app-scaledobject
  annotations:
    # 可選：排除某些標籤不傳播到 HPA
    scaledobject.keda.sh/hpa-excluded-labels: "version"
spec:
  scaleTargetRef:
    name: my-deployment                    # 目標 Deployment 名稱
  pollingInterval: 30                      # 輪詢間隔（秒）
  cooldownPeriod: 300                      # 冷卻期（秒）
  minReplicaCount: 0                       # 最小副本數（可縮到 0）
  maxReplicaCount: 100                     # 最大副本數
  advanced:
    horizontalPodAutoscalerConfig:
      behavior:
        scaleDown:
          stabilizationWindowSeconds: 300  # 縮減穩定窗口
    scalingModifiers:
      formula: "(trigger1 + trigger2) / 2" # 組合多個觸發器
      target: "5"
  triggers:
    - type: rabbitmq
      name: trigger1
      metadata:
        queueName: my-queue
        queueLength: "10"
      authenticationRef:
        name: rabbitmq-auth
    - type: prometheus
      name: trigger2
      metadata:
        serverAddress: http://prometheus:9090
        query: sum(rate(http_requests_total[1m]))
        threshold: "100"
```

```bash
# 暫停擴縮
kubectl annotate scaledobject my-app-scaledobject \
  autoscaling.keda.sh/paused="true"

# 恢復擴縮
kubectl annotate scaledobject my-app-scaledobject \
  autoscaling.keda.sh/paused-

# 暫停並設定固定副本數
kubectl annotate scaledobject my-app-scaledobject \
  autoscaling.keda.sh/paused-replicas="3"

# 強制啟動（從 0 擴展）
kubectl annotate scaledobject my-app-scaledobject \
  autoscaling.keda.sh/force-activation="true"
```

---

## 常見錯誤與提示 (Common Mistakes & Tips)

| 常見錯誤 | 正確做法 |
|----------|----------|
| 長時間任務被 HPA 意外終止 | 使用 terminationGracePeriodSeconds 或改用 ScaledJob |
| 不了解啟動閾值和擴縮閾值的區別 | 啟動控制 0↔1，擴縮控制 1↔N |
| 使用 useCachedMetrics 於不支援的擴縮器 | cpu、memory、cron 不支援快取 |
| 兩個 ScaledObject 指向同一個 Deployment | 確保每個 Deployment 只有一個 ScaledObject |

### 提示

- 對於處理時間很長的任務，考慮使用 ScaledJob 而不是 ScaledObject
- 使用 `scalingModifiers.formula` 可以組合多個觸發器的指標
- 設定適當的 `cooldownPeriod` 避免頻繁的擴縮抖動
- 使用 `activationTarget` 可以設定不同於擴縮閾值的啟動閾值

---

## 快速參考 (Quick Reference)

| Annotation | 說明 |
|------------|------|
| `autoscaling.keda.sh/paused: "true"` | 暫停擴縮，保持當前副本數 |
| `autoscaling.keda.sh/paused-replicas: "N"` | 設定為 N 個副本並暫停 |
| `autoscaling.keda.sh/paused-scale-in: "true"` | 只暫停縮減 |
| `autoscaling.keda.sh/paused-scale-out: "true"` | 只暫停擴展 |
| `autoscaling.keda.sh/force-activation: "true"` | 強制啟動 |
| `scaledobject.keda.sh/transfer-hpa-ownership: "true"` | 接管現有 HPA |
| `scaledobject.keda.sh/hpa-excluded-labels: "a,b"` | 排除標籤 |
