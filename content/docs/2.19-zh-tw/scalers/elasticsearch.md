+++
title = "Elasticsearch"
availability = "v2.5+"
maintainer = "Community"
category = "Data & Storage"
description = "根據 Elasticsearch 查詢結果擴縮應用程式。"
go_file = "elasticsearch_scaler"
+++

## TL;DR

Elasticsearch 擴縮器根據 Elasticsearch 搜尋模板（Search Template）或查詢（Query）結果來擴縮應用程式。可用於監控 Elasticsearch 中的文件數量、交易量等指標。支援使用者名稱/密碼和 Cloud ID/API Key 兩種驗證方式。

---

## 翻譯 (Translation)

### 觸發器規格

此規格描述根據 [Elasticsearch 搜尋模板](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-template.html)查詢或 [Elasticsearch 查詢](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html)進行擴縮的 `elasticsearch` 觸發器。

觸發器需要以下資訊，但只需要 searchTemplateName **或** query 其中之一：

```yaml
triggers:
  - type: elasticsearch
    metadata:
      addresses: "http://localhost:9200"
      username: "elastic"
      passwordFromEnv: "ELASTIC_PASSWORD"
      index: "my-index"
      searchTemplateName: "my-search-template-name"
      query: "my-query"
      parameters: "param1:value1;param2:value2"
      valueLocation: "hits.total.value"
      targetValue: "1.1"
      activationTargetValue: "5.5"
```

**參數列表：**

