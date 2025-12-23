
## TL;DR

KEDA 的 Admission Webhook 會在資源建立或更新時自動驗證設定，防止常見的錯誤配置，如：多個 ScaledObject 指向同一工作負載、CPU/Memory 觸發器缺少資源請求、觸發器名稱重複等。

---

## 翻譯 (Translation)

有幾種錯誤設定場景可能會在生產工作負載中產生擴縮問題，例如：在 Kubernetes 中，單一工作負載永遠不應該被 2 個或更多 HPA 擴縮，因為這會產生衝突和非預期的行為。

某些資料格式錯誤可以在模型驗證期間偵測到，但這些錯誤設定無法在該步驟中偵測到，因為模型本身確實是正確的。為了嘗試在資料平面早期偵測這些錯誤設定，Admission Webhook 會驗證所有傳入的（KEDA）資源（新的或更新的），並拒絕任何不符合以下規則的資源。

### 預防規則

KEDA 會阻止所有不符合這些規則的 `ScaledObject` 傳入變更：

- 擴縮的工作負載（`scaledobject.spec.scaleTargetRef`）已經被其他來源（其他 ScaledObject 或 HPA）自動擴縮。
- 使用了 CPU 和/或 Memory 觸發器，且擴縮的工作負載沒有定義 requests。**此規則不適用於所有工作負載類型，僅適用於 `Deployment` 和 `StatefulSet`。**
- CPU 和/或 Memory 觸發器是**唯一使用的觸發器**，且 ScaledObject 定義了 `minReplicaCount:0`。**此規則不適用於所有工作負載類型，僅適用於 `Deployment` 和 `StatefulSet`。**
- 在多個觸發器中**指定了** `name` 的情況下，名稱必須是**唯一的**（不允許有多個具有相同名稱的觸發器）
- 回退功能（Fallback）設定使用了 AverageValue 以外的指標類型。回退功能僅支援 AverageValue 指標類型，不支援 CPU 和 Memory 指標。

KEDA 會阻止所有不符合這些規則的 `TriggerAuthentication`/`ClusterTriggerAuthentication` 傳入變更：

- 為 Azure AD Workload Identity 和/或 Pod Identity 指定的身分識別 ID 為空。（預設/未設定的身分識別 ID 會被允許通過。）
  > 注意：這僅適用於 `TriggerAuthentication/ClusterTriggerAuthentication` 正在覆蓋安裝期間提供給 KEDA 的預設 identityId 時

---

## 說明 (Explanation)

### 為什麼需要 Admission Webhook？

Admission Webhook 提供了一層保護，在資源套用到叢集之前進行驗證。這可以防止：

1. **擴縮衝突**：同一工作負載被多個 HPA 控制
2. **設定錯誤**：缺少必要的資源定義
3. **邏輯錯誤**：不可能的擴縮設定（如只用 CPU 觸發器但設定縮減到 0）

### 驗證規則詳解

| 規則 | 說明 | 原因 |
|------|------|------|
| 單一 HPA 來源 | 每個工作負載只能有一個擴縮來源 | 防止衝突和不可預測的行為 |
| CPU/Memory 需要 requests | 使用 CPU/Memory 觸發器時必須定義資源請求 | HPA 需要基於 requests 計算使用率 |
| CPU/Memory 不能縮到 0 | 只使用 CPU/Memory 觸發器時不能設定 minReplicaCount: 0 | 沒有 Pod 就無法測量 CPU/Memory |
| 觸發器名稱唯一 | 多個觸發器的名稱不能重複 | 防止指標混淆 |
| Fallback 只支援 AverageValue | 回退功能的指標類型限制 | 技術實作限制 |

---

## 實作範例 (Practical Example)

```yaml
# ❌ 錯誤範例：會被 Admission Webhook 拒絕

# 範例 1：minReplicaCount: 0 但只使用 CPU 觸發器
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: bad-example-1
spec:
  scaleTargetRef:
    name: my-deployment
  minReplicaCount: 0                    # ❌ 不能縮到 0
  triggers:
    - type: cpu                         # ❌ 只有 CPU 觸發器
      metadata:
        type: Utilization
        value: "50"

# 範例 2：重複的觸發器名稱
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: bad-example-2
spec:
  scaleTargetRef:
    name: my-deployment
  triggers:
    - type: prometheus
      name: my-trigger                  # ❌ 名稱重複
      metadata:
        query: sum(rate(requests[1m]))
    - type: rabbitmq
      name: my-trigger                  # ❌ 名稱重複
      metadata:
        queueName: my-queue
```

```yaml
# ✅ 正確範例

# 範例 1：使用 CPU 觸發器但 minReplicaCount >= 1
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: good-example-1
spec:
  scaleTargetRef:
    name: my-deployment
  minReplicaCount: 1                    # ✅ 至少 1 個副本
  triggers:
    - type: cpu
      metadata:
        type: Utilization
        value: "50"

# 範例 2：混合使用 CPU 和其他觸發器
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: good-example-2
spec:
  scaleTargetRef:
    name: my-deployment
  minReplicaCount: 0                    # ✅ 可以縮到 0
  triggers:
    - type: cpu
      metadata:
        type: Utilization
        value: "50"
    - type: prometheus                  # ✅ 有其他觸發器
      metadata:
        query: sum(rate(requests[1m]))
        threshold: "100"
```

```bash
# 查看 Admission Webhook 狀態
kubectl get validatingwebhookconfigurations | grep keda

# 查看 Webhook 詳細設定
kubectl describe validatingwebhookconfiguration keda-admission

# 如果需要暫時停用驗證（不建議用於生產環境）
kubectl delete validatingwebhookconfiguration keda-admission
```

---

## 常見錯誤與提示 (Common Mistakes & Tips)

| 常見錯誤 | 正確做法 |
|----------|----------|
| 只用 CPU/Memory 觸發器但設定 minReplicaCount: 0 | 至少設定 minReplicaCount: 1，或新增其他觸發器 |
| 多個 ScaledObject 指向同一 Deployment | 合併為單一 ScaledObject，使用多個觸發器 |
| 使用重複的觸發器名稱 | 為每個觸發器使用唯一名稱 |
| Deployment 沒有設定資源 requests | 在 Pod spec 中設定 resources.requests |

### 提示

- Admission Webhook 的錯誤訊息通常很清楚，仔細閱讀可以快速定位問題
- 使用 `kubectl apply --dry-run=server` 可以在不實際套用的情況下測試驗證
- 生產環境強烈建議保持 Admission Webhook 啟用
- 如果使用 YAML 安裝，可選擇不含 Webhook 的版本（但不建議）

---

## 快速參考 (Quick Reference)

| 驗證規則 | 影響的資源 | 說明 |
|----------|------------|------|
| 單一 HPA 來源 | ScaledObject | 每個工作負載只能有一個擴縮來源 |
| CPU/Memory 需要 requests | ScaledObject | Deployment/StatefulSet 必須定義 requests |
| CPU/Memory 不能縮到 0 | ScaledObject | 不能設定 minReplicaCount: 0 |
| 觸發器名稱唯一 | ScaledObject | 多個觸發器名稱不能重複 |
| Identity ID 不能為空 | TriggerAuthentication | 覆蓋預設身分時必須提供有效 ID |

| 指令 | 說明 |
|------|------|
| `kubectl get validatingwebhookconfigurations` | 列出驗證 Webhook |
| `kubectl apply --dry-run=server -f scaledobject.yaml` | 測試驗證而不實際套用 |
| `kubectl logs -n keda -l app=keda-admission-webhooks` | 查看 Webhook 日誌 |
