+++
title = "AWS Kinesis Stream"
availability = "v1.1+"
maintainer = "Community"
category = "Messaging"
description = "根據 AWS Kinesis Stream 擴縮應用程式。"
go_file = "aws_kinesis_stream_scaler"
+++

## TL;DR

AWS Kinesis Stream 擴縮器根據 AWS Kinesis Stream 的分片（shard）數量自動擴縮應用程式。適用於處理 Kinesis 資料串流的消費者應用程式。

---

## 翻譯 (Translation)

### 觸發器規格

此規格描述基於 AWS Kinesis Stream 分片數量進行擴縮的 `aws-kinesis-stream` 觸發器。

```yaml
triggers:
- type: aws-kinesis-stream
  metadata:
    # 必填
    streamName: myKinesisStream
    # 必填
    awsRegion: "eu-west-1"
    # 可選：awsEndpoint
    awsEndpoint: ""
    # 可選：預設：2
    shardCount: "2"
```

**參數列表：**

- `streamName` - AWS Kinesis Stream 的名稱。
- `shardCount` - Kinesis 資料串流消費者可處理的目標值。（預設：`2`，可選）
- `activationShardCount` - 啟動擴縮器的目標值。在[這裡](./../concepts/scaling-deployments.md#activating-and-scaling-thresholds)了解更多。（預設：`0`，可選）
- `awsRegion` - Kinesis Stream 的 AWS 區域。
- `awsEndpoint` - 覆寫預設 AWS 端點的端點 URL。（預設：`""`，可選）

### 驗證參數

您可以使用 `TriggerAuthentication` CRD 透過提供角色 ARN 或一組 IAM 憑證來設定驗證，或使用其他 KEDA 支援的驗證方法。

**基於角色的驗證：**
- `awsRoleArn` - Amazon 資源名稱（ARN）唯一識別 AWS 資源。

**基於憑證的驗證：**
- `awsAccessKeyID` - 使用者的 ID。
- `awsSecretAccessKey` - 使用者進行驗證的存取金鑰。
- `awsSessionToken` - 會話令牌，僅在使用臨時憑證時需要。

使用者需要 `DescribeStreamSummary` IAM 權限策略才能從 AWS Kinesis Streams 讀取資料。

---

## 實作範例 (Practical Example)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: test-secrets
  namespace: keda-test
data:
  AWS_ACCESS_KEY_ID: <encoded-user-id>
  AWS_SECRET_ACCESS_KEY: <encoded-key>
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: keda-trigger-auth-aws-credentials
  namespace: keda-test
spec:
  secretTargetRef:
  - parameter: awsAccessKeyID
    name: test-secrets
    key: AWS_ACCESS_KEY_ID
  - parameter: awsSecretAccessKey
    name: test-secrets
    key: AWS_SECRET_ACCESS_KEY
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: aws-kinesis-stream-scaledobject
  namespace: keda-test
spec:
  scaleTargetRef:
    name: nginx-deployment
  triggers:
    - type: aws-kinesis-stream
      authenticationRef:
        name: keda-trigger-auth-aws-credentials
      metadata:
        streamName: myKinesisStream
        awsRegion: "eu-west-1"
        shardCount: "2"
```

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `streamName` | Kinesis Stream 名稱 | 無（必填）|
| `awsRegion` | AWS 區域 | 無（必填）|
| `shardCount` | 目標分片數量 | 2 |
| `activationShardCount` | 啟動閾值 | 0 |
| `awsEndpoint` | 自訂 AWS 端點 | "" |

| 所需 IAM 權限 |
|---------------|
| `kinesis:DescribeStreamSummary` |
