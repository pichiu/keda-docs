
## TL;DR

KEDA Metrics Server 負責將擴縮器的指標暴露給 Kubernetes API 伺服器。本文件說明如何查詢這些指標以及如何從 ScaledObject 取得指標名稱。

---

## 翻譯 (Translation)

### 查詢 KEDA Metrics Server 暴露的指標

KEDA Metrics Server 暴露的指標可以直接使用 `kubectl` 查詢：

```bash
kubectl get --raw "/apis/external.metrics.k8s.io/v1beta1"
```

這將返回 KEDA 暴露的指標列表（僅外部指標）的 JSON：

```json
{
  "kind": "APIResourceList",
  "apiVersion": "v1",
  "groupVersion": "external.metrics.k8s.io/v1beta1",
  "resources": [
    {
      "name": "externalmetrics",
      "singularName": "",
      "namespaced": true,
      "kind": "ExternalMetricValueList",
      "verbs": [
        "get"
      ]
    }
  ]
}
```

若要查詢特定指標值，也可以使用 `kubectl`：

```bash
kubectl get --raw "/apis/external.metrics.k8s.io/v1beta1/namespaces/YOUR_NAMESPACE/YOUR_METRIC_NAME?labelSelector=scaledobject.keda.sh%2Fname%3D{SCALED_OBJECT_NAME}"
```

此時，您應該考慮到 KEDA 指標是有命名空間的，這意味著您必須指定 `ScaledObject` 所在的命名空間。

例如，如果您想取得名為 `s1-rabbitmq-queueName2` 的指標值，該指標由命名空間 `sample-ns` 中名為 `my-scaled-object` 的 ScaledObject 使用，查詢將如下：

```bash
kubectl get --raw "/apis/external.metrics.k8s.io/v1beta1/namespaces/sample-ns/s1-rabbitmq-queueName2?labelSelector=scaledobject.keda.sh%2Fname%3Dmy-scaled-object"
```

這將顯示如下 JSON：

```json
{
  "kind": "ExternalMetricValueList",
  "apiVersion": "external.metrics.k8s.io/v1beta1",
  "metadata": {},
  "items": [
    {
      "metricName": "s1-rabbitmq-queueName2",
      "metricLabels": null,
      "timestamp": "2021-10-20T10:48:17Z",
      "value": "0"
    }
  ]
}
```

> **注意：** 查詢指標有 2 個例外，即 `cpu` 和 `memory` 擴縮器。當 KEDA 建立 HPA 物件時，它使用 Kubernetes Metrics Server 的標準 `cpu` 和 `memory` 指標。如果您想查詢這 2 個特定值，應使用 `/apis/metrics.k8s.io/v1beta1` 而不是 `/apis/external.metrics.k8s.io/v1beta1`。

### 如何從 ScaledObject 取得指標名稱

在運作期間，KEDA 會使用一些需要的相關資訊更新每個 ScaledObject。這些資訊的一部分是從 ScaledObject 內部的觸發器產生的指標名稱。

您可以使用 `kubectl` 從 ScaledObject 取得指標名稱：

```bash
kubectl get scaledobject SCALEDOBJECT_NAME -n NAMESPACE -o jsonpath={.status.externalMetricNames}
```

---

## 實作範例 (Practical Example)

```bash
# 查詢所有外部指標 API
kubectl get --raw "/apis/external.metrics.k8s.io/v1beta1"

# 查詢特定命名空間的指標
kubectl get --raw "/apis/external.metrics.k8s.io/v1beta1/namespaces/default"

# 查詢特定 ScaledObject 的指標
kubectl get --raw "/apis/external.metrics.k8s.io/v1beta1/namespaces/default/s0-kafka-mytopic?labelSelector=scaledobject.keda.sh%2Fname%3Dkafka-scaledobject"

# 取得 ScaledObject 的指標名稱
kubectl get scaledobject kafka-scaledobject -n default -o jsonpath='{.status.externalMetricNames}'

# 查看 ScaledObject 完整狀態
kubectl get scaledobject kafka-scaledobject -n default -o yaml
```

---

## 快速參考 (Quick Reference)

| API 端點 | 說明 |
|----------|------|
| `/apis/external.metrics.k8s.io/v1beta1` | 列出所有外部指標 |
| `/apis/external.metrics.k8s.io/v1beta1/namespaces/{ns}` | 列出命名空間中的指標 |
| `/apis/external.metrics.k8s.io/v1beta1/namespaces/{ns}/{metric}` | 查詢特定指標 |
| `/apis/metrics.k8s.io/v1beta1` | CPU/Memory 指標（標準 Metrics Server）|

| 指令 | 說明 |
|------|------|
| `kubectl get --raw "/apis/external.metrics.k8s.io/v1beta1"` | 查詢外部指標 API |
| `kubectl get scaledobject -o jsonpath='{.status.externalMetricNames}'` | 取得指標名稱 |
