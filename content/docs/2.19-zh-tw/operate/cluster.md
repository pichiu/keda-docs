+++
title = "叢集"
description = "在叢集中執行 KEDA 的指南與需求"
weight = 100
+++

## TL;DR

本文件說明在 Kubernetes 叢集中執行 KEDA 的需求和設定，包括 Kubernetes 版本相容性、叢集容量需求、防火牆設定、高可用性配置、HTTP 設定、日誌設定等維運相關主題。

---

## 翻譯 (Translation)

### 需求

#### Kubernetes 相容性

KEDA 支援的 Kubernetes 版本窗口稱為「N-2」，這意味著 KEDA 將至少支援在 N-2 版本上執行。

維護者可以根據所需的 CRD 決定支援更多次要版本，但不保證。

以下相容性矩陣顯示每個 KEDA 版本支援的 Kubernetes 版本：

| KEDA  | Kubernetes    |
| ----- | ------------- |
| v2.19 | v1.32 - v1.34 |
| v2.18 | v1.31 - v1.33 |
| v2.17 | v1.30 - v1.32 |
| v2.16 | v1.29 - v1.31 |
| v2.15 | v1.28 - v1.30 |
| v2.14 | v1.27 - v1.29 |
| v2.13 | v1.27 - v1.29 |
| v2.12 | v1.26 - v1.28 |
| v2.11 | v1.25 - v1.27 |
| v2.10 | v1.24 - v1.26 |

#### 叢集容量

KEDA 執行時在生產環境設定中需要以下資源：

| 部署 | CPU | 記憶體 |
|------|-----|--------|
| Admission Webhooks | Limit: 1, Request: 100m | Limit: 1000Mi, Request: 100Mi |
| Metrics Server | Limit: 1, Request: 100m | Limit: 1000Mi, Request: 100Mi |
| Operator | Limit: 1, Request: 100m | Limit: 1000Mi, Request: 100Mi |

這些是透過 YAML 部署時使用的預設值。

#### 防火牆

KEDA 需要在叢集內部可存取才能進行自動擴縮。

以下是 KEDA 運作所需開放的埠口概覽：

| 埠口 | 用途 | 備註 |
|------|------|------|
| `443` | Kubernetes API 伺服器用於取得指標 | 所有平台都需要，使用控制平面 → 埠口 443 到 Service IP 範圍的通訊。不適用於 Google Cloud |
| `6443` | Kubernetes API 伺服器用於取得指標 | 僅 Google Cloud 需要，使用控制平面 → 埠口 6443 到 Pod IP 範圍的通訊 |

### 高可用性

由於上游限制，KEDA 不提供完整的高可用性支援。

以下是所有 KEDA 部署的 HA 說明概覽：

| 部署 | 支援副本數 | 說明 |
|------|------------|------|
| Metrics Server | 1 | 可執行多個 Metrics Server 副本，建議在 kube-apiserver 新增 `--enable-aggregator-routing=true` CLI 參數以負載平衡請求。但是，一個 Kubernetes 叢集中只能有一個活動的 Metrics Server 服務 external.metrics.k8s.io |
| Operator | 2 | 雖然可執行多個 Operator 副本，但只有一個實例會活動。其餘將處於待命狀態，可減少故障時的停機時間。多個副本不會提高 KEDA 的效能 |

### HTTP 逾時

某些擴縮器會向外部伺服器（如雲端服務）發出 HTTP 請求。每個適用的擴縮器使用自己的專用 HTTP 客戶端和連線池，預設情況下每個客戶端會在 3 秒後使 HTTP 請求逾時。

您可以透過在 KEDA Operator 部署上設定 `KEDA_HTTP_DEFAULT_TIMEOUT` 環境變數（以毫秒為單位）來覆蓋此預設值。

### HTTP 連線：停用 Keep Alive

預設情況下每個 HTTP 連線都啟用 keep alive 行為，這在某些情況下可能會堆積大量連線。

您可以透過新增相關環境變數到 KEDA Operator 和 KEDA Metrics Server 部署來停用所有 HTTP 連線的 keep alive：

```yaml
- env:
    KEDA_HTTP_DISABLE_KEEP_ALIVE: true
```

### HTTP 代理

某些公司要求透過代理伺服器存取外部伺服器，可以在 KEDA Operator 和 KEDA Metrics Server 部署中新增相關環境變數：

```yaml
- env:
    HTTP_PROXY: http://proxy.server:port
    HTTPS_PROXY: http://proxy.server:port
    NO_PROXY: 10.0.0.0/8
```

### HTTP TLS 最低版本

預設情況下，KEDA 使用 TLS1.2 作為最低 TLS 版本。可以透過環境變數 `KEDA_HTTP_MIN_TLS_VERSION` 設定。

```yaml
- env:
    KEDA_HTTP_MIN_TLS_VERSION: TLS13
```

允許的值：`TLS13`、`TLS12`、`TLS11` 和 `TLS10`。

### 限制 KEDA 監控的命名空間

預設情況下，KEDA 控制器監控 Kubernetes 叢集中所有命名空間的事件。可以透過環境變數 `WATCH_NAMESPACE` 限制。

```yaml
- env:
    WATCH_NAMESPACE: keda,production
```

### 限制 Secret 存取

預設情況下，KEDA 需要在叢集角色中新增 `secrets` 權限。這可能導致安全風險。

若要限制 secret 存取，可以在 KEDA Operator 和 KEDA Metrics Server 中新增環境變數：

```yaml
env:
  - name: KEDA_RESTRICT_SECRET_ACCESS
    value: "true"
```

### 日誌

KEDA 使用 zap 發出日誌。以下參數可用於設定日誌行為：

- `zap-encoder`：Zap 日誌編碼（`json` 或 `console`）。預設：`console`
- `zap-log-level`：Zap 級別（`debug`、`info`、`error`）。預設：`info`
- `zap-time-encoding`：Zap 時間編碼。預設：`rfc3339`

---

## 快速參考 (Quick Reference)

| 環境變數 | 說明 | 預設值 |
|----------|------|--------|
| `KEDA_HTTP_DEFAULT_TIMEOUT` | HTTP 逾時（毫秒）| 3000 |
| `KEDA_HTTP_DISABLE_KEEP_ALIVE` | 停用 keep alive | false |
| `KEDA_HTTP_MIN_TLS_VERSION` | 最低 TLS 版本 | TLS12 |
| `WATCH_NAMESPACE` | 監控的命名空間 | 所有 |
| `KEDA_RESTRICT_SECRET_ACCESS` | 限制 secret 存取 | false |

| 指令 | 說明 |
|------|------|
| `kubectl get pods -n keda` | 查看 KEDA Pods |
| `kubectl logs -n keda deployment/keda-operator` | 查看 Operator 日誌 |
