
## TL;DR

透過定義 Kubernetes Secret 的名稱和鍵，可以將一個或多個 Secret 值拉取到 KEDA 觸發器中。這是最常用的驗證方式，適合大多數場景。

---

## 翻譯 (Translation)

您可以透過定義 Kubernetes Secret 的 `name` 和要使用的 `key` 來將一個或多個 Secret 拉取到觸發器中。

```yaml
secretTargetRef:                          # 可選。
  - parameter: connectionString           # 必填 - 由擴縮觸發器定義
    name: my-keda-secret-entity           # 必填。
    key: azure-storage-connectionstring   # 必填。
```

**假設：** `namespace` 與 ScaledObject 中 `scaleTargetRef.name` 引用的資源在同一命名空間，除非另有指定。

---

## 說明 (Explanation)

### 參數說明

| 參數 | 必填 | 說明 |
|------|------|------|
| `parameter` | 是 | 擴縮觸發器期望的參數名稱 |
| `name` | 是 | Kubernetes Secret 的名稱 |
| `key` | 是 | Secret 中要讀取的鍵名 |

### 使用場景

- 連接字串（如資料庫、訊息佇列）
- API 金鑰
- 密碼
- 憑證

---

## 實作範例 (Practical Example)

```yaml
# 1. 建立 Kubernetes Secret
apiVersion: v1
kind: Secret
metadata:
  name: rabbitmq-credentials           # Secret 名稱
  namespace: default
type: Opaque
data:
  # base64 編碼的連接字串
  connection-string: YW1xcDovL3VzZXI6cGFzc3dvcmRAcmFiYml0bXE6NTY3Mg==

---
# 2. 建立 TriggerAuthentication 引用 Secret
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: rabbitmq-auth
  namespace: default
spec:
  secretTargetRef:
    - parameter: host                  # RabbitMQ 擴縮器需要的參數
      name: rabbitmq-credentials       # 上面建立的 Secret 名稱
      key: connection-string           # Secret 中的鍵名

---
# 3. 在 ScaledObject 中引用 TriggerAuthentication
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: rabbitmq-scaledobject
  namespace: default
spec:
  scaleTargetRef:
    name: my-deployment
  triggers:
    - type: rabbitmq
      metadata:
        queueName: my-queue
        queueLength: "10"
      authenticationRef:
        name: rabbitmq-auth            # 引用 TriggerAuthentication
```

```bash
# 建立 Secret 的快速方式
kubectl create secret generic rabbitmq-credentials \
  --from-literal=connection-string='amqp://user:password@rabbitmq:5672'

# 查看 Secret 內容（解碼）
kubectl get secret rabbitmq-credentials -o jsonpath='{.data.connection-string}' | base64 -d

# 驗證 TriggerAuthentication 設定
kubectl describe triggerauthentication rabbitmq-auth
```

---

## 常見錯誤與提示 (Common Mistakes & Tips)

| 常見錯誤 | 正確做法 |
|----------|----------|
| Secret 和 TriggerAuthentication 在不同命名空間 | 確保它們在同一命名空間 |
| Secret 的 data 未進行 base64 編碼 | 使用 `kubectl create secret` 會自動編碼 |
| 鍵名拼寫錯誤 | 仔細確認 Secret 中的鍵名 |
| 忘記在觸發器中新增 authenticationRef | 每個需要認證的觸發器都要引用 |

### 提示

- 使用 `kubectl create secret generic` 建立 Secret 更方便
- 可以在同一個 TriggerAuthentication 中引用多個 Secret 鍵
- Secret 的值會在 KEDA 讀取時自動解碼

---

## 快速參考 (Quick Reference)

| 指令 | 說明 |
|------|------|
| `kubectl create secret generic <name> --from-literal=key=value` | 建立 Secret |
| `kubectl get secret <name> -o yaml` | 查看 Secret YAML |
| `kubectl get secret <name> -o jsonpath='{.data.key}' \| base64 -d` | 解碼 Secret 值 |
| `kubectl describe triggerauthentication <name>` | 查看 TriggerAuth |
