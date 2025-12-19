+++
title = "AWS EKS Pod Identity Webhook"
+++

## TL;DR

AWS EKS Pod Identity Webhook 允許透過服務帳戶註解來提供 IAM 角色名稱，實現細粒度的 IAM 權限控制。此功能已被棄用，建議遷移至 `aws` 驗證方式。

---

## 翻譯 (Translation)

[**EKS Pod Identity Webhook**](https://github.com/aws/amazon-eks-pod-identity-webhook)，在[這裡](https://aws.amazon.com/blogs/opensource/introducing-fine-grained-iam-roles-service-accounts/)有更深入的說明，允許您透過與 Pod 關聯的服務帳戶上的註解來提供角色名稱。

> ⚠️ **警告：** [`aws-eks` 驗證已被棄用](https://github.com/kedacore/keda/discussions/5343)，並將在 KEDA v3 中移除支援。我們強烈建議遷移至 [`aws` 驗證](./aws.md)。

您可以透過 `podIdentity.provider` 告訴 KEDA 使用 EKS Pod Identity Webhook。

```yaml
podIdentity:
  provider: aws-eks # 可選。預設：none
```

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `provider` | Pod Identity 提供者 | none |

| 狀態 | 說明 |
|------|------|
| 已棄用 | 此功能已棄用，將在 v3 中移除 |
| 建議 | 遷移至 `aws` 驗證方式 |
