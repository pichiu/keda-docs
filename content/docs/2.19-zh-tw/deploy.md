+++
title = "部署 KEDA"
+++

## TL;DR

KEDA 提供多種安裝方式：Helm（最靈活）、Operator Hub（最簡單）、YAML 檔案（最精確控制）、MicroK8s（本機開發最佳）。選擇符合您環境需求的安裝方式即可。KEDA 需要 Kubernetes 1.30 或更高版本。

---

## 翻譯 (Translation)

KEDA 提供多種安裝方式，每種方式都有其獨特優勢，以適應各種環境和需求。如果您需要靈活性和自訂設定，使用 **Helm** 部署是理想選擇；它與已建立 Helm 工作流程的環境整合良好，並允許輕鬆調整設定。若要簡單快速的設定，透過 **Operator Hub** 安裝提供一鍵式部署和自動更新，非常適合希望減少自訂設定的使用者。

使用 **YAML 檔案** 可提供對設定的最大控制權，非常適合需要嚴格設定或無法使用 Helm 和 Operator Hub 的環境。最後，在 **MicroK8s** 上部署 KEDA 非常適合本機或開發測試，提供輕量級的 Kubernetes 環境，可快速設定而無需建立完整叢集。

每種方式在便利性、控制權和相容性之間取得不同的平衡：Helm 最適合廣泛自訂、Operator Hub 最簡單、YAML 檔案用於精確設定、MicroK8s 用於本機實驗。請選擇符合您部署需求和環境的選項。

> 💡 **注意：** KEDA 需要 Kubernetes 叢集版本 1.30 或更高版本

