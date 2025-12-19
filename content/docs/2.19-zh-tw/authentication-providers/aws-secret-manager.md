+++
title = "AWS Secret Manager"
+++

## TL;DR

AWS Secret Manager 允許您將 AWS Secrets Manager 中的 Secret 整合到 KEDA 觸發器中。支援透過 Pod Identity 或 AWS 憑證進行驗證。

---

## 翻譯 (Translation)

您可以透過在 KEDA 擴縮規格中設定 `awsSecretManager` 金鑰，將 AWS Secret Manager 的 Secret 整合到觸發器中。

`podIdentity` 區段設定 AWS Pod Identity 的使用，提供者設為 AWS。

`credentials` 區段指定 AWS 憑證，包括 `accessKey` 和 `secretAccessKey`。

- **accessKey：** AWS 存取金鑰的設定。
- **secretAccessKey：** AWS 秘密存取金鑰的設定。

`region` 參數是可選的，表示 Secret 所在的 AWS 區域，如果未指定則使用預設區域。

`awsSecretManager` 中的 `secrets` 清單定義 AWS Secret Manager 的 Secret 與應用程式中使用的驗證參數之間的對應，包括參數名稱、AWS Secret Manager 的 Secret 名稱，以及可選的版本參數（如果未指定則預設為最新版本）。

### 設定

```yaml
awsSecretManager:
  podIdentity:                                     # 可選。
    provider: aws                                  # 必填。
  credentials:                                     # 可選。
    accessKey:                                     # 必填。
      valueFrom:                                   # 必填。
        secretKeyRef:                              # 必填。
          name: {k8s-secret-with-aws-credentials}  # 必填。
          key: {key-in-k8s-secret}                 # 必填。
    accessSecretKey:                               # 必填。
      valueFrom:                                   # 必填。
        secretKeyRef:                              # 必填。
          name: {k8s-secret-with-aws-credentials}  # 必填。
          key: {key-in-k8s-secret}                 # 必填。
  region: {aws-region}                             # 可選。
  secrets:                                         # 必填。
  - parameter: {param-name-used-for-auth}          # 必填。
    name: {aws-secret-name}                        # 必填。
    version: {aws-secret-version}                  # 可選。
    secretKey: {aws-secret-key}                    # 可選。
```

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `podIdentity.provider` | Pod Identity 提供者 | 無（必須為 `aws`）|
| `credentials.accessKey` | AWS 存取金鑰 | 無 |
| `credentials.accessSecretKey` | AWS 秘密存取金鑰 | 無 |
| `region` | AWS 區域 | 預設區域 |
| `secrets[].parameter` | 驗證參數名稱 | 無（必填）|
| `secrets[].name` | AWS Secret 名稱 | 無（必填）|
| `secrets[].version` | Secret 版本 | 最新版本 |
| `secrets[].secretKey` | Secret 金鑰 | 無 |

| 驗證方式 | 說明 |
|----------|------|
| Pod Identity | 使用 AWS Pod Identity |
| 憑證 | 使用 Kubernetes Secret 中的 AWS 憑證 |
