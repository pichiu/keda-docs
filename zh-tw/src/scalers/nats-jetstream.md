
## TL;DR

NATS JetStream 擴縮器根據 NATS JetStream 串流中的消費者延遲（lag）自動擴縮應用程式。透過監控端點取得延遲指標。

---

## 翻譯 (Translation)

### 觸發器規格

此規格描述 NATS JetStream 的 `nats-jetstream` 觸發器。

```yaml
triggers:
- type: nats-jetstream
  metadata:
    natsServerMonitoringEndpoint: "nats.nats.svc.cluster.local:8222"
    account: "$G"
    stream: "mystream"
    consumer: "pull_consumer"
    lagThreshold: "10"
    activationLagThreshold: "15"
    useHttps: "false"
```

**參數列表：**

- `natsServerMonitoringEndpoint` - NATS 伺服器監控端點的位置。
- `account` - NATS 帳戶名稱。未設定帳戶時，預設為 "$G"。
- `stream` - 帳戶內 JS 串流的名稱。
- `consumer` - 給定串流的消費者名稱。
- `lagThreshold` - 觸發擴縮動作的平均目標值。
- `activationLagThreshold` - 啟動擴縮器的目標值。在[這裡](./../concepts/scaling-deployments.md#activating-and-scaling-thresholds)了解更多。（預設：`0`，可選）
- `useHttps` - 指定 NATS 伺服器監控端點是否使用 HTTPS。（預設：`false`，可選）

### 驗證參數

JetStream 詳細資訊的某些參數可以從 `TriggerAuthentication` 物件中拉取：

- `natsServerMonitoringEndpoint` - NATS Streaming 監控端點的位置。
- `account` - NATS 帳戶名稱。未設定帳戶時，預設為 "$G"。

---

## 實作範例 (Practical Example)

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: nats-jetstream-scaledobject
  namespace: nats-jetstream
spec:
  pollingInterval: 3   # 可選。預設：30 秒
  cooldownPeriod: 10   # 可選。預設：300 秒
  minReplicaCount: 0   # 可選。預設：0
  maxReplicaCount: 2   # 可選。預設：100
  scaleTargetRef:
    name: sub
  triggers:
  - type: nats-jetstream
    metadata:
      natsServerMonitoringEndpoint: "nats.nats.svc.cluster.local:8222"
      account: "$G"
      stream: "mystream"
      consumer: "pull_consumer"
      lagThreshold: "10"
      activationLagThreshold: "15"
      useHttps: "false"
```

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `natsServerMonitoringEndpoint` | NATS 監控端點 | 無（必填）|
| `account` | NATS 帳戶 | $G |
| `stream` | 串流名稱 | 無（必填）|
| `consumer` | 消費者名稱 | 無（必填）|
| `lagThreshold` | 延遲閾值 | 無（必填）|
| `activationLagThreshold` | 啟動閾值 | 0 |
| `useHttps` | 使用 HTTPS | false |
