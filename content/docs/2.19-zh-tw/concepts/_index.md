+++
title = "KEDA 概念"
description = "什麼是 KEDA 以及它如何運作"
weight = 1
+++

## TL;DR

KEDA 是一個輕量級的 Kubernetes 事件驅動自動擴縮器，透過監控外部事件來源（如訊息佇列、資料庫）來動態調整應用程式規模。它與 Kubernetes HPA 協同運作，不會取代任何現有元件。

---

## 翻譯 (Translation)

### 什麼是 KEDA？

**KEDA** 是一個幫助 [Kubernetes](https://kubernetes.io) 根據真實世界事件來擴縮應用程式的工具。它由 Microsoft 和 Red Hat 共同創建。透過 KEDA，您可以根據工作負載（如佇列中的訊息數量或傳入的請求）自動調整容器的規模。

它是輕量級的，與 Kubernetes 元件（如水平 Pod 自動擴縮器，Horizontal Pod Autoscaler, HPA）協同運作。它不會取代任何東西，而是增加更多功能。您可以選擇哪些應用程式要用 KEDA 來擴縮，而其他的保持不變。這使它具有靈活性且易於與現有設定整合。

### KEDA 如何運作

KEDA 監控外部事件來源，並根據需求調整您的應用程式資源。它的主要元件協同運作以實現這一點：

* **KEDA Operator（操作器）** 追蹤事件來源，並根據需求增加或減少應用程式實例的數量。
* **Metrics Server（指標伺服器）** 向 Kubernetes 的 HPA 提供外部指標，以便它可以做出擴縮決策。
* **Scalers（擴縮器）** 連接到事件來源（如訊息佇列或資料庫），提取當前使用率或負載的資料。
* **Custom Resource Definitions（自訂資源定義，CRDs）** 定義您的應用程式應如何根據觸發器（如佇列長度或 API 請求率）進行擴縮。

簡單來說，KEDA 監聽 Kubernetes 外部發生的事情，取得所需的資料，並相應地擴縮您的應用程式。它高效且與 Kubernetes 良好整合，能動態處理擴縮。

### KEDA 架構

下圖顯示 KEDA 如何與 Kubernetes 水平 Pod 自動擴縮器、外部事件來源和 Kubernetes 的 [etcd](https://etcd.io) 資料儲存協同運作：

![KEDA 架構](/img/keda-arch.png)

外部事件（如佇列訊息增加）觸發 **ScaledObject（擴縮物件）**，它設定擴縮規則。**Controller（控制器）** 處理擴縮，而 **Metrics Adapter（指標適配器）** 將資料發送到 HPA 以進行即時擴縮決策。**Admission Webhooks（准入 Webhook）** 確保您的設定正確且不會造成問題。

這種設定讓 Kubernetes 根據外部發生的事情自動調整資源，保持效率和響應能力。

### KEDA 自訂資源（CRDs）

KEDA 使用 **自訂資源定義（Custom Resource Definitions, CRDs）** 來管理擴縮行為：

* **ScaledObject（擴縮物件）**：將您的應用程式（如 **Deployment** 或 **StatefulSet**）連結到外部事件來源，定義擴縮如何運作。
* **ScaledJob（擴縮任務）**：根據外部指標擴縮 Job 來處理批次處理任務。
* **TriggerAuthentication（觸發器驗證）**：提供安全的方式來存取事件來源，支援環境變數或雲端特定憑證等方法。

這些 CRD 讓您可以控制擴縮，同時保持應用程式的安全性和對需求的響應能力。

### 擴縮 Deployment、StatefulSet 和自訂資源

KEDA 超越了基於 CPU 或記憶體的擴縮，透過連接到外部資料來源（如訊息佇列、資料庫或 API）。這意味著您的應用程式可以根據實際工作負載需求即時擴縮。

#### 擴縮 Deployment 和 StatefulSet

使用 KEDA，您可以輕鬆擴縮 Deployment 和 StatefulSet。透過建立 ScaledObject，您將工作負載連結到事件來源，如佇列或請求率。KEDA 根據需求調整實例數量。

Deployment 非常適合需要快速擴縮的無狀態應用程式。StatefulSet 適合需要穩定儲存或身分的應用程式，如資料庫。KEDA 確保您的資源被有效使用，同時跟上需求。

#### 擴縮自訂資源

KEDA 也支援自訂 Kubernetes 資源。您設定一個針對您資源量身定制的 ScaledObject，並將其連接到事件觸發器，如資料庫變更。然後，您定義擴縮限制，KEDA 會處理其餘的事情，確保您的自訂應用程式動態擴縮。

#### 擴縮 Job

KEDA 可以為批次處理擴縮 Kubernetes Job。透過建立 ScaledJob，您將任務連結到外部事件，如佇列大小。KEDA 即時調整 Job 實例的數量，自動清理已完成的 Job。這確保您只在需要時使用資源。

#### 驗證

KEDA 支援使用 TriggerAuthentication 安全連接到外部事件來源。您可以設定它與 Secret、雲端原生驗證（如 AWS IAM 角色）或 Azure Active Directory 一起使用。這保持您的連接安全和資料安全。

#### 外部擴縮器

KEDA 透過擴縮器連接到各種服務，如訊息佇列或雲端 API。這些擴縮器取得即時指標以決定何時以及如何擴縮。KEDA 包含常用服務的內建擴縮器，但您也可以根據需要建立自訂擴縮器。這讓您的工作負載可以輕鬆響應真實世界的需求。

#### 外部消費原始擴縮器指標

KEDA 還允許將內部指標（來自內部或外部擴縮器）消費給有興趣的第三方。此功能使用 gRPC 伺服器串流 API 公開，需要先將 `RAW_METRICS_GRPC_PROTOCOL` 設定為「`enabled`」來啟用。然後可以使用任何 gRPC 用戶端（例如 [grpcurl](https://github.com/kedacore/keda/pull/7093#issuecomment-3333530716)）透過 ScaledObject/ScaledJob 名稱、命名空間和觸發器名稱來訂閱指標。

您可以使用 `RAW_METRICS_MODE` 環境變數控制何時發送原始指標：

* `all` 或 `""`（空）：發送所有原始指標，包括當指標伺服器請求它們（HPA）時和每個 ScaledObject 或 ScaledJob 的常規輪詢間隔期間。這是預設行為。
* `hpa`：僅當 Kubernetes 指標伺服器明確請求 ScaledObject 的指標時才發送原始指標。這意味著指標是響應 HPA 查詢發送的，而不是按常規時間表發送。
* `pollinginterval`：僅在每個 ScaledObject 或 ScaledJob 的輪詢間隔期間發送原始指標。在此模式下，指標在每個輪詢週期推送出去，與 HPA 請求無關。
* 任何未知值將預設為 `all` 模式。

#### Admission Webhooks

KEDA 使用 Admission Webhook 來驗證您的擴縮設定。它們確保您的設定正確，例如防止多個 ScaledObject 指向同一個應用程式。這減少錯誤並使擴縮更順暢。

---

## 說明 (Explanation)

### KEDA 核心元件詳解

| 元件 | 功能 | 說明 |
|------|------|------|
| KEDA Operator | 核心控制器 | 監控 ScaledObject/ScaledJob，觸發擴縮動作 |
| Metrics Server | 指標伺服器 | 向 HPA 提供外部指標 |
| Scaler | 擴縮器 | 連接外部事件來源，取得指標 |
| ScaledObject | 擴縮物件 | 定義 Deployment/StatefulSet 的擴縮規則 |
| ScaledJob | 擴縮任務 | 定義 Job 的擴縮規則 |
| TriggerAuthentication | 觸發器驗證 | 管理連接外部服務的認證 |

### KEDA 與 HPA 的關係

KEDA 不是要取代 HPA，而是增強它：

1. **0 → 1 擴縮**：由 KEDA Operator 負責（HPA 無法從 0 擴縮）
2. **1 → N 擴縮**：由 HPA 根據 KEDA 提供的指標負責
3. **N → 0 擴縮**：由 KEDA Operator 負責

---

## 實作範例 (Practical Example)

```bash
# 查看 KEDA 元件狀態
kubectl get pods -n keda                    # 列出 KEDA 命名空間中的 Pod
kubectl get crd | grep keda                 # 列出 KEDA 相關的 CRD

# 查看 ScaledObject
kubectl get scaledobject                    # 列出所有 ScaledObject
kubectl describe scaledobject <name>        # 查看特定 ScaledObject 的詳細資訊

# 查看 ScaledJob
kubectl get scaledjob                       # 列出所有 ScaledJob
kubectl describe scaledjob <name>           # 查看特定 ScaledJob 的詳細資訊

# 查看 TriggerAuthentication
kubectl get triggerauthentication           # 列出所有 TriggerAuthentication
kubectl get clustertriggerauthentication    # 列出所有 ClusterTriggerAuthentication
```

---

## 常見錯誤與提示 (Common Mistakes & Tips)

| 常見錯誤 | 正確做法 |
|----------|----------|
| 將 KEDA 視為 HPA 的替代品 | KEDA 是 HPA 的補充，兩者協同運作 |
| 忘記設定 TriggerAuthentication | 連接需要認證的外部服務時必須設定 |
| ScaledObject 和 ScaledJob 混淆 | Deployment/StatefulSet 用 ScaledObject，批次任務用 ScaledJob |
| 不了解 0 → 1 和 1 → N 的區別 | KEDA 負責 0 → 1，HPA 負責 1 → N |

### 提示

- KEDA 可以將 Pod 縮減到 0，這對於不活躍的工作負載很有用（節省成本）
- 使用 `useCachedMetrics` 可以減少對擴縮器服務的負載
- 每個擴縮器都有其特定的設定參數，請參閱個別文件

---

## 快速參考 (Quick Reference)

| 指令 | 說明 |
|------|------|
| `kubectl get scaledobject` | 列出所有 ScaledObject |
| `kubectl get scaledjob` | 列出所有 ScaledJob |
| `kubectl get triggerauthentication` | 列出命名空間級別的 TriggerAuthentication |
| `kubectl get clustertriggerauthentication` | 列出叢集級別的 TriggerAuthentication |
| `kubectl describe scaledobject <name>` | 查看 ScaledObject 詳細資訊 |
| `kubectl get hpa` | 查看 KEDA 產生的 HPA |
| `kubectl logs -n keda -l app=keda-operator` | 查看 KEDA Operator 日誌 |
