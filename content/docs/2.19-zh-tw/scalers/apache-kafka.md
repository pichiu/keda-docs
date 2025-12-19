+++
title = "Apache Kafka"
availability = "v1.0+"
maintainer = "Microsoft"
category = "Messaging"
description = "根據 Apache Kafka 主題或其他支援 Kafka 協定的服務擴縮應用程式。"
go_file = "kafka_scaler"
+++

## TL;DR

Apache Kafka 擴縮器根據 Consumer Group 的 lag（延遲）來擴縮應用程式。預設情況下，副本數不會超過主題的分割區（Partition）數量。支援多種 SASL 驗證模式和 TLS。

---

## 翻譯 (Translation)

> **注意：**
> - 預設情況下，副本數不會超過：
>   - 指定主題時的分割區數量；
>   - 未指定主題時 Consumer Group 中*所有主題*的分割區數量；
>   - `ScaledObject`/`ScaledJob` 中指定的 `maxReplicaCount`；
>   - 如果 `limitToPartitionsWithLag` 設為 `true`，則為有非零 lag 的分割區數量
> - 這是因為如果 Consumer 數量超過主題的分割區數量，額外的 Consumer 將處於閒置狀態。

### 觸發器規格

此規格描述 Apache Kafka 主題的 `kafka` 觸發器。

```yaml
triggers:
- type: kafka
  metadata:
    bootstrapServers: kafka.svc:9092     # Kafka broker 列表
    consumerGroup: my-group              # Consumer Group 名稱
    topic: test-topic                    # 主題名稱（可選）
    lagThreshold: '5'                    # Lag 閾值（預設：10）
    activationLagThreshold: '3'          # 啟動閾值（預設：0）
    offsetResetPolicy: latest            # offset 重設策略（latest/earliest）
    allowIdleConsumers: 'false'          # 允許閒置 Consumer
    scaleToZeroOnInvalidOffset: 'false'  # 無效 offset 時縮到 0
    excludePersistentLag: 'false'        # 排除持久 lag
    limitToPartitionsWithLag: 'false'    # 限制為有 lag 的分割區
    version: 1.0.0                       # Kafka 版本
    sasl: plaintext                      # SASL 模式
    tls: enable                          # TLS 設定
```

**參數列表：**

- `bootstrapServers` - Kafka broker 的「hostname:port」逗號分隔列表。
- `consumerGroup` - 用於檢查主題 offset 和處理相關 lag 的 Consumer Group 名稱。
- `topic` - 處理 offset lag 的主題名稱。（可選，見下方注意事項）
- `lagThreshold` - 觸發擴縮動作的總 lag 目標值（所有分割區 lag 的總和）。（預設：`10`，可選）
- `activationLagThreshold` - 啟動擴縮器的目標值。（預設：`0`，可選）
- `offsetResetPolicy` - Consumer 的 offset 重設策略。（值：`latest`、`earliest`，預設：`latest`，可選）
- `allowIdleConsumers` - 設為 `true` 時，副本數可以超過主題的分割區數量，允許閒置 Consumer。（預設：`false`，可選）
- `scaleToZeroOnInvalidOffset` - 控制當分割區沒有有效 offset 時擴縮器的行為。
- `excludePersistentLag` - 設為 `true` 時，擴縮器會排除當前 offset 與上次輪詢相同的分割區 lag。
- `limitToPartitionsWithLag` - 設為 `true` 時，副本數不會超過有非零 lag 的分割區數量。
- `version` - Kafka broker 版本。（預設：`1.0.0`，可選）
- `sasl` - Kafka SASL 驗證模式。（值：`plaintext`、`scram_sha256`、`scram_sha512`、`gssapi`、`oauthbearer` 或 `none`，預設：`none`，可選）
- `tls` - 啟用 Kafka SSL 驗證。（值：`enable`、`disable`，預設：`disable`，可選）

### 驗證參數

**SASL 驗證：**
- `sasl` - SASL 驗證模式
- `username` - SASL 驗證的使用者名稱
- `password` - SASL 驗證的密碼

**TLS 驗證：**
- `tls` - 設為 `enable` 啟用 SSL
- `ca` - TLS 客戶端驗證的 CA 憑證
- `cert` - 客戶端驗證的憑證
- `key` - 客戶端驗證的金鑰

