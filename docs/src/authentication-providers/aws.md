
## TL;DR

AWS IAM Roles for Service Accounts (IRSA) Pod Identity Webhook 允許您使用服務帳戶上的註解提供角色名稱。KEDA 可以使用 KEDA 的角色或工作負載的角色來存取 AWS 資源。支援 AssumeRoleWithWebIdentity 和 AssumeRole 兩種方式。

---

## 翻譯 (Translation)

[**AWS IAM Roles for Service Accounts (IRSA) Pod Identity Webhook**](https://github.com/aws/amazon-eks-pod-identity-webhook) 允許您使用與 Pod 關聯的服務帳戶上的註解提供角色名稱。

您可以透過 `podIdentity.provider` 告訴 KEDA 使用 AWS Pod Identity Webhook。

```yaml
podIdentity:
  provider: aws
  roleArn: <role-arn>           # 可選
  identityOwner: keda|workload  # 可選。如未設定，預設為 'keda'。與 'roleArn' 互斥
```

**參數列表：**

- `roleArn` - KEDA 使用的角色 ARN。如未設定，將使用 KEDA Operator 使用的 IAM 角色。與 `identityOwner: workload` 互斥
- `identityOwner` - 要使用的身分識別擁有者。（值：`keda`、`workload`，預設：`keda`，可選）

> ⚠️ **注意：** `podIdentity.roleArn` 和 `podIdentity.identityOwner` 互斥，不支援同時設定兩者。

### 如何使用

AWS IRSA 會授予具有適當註解的服務帳戶的 Pod 存取權限。您可以在 KEDA Operator 服務帳戶上設定這些註解。

使用 Helm 部署時可以透過以下參數設定：

1. `--set podIdentity.aws.irsa.enabled=true`
2. `--set podIdentity.aws.irsa.roleArn={aws-arn-role}`

您可以透過在 `podIdentity` 欄位下指定 `roleArn` 參數來覆蓋預設 KEDA Operator IAM 角色。

如果您想使用與工作負載目前相同的 IAM 憑證，可以將 `podIdentity.identityOwner` 設為 `workload`，KEDA 將檢查工作負載服務帳戶是否有 IRSA 註解，並承擔該角色。

**區域 STS 端點**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: <SERVICE_ACCOUNT_NAME>
  namespace: <NAMESPACE>
  annotations:
    eks.amazonaws.com/role-arn: <YOUR_IRSA_ROLE_ARN>
    # 區域 STS 端點所需（例如 eu-west-1）
    eks.amazonaws.com/sts-regional-endpoints: "true"
```

### 何時需要區域 STS 端點？

預設情況下，AWS STS 呼叫會發送到全域端點（`sts.amazonaws.com`）。某些區域（例如 **eu-west-1**、**ap-northeast-1**、**ap-southeast-2**）需要或強烈建議使用其**區域 STS 端點**（例如 `sts.eu-west-1.amazonaws.com`）。

如果您看到錯誤：

```
AccessDenied: Not authorized to perform sts:AssumeRoleWithWebIdentity
```

而您的 IAM 信任策略看起來正確，您可能需要啟用區域端點：

```yaml
annotations:
  eks.amazonaws.com/sts-regional-endpoints: "true"
```

---

## 說明 (Explanation)

### AssumeRole vs AssumeRoleWithWebIdentity

此驗證自動使用兩者，如果 [AssumeRoleWithWebIdentity](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRoleWithWebIdentity.html) 失敗則回退到 [AssumeRole](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html)。

### 設定 KEDA 角色和策略

#### 使用 KEDA 角色存取基礎設施

將所需策略附加到 KEDA 的角色。例如，用於 SQS 的策略：

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "sqs:GetQueueAttributes",
            "Resource": "arn:aws:sqs:*:YOUR_ACCOUNT:YOUR_QUEUE"
        }
    ]
}
```

#### 使用 KEDA 角色透過 AssumeRoleWithWebIdentity 承擔工作負載角色

角色策略範例：

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "YOUR_OIDC_ARN"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "YOUR_OIDC:sub": "system:serviceaccount:keda:keda-operator",
                    "YOUR_OIDC:aud": "sts.amazonaws.com"
                }
            }
        }
    ]
}
```

#### 使用 KEDA 角色透過 AssumeRole 承擔工作負載角色

KEDA 角色策略範例：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": [
        "arn:aws:iam::ACCOUNT_1:role/ROLE_NAME"
      ]
    }
  ]
}
```

工作負載角色信任關係：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT:role/KEDA_ROLE_NAME"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `provider` | Pod Identity 提供者 | none |
| `roleArn` | IAM 角色 ARN | KEDA Operator 角色 |
| `identityOwner` | 身分識別擁有者 | keda |

| Helm 參數 | 說明 |
|-----------|------|
| `podIdentity.aws.irsa.enabled` | 啟用 AWS IRSA |
| `podIdentity.aws.irsa.roleArn` | IAM 角色 ARN |