找不到您需要的內容？歡迎在我們的 GitHub 儲存庫[建立 Issue](https://github.com/kedacore/keda/issues/new)。

### 使用 Helm 部署 {#helm}

#### 前置需求

使用 Helm 部署 KEDA 前，請確保您的系統已安裝並設定 Helm。Helm 是 Kubernetes 的套件管理器（Package Manager），透過處理複雜的設定和範本化來簡化部署過程，這對於管理多個實例或自訂設定特別有用。建議使用最新版本的 Helm 以確保與 KEDA 的相容性並使用最新功能。

如果您是 Helm 新手，請先熟悉基本的 Helm 指令（[`helm install`](https://helm.sh/docs/helm/helm_install/)、`helm upgrade`、`helm repo add`）。確保您有權限在 Kubernetes 叢集上安裝 Chart，因為某些環境可能會限制存取。正確設定的 Helm 將允許您快速部署 KEDA 並輕鬆調整設定。

#### 安裝

1. 使用 Helm 部署 KEDA，首先新增官方 KEDA Helm 儲存庫：

    ```sh
   helm repo add kedacore https://kedacore.github.io/charts
   helm repo update
    ```

2. 執行以下指令安裝 `keda`：

    **Helm 3**

    ```sh
    helm install keda kedacore/keda --namespace keda --create-namespace
    ```

    此指令會在專用的命名空間（keda）中安裝 KEDA。您可以使用 `--set` 傳遞額外的設定值來自訂安裝，允許您調整副本數（Replica Count）、擴縮指標（Scaling Metrics）或日誌等級等參數。安裝完成後，透過檢查 keda 命名空間中執行的 Pod 來驗證部署：

    ```sh
    kubectl get pods -n keda
    ```

若要將 KEDA 的自訂資源定義（Custom Resource Definitions, CRDs）與 Helm Chart 分開部署，請按照以下步驟操作：

1. **下載 CRD YAML 檔案**：造訪 [KEDA GitHub 發布頁面](https://github.com/kedacore/keda/releases)，找到與您需要版本對應的 `keda-2.xx.x-crds.yaml` 檔案。
2. **將 CRD 套用到您的叢集**：使用 `kubectl` 套用 CRD 定義：

    ```sh
    kubectl apply -f keda-2.xx.x-crds.yaml
    ```

    將 `2.xx.x` 替換為您下載的特定版本號碼。

透過分開部署 CRD，您可以獨立於 Helm Chart 管理它們，為您的部署過程提供彈性。

> 💡 **注意：** 升級到 KEDA 2.2.1 或更高版本時，請務必處理 CRD 的潛在問題。從 v2.2.1 開始，KEDA 的 Helm Chart 會自動管理 CRD，如果您之前使用較早版本安裝 KEDA，可能會導致升級失敗。為防止升級過程中出現錯誤（如衝突或部署失敗），請參閱 KEDA 的[疑難排解指南](https://keda.sh/docs/2.0/troubleshooting/)以取得解決 CRD 相關問題的詳細說明。

使用 Helm 部署 KEDA 簡單直接，並允許輕鬆更新和調整設定，使其成為大多數環境的靈活選擇。

#### 解除安裝

要解除安裝 KEDA，請使用以下 Helm 指令：

```sh
helm uninstall keda --namespace keda
```

此指令會從您的叢集移除 KEDA，同時保留設定檔案以便日後需要重新安裝。如果您也想刪除 keda 命名空間，請執行：

```sh
kubectl delete namespace keda
```

使用 Helm 解除安裝既有效率又能保持叢集整潔，特別是在測試設定或升級到新 KEDA 版本時。

您可以使用以下指令移除 Finalizer：

```sh
kubectl patch scaledobject <resource-name> -p '{"metadata":{"finalizers":null}}' --type=merge
kubectl patch scaledjob <resource-name> -p '{"metadata":{"finalizers":null}}' --type=merge
```

將 \<*resource-name*\> 替換為每個資源的特定名稱。移除 Finalizer 可確保這些資源完全被移除，防止叢集中出現任何非預期的孤立資源（Orphaned Resources）。

### 透過 Operator Hub 部署 {#operatorhub}

#### 前置需求

透過 Operator Hub 部署 KEDA 前，請確保您可以存取支援 Operator Hub 的 Kubernetes 市集（例如 [OpenShift](https://docs.redhat.com/en) 或啟用 [Operator Lifecycle Manager](https://olm.operatorframework.io/docs/)（OLM）的叢集）。您還需要在叢集中安裝 Operator 的適當權限，因為某些環境可能會限制存取。

如果您使用 OpenShift，可以直接透過 OpenShift 主控台存取 Operator Hub。對於其他 Kubernetes 發行版，請驗證 OLM 已安裝，因為它負責管理 Operator Hub 中 Operator 的安裝和生命週期。確保滿足這些前置需求將使 KEDA 從 Operator Hub 順利安裝。

#### 安裝

要透過 Operator Hub 部署 KEDA，請先導覽至您叢集的 Operator Hub 介面。如果您使用 OpenShift，直接從 OpenShift 主控台存取 Operator Hub。對於其他 Kubernetes 環境，請確保已安裝 **Operator Lifecycle Manager（OLM）**。

在 Operator Hub 中搜尋「KEDA」，選擇 KEDA Operator，然後點選 **Install（安裝）**。選擇您偏好的安裝選項，如目標命名空間，然後確認安裝。KEDA 安裝完成後，透過檢查 KEDA Operator Pod 是否在指定的命名空間中執行來驗證部署。

1. 在 Operator Hub 市集中找到並安裝 KEDA Operator 到 `keda` 命名空間
2. 在 `keda` 命名空間中建立名為 `keda` 的 `KedaController` 資源

![Operator Hub 安裝](https://raw.githubusercontent.com/kedacore/keda-olm-operator/main/images/keda-olm-install.gif)

使用 Operator Hub 簡化了 KEDA 部署，在您的 Kubernetes 環境中提供簡易設定和自動化生命週期管理。

> 💡 **注意：** 有關使用 Operator Hub 安裝方式部署 KEDA 的更多詳細資訊，請參閱官方儲存庫：
>
> [KEDA Operator Hub 儲存庫](https://github.com/kedacore/keda-olm-operator)
>
> 此儲存庫提供在各種 Kubernetes 環境中透過 Operator Hub 安裝 KEDA 的額外指南、設定選項和疑難排解提示。
>
> 對於初學者探索 [`keda-olm-operator 儲存庫`](https://github.com/kedacore/keda-olm-operator)，以下檔案和目錄特別有幫助：
>
> \- **`README.md`：** 此檔案提供專案概述，包括安裝說明和使用範例。這是了解 Operator 目的和功能的絕佳起點。
>
> \- **`config/samples/`**：此目錄包含示範如何設定 KEDA 資源的範例 YAML 檔案。檢閱這些範例可以幫助您學習如何在 Kubernetes 叢集中定義和套用自訂資源。
>
> \- **`Makefile`**：`Makefile` 包含建置和部署 Operator 的指令。檢視此檔案可以讓您了解專案中使用的開發和部署流程。

#### 解除安裝

要解除安裝 KEDA，請前往您叢集的 Operator Hub 介面，找到 **Installed Operators（已安裝的 Operators）** 區段。在清單中找到 KEDA Operator，選擇它，然後選擇 **Uninstall（解除安裝）**。確認解除安裝以從您的叢集移除 Operator。

如果您在特定命名空間中部署了 KEDA，您可能還想刪除該命名空間以完全清理任何剩餘的資源。使用 Operator Hub 解除安裝可透過幾次點選移除所有 KEDA 相關元件，保持叢集井井有條。

### 使用 YAML 檔案部署 KEDA {#yaml}

#### 前置需求

在使用 YAML 檔案部署 KEDA 之前，請確保您已安裝 `kubectl` 並設定為與您的 Kubernetes 叢集互動。您還需要 KEDA YAML 清單檔案（Manifest），可從 [KEDA GitHub 發布頁面](https://github.com/kedacore/keda/releases)下載。此方式提供對設定的完全控制，如果您需要高度自訂設定或無法使用 Helm 或 Operator Hub，這是理想選擇。請確保您有在叢集中套用這些設定的適當權限。

#### 安裝

下載 KEDA YAML 清單檔案後，使用以下指令將檔案套用到您的叢集：

```sh
# 包含 Admission Webhook
kubectl apply --server-side -f https://github.com/kedacore/keda/releases/download/v2.19.0/keda-2.19.0.yaml
# 不包含 Admission Webhook
kubectl apply --server-side -f https://github.com/kedacore/keda/releases/download/v2.19.0/keda-2.19.0-core.yaml
```

或者，您可以下載檔案並從本機路徑部署：

```sh
# 包含 Admission Webhook
kubectl apply --server-side -f keda-2.19.0.yaml
# 不包含 Admission Webhook
kubectl apply --server-side -f keda-2.19.0-core.yaml
```

`--server-side` 旗標允許 Kubernetes 直接在伺服器上管理複雜資源，如 CRD 和 Admission Webhook。這種方式減少衝突並確保設定有效合併。更多資訊請參閱[此 Issue](https://github.com/kedacore/keda/issues/4740)。

> 💡 **注意：** 如果您偏好直接從 [KEDA GitHub 儲存庫](https://github.com/kedacore/keda)操作，您可以在 `/config` 目錄中找到必要的 YAML 檔案。複製儲存庫允許您在本機管理和部署 KEDA 設定：
>
> ```sh
> git clone https://github.com/kedacore/keda && cd keda
>
> VERSION=2.19.0 make deploy
> ```
>
> 這種方式讓您可以完全存取 KEDA 的設定檔案，允許您在部署前探索、修改或調整 YAML 清單檔案。使用指定版本的 make deploy 將直接從您的本機設定安裝 KEDA，提供自訂的彈性。

套用 YAML 後，透過檢查 keda 命名空間來驗證部署：

```sh
kubectl get pods -n keda
```

這種方式部署 KEDA 提供對設定的控制，同時利用伺服器端合併以實現更順暢的更新。

#### 解除安裝

如果您使用發布的 YAML 檔案安裝 KEDA，可以執行以下指令來解除安裝：

```sh
# 包含 Admission Webhook
kubectl delete -f https://github.com/kedacore/keda/releases/download/v2.19.0/keda-2.19.0.yaml
# 不包含 Admission Webhook
kubectl delete -f https://github.com/kedacore/keda/releases/download/v2.19.0/keda-2.19.0-core.yaml
```

如果您在本機下載了檔案，使用以下指令解除安裝：

```sh
# 包含 Admission Webhook
kubectl delete -f keda-2.19.0.yaml
# 不包含 Admission Webhook
kubectl delete -f keda-2.19.0-core.yaml
```

對於複製了 KEDA GitHub 儲存庫的使用者，導覽到複製的目錄並使用：

```sh
VERSION=2.19.0 make undeploy
```

### 在 MicroK8s 上部署 KEDA {#microk8s}

#### 前置需求

在 [**MicroK8s**](https://microk8s.io/) 上部署 KEDA 之前，請確保您的本機已安裝並執行 MicroK8s。MicroK8s 是輕量級的 Kubernetes 發行版，非常適合測試和本機開發。您需要設定 `kubectl` 來與您的 MicroK8s 叢集互動，這通常包含在 MicroK8s 中，但可能需要啟用（`microk8s kubectl`）。

此外，請確認您的 MicroK8s 設定包含 **Helm 3** 和 **DNS** 附加元件：

* **Helm 3**：KEDA 使用 Helm Chart 進行部署，因此 Helm 3 對於管理 KEDA 的安裝和設定是必要的。
* **DNS**：Kubernetes 服務依賴 DNS 進行內部通訊。啟用 DNS 附加元件可確保 KEDA 元件能夠在叢集內解析服務名稱，促進正常運作。

#### 安裝

要在 MicroK8s 上安裝 KEDA，首先啟用必要的附加元件，然後使用 Helm 3 附加元件部署 KEDA。

1. 啟用 Helm 和 DNS 附加元件（如果尚未啟用）：

   ```sh
   microk8s enable dns helm3
   ```

2. 新增 KEDA Helm 儲存庫：

   ```sh
   microk8s helm3 repo add kedacore https://kedacore.github.io/charts

   microk8s helm3 repo update
   ```

3. 使用 Helm 安裝 KEDA。

   執行以下指令將 KEDA 部署到您的 MicroK8s 叢集：

   ```sh
   microk8s helm3 install keda kedacore/keda --namespace keda --create-namespace
   ```

4. 驗證安裝。

   透過列出 keda 命名空間中的 Pod 來檢查 KEDA 是否正在執行：

   ```sh
   microk8s kubectl get pods -n keda
   ```

這種方式讓您可以在 MicroK8s 上快速設定 KEDA，為本機測試和開發提供簡化的環境。

#### 解除安裝

要從 MicroK8s 環境解除安裝 KEDA，請停用 KEDA 附加元件：

```sh
microk8s disable keda
```

此指令會從您的叢集移除 KEDA 及其相關元件，確保乾淨的解除安裝。

如果您使用 Helm 部署 KEDA，請使用以下指令解除安裝：

```sh
microk8s helm3 uninstall keda --namespace keda
```

執行這些指令後，KEDA 將從您的 MicroK8s 設定中完全移除。

### KEDA 入門：簡單範例

為了幫助您開始使用 KEDA，我們將透過一個簡單的範例來示範其事件驅動擴縮功能。這個「Hello KEDA」練習將引導您設定一個根據外部事件進行擴縮的基本應用程式，提供 KEDA 功能的實作介紹。

開始之前，請確保您具備以下條件：

* **Kubernetes 叢集**：一個正在執行的 Kubernetes 叢集。您可以使用 Minikube、Kind 或任何雲端 Kubernetes 服務。
* **kubectl**：Kubernetes 命令列工具，已設定為與您的叢集互動。
* **KEDA 已安裝**：KEDA 應該已安裝在您的叢集中。

#### 步驟 1：部署範例應用程式

我們將部署一個回應 HTTP 請求的簡單應用程式。在此範例中，我們將使用基本的 Python HTTP 伺服器。

1. **建立 Deployment 清單檔案**：將以下 YAML 儲存為 `deployment.yaml`：

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
      name: http-app
   spec:
      replicas: 1
      selector:
         matchLabels:
            app: http-app
      template:
         metadata:
            labels:
               app: http-app
         spec:
            containers:
            - name: http-app
              image: hashicorp/http-echo
              args:
                 - "-text=Hello, KEDA!"
              ports:
                 - containerPort: 5678
      ```

2. **套用 Deployment**：執行以下指令建立 Deployment：

   ```sh
   kubectl apply -f deployment.yaml
   ```

#### 步驟 2：公開應用程式

要存取應用程式，我們將建立一個 Service。

1. **建立 Service 清單檔案**：將以下 YAML 儲存為 `service.yaml`：

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
      name: http-app-service
   spec:
      selector:
         app: http-app
      ports:
         - protocol: TCP
           port: 80
           targetPort: 5678
      type: LoadBalancer
   ```

2. **套用 Service**：執行以下指令建立 Service：

   ```sh
   kubectl apply -f service.yaml
   ```

3. **取得外部 IP**：稍後，取得外部 IP 位址：

   ```sh
   kubectl get service http-app-service
   ```

#### 步驟 3：建立 ScaledObject

我們將建立一個 `ScaledObject`（擴縮物件）來讓 KEDA 根據 HTTP 請求率擴縮我們的 Deployment。

1. **建立 ScaledObject 清單檔案**：將以下 YAML 儲存為 `scaledobject.yaml`：

   ```yaml
   apiVersion: keda.sh/v1alpha1
   kind: ScaledObject
   metadata:
      name: http-app-scaledobject
   spec:
      scaleTargetRef:
         name: http-app
      minReplicaCount: 1
      maxReplicaCount: 10
      triggers:
         - type: prometheus
           metadata:
              serverAddress: http://prometheus-server.default.svc.cluster.local:9090
              threshold: '5'
              query: sum(rate(http_requests_total[1m]))
   ```

   > 💡 **注意：** 此範例假設您的叢集中已安裝 Prometheus 並從您的應用程式抓取指標。請根據需要調整 `serverAddress` 和 `query`。

2. **套用 ScaledObject**：執行以下指令建立 ScaledObject：

   ```sh
   kubectl apply -f scaledobject.yaml
   ```

#### 步驟 4：測試擴縮行為

要觀察 KEDA 的擴縮功能：

1. **產生負載**：使用 curl 或 hey 等工具向您的應用程式外部 IP 發送多個請求：

   ```sh
   hey -z 1m -c 10 http://<EXTERNAL-IP>
   ```

   將 `<EXTERNAL-IP>` 替換為先前取得的外部 IP 位址。

2. **監控擴縮：** 執行以下指令觀察擴縮行為：

   ```sh
   kubectl get pods -w
   ```

   您應該會看到 Pod 數量隨著負載增加而增加，當負載減少時則減少。

#### 清理

完成練習後，清理資源：

   ```sh
   kubectl delete -f scaledobject.yaml
   kubectl delete -f service.yaml
   kubectl delete -f deployment.yaml
   ```

此範例提供了 KEDA 事件驅動擴縮功能的實作介紹。透過遵循這些步驟，您可以看到 KEDA 如何與 Kubernetes 整合，根據外部事件擴縮應用程式。

---

## 說明 (Explanation)

### 為什麼需要 KEDA？

傳統的 Kubernetes HPA（Horizontal Pod Autoscaler，水平 Pod 自動擴縮器）只能根據 CPU 和記憶體指標進行擴縮。但在現代微服務架構中，我們常常需要根據其他指標來擴縮，例如：

- 訊息佇列（Message Queue）中的待處理訊息數量
- 資料庫中的未處理記錄數
- 外部 API 的回應時間
- 排程時間（例如尖峰時段）

KEDA 填補了這個空白，讓您可以根據幾乎任何可量化的指標來擴縮工作負載。

### 安裝方式比較

| 方式 | 優點 | 缺點 | 適用場景 |
|------|------|------|----------|
| Helm | 高度可自訂、易於升級 | 需要熟悉 Helm | 生產環境、需要自訂設定 |
| Operator Hub | 一鍵安裝、自動更新 | 需要 OLM | OpenShift 環境 |
| YAML 檔案 | 完全控制、無需額外工具 | 升級較繁瑣 | 嚴格控管的環境 |
| MicroK8s | 快速設定、輕量級 | 僅適合開發測試 | 本機開發、概念驗證 |

---

## 實作範例 (Practical Example)

```bash
# === Helm 安裝方式 ===
helm repo add kedacore https://kedacore.github.io/charts  # 新增 KEDA Helm 儲存庫
helm repo update                                           # 更新儲存庫
helm install keda kedacore/keda \                         # 安裝 KEDA
  --namespace keda \                                       # 指定命名空間
  --create-namespace                                       # 自動建立命名空間

# === 驗證安裝 ===
kubectl get pods -n keda                                   # 查看 KEDA Pod
kubectl get crd | grep keda                                # 查看 KEDA CRD

# === 基本 ScaledObject 範例 ===
cat <<EOF | kubectl apply -f -
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: my-scaledobject
spec:
  scaleTargetRef:
    name: my-deployment          # 要擴縮的 Deployment 名稱
  minReplicaCount: 0             # 最小副本數（可縮到零）
  maxReplicaCount: 100           # 最大副本數
  triggers:
    - type: rabbitmq             # 使用 RabbitMQ 擴縮器
      metadata:
        queueName: my-queue      # 佇列名稱
        queueLength: "5"         # 每個 Pod 處理 5 個訊息
EOF
```

---

## 常見錯誤與提示 (Common Mistakes & Tips)

| 常見錯誤 | 正確做法 |
|----------|----------|
| CRD 版本衝突導致升級失敗 | 升級前先備份並刪除舊 CRD |
| Finalizer 導致資源無法刪除 | 使用 `kubectl patch` 移除 Finalizer |
| 忘記建立命名空間 | 使用 `--create-namespace` 參數 |
| ScaledObject 指向不存在的 Deployment | 確認 Deployment 名稱正確且在同一命名空間 |
| Admission Webhook 問題 | 可選擇使用 `-core.yaml` 版本避開 Webhook |

### 提示

- 生產環境建議使用 Helm 安裝，方便版本管理和升級
- 使用 `--server-side` 旗標可避免大型 CRD 的客戶端處理問題
- 定期備份 ScaledObject 設定，方便災難復原

---

## 快速參考 (Quick Reference)

| 指令 | 說明 |
|------|------|
| `helm repo add kedacore https://kedacore.github.io/charts` | 新增 KEDA Helm 儲存庫 |
| `helm install keda kedacore/keda -n keda --create-namespace` | 使用 Helm 安裝 KEDA |
| `helm uninstall keda -n keda` | 使用 Helm 解除安裝 KEDA |
| `kubectl apply --server-side -f keda-2.19.0.yaml` | 使用 YAML 安裝 KEDA |
| `kubectl delete -f keda-2.19.0.yaml` | 使用 YAML 解除安裝 KEDA |
| `kubectl get pods -n keda` | 查看 KEDA Pod 狀態 |
| `kubectl get scaledobject` | 列出所有 ScaledObject |
| `kubectl describe scaledobject <name>` | 查看 ScaledObject 詳細資訊 |
| `kubectl patch scaledobject <name> -p '{"metadata":{"finalizers":null}}' --type=merge` | 移除 Finalizer |
| `microk8s enable dns helm3` | 啟用 MicroK8s 附加元件 |
