+++
title = "環境變數"
+++

## TL;DR

環境變數驗證提供者允許您從容器的環境變數中拉取資訊到 KEDA 觸發器中，適用於已透過環境變數注入的設定值。

---

## 翻譯 (Translation)

您可以透過提供給定 `containerName` 的變數 `name` 來從一個或多個環境變數中拉取資訊。

```yaml
env:                              # 可選。
  - parameter: region             # 必填 - 由擴縮觸發器定義
    name: my-env-var              # 必填。
    containerName: my-container   # 可選。預設：ScaledObject 的 scaleTargetRef.envSourceContainerName
```

**假設：** 除非另有指定，否則 `containerName` 與 ScaledObject 中 `scaleTargetRef.name` 參照的資源位於相同的資源中。

---

## 實作範例 (Practical Example)

```yaml
# Deployment 包含環境變數
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  template:
    spec:
      containers:
      - name: my-container
        env:
        - name: AWS_REGION
          value: "ap-northeast-1"
        - name: QUEUE_URL
          value: "https://sqs.ap-northeast-1.amazonaws.com/..."

---
# 使用環境變數的 TriggerAuthentication
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: env-var-auth
  namespace: default
spec:
  env:
    - parameter: awsRegion
      name: AWS_REGION
      containerName: my-container
    - parameter: queueURL
      name: QUEUE_URL
      containerName: my-container
```

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `parameter` | 觸發器使用的參數名稱 | 無（必填）|
| `name` | 環境變數名稱 | 無（必填）|
| `containerName` | 容器名稱 | scaleTargetRef.envSourceContainerName |

| 注意事項 | 說明 |
|----------|------|
| 容器來源 | 環境變數從指定容器中讀取 |
| 預設容器 | 如未指定則使用 ScaledObject 中定義的容器 |
| 適用場景 | 已透過環境變數注入設定的應用程式 |
