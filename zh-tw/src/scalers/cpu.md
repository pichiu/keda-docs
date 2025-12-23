
## TL;DR

CPU 擴縮器根據 Pod 的 CPU 使用率來擴縮應用程式。需要 Kubernetes Metrics Server 和 Pod 的資源請求（requests）定義。此擴縮器無法單獨將 Pod 縮減到 0，必須搭配其他擴縮器使用。

---

## 翻譯 (Translation)

> **注意：**
> - 此擴縮器**需要前置條件**。請參閱「前置條件」章節。
> - 此擴縮器只有在使用者定義了至少一個非 CPU 或 Memory 的額外擴縮器（例如 Kafka + CPU，或 Prometheus + CPU）且 `minReplicaCount` 為 0 時，才能縮減到 0。
> - 此擴縮器僅適用於 ScaledObject，不適用於 ScaledJob。

### 前置條件

KEDA 使用來自 Kubernetes Metrics Server 的標準 `cpu` 和 `memory` 指標，某些 Kubernetes 部署（如 AWS 上的 EKS）預設未安裝此服務。此外，相關 Kubernetes Pod 的 `resources` 區段必須至少包含 `requests` 或 `limits` 之一。

- 必須安裝 Kubernetes Metrics Server。安裝說明因您的 Kubernetes 提供者而異。
- 您的 Kubernetes Pod 設定必須包含帶有指定 `requests`（或 `limits`）的 `resources` 區段。請參閱 [Pod 和容器的資源管理](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)。

```yaml
# resources 的工作範例，帶有指定的 requests
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

此規格描述根據 CPU 指標擴縮的 `cpu` 觸發器。

```yaml
triggers:
- type: cpu
  metricType: Utilization # 允許的類型為 'Utilization' 或 'AverageValue'
  metadata:
    value: "60"
    containerName: "" # 可選。您可以使用此參數指向 Pod 中的特定容器
```

**參數列表：**

- `type` - 要使用的指標類型。選項為 `Utilization` 或 `AverageValue`。
- `value` - 觸發擴縮動作的值：
  - 使用 `Utilization` 時，目標值是所有相關 Pod 的資源指標的平均值，表示為 Pod 請求的資源值的百分比。
  - 使用 `AverageValue` 時，目標值是所有相關 Pod 的指標平均值的目標值（數量）。
- `containerName` - 要根據其 CPU 進行擴縮的特定容器名稱，而不是整個 Pod。如未指定，預設為空。

> 💡 **注意：** `containerName` 參數需要 Kubernetes 叢集版本 1.20 或更高，並啟用 `HPAContainerMetrics` 功能。

---

## 說明 (Explanation)

### 指標類型比較

| 類型 | 說明 | 使用場景 |
|------|------|----------|
| `Utilization` | 相對於 requests 的使用率百分比 | 最常見，設定目標如 70% |
| `AverageValue` | 絕對值（如 500m）| 需要固定閾值時 |

### CPU 單位

| 單位 | 說明 | 範例 |
|------|------|------|
| `m` | 毫核心 (millicores) | `500m` = 0.5 核心 |
| 整數 | 核心數 | `2` = 2 核心 |

### 計算範例

假設設定：
- Pod requests: `500m`
- 目標 Utilization: `80%`

當前使用量為 `400m` 時：
- 使用率 = 400m / 500m = 80%
- 達到目標，不擴縮

當前使用量為 `600m` 時：
- 使用率 = 600m / 500m = 120%
- 超過目標，需要擴展

---

## 實作範例 (Practical Example)

```yaml
# 基本 CPU 擴縮（整個 Pod）
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: cpu-scaledobject
spec:
  scaleTargetRef:
    name: my-deployment
  minReplicaCount: 1                  # CPU 擴縮器無法單獨縮到 0
  maxReplicaCount: 10
  triggers:
    - type: cpu
      metricType: Utilization
      metadata:
        value: "70"                   # 目標 CPU 使用率 70%
```

```yaml
# 針對特定容器的 CPU 擴縮
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: cpu-container-scaledobject
spec:
  scaleTargetRef:
    name: my-deployment
  triggers:
    - type: cpu
      metricType: Utilization
      metadata:
        value: "60"
        containerName: "main-app"     # 只監控 main-app 容器
```

```yaml
# 結合 Prometheus 實現縮到 0
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: cpu-prometheus-scaledobject
spec:
  scaleTargetRef:
    name: my-deployment
  minReplicaCount: 0                  # 可以縮到 0（因為有 Prometheus 觸發器）
  maxReplicaCount: 20
  triggers:
    - type: cpu
      metricType: Utilization
      metadata:
        value: "80"
    - type: prometheus
      metadata:
        serverAddress: http://prometheus:9090
        query: sum(rate(http_requests_total[1m]))
        threshold: "10"
```

```yaml
# Deployment 必須設定 resources.requests
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  template:
    spec:
      containers:
        - name: main-app
          image: my-app:latest
          resources:
            requests:
              cpu: "500m"             # 必填！CPU 擴縮器需要此設定
              memory: "256Mi"
            limits:
              cpu: "1000m"
              memory: "512Mi"
```

```bash
# 查看 Pod 的 CPU 使用量
kubectl top pods

# 查看 HPA 狀態（包含當前指標）
kubectl get hpa -w

# 產生 CPU 負載測試
kubectl run stress --rm -it --image=alpine -- \
  sh -c "apk add stress-ng && stress-ng --cpu 2 --timeout 60s"

# 檢查 Metrics Server 是否運作
kubectl top nodes
```

---

## 常見錯誤與提示 (Common Mistakes & Tips)

| 常見錯誤 | 正確做法 |
|----------|----------|
| Pod 沒有設定 resources.requests | 必須在 Deployment 中設定 CPU requests |
| 期望只用 CPU 擴縮器縮到 0 | CPU 擴縮器無法單獨縮到 0，需搭配其他擴縮器 |
| Metrics Server 未安裝 | 確保 Kubernetes 叢集安裝了 Metrics Server |
| 混淆 Utilization 和 AverageValue | Utilization 是百分比，AverageValue 是絕對值 |

### 提示

- CPU 擴縮器與標準 Kubernetes HPA 的行為相同
- 使用 `containerName` 可以只監控多容器 Pod 中的特定容器
- 結合 Cron 擴縮器可實現工作時間 CPU 擴縮、非工作時間縮到 0
- 建議 `value` 設定在 60-80% 之間，保留緩衝空間

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `metricType` | 指標類型 | 必填 |
| `value` | 目標值 | 必填 |
| `containerName` | 特定容器名稱 | 空（整個 Pod）|

| metricType | 說明 | 範例值 |
|------------|------|--------|
| `Utilization` | 相對於 requests 的百分比 | `"70"` (70%) |
| `AverageValue` | 絕對 CPU 數量 | `"500m"` |

| 指令 | 說明 |
|------|------|
| `kubectl top pods` | 查看 Pod CPU 使用量 |
| `kubectl top nodes` | 查看節點 CPU 使用量 |
| `kubectl get hpa` | 查看 HPA 狀態 |
| `kubectl describe hpa <name>` | 查看 HPA 詳細資訊 |
