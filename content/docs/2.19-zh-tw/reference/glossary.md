+++
title = "術語表"
weight = 1000
+++

本文件定義理解文件和設定使用 KEDA 所需的各種術語。

## Admission Webhook

[在 Kubernetes 中](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)，處理 Admission 請求的 HTTP 回呼。KEDA 使用 Admission Webhook 來驗證和修改 ScaledObject 資源。

## Agent

KEDA Operator 持有的主要角色。Agent 啟動和停用 Kubernetes Deployment 以實現縮減到零和從零擴展。

## Cluster（叢集）

[在 Kubernetes 中](https://kubernetes.io/docs/reference/glossary/?fundamental=true#term-cluster)，執行容器化應用程式的一組或多個節點。

## CRD

Custom Resource Definition（自訂資源定義）。[在 Kubernetes 中](https://kubernetes.io/docs/reference/glossary/?fundamental=true#term-CustomResourceDefinition)，擴展 Kubernetes API 的自訂資源，如 ScaledObjects，具有自訂欄位和行為。

## Event（事件）

由事件來源捕獲的顯著事件，KEDA 可能使用它作為觸發器來擴縮容器或部署。

## Event Source（事件來源）

外部系統，如 Kafka、RabbitMQ，產生 KEDA 可以使用擴縮器監控的事件。

## Grafana

開源監控平台，可以視覺化 KEDA 收集的指標。

## gRPC Remote Procedure Calls (gRPC)

gRPC 遠端程序呼叫。KEDA 元件用於通訊的開源遠端程序呼叫框架。

## HPA

Horizontal Pod Autoscaler（水平 Pod 自動擴縮器）。Kubernetes 自動擴縮器。預設根據 CPU/記憶體使用量進行擴縮。KEDA 使用 HPA 來擴縮 Kubernetes 叢集和部署。

## KEDA

Kubernetes Event-Driven Autoscaling（Kubernetes 事件驅動自動擴縮）。單一用途、輕量級的自動擴縮器，可以根據事件指標擴縮 Kubernetes 工作負載。

## Metric（指標）

事件來源的測量值，如佇列長度或回應延遲，KEDA 使用它來決定擴縮。

## OpenTelemetry

KEDA 用於檢測應用程式和收集指標的可觀測性框架。

## Operator

核心 KEDA 元件，監控指標並相應地擴縮工作負載。

## Prometheus

開源監控系統，可以從 KEDA 抓取和儲存指標。

## Scaled Object

自訂資源，定義 KEDA 應如何根據事件擴縮工作負載。

## Scaled Job

KEDA 用於擴縮應用程式的自訂資源。

## Scaler（擴縮器）

將 KEDA 與特定事件來源整合以收集指標的元件。

## Stateful Set

具有持久資料的 Kubernetes 工作負載。KEDA 可以擴縮 StatefulSet。

## TLS

Transport Layer Security（傳輸層安全性）。KEDA 使用 TLS 加密 KEDA 元件之間的通訊。

## Webhook

用於從外部來源通知 KEDA 事件的 HTTP 回呼。

[在 Kubernetes 中](https://kubernetes.io/docs/reference/access-authn-authz/webhook/)，用作事件通知機制的 HTTP 回呼。
