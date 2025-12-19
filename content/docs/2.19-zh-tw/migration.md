+++
title = "遷移指南"
+++

## TL;DR

從 KEDA v1 遷移到 v2 需要先解除安裝 v1（包括 CRD），然後安裝 v2。主要變更包括：API 版本從 `keda.k8s.io/v1alpha1` 改為 `keda.sh/v1alpha1`、Job 擴縮改用獨立的 ScaledJob 資源、部分屬性名稱變更。

---

## 翻譯 (Translation)

### 從 KEDA v1 遷移到 v2

請注意，您**不能**在同一個 Kubernetes 叢集上同時執行 KEDA v1 和 v2。您需要先[解除安裝](../../1.5/deploy) KEDA v1，才能[安裝](../deploy)和使用 KEDA v2。

> 💡 **注意：** 解除安裝 KEDA v1 時，請確保 v1 CRD 也從叢集中解除安裝。

KEDA v2 使用新的 API 命名空間來定義其自訂資源定義（CRD）：`keda.sh` 取代 `keda.k8s.io`，並引入了用於擴縮 Job 的新自訂資源。請參閱 KEDA 自訂資源的完整詳細資訊[這裡](../concepts/#keda-custom-resources-crds)。

以下是變更概述：

- [擴縮 Deployment](#擴縮-deployment)
- [擴縮 Job](#擴縮-job)
- [觸發器 metadata 的改進靈活性和可用性](#觸發器-metadata-的改進靈活性和可用性)
- [擴縮器](#擴縮器)
- [TriggerAuthentication](#triggerauthentication)

### 擴縮 Deployment

為了使用 KEDA v2 擴縮 `Deployment`，您只需要對現有的 v1 `ScaledObjects` 定義進行少量修改，使其符合 v2：

- 將 `apiVersion` 屬性的值從 `keda.k8s.io/v1alpha1` 改為 `keda.sh/v1alpha1`
- 將屬性 `spec.scaleTargetRef.deploymentName` 重新命名為 `spec.scaleTargetRef.name`
- 將屬性 `spec.scaleTargetRef.containerName` 重新命名為 `spec.scaleTargetRef.envSourceContainerName`
- 標籤 `deploymentName`（在 `metadata.labels.` 中）在 v2 ScaledObject 上不再需要指定（在較舊版本的 v1 中是強制的）

請參閱以下範例或參考完整的 [v2 ScaledObject 規格](./reference/scaledobject-spec)

**v1 ScaledObject 範例**

```yaml
apiVersion: keda.k8s.io/v1alpha1
kind: ScaledObject
metadata:
  name: { scaled-object-name }
  labels:
    deploymentName: { deployment-name }
spec:
  scaleTargetRef:
    deploymentName: { deployment-name }
    containerName: { container-name }
  pollingInterval: 30
  cooldownPeriod: 300
  minReplicaCount: 0
  maxReplicaCount: 100
  triggers:
  # {觸發器列表以啟動 deployment}
```

**v2 ScaledObject 範例**

```yaml
apiVersion: keda.sh/v1alpha1 #  <--- 屬性值已變更
kind: ScaledObject
metadata: #  <--- labels.deploymentName 不再需要
  name: { scaled-object-name }
spec:
  scaleTargetRef:
    name: { deployment-name } #  <--- 屬性名稱已變更
    envSourceContainerName: { container-name } #  <--- 屬性名稱已變更
  pollingInterval: 30
  cooldownPeriod: 300
  minReplicaCount: 0
  maxReplicaCount: 100
  triggers:
  # {觸發器列表以啟動 deployment}
```

### 擴縮 Job

為了使用 KEDA v2 擴縮 `Job`，您只需要對現有的 v1 `ScaledObjects` 定義進行少量修改，使其符合 v2：

- 將 `apiVersion` 屬性的值從 `keda.k8s.io/v1alpha1` 改為 `keda.sh/v1alpha1`
- 將 `kind` 屬性的值從 `ScaledObject` 改為 `ScaledJob`
- 移除屬性 `spec.scaleType`
- 移除屬性 `spec.cooldownPeriod` 和 `spec.minReplicaCount`

您可以設定 `successfulJobsHistoryLimit` 和 `failedJobsHistoryLimit`。它們會自動移除舊的 Job 歷史記錄。

請參閱以下範例或參考完整的 [v2 ScaledJob 規格](./reference/scaledjob-spec/)

**v1 用於 Job 擴縮的 ScaledObject 範例**

```yaml
apiVersion: keda.k8s.io/v1alpha1
kind: ScaledObject
metadata:
  name: { scaled-object-name }
spec:
  scaleType: job
  jobTargetRef:
    parallelism: 1
    completions: 1
    activeDeadlineSeconds: 600
    backoffLimit: 6
    template:
      # {job 範本}
  pollingInterval: 30
  cooldownPeriod: 300
  minReplicaCount: 0
  maxReplicaCount: 100
  triggers:
  # {建立 job 的觸發器列表}
```

**v2 ScaledJob 範例**

```yaml
apiVersion: keda.sh/v1alpha1 #  <--- 屬性值已變更
kind: ScaledJob #  <--- 屬性值已變更
metadata:
  name: { scaled-job-name }
spec: #  <--- spec.scaleType 不再需要
  jobTargetRef:
    parallelism: 1
    completions: 1
    activeDeadlineSeconds: 600
    backoffLimit: 6
    template:
      # {job 範本}
  pollingInterval: 30 #  <--- spec.cooldownPeriod 和 spec.minReplicaCount 不再需要
  successfulJobsHistoryLimit: 5 #  <--- 新增屬性
  failedJobsHistoryLimit: 5 #  <--- 新增屬性
  maxReplicaCount: 100
  triggers:
  # {建立 job 的觸發器列表}
```

### 觸發器 metadata 的改進靈活性和可用性

我們引入了更多選項來設定觸發器 metadata，為使用者提供更大的靈活性。

> 💡 **注意：** 變更僅適用於觸發器 metadata，不影響 `TriggerAuthentication` 的使用

### 擴縮器

**Azure Service Bus**

- `queueLength` 已重新命名為 `messageCount`

**Kafka**

- `authMode` 屬性已被 `sasl` 和 `tls` 屬性取代。請參閱 [Kafka 驗證參數文件](../scalers/apache-kafka/#authentication-parameters)了解詳細資訊。

**RabbitMQ**

在 KEDA 2.0 中，RabbitMQ 擴縮器只有 `host` 參數，通訊協定可以透過 `protocol`（http 或 amqp）指定。預設值為 `amqp`。行為變更僅影響使用 HTTP 協定的擴縮器。

2.0 之前的 RabbitMQ 觸發器範例：

```yaml
triggers:
  - type: rabbitmq
    metadata:
      queueLength: "20"
      queueName: testqueue
      includeUnacked: "true"
      apiHost: "https://guest:password@localhost:443/vhostname"
```

2.0 中相同的觸發器：

```yaml
triggers:
  - type: rabbitmq
    metadata:
      queueLength: "20"
      queueName: testqueue
      protocol: "http"
      host: "https://guest:password@localhost:443/vhostname"
```

### TriggerAuthentication

為了在 KEDA v2 中透過 `TriggerAuthentication` 使用驗證，您需要變更：

- 將 `apiVersion` 屬性的值從 `keda.k8s.io/v1alpha1` 改為 `keda.sh/v1alpha1`

更多詳細資訊請參閱完整的 [v2 TriggerAuthentication 規格](../concepts/authentication/#re-use-credentials-and-delegate-auth-with-triggerauthentication)

---

## 說明 (Explanation)

### v1 vs v2 主要差異

| 項目 | v1 | v2 |
|------|----|----|
| API 版本 | `keda.k8s.io/v1alpha1` | `keda.sh/v1alpha1` |
| Job 擴縮 | ScaledObject with scaleType: job | 獨立的 ScaledJob 資源 |
| Deployment 名稱屬性 | `deploymentName` | `name` |
| 容器名稱屬性 | `containerName` | `envSourceContainerName` |
| 觸發器 metadata | 特定命名 | 支援 `FromEnv` 後綴 |

### 遷移步驟

1. 備份所有現有的 ScaledObject 和 TriggerAuthentication
2. 解除安裝 KEDA v1（包括 CRD）
3. 安裝 KEDA v2
4. 修改資源定義以符合 v2 格式
5. 套用更新後的資源

---

## 實作範例 (Practical Example)

```bash
# === 遷移步驟 ===

# 1. 備份現有資源
kubectl get scaledobjects -A -o yaml > scaledobjects-backup.yaml
kubectl get triggerauthentications -A -o yaml > triggerauth-backup.yaml

# 2. 解除安裝 KEDA v1
helm uninstall keda -n keda
# 或如果使用 YAML 安裝
kubectl delete -f keda-v1.yaml

# 3. 刪除 v1 CRD（重要！）
kubectl delete crd scaledobjects.keda.k8s.io
kubectl delete crd triggerauthentications.keda.k8s.io

# 4. 安裝 KEDA v2
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda --namespace keda --create-namespace

# 5. 套用更新後的資源
kubectl apply -f scaledobjects-v2.yaml
```

```bash
# 使用 sed 批量更新 ScaledObject（僅供參考，建議手動檢查）
# 更新 apiVersion
sed -i 's/keda.k8s.io\/v1alpha1/keda.sh\/v1alpha1/g' scaledobjects.yaml

# 更新 deploymentName 為 name
sed -i 's/deploymentName:/name:/g' scaledobjects.yaml

# 更新 containerName 為 envSourceContainerName
sed -i 's/containerName:/envSourceContainerName:/g' scaledobjects.yaml
```

---

## 常見錯誤與提示 (Common Mistakes & Tips)

| 常見錯誤 | 正確做法 |
|----------|----------|
| 沒有刪除 v1 CRD 就安裝 v2 | 必須先刪除 v1 CRD |
| Job 擴縮仍使用 ScaledObject | 改用獨立的 ScaledJob 資源 |
| 忘記更新 apiVersion | 從 `keda.k8s.io` 改為 `keda.sh` |
| 保留 scaleType: job 屬性 | v2 的 ScaledJob 不需要此屬性 |

### 遷移提示

- 在非生產環境先測試遷移流程
- 使用版本控制追蹤資源定義的變更
- 遷移後仔細監控擴縮行為
- 保留 v1 資源備份以備回滾

---

## 快速參考 (Quick Reference)

| v1 屬性 | v2 屬性 | 說明 |
|---------|---------|------|
| `apiVersion: keda.k8s.io/v1alpha1` | `apiVersion: keda.sh/v1alpha1` | API 版本 |
| `spec.scaleTargetRef.deploymentName` | `spec.scaleTargetRef.name` | 目標名稱 |
| `spec.scaleTargetRef.containerName` | `spec.scaleTargetRef.envSourceContainerName` | 容器名稱 |
| `spec.scaleType: job` | 使用 `kind: ScaledJob` | Job 擴縮 |
| `metadata.labels.deploymentName` | 移除 | 不再需要 |

| 擴縮器 | v1 → v2 變更 |
|--------|--------------|
| Azure Service Bus | `queueLength` → `messageCount` |
| RabbitMQ | `apiHost` → `host` + `protocol: http` |
| Kafka | `authMode` → `sasl` + `tls` |

| 遷移指令 | 說明 |
|----------|------|
| `kubectl get scaledobjects -A -o yaml > backup.yaml` | 備份 ScaledObject |
| `kubectl delete crd scaledobjects.keda.k8s.io` | 刪除 v1 CRD |
| `helm install keda kedacore/keda -n keda` | 安裝 v2 |
