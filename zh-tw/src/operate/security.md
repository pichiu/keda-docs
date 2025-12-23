
## TL;DR

本文件說明 KEDA 的安全性設定，包括如何使用自己的 TLS 憑證和如何註冊自訂 CA 到 KEDA Operator 的信任存放區。

---

## 翻譯 (Translation)

### 使用自己的 TLS 憑證

KEDA 使用自簽憑證進行不同的用途。這些憑證由 Operator 產生和輪換。憑證存放在 Kubernetes secret（`kedaorg-certs`）中，該 secret 掛載到所有 KEDA 元件的（預設）路徑 `/certs`。產生的檔案命名為 `tls.crt` 和 `tls.key`（TLS 憑證），以及 `ca.crt` 和 `ca.key`（CA 憑證）。KEDA 還會修補 Kubernetes 資源以包含 `caBundle`，使 Kubernetes 信任該 CA。

KEDA Operator 負責為所有服務產生憑證，預設情況下憑證為以下 DNS 名稱產生：

```
<KEDA_OPERATOR_SERVICE>                          -> 例如 keda-operator
<KEDA_OPERATOR_SERVICE>.svc                      -> 例如 keda-operator.svc
<KEDA_OPERATOR_SERVICE>.svc.<CLUSTER_DOMAIN>     -> 例如 keda-operator.svc.cluster.local
```

若要變更預設叢集網域（`cluster.local`），可以在 KEDA Operator 上使用參數 `--k8s-cluster-domain="my-domain"`。Helm Charts 會自動從 `clusterDomain` 值設定此參數。

雖然這是一個好的起點，但某些使用者可能希望使用從自己的 CA 產生的憑證以提高安全性。這可以透過停用 Operator 中的憑證產生/輪換並更新其他元件中的預設值（如需要）來完成。

若要停用 KEDA Operator 中的憑證產生，可移除主控台參數 `--enable-cert-rotation=true` 或將其設為 `false`。停用此設定後，使用者提供的憑證可以放置在 secret `kedaorg-certs` 中（該 secret 自動掛載到所有元件），或者可以修補元件使用其他 secret（也可以透過 helm 值完成）。

此外，KEDA 包含一個新的 `--enable-webhook-patching` 參數，用於控制 Operator 是否修補 webhook 資源。預設設為 `true`，確保 Kubernetes 信任 Operator 的 CA。但是，如果您的部署中停用或不需要 webhooks，可以將此參數設為 `false` 以避免與缺少 webhook 資源相關的錯誤。

所有元件都會檢查 `/certs` 資料夾中的任何憑證。可以使用參數 `--cert-dir` 指定另一個資料夾作為憑證來源。由於這些憑證也用於 KEDA 元件之間的內部通訊，CA 也需要在 KEDA 元件中註冊為受信任的 CA。

### 在 KEDA Operator 信任存放區中註冊自己的 CA

有些使用情況需要使用自簽 CA（例如 AWS 的 CA 未註冊為受信任的情況等）。某些擴縮器允許透過設定 `unsafeSsl` 參數跳過憑證驗證，但這不理想，因為它允許任何憑證，這不安全。

為了解決這個問題，KEDA 支援在可能的 SDK 中註冊自訂 CA 作為受信任的 CA。若要註冊自訂 CA，將憑證放置在目錄中，然後使用 `--ca-dir=` 參數將目錄傳遞給 KEDA Operator。預設情況下，KEDA Operator 會查看 `/custom/ca` 目錄。可以透過多次提供 `--ca-dir=` 參數來指定多個目錄。KEDA 會嘗試將這些目錄中的所有憑證註冊為受信任的 CA。

---

## 說明 (Explanation)

### 憑證設定選項

| 選項 | 說明 |
|------|------|
| 自動產生（預設）| KEDA Operator 自動產生和輪換憑證 |
| 使用者提供 | 停用自動產生，使用自己的憑證 |

### 相關參數

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `--enable-cert-rotation` | 啟用憑證輪換 | true |
| `--enable-webhook-patching` | 修補 webhook 資源 | true |
| `--cert-dir` | 憑證目錄 | /certs |
| `--ca-dir` | 自訂 CA 目錄 | /custom/ca |
| `--k8s-cluster-domain` | 叢集網域 | cluster.local |

---

## 快速參考 (Quick Reference)

| 指令 | 說明 |
|------|------|
| `kubectl get secret kedaorg-certs -n keda` | 查看 KEDA 憑證 secret |
| `kubectl describe secret kedaorg-certs -n keda` | 查看 secret 詳細資訊 |

| 檔案 | 說明 |
|------|------|
| `tls.crt` | TLS 憑證 |
| `tls.key` | TLS 金鑰 |
| `ca.crt` | CA 憑證 |
| `ca.key` | CA 金鑰 |
