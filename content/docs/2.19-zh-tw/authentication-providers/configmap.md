+++
title = "ConfigMap"
+++

## TL;DR

ConfigMap 驗證提供者允許您從 Kubernetes ConfigMap 中拉取非機密設定資訊到 KEDA 觸發器中，適用於連線字串或設定參數等資料。

---

## 翻譯 (Translation)

您可以透過定義 Kubernetes `ConfigMap` 的 `name` 和要使用的 `key` 來將資訊拉取到觸發器中。

```yaml
configMapTargetRef:                       # 可選。
  - parameter: connectionString           # 必填 - 由擴縮觸發器定義
    name: my-keda-configmap-resource-name # 必填。
    key: azure-storage-connectionstring   # 必填。
```

**假設：** 除非另有指定，否則 `namespace` 與 ScaledObject 中 `scaleTargetRef.name` 參照的資源位於相同的命名空間。

---

## 實作範例 (Practical Example)

```yaml
# 建立 ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-keda-configmap-resource-name
  namespace: default
data:
  azure-storage-connectionstring: "DefaultEndpointsProtocol=https;AccountName=..."

---
# 使用 ConfigMap 的 TriggerAuthentication
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: azure-queue-auth
  namespace: default
spec:
  configMapTargetRef:
    - parameter: connection
      name: my-keda-configmap-resource-name
      key: azure-storage-connectionstring
```

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `parameter` | 觸發器使用的參數名稱 | 無（必填）|
| `name` | ConfigMap 資源名稱 | 無（必填）|
| `key` | ConfigMap 中的金鑰名稱 | 無（必填）|

| 注意事項 | 說明 |
|----------|------|
| 命名空間 | 預設與 ScaledObject 相同命名空間 |
| 用途 | 適用於非機密設定資料 |
| 建議 | 機密資料應使用 Secret 而非 ConfigMap |