**AWS MSK IAM 驗證：**
- `awsAccessKeyID` - AWS 存取金鑰 ID
- `awsSecretAccessKey` - AWS 秘密存取金鑰
- `awsRegion` - AWS 區域

---

## 說明 (Explanation)

### Lag 計算

```
Lag = 最新 Offset - 當前消費 Offset
```

| 參數 | 說明 |
|------|------|
| `lagThreshold` | 每個 Pod 處理的目標 lag |
| 副本數公式 | 總 Lag / lagThreshold |

### 分割區與副本數關係

預設情況下：
- 10 個分割區 + 100 lag + lagThreshold=10 → 10 個副本（受分割區數限制）
- 設定 `allowIdleConsumers: true` 可突破此限制

### SASL 模式

| 模式 | 說明 |
|------|------|
| `plaintext` | 純文字使用者名稱/密碼 |
| `scram_sha256` | SCRAM-SHA-256 |
| `scram_sha512` | SCRAM-SHA-512 |
| `gssapi` | Kerberos |
| `oauthbearer` | OAuth 2.0 |

---

## 實作範例 (Practical Example)

```yaml
# 基本範例（無驗證）
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-scaledobject
spec:
  scaleTargetRef:
    name: my-consumer-deployment
  pollingInterval: 30
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka:9092
        consumerGroup: my-group         # 與消費者使用相同的 Consumer Group
        topic: my-topic
        lagThreshold: "50"
        offsetResetPolicy: latest
```

```yaml
# 使用 SASL/TLS 驗證
apiVersion: v1
kind: Secret
metadata:
  name: kafka-secrets
data:
  sasl: cGxhaW50ZXh0           # base64: plaintext
  username: YWRtaW4=           # base64: admin
  password: cGFzc3dvcmQ=       # base64: password
  tls: ZW5hYmxl                # base64: enable
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: kafka-auth
spec:
  secretTargetRef:
    - parameter: sasl
      name: kafka-secrets
      key: sasl
    - parameter: username
      name: kafka-secrets
      key: username
    - parameter: password
      name: kafka-secrets
      key: password
    - parameter: tls
      name: kafka-secrets
      key: tls
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-scaledobject
spec:
  scaleTargetRef:
    name: my-consumer-deployment
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka:9092
        consumerGroup: my-group
        topic: my-topic
        lagThreshold: "50"
      authenticationRef:
        name: kafka-auth
```

```bash
# 查看 Consumer Group lag
kubectl exec -it kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server kafka:9092 \
  --describe --group my-group

# 查看 ScaledObject 狀態
kubectl describe scaledobject kafka-scaledobject
```

---

## 常見錯誤與提示 (Common Mistakes & Tips)

| 常見錯誤 | 正確做法 |
|----------|----------|
| Consumer Group 名稱與實際消費者不符 | 確保 ScaledObject 中的 consumerGroup 與應用程式相同 |
| 副本數始終為分割區數 | 這是預設行為，設定 `allowIdleConsumers: true` 可改變 |
| 新 Consumer 未開始消費 | 檢查 `offsetResetPolicy` 設定（latest vs earliest）|
| 未指定 topic 導致無法擴縮 | 指定 topic 或確保 Consumer 已有提交記錄 |

### 提示

- 使用 `ensureEvenDistributionOfPartitions: true` 確保分割區均勻分配
- 設定 `excludePersistentLag: true` 可排除無法消費的訊息造成的 lag
- AWS MSK 使用 IAM 驗證時需設定 `saslTokenProvider: aws_msk_iam`
- 監控 Consumer lag 可使用 `kafka-consumer-groups.sh` 工具

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `bootstrapServers` | Kafka broker 列表 | 必填 |
| `consumerGroup` | Consumer Group 名稱 | 必填 |
| `topic` | 主題名稱 | 可選 |
| `lagThreshold` | Lag 閾值 | 10 |
| `activationLagThreshold` | 啟動閾值 | 0 |
| `offsetResetPolicy` | Offset 重設策略 | latest |
| `sasl` | SASL 模式 | none |
| `tls` | TLS 設定 | disable |

| 指令 | 說明 |
|------|------|
| `kubectl get scaledobject` | 列出 ScaledObject |
| `kafka-consumer-groups.sh --describe --group <name>` | 查看 Consumer Group 詳細資訊 |
