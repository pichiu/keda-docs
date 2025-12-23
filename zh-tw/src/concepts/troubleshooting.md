
## TL;DR

排解 KEDA 問題首先查看 KEDA Operator 的日誌，使用 `kubectl logs -n keda` 指令。如果問題無法解決，可以在 KEDA GitHub 儲存庫提交 Issue。

---

## 翻譯 (Translation)

### KEDA 日誌和遙測

如果某些功能沒有正確運作，首先要查看的是 KEDA 產生的日誌。部署後，您應該在命名空間中（預設為：`keda`）有一個包含兩個容器的 Pod 在執行。

您可以透過 kubectl 查看 KEDA Operator Pod：

```sh
kubectl get pods -n keda
```

您可以使用以下指令查看 keda Operator 容器的日誌：

```sh
kubectl logs -n keda {keda-pod-name} -c keda-operator
```

### 回報問題

如果您遇到問題或發現潛在的錯誤，請在 [KEDA GitHub 儲存庫](https://github.com/kedacore/keda/issues/new/choose)提交 Issue，包含詳細資訊、日誌和重現行為的步驟。

---

## 說明 (Explanation)

### KEDA 元件架構

KEDA 部署後會建立以下主要元件：

| 元件 | 說明 | 容器 |
|------|------|------|
| keda-operator | 核心控制器，處理 ScaledObject/ScaledJob | keda-operator |
| keda-metrics-apiserver | 提供自訂指標給 HPA | keda-metrics-apiserver |
| keda-admission-webhooks | 驗證資源設定 | keda-admission-webhooks |

### 常見問題類別

1. **安裝問題**：CRD 未正確安裝、權限不足
2. **連接問題**：無法連接到外部事件來源
3. **擴縮問題**：擴縮行為不符預期
4. **驗證問題**：TriggerAuthentication 設定錯誤

---

## 實作範例 (Practical Example)

```bash
# === 基本診斷指令 ===

# 1. 檢查 KEDA Pod 狀態
kubectl get pods -n keda
# 預期輸出：所有 Pod 應為 Running 狀態

# 2. 檢查 KEDA CRD 是否已安裝
kubectl get crd | grep keda
# 預期輸出：scaledobjects.keda.sh, scaledjobs.keda.sh, triggerauthentications.keda.sh 等

# 3. 查看 KEDA Operator 日誌
kubectl logs -n keda -l app=keda-operator -c keda-operator --tail=100

# 4. 查看 Metrics Server 日誌
kubectl logs -n keda -l app=keda-operator -c keda-metrics-apiserver --tail=100

# 5. 查看 Admission Webhook 日誌
kubectl logs -n keda -l app=keda-admission-webhooks --tail=100

# === 擴縮問題診斷 ===

# 6. 檢查 ScaledObject 狀態
kubectl get scaledobject
kubectl describe scaledobject <name>

# 7. 檢查產生的 HPA
kubectl get hpa
kubectl describe hpa keda-hpa-<scaledobject-name>

# 8. 檢查 HPA 的指標
kubectl get --raw "/apis/external.metrics.k8s.io/v1beta1" | jq .

# === 事件和狀態 ===

# 9. 查看相關事件
kubectl get events --sort-by='.lastTimestamp' | grep -i keda

# 10. 檢查 ScaledObject 的條件
kubectl get scaledobject <name> -o jsonpath='{.status.conditions}' | jq .
```

```bash
# === 進階診斷 ===

# 檢查 KEDA 是否可以存取指標來源
# 範例：測試 Prometheus 連接
kubectl run test-pod --rm -it --image=curlimages/curl -- \
  curl -s "http://prometheus-server.monitoring:9090/api/v1/query?query=up"

# 檢查 TriggerAuthentication
kubectl get triggerauthentication
kubectl describe triggerauthentication <name>

# 檢查 Secret 是否存在且有正確的鍵
kubectl get secret <secret-name> -o jsonpath='{.data}' | jq 'keys'

# 強制重新協調 ScaledObject
kubectl annotate scaledobject <name> force-reconcile=$(date +%s) --overwrite
```

---

## 常見錯誤與提示 (Common Mistakes & Tips)

| 常見問題 | 可能原因 | 解決方案 |
|----------|----------|----------|
| ScaledObject 狀態為 Unknown | 無法連接到指標來源 | 檢查網路連接和認證設定 |
| HPA 沒有被建立 | ScaledObject 設定錯誤 | 查看 Operator 日誌和事件 |
| Pod 沒有擴縮 | 指標值在閾值內 | 確認指標查詢和閾值設定 |
| 無法縮減到 0 | minReplicaCount 設定不正確 | 確認 minReplicaCount: 0 且有非 CPU/Memory 觸發器 |
| 認證失敗 | TriggerAuthentication 設定錯誤 | 檢查 Secret 和認證參數 |

### 診斷提示

- 開啟 KEDA 的 debug 日誌：在 Helm values 中設定 `logging.operator.level: debug`
- 使用 `kubectl describe` 查看資源的詳細狀態和事件
- 檢查 ScaledObject 的 `.status.conditions` 欄位了解當前狀態
- 確認外部服務（如 Prometheus、RabbitMQ）是否可從 KEDA Pod 存取

### 提交 Issue 時應包含的資訊

1. KEDA 版本和 Kubernetes 版本
2. ScaledObject/ScaledJob YAML 設定（移除敏感資訊）
3. KEDA Operator 日誌
4. `kubectl describe scaledobject` 輸出
5. 重現問題的步驟

---

## 快速參考 (Quick Reference)

| 診斷目標 | 指令 |
|----------|------|
| 查看 KEDA Pod | `kubectl get pods -n keda` |
| 查看 Operator 日誌 | `kubectl logs -n keda -l app=keda-operator -c keda-operator` |
| 查看 Metrics Server 日誌 | `kubectl logs -n keda -l app=keda-operator -c keda-metrics-apiserver` |
| 查看 ScaledObject 狀態 | `kubectl describe scaledobject <name>` |
| 查看 HPA 狀態 | `kubectl describe hpa keda-hpa-<name>` |
| 查看事件 | `kubectl get events -n <namespace> --sort-by='.lastTimestamp'` |
| 檢查 CRD | `kubectl get crd \| grep keda` |
| 查看外部指標 | `kubectl get --raw "/apis/external.metrics.k8s.io/v1beta1"` |

| 常見狀態 | 意義 |
|----------|------|
| `Active: True` | 擴縮器正常運作，指標值超過啟動閾值 |
| `Active: False` | 擴縮器正常運作，指標值未超過啟動閾值 |
| `Active: Unknown` | 無法取得指標，可能是連接問題 |
| `Paused: True` | 擴縮已被手動暫停 |
