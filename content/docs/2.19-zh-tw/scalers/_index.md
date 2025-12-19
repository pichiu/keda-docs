+++
title = "擴縮器"
weight = 2
+++

## TL;DR

KEDA 擴縮器（Scalers）可以偵測 Deployment 是否應該啟動或停用，並為特定事件來源提供自訂指標。KEDA 支援超過 60 種內建擴縮器，涵蓋訊息佇列、資料庫、雲端服務、監控系統等。

---

## 翻譯 (Translation)

KEDA **擴縮器（Scalers）** 可以偵測 Deployment 是否應該啟動或停用，並為特定事件來源提供自訂指標。

---

## 說明 (Explanation)

### 擴縮器類別

| 類別 | 範例 | 說明 |
|------|------|------|
| 訊息佇列 | RabbitMQ, Kafka, AWS SQS | 根據佇列長度擴縮 |
| 資料庫 | PostgreSQL, MySQL, MongoDB | 根據查詢結果擴縮 |
| 雲端服務 | Azure Event Hub, GCP Pub/Sub | 雲端原生事件來源 |
| 監控系統 | Prometheus, Datadog, Grafana | 根據監控指標擴縮 |
| 排程 | Cron | 根據時間排程擴縮 |
| 資源 | CPU, Memory | 根據資源使用率擴縮 |

### 常用擴縮器

- **Apache Kafka** - 根據 Consumer Group 的 lag 擴縮
- **RabbitMQ** - 根據佇列長度擴縮
- **Prometheus** - 根據 Prometheus 查詢結果擴縮
- **AWS SQS** - 根據 SQS 佇列長度擴縮
- **Azure Service Bus** - 根據訊息數量擴縮
- **Cron** - 根據排程時間擴縮

---

## 快速參考 (Quick Reference)

| 擴縮器類型 | 常見用途 |
|------------|----------|
| `rabbitmq` | 處理 RabbitMQ 訊息 |
| `kafka` | 處理 Kafka 事件 |
| `prometheus` | 基於 Prometheus 指標擴縮 |
| `aws-sqs-queue` | 處理 AWS SQS 訊息 |
| `azure-servicebus` | 處理 Azure Service Bus 訊息 |
| `cron` | 定時擴縮 |
| `cpu` | 基於 CPU 使用率擴縮 |
| `memory` | 基於記憶體使用率擴縮 |
| `external` | 自訂外部擴縮器 |
