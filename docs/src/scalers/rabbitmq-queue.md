
## TL;DR

RabbitMQ 擴縮器根據 RabbitMQ 佇列的訊息數量、發布速率或消費速率來擴縮應用程式。支援 AMQP 和 HTTP 兩種協定，提供多種觸發模式（QueueLength、MessageRate、DeliverGetRate 等）以適應不同場景。

---

## 翻譯 (Translation)

### 觸發器規格

此規格描述 RabbitMQ 佇列的 `rabbitmq` 觸發器。

```yaml
triggers:
- type: rabbitmq
  metadata:
    host: amqp://<host>:<port>/<vhost> # 可選。如未指定，必須使用 TriggerAuthentication。
    protocol: auto # 可選。指定使用的協定，`amqp`、`http` 或 `auto`（根據 `host` 值自動偵測）。預設值為 `auto`。
    mode: QueueLength # 支援的模式有 `QueueLength`、`MessageRate`、`DeliverGetRate`、`PublishedToDeliveredRatio` 或 `ExpectedQueueConsumptionTime`
    value: "100.50" # 定義閾值（每個實例），超過時觸發擴縮。
    activationValue: "10.5" # 可選。啟動閾值，預設為 0。
    queueName: testqueue # 佇列名稱。
    vhostName: / # 可選。如未指定，使用 `host` 連接字串中的 VHost。Azure AD Workload Identity 授權時為必填。
    hostFromEnv: RABBITMQ_HOST # 可選。可用來替代 `host` 參數。
    usernameFromEnv: RABBITMQ_USERNAME # 可選。可用來替代 `TriggerAuthentication`。
    passwordFromEnv: RABBITMQ_PASSWORD # 可選。可用來替代 `TriggerAuthentication`。
    unsafeSsl: true # 可選。是否允許不安全的 SSL 連接。
    timeout: 1000 # 可選。此擴縮器使用的 HTTP 客戶端自訂逾時。
```

**參數列表：**

- `host` - RabbitMQ 主機位址，格式為 `<protocol>://<host>:<port>/vhost`。
- `queueName` - 要讀取資訊和統計的佇列名稱。
- `mode` - 觸發模式。可以是以下之一：
  - `QueueLength` - 根據佇列中的訊息數量觸發
  - `MessageRate` - 根據佇列報告的發布速率觸發
  - `DeliverGetRate` - 根據已傳遞訊息的速率觸發
  - `PublishedToDeliveredRatio` - 根據 MessageRate 與 DeliverGetRate 的比值觸發
  - `ExpectedQueueConsumptionTime` - 根據預期消費所有可用訊息的時間（秒）觸發
- `value` - 定義閾值（每個實例），超過時觸發擴縮（根據觸發模式可以是浮點數）。
- `activationValue` - 啟動擴縮器的目標值（預設：`0`；可選；可以是浮點數）。
- `protocol` - 通訊協定（值：`auto`、`http`、`amqp`；預設：`auto`；可選）。

### 選擇正確的觸發模式

- `QueueLength` - 主要用於擴縮訊息消費服務，保持佇列中的訊息數量在選定的標稱水平。
- `MessageRate` - 主要用於擴縮訊息消費服務，保持消費速率達到或超過訊息發布速率。
- `DeliverGetRate` - 可用於擴展訊息發布服務以增加發布速率。
- `PublishedToDeliveredRatio` - 用於控制消費者 Pod 擴縮，保持發布和消費速率穩定。
- `ExpectedQueueConsumptionTime` - 用於擴縮消費者 Pod，使用傳遞所有可用訊息的預估待處理時間。

### 驗證參數

`TriggerAuthentication` CRD 用於連接和驗證 RabbitMQ：
- AMQP URI 格式：`amqp://guest:password@localhost:5672/vhost`
- HTTP URI 格式：`http://guest:password@localhost:15672/path/vhost`

#### 使用者名稱和密碼驗證

- `username` - 連接到代理管理端點的使用者名稱。
- `password` - 連接到代理管理端點的密碼。

#### TLS 驗證

- `tls` - 設定為 `enable` 啟用 RabbitMQ 的 SSL 驗證。
- `ca` - TLS 客戶端驗證的憑證授權單位檔案。
- `cert` - 客戶端驗證的憑證。
- `key` - 客戶端驗證的憑證金鑰。

---

## 說明 (Explanation)

### 觸發模式比較

