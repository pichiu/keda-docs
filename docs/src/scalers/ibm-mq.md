
## TL;DR

IBM MQ 擴縮器根據 IBM MQ 佇列深度自動擴縮應用程式。透過 Queue Manager Admin REST API 監控佇列中的訊息數量。

---

## 翻譯 (Translation)

### 觸發器規格

此規格描述 IBM MQ 佇列的 `ibmmq` 觸發器。

```yaml
triggers:
- type: ibmmq
  metadata:
    host: <ibm-host> # 必填 - IBM MQ Queue Manager Admin REST 端點
    queueName: <queue-name> # 必填 - 您的佇列名稱
    queueDepth: <queue-depth> # 可選 - HPA 的佇列深度目標。預設：20 則訊息
    activationQueueDepth: <activation-queue-depth> # 可選 - 啟動佇列深度目標。預設：0 則訊息
    usernameFromEnv: <admin-user> # 可選 - 從環境變數提供管理員使用者名稱
    passwordFromEnv: <admin-password> # 可選 - 從環境變數提供管理員密碼
    unsafeSsl: "false" # 可選 - 用於跳過自簽憑證檢查 'true'。預設：false
```

**參數列表：**

- `host` - IBM MQ Queue Manager Admin REST 端點。IBM Cloud 上的範例 URI 端點結構 `https://example.mq.appdomain.cloud/ibmmq/rest/v2/admin/action/qmgr/QM/mqsc`。
- `queueName`（或 `queueNames`）- 佇列名稱。支援以逗號（`,`）分隔的多個佇列。
- `operation` - 用於計算訊息數量的操作。可以是 `max`（預設）、`sum` 或 `avg`。（可選）
- `queueDepth` - HPA 的佇列深度目標。（預設：`20`，可選）
- `activationQueueDepth` - 啟動擴縮器的目標值。（預設：`0`，可選）
- `usernameFromEnv` - 從環境變數提供管理員使用者名稱，而非作為 Secret。（可選）
- `passwordFromEnv` - 從環境變數提供管理員密碼，而非作為 Secret。（可選）
- `unsafeSsl` - 是否允許不安全的 SSL（值：`true`、`false`，預設：`false`，可選）

### 驗證參數

TriggerAuthentication CRD 用於連線和驗證 IBM MQ：

- `ADMIN_USER` - 必填 - MQ Queue Manager 的 Admin REST 端點使用者名稱。
- `ADMIN_PASSWORD` - 必填 - MQ Queue Manager 的 Admin REST 端點 API 金鑰。
- `ca` - TLS 用戶端驗證的憑證授權單位檔案。（可選）
- `cert` - 用戶端驗證的憑證。（可選）
- `key` - 用戶端驗證的金鑰。（可選）
- `keyPassword` - 如果設定，keyPassword 用於解密提供的金鑰。（可選）

---

## 實作範例 (Practical Example)

使用基本驗證的範例：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: keda-ibmmq-secret
data:
  ADMIN_USER: <encoded-username>
  ADMIN_PASSWORD: <encoded-password>
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: ibmmq-scaledobject
  namespace: default
  labels:
    deploymentName: ibmmq-deployment
spec:
  scaleTargetRef:
    name: ibmmq-deployment
  pollingInterval: 5
  cooldownPeriod: 30
  maxReplicaCount: 18
  triggers:
    - type: ibmmq
      metadata:
        host: <ibm-host>
        queueName: <queue-name>
        queueDepth: "20"
      authenticationRef:
        name: keda-ibmmq-trigger-auth
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: keda-ibmmq-trigger-auth
  namespace: default
spec:
  secretTargetRef:
    - parameter: username
      name: keda-ibmmq-secret
      key: ADMIN_USER
    - parameter: password
      name: keda-ibmmq-secret
      key: ADMIN_PASSWORD
```

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `host` | IBM MQ Admin REST 端點 | 無（必填）|
| `queueName` | 佇列名稱 | 無（必填）|
| `queueDepth` | 佇列深度目標 | 20 |
| `activationQueueDepth` | 啟動閾值 | 0 |
| `operation` | 聚合操作（max/sum/avg）| max |
| `unsafeSsl` | 跳過 SSL 驗證 | false |

| 驗證參數 | 說明 |
|----------|------|
| `username` | 管理員使用者名稱 |
| `password` | 管理員密碼 |
| `ca` | CA 憑證 |
| `cert` | 用戶端憑證 |
| `key` | 用戶端金鑰 |
