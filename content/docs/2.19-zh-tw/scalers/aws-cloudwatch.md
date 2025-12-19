+++
title = "AWS CloudWatch"
availability = "v1.0+"
maintainer = "Community"
category = "Metrics"
description = "根據 AWS CloudWatch 擴縮應用程式。"
go_file = "aws_cloudwatch_scaler"
+++

## TL;DR

AWS CloudWatch 擴縮器根據 AWS CloudWatch 指標來擴縮應用程式。可使用維度查詢或 CloudWatch Metrics Insights 表達式查詢。支援 Pod Identity、IAM 角色和 IAM 使用者等多種驗證方式。

---

## 翻譯 (Translation)

### 觸發器規格

此規格描述根據 AWS CloudWatch 進行擴縮的 `aws-cloudwatch` 觸發器。

```yaml
triggers:
- type: aws-cloudwatch
  metadata:
    # 可選：命名空間
    namespace: AWS/SQS
    # 可選：維度名稱
    dimensionName: QueueName
    # 可選：維度值
    dimensionValue: keda
    # 可選：表達式查詢
    expression: SELECT MAX("ApproximateNumberOfMessagesVisible") FROM "AWS/SQS" WHERE QueueName = 'keda'
    # 可選：指標名稱
    metricName: ApproximateNumberOfMessagesVisible
    targetMetricValue: "2.1"
    minMetricValue: "1.5"
    # 可選：忽略空值
    ignoreNullValues: "false"
    # 必填：區域
    awsRegion: "eu-west-1"
    # 可選：指定指標來源 AWS 帳戶 ID（用於 CloudWatch 跨帳戶可觀測性）
    awsAccountId: ""
    # 可選：AWS 端點 URL
    awsEndpoint: ""
    # 可選：指標收集時間
    metricCollectionTime: "300"    # 預設 300
    # 可選：指標統計方法
    metricStat: "Average"          # 預設 "Average"
    # 可選：指標統計週期
    metricStatPeriod: "300"        # 預設 300
    # 可選：指標單位
    metricUnit: "Count"            # 預設 ""
    # 可選：指標結束時間偏移
    metricEndTimeOffset: "60"      # 預設 0
```

**參數列表：**

- `awsRegion` - AWS CloudWatch 的 AWS 區域。
- `awsAccountId` - 使用 CloudWatch 跨帳戶可觀測性時，指定指標來源 AWS 帳戶 ID。
- `awsEndpoint` - 覆蓋預設 AWS 端點的端點 URL。（預設：`""`，可選）
- `namespace` - 指標所在的 AWS CloudWatch 命名空間（未指定 `expression` 時必填）
- `metricName` - AWS CloudWatch 指標名稱（未指定 `expression` 時必填）
- `dimensionName` - 支援使用 ";" 分隔符指定多個維度名稱，例如 `dimensionName: QueueName;QueueName`（未指定 `expression` 時必填）
- `dimensionValue` - 支援使用 ";" 分隔符指定多個維度值，例如 `dimensionValue: queue1;queue2`（未指定 `expression` 時必填）
- `expression` - 支援使用[表達式](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch-metrics-insights-querylanguage.html)進行查詢（未指定 `dimensionName` 和 `dimensionValue` 時必填）
- `metricCollectionTime` - 擴縮器應檢查多久以前（秒）的 AWS CloudWatch。用於定義 **StartTime**。值必須大於 `metricStatPeriod`。（預設：`300`，可選）
- `metricStat` - 查詢使用的統計指標。用於定義 **Stat**。支援所有 CloudWatch 統計方法，包括百分位數（如 `p95`、`p50`）和修剪平均值（如 `tm99`、`tm95`）。（預設：`Average`，可選）
- `metricStatPeriod` - 相關查詢使用的頻率。用於定義 **Period**。值必須是 CloudWatch 支援的值（1、5、10、30 或 60 的倍數）。（預設：`300`，可選）
- `metricUnit` - 查詢使用的單位。用於定義 **Unit**。（預設：`none`，可選）
- `metricEndTimeOffset` - 偏移 **EndTime** 的秒數。由於 CloudWatch 使用的最終一致性模型，最新的資料點可能不準確。（預設：`0`，可選）
- `minMetricValue` - CloudWatch 返回空響應時的返回值。（預設：0，可以是浮點數）
- `ignoreNullValues` - 描述指標查詢在響應中未返回任何指標值時的行為。設為 `true` 時，擴縮器將根據提供的 `minMetricValue` 擴縮工作負載。設為 `false` 時，擴縮器將返回錯誤且不調整工作負載的擴縮。（預設：`true`，可選）
- `targetMetricValue` - 指標的目標值。（預設：0，可以是浮點數）
- `activationTargetMetricValue` - 啟動擴縮器的目標值。（預設：`0`，可選，可以是浮點數）

