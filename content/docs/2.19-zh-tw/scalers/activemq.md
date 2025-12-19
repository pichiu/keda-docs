+++
title = "ActiveMQ"
availability = "v2.6+"
maintainer = "Community"
category = "Messaging"
description = "根據 ActiveMQ 佇列擴縮應用程式。"
go_file = "activemq_scaler"
+++

## TL;DR

ActiveMQ 擴縮器根據 ActiveMQ 佇列中的訊息數量自動擴縮應用程式。透過 REST API 輪詢佇列長度，需要基本驗證。

---

## 翻譯 (Translation)

### 觸發器規格

此規格描述基於 ActiveMQ 佇列進行擴縮的 `activemq` 觸發器。

```yaml
triggers:
- type: activemq
  metadata:
    managementEndpoint: "activemq.activemq-test:8161"
    destinationName: "testQueue"
    brokerName: "activemq_broker"
    targetQueueSize: "100"
    activationTargetQueueSize: "10"
```

**參數列表：**

- `managementEndpoint` - ActiveMQ 管理端點，格式：`<hostname>:<port>`。
- `destinationName` - 要檢查訊息數量的佇列名稱。
- `brokerName` - ActiveMQ 中定義的代理程式名稱。
- `targetQueueSize` - 傳遞給擴縮器的佇列長度目標值。如果每個活動副本的佇列訊息數量大於目標值，擴縮器將增加副本數。（預設：`10`，可選）
- `activationTargetQueueSize` - 啟動擴縮器的目標值。在[這裡](./../concepts/scaling-deployments.md#activating-and-scaling-thresholds)了解更多關於啟動的資訊。（預設：`0`，可選）
- `restAPITemplate` - 用於建構取得佇列大小的 REST API URL 的範本。（預設：`"http://{{.ManagementEndpoint}}/api/jolokia/read/org.apache.activemq:type=Broker,brokerName={{.BrokerName}},destinationType=Queue,destinationName={{.DestinationName}}/QueueSize"`，可選）
- `corsHeader` - 用於 CORS 過濾的 Origin 標頭欄位值。（預設：`"http://<<managmentEndpoint>>"`，可選）

**參數需求：**

- 如果未使用 `restAPITemplate` 參數，解析 REST API 範本的參數都是**必填**的：`managementEndpoint`、`destinationName`、`brokerName`。
- ActiveMQ 擴縮器透過輪詢 ActiveMQ REST API 來監控目標佇列的訊息數量。目前擴縮器支援基本驗證。`username` 和 `password` 是**必填**的。請參見下方的[驗證參數](#驗證參數)。

### 驗證參數

您可以透過 `TriggerAuthentication` 設定使用使用者名稱和密碼進行驗證。

**基於使用者名稱和密碼的驗證：**

- `username` - 連線到 ActiveMQ 管理端點的使用者名稱。
- `password` - 連線到 ActiveMQ 管理端點的密碼。

---

## 實作範例 (Practical Example)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: activemq-secret
type: Opaque
data:
  activemq-password: ACTIVEMQ_PASSWORD
  activemq-username: ACTIVEMQ_USERNAME
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: trigger-auth-activemq
spec:
  secretTargetRef:
  - parameter: username
    name: activemq-secret
    key: activemq-username
  - parameter: password
    name: activemq-secret
    key: activemq-password
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: activemq-scaledobject
spec:
  scaleTargetRef:
    name: nginx-deployment
  triggers:
  - type: activemq
    metadata:
      managementEndpoint: "activemq.activemq-test:8161"
      destinationName: "testQ"
      brokerName: "localhost"
      targetQueueSize: "50"
    authenticationRef:
      name: trigger-auth-activemq
```

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `managementEndpoint` | ActiveMQ 管理端點 | 無（必填）|
| `destinationName` | 佇列名稱 | 無（必填）|
| `brokerName` | 代理程式名稱 | 無（必填）|
| `targetQueueSize` | 目標佇列大小 | 10 |
| `activationTargetQueueSize` | 啟動閾值 | 0 |
| `username` | 驗證使用者名稱 | 無（必填）|
| `password` | 驗證密碼 | 無（必填）|
