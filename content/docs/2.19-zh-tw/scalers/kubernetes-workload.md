+++
title = "Kubernetes Workload"
availability = "v2.4+"
maintainer = "Community"
category = "Apps"
description = "根據符合給定選擇器的執行中 Pod 數量擴縮應用程式。"
go_file = "kubernetes_workload_scaler"
+++

## TL;DR

Kubernetes Workload 擴縮器根據命名空間中符合標籤選擇器的 Pod 數量與擴縮工作負載 Pod 數量之間的比例來進行擴縮。適用於需要維持特定 Pod 比例的場景。

---

## 翻譯 (Translation)

### 觸發器規格

```yaml
triggers:
- type: kubernetes-workload
  metadata:
    podSelector: 'app=backend'
    value: '0.5'
    activationValue: '3.1'
```

**參數列表：**

- `podSelector` - 用於取得 Pod 數量的[標籤選擇器](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors)。支援以逗號（`,`）分隔的多個選擇器。也支援[基於集合的需求](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#set-based-requirement)及其混合使用。
- `value` - 擴縮工作負載與符合選擇器的 Pod 數量之間的目標比例。計算公式為：`比例 = (符合選擇器的 Pod) / (擴縮工作負載的 Pod)`。（此值可以是浮點數）
- `activationValue` - 啟動擴縮器的目標值。在[這裡](./../concepts/scaling-deployments.md#activating-and-scaling-thresholds)了解更多。（預設：`0`，可選，此值可以是浮點數）

> 💡 **注意：** 搜尋範圍限於部署 `ScaledObject` 的命名空間。

計數不包括已終止的 Pod，即 [Pod 狀態](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.22/#podstatus-v1-core) `phase` 等於 `Succeeded` 或 `Failed`。

### 驗證參數

使用 KEDA 自身的身分來列出 Pod，因此不需要額外的設定。

---

## 實作範例 (Practical Example)

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: workload-scaledobject
spec:
  scaleTargetRef:
    name: workload-deployment
  triggers:
  - type: kubernetes-workload
    metadata:
      podSelector: 'app=backend, deploy notin (critical, monolith)'
      value: '3'
```

這個範例會根據標籤為 `app=backend` 且 `deploy` 不在 `critical` 或 `monolith` 中的 Pod 數量，以 3:1 的比例擴縮 `workload-deployment`。

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `podSelector` | Pod 標籤選擇器 | 無（必填）|
| `value` | 目標比例值 | 無（必填）|
| `activationValue` | 啟動閾值 | 0 |

| 選擇器語法 | 說明 |
|------------|------|
| `key=value` | 相等選擇器 |
| `key!=value` | 不相等選擇器 |
| `key in (v1,v2)` | 值在集合中 |
| `key notin (v1,v2)` | 值不在集合中 |
| `key` | 金鑰存在 |
| `!key` | 金鑰不存在 |
