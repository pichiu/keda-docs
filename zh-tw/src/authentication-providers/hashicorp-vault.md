
## TL;DR

Hashicorp Vault 驗證提供者允許您從 Hashicorp Vault 拉取一個或多個 Secret 到觸發器中。支援 token 和 kubernetes 兩種驗證方法。可用於提供擴縮器所需的各種憑證，包括動態產生的 PKI 憑證。

---

## 翻譯 (Translation)

您可以透過定義驗證元資料（如 Vault `address` 和 `authentication` 方法（token | kubernetes））從 Hashicorp Vault 拉取一個或多個 Secret 到觸發器中。如果選擇 kubernetes 驗證方法，還應提供 `role` 和 `mount`。

`credential` 根據驗證方法定義 Hashicorp Vault 憑證。有多種驗證方法：

- **kubernetes**：提供服務帳戶令牌的路徑（`/var/run/secrets/kubernetes.io/serviceaccount/token`），或提供可以在 ScaledObject/ScaledJob 資源的命名空間中驗證 kubernetes 的 serviceAccountName（通常是 `default`）。如果使用 serviceAccountName，請確保授予 KEDA Operator `create serviceaccounts/token` 權限。
- **token**：提供令牌。

`secrets` 列表定義 Vault 中 Secret 的路徑和金鑰到參數的對應。

`namespace` 可用於針對給定的 Vault Enterprise 命名空間。

> 自版本 `1.5.0` 起支援 Vault secrets 後端**版本 2**。
> 版本 `2.10` 新增對 Vault secrets 後端**版本 1** 的支援。

```yaml
hashiCorpVault:                                               # 可選
  address: {hashicorp-vault-address}                          # 必填
  namespace: {hashicorp-vault-namespace}                      # 可選。預設為根命名空間。對 Vault Enterprise 有用
  authentication: token | kubernetes                          # 必填
  role: {hashicorp-vault-role}                                # 可選
  mount: {hashicorp-vault-mount}                              # 可選
  credential:                                                 # 可選
    token: {hashicorp-vault-token}                            # 可選。透過提供的令牌驗證 Vault
    serviceAccount: {path-to-service-account-file}            # 可選。透過 KEDA Operator Pod 中的 JWT 令牌驗證 Vault
    serviceAccountName: {service-account-name-for-auth}       # 可選。需要 serviceaccounts/token create 權限
  secrets:                                                    # 必填
  - parameter: {scaledObject-parameter-name}                  # 必填
    key: {hashicorp-vault-secret-key-name}                    # 必填
    path: {hashicorp-vault-secret-path}                       # 必填
    type: {hashicorp-vault-secret-type}                       # 可選。預設為 ""。允許值：secret、secretV2、pki
    pkidata: {hashicorp-vault-secret-pkidata}                 # 可選。如果類型是 pki 請求，要發送的資料
      commonName: {hashicorp-vault-secret-pkidata-commonName} # 可選
      altNames: {hashicorp-vault-secret-pkidata-altNames}     # 可選
      ipSans: {hashicorp-vault-secret-pkidata-ipSans}         # 可選
      uriSans: {hashicorp-vault-secret-pkidata-uriSans}       # 可選
      otherSans: {hashicorp-vault-secret-pkidata-otherSans}   # 可選
      ttl: {hashicorp-vault-secret-pkidata-ttl}               # 可選
      format: {hashicorp-vault-secret-pkidata-format}         # 可選
```

---

## 實作範例 (Practical Example)

Vault Secret 可用於為擴縮器提供驗證。例如，使用 [Prometheus 擴縮器](https://keda.sh/docs/2.3/scalers/prometheus/)時，可以使用 mTLS 由 `ScaledObject` 驗證 Prometheus 伺服器。以下範例會動態向 Vault 請求憑證。

```yaml
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: vault-trigger-auth
  namespace: default
spec:
  hashiCorpVault:
    address: {hashicorp-vault-address}
    authentication: token
    credential:
      token: {hashicorp-vault-token}
    secrets:
      - key: "ca_chain"
        parameter: "ca"
        path: {hashicorp-vault-secret-path}
        type: pki
        pki_data:
          common_name: {common-name}
      - key: "private_key"
        parameter: "key"
        path: {hashicorp-vault-secret-path}
        type: pki
        pki_data:
          common_name: {common-name}
      - key: "certificate"
        parameter: "cert"
        path: {hashicorp-vault-secret-path}
        type: pki
        pki_data:
          common_name: {common-name}
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: prometheus-scaledobject
  namespace: default
spec:
  scaleTargetRef:
    name: my-deployment
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://<prometheus-host>:9090
        query: sum(rate(http_requests_total{deployment="my-deployment"}[2m]))
        authModes: "tls"
      authenticationRef:
        name: vault-trigger-auth
```

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `address` | Vault 伺服器位址 | 無（必填）|
| `namespace` | Vault 命名空間（Enterprise）| 根命名空間 |
| `authentication` | 驗證方法 | 無（必填：token 或 kubernetes）|
| `role` | Vault 角色 | 無（kubernetes 驗證時需要）|
| `mount` | Vault 掛載點 | 無 |

| Secret 類型 | 說明 |
|-------------|------|
| `secret` | Vault secrets 後端 v1 |
| `secretV2` | Vault secrets 後端 v2 |
| `pki` | PKI 憑證 |

| 驗證方法 | 說明 |
|----------|------|
| `token` | 使用 Vault 令牌 |
| `kubernetes` | 使用 Kubernetes 服務帳戶 JWT |