- `addresses` - Elasticsearch 叢集客戶端節點的主機和埠口，以逗號分隔。
- `username` - 連接 Elasticsearch 叢集的使用者名稱。
- `passwordFromEnv` - 包含驗證密碼的環境變數名稱。
- `index` - 執行搜尋模板查詢的索引。支援使用分號（`;`）分隔多個索引。
- `searchTemplateName` - 要執行的搜尋模板名稱。
- `query` - 要執行的查詢。
- `targetValue` - 擴縮的目標值。當 API 提供的指標等於或高於此值時，KEDA 將開始擴展。當指標為 0 或更低時，KEDA 將縮減到 0。（可以是浮點數）
- `activationTargetValue` - 啟動擴縮器的目標值。（預設：`0`，可選，可以是浮點數）
- `parameters` - 搜尋模板使用的參數。支援使用分號（`;`）分隔多個參數。
- `valueLocation` - [GJSON 路徑表示法](https://github.com/tidwall/gjson#path-syntax)，指向包含指標值的 payload 欄位。
- `unsafeSsl` - 透過 HTTPS 連線時跳過憑證驗證。（值：`true`、`false`，預設：`false`，可選）
- `ignoreNullValues` - 設為 `true` 時忽略發現 Null 值時的錯誤。設為 `false` 時，發現 Null 值將返回錯誤。（值：`true`、`false`，預設：`false`，可選）

### 驗證參數

可使用使用者名稱/密碼或 Elastic Cloud 的 cloudID/apiKey 進行驗證。

**密碼驗證：**

- `username` - 連接 Elasticsearch 叢集的使用者名稱。
- `password` - 設定使用者登入 Elasticsearch 叢集的密碼。

**Cloud ID 和 API Key 驗證：**

[Cloud ID](https://www.elastic.co/guide/en/cloud/current/ec-cloud-id.html) 和 API Key 可用於 Elastic Cloud Service。

- `cloudID` - 連接 Elastic Cloud 上 ElasticSearch 的 CloudID。
- `apiKey` - 連接 Elastic Cloud 上 ElasticSearch 的 API 金鑰。

---

## 說明 (Explanation)

### 查詢方式比較

| 方式 | 說明 | 使用場景 |
|------|------|----------|
| `searchTemplateName` | 使用預定義的搜尋模板 | 可重複使用的查詢 |
| `query` | 直接在設定中定義查詢 | 一次性查詢 |

### valueLocation 說明

使用 GJSON 路徑語法指向結果中的值：

| 範例 | 說明 |
|------|------|
| `hits.total.value` | 匹配文件總數 |
| `aggregations.my_agg.value` | 聚合結果值 |
| `aggregations.transaction_count.value` | 交易計數聚合 |

---

## 實作範例 (Practical Example)

```yaml
# 使用搜尋模板
apiVersion: v1
kind: Secret
metadata:
  name: elasticsearch-secrets
type: Opaque
data:
  password: cGFzc3cwcmQh  # base64 編碼
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: elasticsearch-auth
spec:
  secretTargetRef:
    - parameter: password
      name: elasticsearch-secrets
      key: password
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: elasticsearch-scaledobject
spec:
  scaleTargetRef:
    name: my-processor-deployment
  pollingInterval: 15
  cooldownPeriod: 120
  minReplicaCount: 0
  maxReplicaCount: 20
  triggers:
    - type: elasticsearch
      metadata:
        addresses: "http://elasticsearch:9200"
        username: "elastic"
        index: "logs-*"
        searchTemplateName: "pending-tasks-template"
        valueLocation: "hits.total.value"
        targetValue: "100"
        parameters: "status:pending"
      authenticationRef:
        name: elasticsearch-auth
```

```yaml
# 使用直接查詢 - 監控 APM 交易
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: elasticsearch-query-scaledobject
spec:
  scaleTargetRef:
    name: my-processor-deployment
  triggers:
    - type: elasticsearch
      metadata:
        addresses: "http://elasticsearch:9200"
        username: "elastic"
        index: "apm-*"
        query: |
          {
            "size": 0,
            "query": {
              "bool": {
                "must": [
                  {
                    "term": {
                      "service.name": "my-application"
                    }
                  },
                  {
                    "term": {
                      "service.environment": "production"
                    }
                  },
                  {
                    "range": {
                      "@timestamp": {
                        "gte": "now-2m",
                        "lte": "now-1m"
                      }
                    }
                  }
                ]
              }
            },
            "aggs": {
              "transaction_count": {
                "cardinality": {
                  "field": "transaction.id"
                }
              }
            }
          }
        valueLocation: "aggregations.transaction_count.value"
        targetValue: "1000"
      authenticationRef:
        name: elasticsearch-auth
```

```yaml
# 使用 Elastic Cloud
apiVersion: v1
kind: Secret
metadata:
  name: elastic-cloud-secrets
type: Opaque
data:
  cloudID: <base64-encoded-cloud-id>
  apiKey: <base64-encoded-api-key>
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: elastic-cloud-auth
spec:
  secretTargetRef:
    - parameter: cloudID
      name: elastic-cloud-secrets
      key: cloudID
    - parameter: apiKey
      name: elastic-cloud-secrets
      key: apiKey
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: elastic-cloud-scaledobject
spec:
  scaleTargetRef:
    name: my-processor-deployment
  triggers:
    - type: elasticsearch
      metadata:
        index: "logs-*"
        query: '{"query":{"match":{"status":"pending"}}}'
        valueLocation: "hits.total.value"
        targetValue: "50"
      authenticationRef:
        name: elastic-cloud-auth
```

```bash
# 測試 Elasticsearch 查詢
curl -X GET "http://localhost:9200/my-index/_search" \
  -H "Content-Type: application/json" \
  -u elastic:password \
  -d '{"query":{"match":{"status":"pending"}}}'

# 使用搜尋模板
curl -X GET "http://localhost:9200/my-index/_search/template" \
  -H "Content-Type: application/json" \
  -u elastic:password \
  -d '{"id":"my-template","params":{"status":"pending"}}'

# 查看 ScaledObject 狀態
kubectl describe scaledobject elasticsearch-scaledobject
```

---

## 常見錯誤與提示 (Common Mistakes & Tips)

| 常見錯誤 | 正確做法 |
|----------|----------|
| 同時設定 searchTemplateName 和 query | 只需選擇其中之一 |
| valueLocation 路徑錯誤 | 使用正確的 GJSON 路徑語法 |
| 查詢結果為 null 導致錯誤 | 設定 `ignoreNullValues: true` |
| SSL 憑證問題 | 使用 `unsafeSsl: true` 跳過驗證 |

### 提示

- 使用 `size: 0` 可以只返回聚合結果，減少資料傳輸
- 對於 Elastic Cloud，使用 Cloud ID 和 API Key 而非使用者名稱/密碼
- 多個索引可使用分號分隔，如 `index: "logs-2024-01;logs-2024-02"`
- 使用時間範圍查詢可以監控近期資料

### 查詢範例

```json
// 計算待處理任務
{
  "query": {
    "match": {
      "status": "pending"
    }
  }
}

// 使用時間範圍的聚合查詢
{
  "size": 0,
  "query": {
    "range": {
      "@timestamp": {
        "gte": "now-5m"
      }
    }
  },
  "aggs": {
    "count": {
      "value_count": {
        "field": "_id"
      }
    }
  }
}
```

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `addresses` | Elasticsearch 節點位址 | 無（必填）|
| `index` | 索引名稱 | 無（必填）|
| `searchTemplateName` | 搜尋模板名稱 | 無（與 query 二選一）|
| `query` | 查詢語句 | 無（與 searchTemplateName 二選一）|
| `valueLocation` | 值的 GJSON 路徑 | 無（必填）|
| `targetValue` | 目標值 | 無（必填）|
| `activationTargetValue` | 啟動閾值 | 0 |
| `unsafeSsl` | 跳過 SSL 驗證 | false |
| `ignoreNullValues` | 忽略 null 值 | false |

| 驗證方式 | 必要參數 |
|----------|----------|
| 密碼 | username, password |
| Elastic Cloud | cloudID, apiKey |

| 指令 | 說明 |
|------|------|
| `kubectl get scaledobject` | 列出 ScaledObject |
| `curl -X GET "http://es:9200/index/_search"` | 測試查詢 |
