
## TL;DR

Memory 擴縮器根據 Pod 的記憶體使用量來擴縮應用程式。可設定使用率（Utilization）百分比或絕對值（AverageValue）作為目標。此擴縮器需要 Kubernetes Metrics Server，且 Pod 必須設定資源請求（requests）或限制（limits）。

---

## 翻譯 (Translation)

> **注意：**
> - 此擴縮器**需要前置條件**。請參閱「前置條件」區段。
> - 此擴縮器只有在使用者定義了至少一個非 CPU 或 Memory 的額外擴縮器（例如 Kafka + Memory 或 Prometheus + Memory）且 `minReplicaCount` 為 0 時，才能縮減到 0。
> - 此擴縮器僅適用於 ScaledObject，不適用於 ScaledJob。

### 前置條件

KEDA 使用 Kubernetes Metrics Server 的標準 `cpu` 和 `memory` 指標，該服務在某些 Kubernetes 部署（如 AWS 上的 EKS）中預設未安裝。此外，相關 Kubernetes Pod 的 `resources` 區段必須至少包含 `requests` 或 `limits` 其中之一。

- 必須安裝 Kubernetes Metrics Server。安裝說明因 Kubernetes 提供者而異。
- Kubernetes Pod 的設定必須包含指定 `requests`（或 `limits`）的 `resources` 區段。如果 resources 區段為空（`resources: {}` 或類似），KEDA 會檢查同一命名空間中是否為 `Container` 類型設定了 `LimitRange` 的 `defaultRequest`（或 limits 的 `default`）。如果 `defaultRequest`（或 `default`）也缺失，會發生 `missing request for {cpu/memory}` 錯誤。

```yaml
# resources 設定範例
spec:
  containers:
  - name: app
    image: images.my-company.example/app:v4
    resources:
      requests:
        memory: "128Mi"
        cpu: "500m"
```

### 觸發器規格

此規格描述根據記憶體指標進行擴縮的 `memory` 觸發器。

```yaml
triggers:
- type: memory
  metricType: Utilization  # 允許的類型：'Utilization' 或 'AverageValue'
  metadata:
    value: "60"
    containerName: ""      # 可選。可用來針對 Pod 中的特定容器
```

**參數列表：**

- `metricType` - 使用的指標類型。選項為 `Utilization` 或 `AverageValue`。
- `value` - 觸發擴縮動作的值：
  - 使用 `Utilization` 時，目標值是所有相關 Pod 資源指標的平均值，表示為 Pod 請求資源值的百分比。
  - 使用 `AverageValue` 時，目標值是所有相關 Pod 指標平均值的目標值（數量）。
- `containerName` - 根據特定容器的記憶體進行擴縮，而非整個 Pod。如未指定則預設為空。

> 💡 **注意：** `containerName` 參數需要 Kubernetes 叢集版本 1.20 或更高，並啟用 `HPAContainerMetrics` 功能。請參閱[容器資源指標](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#container-resource-metrics)了解更多資訊。

---

## 說明 (Explanation)

### 指標類型比較

| 類型 | 說明 | 使用場景 |
|------|------|----------|
| `Utilization` | 相對於 requests 的百分比 | 希望保持固定使用率 |
| `AverageValue` | 絕對記憶體量（如 500Mi）| 希望維持固定記憶體用量 |

### 計算公式

| 類型 | 公式 |
|------|------|
| Utilization | 期望副本數 = ceil(當前副本數 × (當前使用率 / 目標使用率)) |
| AverageValue | 期望副本數 = ceil(當前副本數 × (當前平均值 / 目標平均值)) |

---

## 實作範例 (Practical Example)

```yaml
# 針對整個 Pod 的記憶體使用率
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: memory-scaledobject
  namespace: default
spec:
  scaleTargetRef:
    name: my-deployment
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
    - type: memory
      metricType: Utilization
      metadata:
        value: "50"           # 當記憶體使用率超過 50% 時擴縮
```

```yaml
# 針對特定容器的記憶體使用率
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: memory-container-scaledobject
  namespace: default
spec:
  scaleTargetRef:
    name: my-deployment
  triggers:
    - type: memory
      metricType: Utilization
      metadata:
        value: "70"
        containerName: "main-app"  # 只監控此容器
```

```yaml
# 使用絕對值（AverageValue）
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: memory-average-scaledobject
spec:
  scaleTargetRef:
    name: my-deployment
  triggers:
    - type: memory
      metricType: AverageValue
      metadata:
        value: "500Mi"         # 每個 Pod 平均 500Mi
```

```yaml
# 結合其他擴縮器以支援縮減到 0
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: memory-kafka-scaledobject
spec:
  scaleTargetRef:
    name: my-deployment
  minReplicaCount: 0
  maxReplicaCount: 20
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka:9092
        consumerGroup: my-group
        topic: my-topic
        lagThreshold: "50"
    - type: memory
      metricType: Utilization
      metadata:
        value: "70"
```

```bash
# 安裝 Metrics Server（範例）
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# 查看 Pod 記憶體使用量
kubectl top pods

# 查看 HPA 狀態
kubectl get hpa -w

# 查看 ScaledObject 狀態
kubectl describe scaledobject memory-scaledobject
```

---

## 常見錯誤與提示 (Common Mistakes & Tips)

| 常見錯誤 | 正確做法 |
|----------|----------|
| 未安裝 Metrics Server | 安裝 Kubernetes Metrics Server |
| Pod 未設定 resources | 在 Deployment 中設定 resources.requests |
| 只用 Memory 擴縮器但 minReplicaCount 為 0 | 新增其他擴縮器（如 Kafka）才能縮減到 0 |
| containerName 不存在 | 確認容器名稱正確 |

### 提示

- Memory 擴縮器單獨使用時無法縮減到 0，需搭配其他擴縮器
- 使用 `Utilization` 類型時，確保 Pod 設定了 memory requests
- 對於有多個容器的 Pod，預設會計算所有容器記憶體的總和
- 設定 `containerName` 可以只監控特定容器的記憶體使用量

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `metricType` | 指標類型 | 無（必填）|
| `value` | 目標值 | 無（必填）|
| `containerName` | 容器名稱 | 空（整個 Pod）|

| metricType | 說明 | 範例值 |
|------------|------|--------|
| `Utilization` | 使用率百分比 | "70"（70%）|
| `AverageValue` | 絕對值 | "500Mi" |

| 指令 | 說明 |
|------|------|
| `kubectl top pods` | 查看 Pod 資源使用量 |
| `kubectl get hpa` | 查看 HPA 狀態 |
| `kubectl describe scaledobject <name>` | 查看 ScaledObject 詳細資訊 |