### 驗證參數

您可以使用 `TriggerAuthentication` CRD 透過提供角色 ARN 或一組 IAM 憑證來設定驗證。

**Pod Identity 驗證：**
- `podIdentity.provider` - 需要在 `TriggerAuthentication` 中設定，且 Pod/服務帳戶必須為您的 Pod Identity 提供者正確設定。

**角色型驗證：**
- `awsRoleArn` - Amazon Resource Names (ARNs) 唯一識別 AWS 資源。

**憑證型驗證：**
- `awsAccessKeyID` - 使用者 ID。
- `awsSecretAccessKey` - 使用者的驗證存取金鑰。
- `awsSessionToken` - 工作階段令牌，僅在使用臨時憑證時需要。

### IAM 權限

用於驗證 AWS CloudWatch 的使用者或角色必須具有 `cloudwatch:GetMetricData` 權限。

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudWatchGetMetricData",
      "Effect": "Allow",
      "Action": "cloudwatch:GetMetricData",
      "Resource": "*"
    }
  ]
}
```

---

## 說明 (Explanation)

### 查詢方式比較

| 方式 | 說明 | 使用場景 |
|------|------|----------|
| 維度查詢 | 使用 namespace + metricName + dimensionName/Value | 單一指標 |
| 表達式查詢 | 使用 CloudWatch Metrics Insights 語法 | 複雜查詢、聚合 |

### 常用統計方法

| 方法 | 說明 |
|------|------|
| `Average` | 平均值（預設）|
| `Sum` | 總和 |
| `Maximum` | 最大值 |
| `Minimum` | 最小值 |
| `SampleCount` | 樣本數 |
| `p99`、`p95` 等 | 百分位數 |

### 時間參數關係

```
StartTime = 現在 - metricCollectionTime
EndTime = 現在 - metricEndTimeOffset
Period = metricStatPeriod
```

---

## 實作範例 (Practical Example)

```yaml
# 使用維度查詢監控 SQS 佇列
apiVersion: v1
kind: Secret
metadata:
  name: aws-secrets
data:
  AWS_ACCESS_KEY_ID: <base64-encoded-key-id>
  AWS_SECRET_ACCESS_KEY: <base64-encoded-secret-key>
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: aws-cloudwatch-auth
spec:
  secretTargetRef:
    - parameter: awsAccessKeyID
      name: aws-secrets
      key: AWS_ACCESS_KEY_ID
    - parameter: awsSecretAccessKey
      name: aws-secrets
      key: AWS_SECRET_ACCESS_KEY
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: aws-cloudwatch-scaledobject
spec:
  scaleTargetRef:
    name: my-deployment
  minReplicaCount: 0
  maxReplicaCount: 20
  triggers:
    - type: aws-cloudwatch
      metadata:
        namespace: AWS/SQS
        dimensionName: QueueName
        dimensionValue: my-queue
        metricName: ApproximateNumberOfMessagesVisible
        targetMetricValue: "5"
        minMetricValue: "0"
        awsRegion: "ap-northeast-1"
      authenticationRef:
        name: aws-cloudwatch-auth
