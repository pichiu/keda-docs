+++
title = "外部擴縮器"
weight = 500
+++

## TL;DR

外部擴縮器（External Scalers）讓使用者可以透過 GRPC 服務擴展 KEDA 的功能，實作自訂的擴縮邏輯。當內建擴縮器無法滿足需求時，可以用 Go、C#、Node.js 等語言開發自己的外部擴縮器。

---

## 翻譯 (Translation)

雖然 KEDA 附帶一組[內建擴縮器](../scalers)，使用者也可以透過實作與內建擴縮器相同介面的 [GRPC](https://grpc.io) 服務來擴展 KEDA。

內建擴縮器在 KEDA 程序/Pod 中執行，而外部擴縮器需要一個外部管理的 GRPC 伺服器，可從 KEDA 存取，並可選擇使用 [TLS 驗證](https://grpc.io/docs/guides/auth/)。KEDA 本身作為 GRPC 用戶端，並為內建擴縮器公開類似的服務介面，因此外部擴縮器可以完全取代內建擴縮器。

本文件描述外部擴縮器介面以及如何在 Go、Node 和 .NET 中實作它們；但有關 GRPC 的更多詳細資訊，請參閱[官方 GRPC 文件](https://grpc.io/docs/)

> 想了解現有的外部擴縮器？探索我們的[外部擴縮器社群](https://github.com/kedacore/external-scalers)。

### 概述

#### 內建擴縮器介面

由於外部擴縮器鏡像內建擴縮器的介面，因此值得熟悉內建擴縮器實作的 Go `interface`：

```go
// Scaler 介面
type Scaler interface {
	// GetMetricsAndActivity 返回指標名稱的指標值和活動狀態
	GetMetricsAndActivity(ctx context.Context, metricName string) ([]external_metrics.ExternalMetricValue, bool, error)
	// GetMetricSpecForScaling 返回此擴縮器用於決定 ScaleTarget 擴縮的指標。這用於建構為此擴縮物件建立的 HPA 規格。使用的標籤應與 GetMetrics 中使用的選擇器匹配
	GetMetricSpecForScaling(ctx context.Context) []v2.MetricSpec
	// Close 當擴縮器不再使用或被銷毀時，關閉任何需要處理的資源
	Close(ctx context.Context) error
}

// PushScaler 介面
type PushScaler interface {
	Scaler

	// Run 是 active 通道的唯一寫入者，完成後必須關閉它。
	Run(ctx context.Context, active chan<- bool)
}
```

`Scaler` 介面定義了 3 個方法：

- `Close` 被呼叫以允許擴縮器清理連接或其他資源。
- `GetMetricSpecForScaling` 返回擴縮器的 HPA 定義的目標值。更多詳細資訊請參閱[實作 `GetMetricSpec`](#5-implementing-getmetricspec)。
- `GetMetricsAndActivity` 在 `pollingInterval` 時被呼叫。當 activity 返回 `true` 時，KEDA 會擴縮到指標返回的值，受 ScaledObject/ScaledJob 上的 `maxReplicaCount` 限制。
  當返回 `false` 時，KEDA 會縮減到 `minReplicaCount` 或可選的 `idleReplicaCount`。更多關於預設值以及這些選項如何協同運作的詳細資訊可以在 [ScaledObjectSpec](../reference/scaledobject-spec) 中找到。

> 請參閱 [HPA 文件](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)了解 HPA 如何根據指標值和目標值計算 `replicaCount`。KEDA 支援外部指標的 `AverageValue` 和 `Value` 指標目標類型。當使用 `AverageValue`（預設指標類型）時，外部擴縮器返回的指標值將除以副本數。

`PushScaler` 介面新增了一個 `Run` 方法。此方法接收一個推送通道（`active`），擴縮器可以隨時在其上發送 `true`。此機制的目的是獨立於 `pollingInterval` 啟動擴縮操作。

#### 外部擴縮器 GRPC 介面

KEDA 附帶 2 個外部擴縮器 [`external`](../scalers/external.md) 和 [`external-push`](../scalers/external-push.md)。

ScaledObject 中的設定指向一個實作 [`externalscaler.proto`](https://github.com/kedacore/keda/blob/main/pkg/scalers/externalscaler/externalscaler.proto) GRPC 契約的 GRPC 服務端點：

```proto
service ExternalScaler {
    rpc IsActive(ScaledObjectRef) returns (IsActiveResponse) {}
    rpc StreamIsActive(ScaledObjectRef) returns (stream IsActiveResponse) {}
    rpc GetMetricSpec(ScaledObjectRef) returns (GetMetricSpecResponse) {}
    rpc GetMetrics(GetMetricsRequest) returns (GetMetricsResponse) {}
}
```

此契約的大部分與內建擴縮器類似：

- `GetMetricsSpec` 對應 `Scaler` 介面中用於建立 HPA 定義的對應方法。
- `IsActive` 和 `GetMetrics` 對應 `Scaler` 介面上的 `GetMetricsAndActivity` 方法。
- `StreamIsActive` 對應 `PushScaler` 介面上的 `Run` 方法。

然而，有一些明顯的差異：

- 沒有 `Close` 方法。擴縮器預期在其整個生命週期內保持功能。
- `IsActive`、`StreamIsActive` 和 `GetMetricsSpec` 使用包含 scaledObject 名稱/命名空間以及觸發器中定義的 `metadata` 內容的 `ScaledObjectRef` 呼叫。

#### 範例

給定以下 `ScaledObject`：

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: scaledobject-name
  namespace: scaledobject-namespace
spec:
  scaleTargetRef:
    name: deployment-name
  triggers:
    - type: external-push
      metadata:
        scalerAddress: service-address.svc.local:9090
        key1: value1
        key2: value2
```

KEDA 會在協調 `ScaledObject` 後立即嘗試建立到 `service-address.svc.local:9090` 的 GRPC 連接。然後它會進行以下 RPC 呼叫：

- `IsActive` - KEDA 進行初始呼叫，然後在每個 `pollingInterval` 進行一次呼叫
- `StreamIsActive` - KEDA 進行初始呼叫，擴縮器預期維護一個長期連接（在 GRPC 術語中稱為 `stream`）。外部推送擴縮器可以隨時向 KEDA 發送 `IsActive` 事件。KEDA 只有在需要重新連接時才會嘗試另一次 `StreamIsActive` 呼叫
- `GetMetricsSpec` - KEDA 會使用傳入的 `ScaledObjectRef` 參數中的以下資料進行初始呼叫
- `GetMetrics` - KEDA 會在每個 `pollingInterval` 呼叫此方法以取得 `GetMetricSpec` 返回的名稱的即時指標值。

```json
{
  "name": "scaledobject-name",
  "namespace": "scaledobject-namespace",
  "scalerMetadata": {
    "scalerAddress": "service-address.svc.local:9090",
    "key1": "value1",
    "key2": "value2"
  }
}
```

> **注意**：如果 `spec.triggers.type` 是 `external`，KEDA 會發出上述所有 RPC 呼叫，除了 `StreamIsActive`。必須是 `external-push` 才會呼叫 `StreamIsActive`。

### 實作 KEDA 外部擴縮器 GRPC 介面

#### 1. 下載 [`externalscaler.proto`](https://github.com/kedacore/keda/blob/main/pkg/scalers/externalscaler/externalscaler.proto)

#### 2. 準備專案（依語言）

**Golang：**
```bash
go get github.com/golang/protobuf/protoc-gen-go@v1.3.2
go mod init example.com/external-scaler/sample
mkdir externalscaler
protoc externalscaler.proto --go_out=plugins=grpc:externalscaler
```

**C#：**
```bash
dotnet new console -o ExternalScalerSample
cd ExternalScalerSample
dotnet add package Grpc.AspNetCore
```

**JavaScript：**
```bash
npm install --save grpc request
```

---

## 說明 (Explanation)

### 外部擴縮器 vs 內建擴縮器

| 特性 | 內建擴縮器 | 外部擴縮器 |
|------|------------|------------|
| 執行位置 | KEDA Pod 內 | 獨立的 GRPC 伺服器 |
| 開發語言 | Go | 任何支援 GRPC 的語言 |
| 維護者 | KEDA 團隊 | 使用者/社群 |
| 適用場景 | 通用場景 | 自訂需求 |

### GRPC 方法說明

| 方法 | 說明 | 呼叫時機 |
|------|------|----------|
| `IsActive` | 判斷是否應該啟動擴縮 | 每個 pollingInterval |
| `StreamIsActive` | 長連接推送方式 | 初始連接後保持 |
| `GetMetricSpec` | 返回 HPA 目標值 | 初始設定時 |
| `GetMetrics` | 返回當前指標值 | 每個 pollingInterval |

---

## 實作範例 (Practical Example)

```yaml
# 外部擴縮器的 ScaledObject 設定
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: external-scaler-demo
spec:
  scaleTargetRef:
    name: my-deployment
  triggers:
    - type: external                          # 使用輪詢模式
      metadata:
        scalerAddress: my-scaler.default:9090 # GRPC 服務地址
        customParam: value                    # 自訂參數會傳給擴縮器
---
# 或使用推送模式
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: external-push-scaler-demo
spec:
  scaleTargetRef:
    name: my-deployment
  triggers:
    - type: external-push                     # 使用推送模式
      metadata:
        scalerAddress: my-scaler.default:9090
```

```bash
# 部署外部擴縮器服務（範例）
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-external-scaler
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-external-scaler
  template:
    metadata:
      labels:
        app: my-external-scaler
    spec:
      containers:
      - name: scaler
        image: my-external-scaler:latest
        ports:
        - containerPort: 9090
---
apiVersion: v1
kind: Service
metadata:
  name: my-scaler
spec:
  selector:
    app: my-external-scaler
  ports:
  - port: 9090
    targetPort: 9090
EOF
```

---

## 常見錯誤與提示 (Common Mistakes & Tips)

| 常見錯誤 | 正確做法 |
|----------|----------|
| 忘記實作所有必要的 GRPC 方法 | 至少實作 IsActive、GetMetricSpec、GetMetrics |
| GRPC 服務無法從 KEDA 存取 | 確認 Service 和網路設定正確 |
| 混淆 external 和 external-push | external 使用輪詢，external-push 使用長連接推送 |
| 沒有處理連接錯誤 | 實作適當的錯誤處理和重試邏輯 |

### 提示

- 使用 `external-push` 可以實現更即時的擴縮響應
- 外部擴縮器社群有許多現成的實作可以參考
- 開發時可以使用 `grpcurl` 測試 GRPC 服務
- 考慮使用 TLS 保護 GRPC 連接的安全性

---

## 快速參考 (Quick Reference)

| 觸發器類型 | 說明 | 使用場景 |
|------------|------|----------|
| `external` | 輪詢模式外部擴縮器 | 定期檢查指標 |
| `external-push` | 推送模式外部擴縮器 | 即時事件響應 |

| GRPC 方法 | 必要性 | 說明 |
|-----------|--------|------|
| `IsActive` | 必要 | 判斷是否啟動 |
| `GetMetricSpec` | 必要 | 返回目標值定義 |
| `GetMetrics` | 必要 | 返回當前指標值 |
| `StreamIsActive` | 僅 external-push | 長連接推送 |

| 指令 | 說明 |
|------|------|
| `grpcurl -plaintext localhost:9090 list` | 列出 GRPC 服務 |
| `grpcurl -plaintext localhost:9090 externalscaler.ExternalScaler/IsActive` | 測試 IsActive 方法 |
