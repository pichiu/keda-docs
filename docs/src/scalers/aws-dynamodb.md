
## TL;DR

AWS DynamoDB 擴縮器使用指定的 DynamoDB 查詢來決定何時以及如何擴縮給定的工作負載。適用於需要根據 DynamoDB 表中的資料量進行擴縮的場景。

---

## 翻譯 (Translation)

### 觸發器規格

此規格描述 AWS DynamoDB 擴縮器。此擴縮器使用指定的 DynamoDB 查詢來決定何時以及如何擴縮給定的工作負載。

```yaml
triggers:
- type: aws-dynamodb
  metadata:
    # 必填：awsRegion
    awsRegion: "eu-west-1"
    # 可選：awsEndpoint
    awsEndpoint: ""
    # 必填：tableName
    tableName: myTableName
    # 可選：indexName
    indexName: ""
    # 必填：targetValue
    targetValue: "1"
    # 必填：expressionAttributeNames
    expressionAttributeNames: '{ "#k" : "partition_key_name"}'
    # 必填：keyConditionExpression
    keyConditionExpression: "#k = :key"
    # 必填：expressionAttributeValues
    expressionAttributeValues: '{ ":key" : {"S":"partition_key_target_value"}}'
    # 可選：filterExpression
    filterExpression: 'filterField = :filterValue'
```

**參數列表：**

- `awsRegion` - DynamoDB 表的 AWS 區域。
- `awsEndpoint` - 覆寫預設 AWS 端點的端點 URL。（預設：`""`，可選）
- `tableName` - 擴縮器執行查詢的目標表。
- `indexName` - DynamoDB 查詢使用的索引。（可選）
- `targetValue` - 查詢擷取的項目數量的目標值。
- `activationTargetValue` - 啟動擴縮器的目標值。（預設：`0`，可選）
- `expressionAttributeNames` - 表達式中屬性名稱的一個或多個替代令牌。定義為 JSON。
- `keyConditionExpression` - 指定 Query 動作要擷取的項目的金鑰值的條件。
- `expressionAttributeValues` - 可在表達式中替代的一個或多個值。定義為 JSON。
- `filterExpression` - 指定 Query 動作要使用的 filterExpression 的條件。

### 驗證參數

**基於 Pod Identity 的驗證：**
- `podIdentity.provider` - 需要在 `TriggerAuthentication` 上設定，且 Pod/服務帳戶必須為您的 Pod Identity 提供者正確設定。

**基於角色的驗證：**
- `awsRoleArn` - Amazon 資源名稱（ARN）唯一識別 AWS 資源。

**基於憑證的驗證：**
- `awsAccessKeyID` - 使用者的 ID。
- `awsSecretAccessKey` - 使用者進行驗證的存取金鑰。
- `awsSessionToken` - 會話令牌，僅在使用臨時憑證時需要。

---

## 實作範例 (Practical Example)

使用 IAM 使用者擴縮部署：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: test-secrets
data:
  AWS_ACCESS_KEY_ID: <encoded-user-id>
  AWS_SECRET_ACCESS_KEY: <encoded-key>
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: keda-trigger-auth-aws-credentials
  namespace: default
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
  name: aws-dynamodb-table-scaledobject
  namespace: keda-test
spec:
  scaleTargetRef:
    name: nginx-deployment
  triggers:
  - type: aws-dynamodb
    authenticationRef:
      name: keda-trigger-auth-aws-credentials
    metadata:
      awsRegion: eu-west-2
      tableName: keda-events
      expressionAttributeNames: '{ "#k" : "event_type"}'
      keyConditionExpression: "#k = :key"
      expressionAttributeValues: '{ ":key" : {"S":"scaling_event"}}'
      targetValue: "5"
```

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `awsRegion` | AWS 區域 | 無（必填）|
| `tableName` | DynamoDB 表名稱 | 無（必填）|
| `targetValue` | 目標值 | 無（必填）|
| `expressionAttributeNames` | 屬性名稱表達式 | 無（必填）|
| `keyConditionExpression` | 金鑰條件表達式 | 無（必填）|
| `expressionAttributeValues` | 屬性值表達式 | 無（必填）|
| `indexName` | 索引名稱 | 無 |
| `filterExpression` | 過濾表達式 | 無 |

| 驗證方式 | 說明 |
|----------|------|
| Pod Identity | 使用 AWS Pod Identity |
| IAM 角色 | 使用 IAM 角色 ARN |
| IAM 憑證 | 使用存取金鑰 ID 和秘密存取金鑰 |