```

```yaml
# 使用 CloudWatch Metrics Insights 表達式
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: cloudwatch-expression-scaledobject
spec:
  scaleTargetRef:
    name: my-deployment
  triggers:
    - type: aws-cloudwatch
      metadata:
        expression: SELECT MAX("ApproximateNumberOfMessagesVisible") FROM "AWS/SQS" WHERE QueueName = 'my-queue'
        targetMetricValue: "10"
        awsRegion: "ap-northeast-1"
      authenticationRef:
        name: aws-cloudwatch-auth
```

```yaml
# 使用 Pod Identity
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: aws-pod-identity-auth
spec:
  podIdentity:
    provider: aws
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: cloudwatch-pod-identity-scaledobject
spec:
  scaleTargetRef:
    name: my-deployment
  triggers:
    - type: aws-cloudwatch
      metadata:
        namespace: AWS/EC2
        dimensionName: InstanceId
        dimensionValue: i-1234567890abcdef0
        metricName: CPUUtilization
        targetMetricValue: "70"
        metricStat: "Average"
        metricStatPeriod: "60"
        awsRegion: "ap-northeast-1"
      authenticationRef:
        name: aws-pod-identity-auth
```

```bash
# 使用 AWS CLI 查詢 CloudWatch 指標
aws cloudwatch get-metric-data \
  --metric-data-queries '[
    {
      "Id": "m1",
      "MetricStat": {
        "Metric": {
          "Namespace": "AWS/SQS",
          "MetricName": "ApproximateNumberOfMessagesVisible",
          "Dimensions": [{"Name": "QueueName", "Value": "my-queue"}]
        },
        "Period": 300,
        "Stat": "Average"
      }
    }
  ]' \
  --start-time $(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --region ap-northeast-1

# 查看 ScaledObject 狀態
kubectl describe scaledobject aws-cloudwatch-scaledobject
```

---

## 常見錯誤與提示 (Common Mistakes & Tips)

| 常見錯誤 | 正確做法 |
|----------|----------|
| IAM 權限不足 | 確保有 `cloudwatch:GetMetricData` 權限 |
| metricCollectionTime < metricStatPeriod | metricCollectionTime 必須大於 metricStatPeriod |
| 維度名稱/值不匹配 | 確認維度名稱和值與 CloudWatch 中的完全一致 |
| 最新資料點不準確 | 使用 metricEndTimeOffset 跳過最近的資料點 |

### 提示

- 建議將 `metricCollectionTime` 設為 `metricStatPeriod` 的 2-3 倍
- 使用 `expression` 可以執行更複雜的查詢，如跨多個維度聚合
- 對於需要精確最新資料的場景，設定 `metricEndTimeOffset` 跳過最近的資料點
- 生產環境建議使用 Pod Identity 而非 IAM 使用者憑證

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `awsRegion` | AWS 區域 | 無（必填）|
| `namespace` | CloudWatch 命名空間 | 無（或使用 expression）|
| `metricName` | 指標名稱 | 無（或使用 expression）|
| `dimensionName` | 維度名稱 | 無（或使用 expression）|
| `dimensionValue` | 維度值 | 無（或使用 expression）|
| `expression` | Metrics Insights 表達式 | 無（或使用維度查詢）|
| `targetMetricValue` | 目標指標值 | 0 |
| `metricStat` | 統計方法 | Average |
| `metricStatPeriod` | 統計週期（秒）| 300 |
| `metricCollectionTime` | 收集時間範圍（秒）| 300 |

| AWS CLI 指令 | 說明 |
|--------------|------|
| `aws cloudwatch list-metrics` | 列出可用指標 |
| `aws cloudwatch get-metric-data` | 查詢指標資料 |

| 指令 | 說明 |
|------|------|
| `kubectl get scaledobject` | 列出 ScaledObject |
| `kubectl describe scaledobject <name>` | 查看詳細狀態 |
