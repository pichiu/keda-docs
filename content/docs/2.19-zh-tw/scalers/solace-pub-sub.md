+++
title = "Solace PubSub+ Event Broker"
availability = "2.4+"
maintainer = "Community"
category = "Messaging"
description = "根據 Solace PubSub+ Event Broker 佇列擴縮應用程式"
go_file = "solace_scaler"
+++

## TL;DR

Solace PubSub+ 擴縮器根據 Solace PubSub+ Event Broker 佇列的指標自動擴縮應用程式。支援訊息數量、訊息 spool 使用量和訊息接收速率等指標。

---

## 翻譯 (Translation)

### 觸發器規格

此規格描述基於 Solace PubSub+ Event Broker 佇列進行擴縮的 `solace-event-queue` 觸發器。

**注意：** 此觸發器僅適用於**保證訊息**（Solace PubSub+ Event Broker 佇列），它提供根據 Solace PubSub+ Event Broker 佇列的指標自動擴縮用戶端實例數量的能力。

```yaml
triggers:
- type: solace-event-queue
  metadata:
    solaceSempBaseURL:                  http://solace_broker:8080
    messageVpn:                         message-vpn
    queueName:                          queue_name
    messageCountTarget:                 '100'
    messageSpoolUsageTarget:            '100'       ### 兆位元組 (MB)
    messageReceiveRateTarget:           '50'        ### 過去 1 分鐘間隔的訊息/秒
    activationMessageCountTarget:       '10'
    activationMessageSpoolUsageTarget:  '5'
    activationMessageReceiveRateTarget: '1'
```

**參數列表：**

- `solaceSempBaseURL` - Solace SEMP 端點，以逗號分隔的主機列表，格式：`<protocol>://<host-or-service>:<port>`。
- `messageVpn` - 託管在 Solace 代理程式上的 Message VPN。
- `queueName` - 要監控的訊息佇列。
- `messageCountTarget` - Pod 可管理的目標訊息數量。如果佇列訊息積壓大於每個活動副本的目標值，擴縮器將增加副本數。
- `activationMessageCountTarget` - 啟動擴縮器的目標訊息數量（從 0->1 或 1->0 副本擴縮）。（預設：`0`，可選）
- `messageSpoolUsageTarget` - 以兆位元組（MB）表示的整數值。Pod 可管理的目標 spool 使用量。
- `activationMessageSpoolUsageTarget` - 啟動擴縮器的目標訊息 spool 積壓（以兆位元組表示儲存在佇列中的資料）。（預設：`0`，可選）
- `messageReceiveRateTarget` - 副本可管理的目標訊息/秒數量。
- `activationMessageReceiveRateTarget` - 啟動擴縮器的每秒傳遞到佇列的目標訊息數量。（預設：`0`，可選）
- `username` - 具有 Solace SEMP RESTful 端點存取權限的使用者帳戶。
- `password` - 使用者帳戶的密碼。

**參數要求：**
- 解析目標佇列的參數都是**必填**的：`solaceSempBaseURL`、`messageVpn`、`queueName`
- **至少**需要 `messageCountTarget`、`messageSpoolUsageTarget` 或 `messageReceiveRateTarget` 之一。

### 驗證參數

- `username` - 必填。用於連線到 Solace PubSub+ Event Broker 的 SEMP 端點的使用者名稱。
- `password` - 必填。用於連線到 Solace PubSub+ Event Broker 的 SEMP 端點的密碼。

---

## 實作範例 (Practical Example)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: solace-secret
  namespace: solace
type: Opaque
data:
  SEMP_USER: YWRtaW4=
  SEMP_PASSWORD: S2VkYUxhYkFkbWluUHdkMQ==
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: solace-scaled-object
  namespace: solace
spec:
  scaleTargetRef:
    name: solace-consumer
  pollingInterval: 20
  cooldownPeriod: 60
  minReplicaCount: 0
  maxReplicaCount: 10
  triggers:
  - type: solace-event-queue
    metadata:
      solaceSempBaseURL: http://broker-pubsubplus.solace.svc.cluster.local:8080
      messageVpn: test_vpn
      queueName: SCALED_CONSUMER_QUEUE1
      messageCountTarget: '50'
      messageSpoolUsageTarget: '100000'
      messageReceiveRateTarget: '20'
      activationMessageCountTarget: '5'
    authenticationRef:
      name: solace-trigger-auth
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: solace-trigger-auth
  namespace: solace
spec:
  secretTargetRef:
    - parameter: username
      name: solace-secret
      key: SEMP_USER
    - parameter: password
      name: solace-secret
      key: SEMP_PASSWORD
```

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `solaceSempBaseURL` | SEMP 端點 URL | 無（必填）|
| `messageVpn` | Message VPN | 無（必填）|
| `queueName` | 佇列名稱 | 無（必填）|
| `messageCountTarget` | 訊息數量目標 | 無 |
| `messageSpoolUsageTarget` | Spool 使用量目標（MB）| 無 |
| `messageReceiveRateTarget` | 訊息接收速率目標 | 無 |

| 指標類型 | 說明 |
|----------|------|
| `messageCountTarget` | 基於佇列積壓的反應式擴縮 |
| `messageReceiveRateTarget` | 基於訊息接收速率的穩定擴縮 |
| `messageSpoolUsageTarget` | 基於儲存使用量的擴縮 |