| 模式 | 適用場景 | 指標類型 |
|------|----------|----------|
| QueueLength | 消費服務，控制佇列長度 | 訊息數量 |
| MessageRate | 消費服務，跟上發布速率 | 每秒訊息數 |
| DeliverGetRate | 發布服務，增加發布速率 | 每秒傳遞數 |
| PublishedToDeliveredRatio | 消費服務，保持穩定比率 | 比率值 |
| ExpectedQueueConsumptionTime | 消費服務，控制處理時間 | 秒 |

### 協定選擇

| 協定 | 埠口 | 功能 | 使用場景 |
|------|------|------|----------|
| AMQP | 5672 | 基本佇列長度 | 簡單場景 |
| HTTP | 15672 | 完整統計 API | 進階指標、正則匹配 |

---

## 實作範例 (Practical Example)

```yaml
# 建立 Secret
apiVersion: v1
kind: Secret
metadata:
  name: rabbitmq-secret
data:
  host: YW1xcDovL2d1ZXN0OnBhc3N3b3JkQHJhYmJpdG1xOjU2NzIv  # base64 編碼

---
# 建立 TriggerAuthentication
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: rabbitmq-auth
spec:
  secretTargetRef:
    - parameter: host
      name: rabbitmq-secret
      key: host

---
# 建立 ScaledObject
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: rabbitmq-scaledobject
spec:
  scaleTargetRef:
    name: my-consumer-deployment   # 消費者 Deployment
  pollingInterval: 15               # 每 15 秒檢查一次
  cooldownPeriod: 300               # 5 分鐘冷卻期
  minReplicaCount: 0                # 可縮減到 0
  maxReplicaCount: 30               # 最多 30 個副本
  triggers:
    - type: rabbitmq
      metadata:
        protocol: amqp              # 使用 AMQP 協定
        queueName: my-queue         # 佇列名稱
        mode: QueueLength           # 根據佇列長度擴縮
        value: "10"                 # 每 10 個訊息一個 Pod
      authenticationRef:
        name: rabbitmq-auth
```

```bash
# 建立 RabbitMQ Secret
kubectl create secret generic rabbitmq-secret \
  --from-literal=host='amqp://guest:password@rabbitmq:5672/'

# 查看 ScaledObject 狀態
kubectl get scaledobject rabbitmq-scaledobject

# 查看產生的 HPA
kubectl get hpa keda-hpa-rabbitmq-scaledobject

# 手動發送測試訊息到 RabbitMQ（測試擴縮）
kubectl run rabbitmq-test --rm -it --image=rabbitmq:3 -- \
  rabbitmqadmin -H rabbitmq -u guest -p password publish \
  exchange=amq.default routing_key=my-queue payload="test"
```

---

## 常見錯誤與提示 (Common Mistakes & Tips)

| 常見錯誤 | 正確做法 |
|----------|----------|
| 使用 HTTP 功能但設定 AMQP 協定 | MessageRate、regex 等功能需要 HTTP 協定 |
| host 字串缺少尾隨斜線 | 沒有 vhost 時需要尾隨斜線，如 `amqp://host:5672/` |
| 密碼包含特殊字元導致解析錯誤 | 使用 TriggerAuthentication 分離密碼 |
| 未考慮未確認訊息 | 使用 HTTP 協定才能計算未確認訊息 |

### 提示

- 如需計算未確認訊息（unacked），必須使用 HTTP 協定
- `useRegex` 功能只能與 HTTP 協定一起使用
- 對於 `DeliverGetRate` 和 `PublishedToDeliveredRatio`，建議設定 `metricType: Value`
- 使用 `ExpectedQueueConsumptionTime` 時，建議將 `pollingInterval` 設定為 1 秒

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `host` | RabbitMQ 連接 URI | 無（必填或使用 TriggerAuth）|
| `queueName` | 佇列名稱 | 無（必填）|
| `mode` | 觸發模式 | QueueLength |
| `value` | 擴縮閾值 | 無（必填）|
| `protocol` | 通訊協定 | auto |
| `activationValue` | 啟動閾值 | 0 |
| `unsafeSsl` | 允許不安全 SSL | false |

| 指令 | 說明 |
|------|------|
| `kubectl get scaledobject` | 列出 ScaledObject |
| `kubectl describe scaledobject <name>` | 查看詳細狀態 |
| `kubectl get hpa` | 查看產生的 HPA |
