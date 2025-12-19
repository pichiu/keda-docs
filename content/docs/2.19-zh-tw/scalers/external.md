+++
title = "External"
availability = "v1.0+"
maintainer = "Microsoft"
category = "Extensibility"
description = "根據外部擴縮器擴縮應用程式。"
go_file = "external_scaler"
+++

## TL;DR

External 擴縮器允許您連接到自訂的外部 GRPC 擴縮器服務，實現完全自訂的擴縮邏輯。這是 KEDA 可擴展性的核心功能。

---

## 翻譯 (Translation)

### 觸發器規格

此規格描述外部擴縮器的 `external` 觸發器。

```yaml
triggers:
- type: external
  metadata:
    scalerAddress: external-scaler-service:8080
    caCert : /path/to/tls/ca.pem
    tlsClientCert: /path/to/tls/cert.pem
    tlsClientKey: /path/to/tls/key.pem
    enableTLS: false
    unsafeSsl: false
```

**參數列表：**

- `scalerAddress` - 外部擴縮器的位址。格式必須為 `host:port`。
- `enableTLS` - 允許使用 TLS 連線，並使用 Operator 已載入的 CA 進行驗證。（值：`true`、`false`，預設：`false`，可選）
- `unsafeSsl` - 透過 HTTPS 連線時跳過憑證驗證。（值：`true`、`false`，預設：`false`，可選）

整個 metadata 物件會在 `ScaledObjectRef.scalerMetadata` 中傳遞給外部擴縮器。

> 有關實作外部擴縮器的詳細資訊，請參閱[外部擴縮器概念](../concepts/external-scalers.md)。

### 驗證參數

- `caCert` - 用於驗證 GRPC 連線的憑證授權單位（CA）憑證。（可選）
- `tlsClientCert` - 用於驗證 GRPC 連線的用戶端憑證。（可選）
- `tlsClientKey` - 用於驗證 GRPC 連線的用戶端私鑰。（可選）

---

## 實作範例 (Practical Example)

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: external-scaledobject
spec:
  scaleTargetRef:
    name: keda-redis-node
  triggers:
  - type: external
    metadata:
      scalerAddress: redis-external-scaler-service:8080
      address: REDIS_HOST
      password: REDIS_PASSWORD
      listName: mylist
      listLength: "5"
```

使用憑證的外部擴縮器範例：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: certificate
data:
  ca.crt: "YOUR_CA_IN_SECRET"
  tls.crt: "YOUR_CERTIFICATE_IN_SECRET"
  tls.key: "YOUR_KEY_IN_SECRET"
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: keda-trigger-auth
spec:
  secretTargetRef:
  - parameter: caCert
    name: certificate
    key: ca.crt
  - parameter: tlsClientCert
    name: certificate
    key: tls.crt
  - parameter: tlsClientKey
    name: certificate
    key: tls.key
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: external-scaledobject
spec:
  scaleTargetRef:
    name: keda-redis-node
  triggers:
  - type: external
    metadata:
      scalerAddress: mydomain.com:443
      metricType: mymetric
      extraKey: "demo"
    authenticationRef:
      name: keda-trigger-auth
```

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `scalerAddress` | 外部擴縮器位址（host:port）| 無（必填）|
| `enableTLS` | 啟用 TLS 連線 | false |
| `unsafeSsl` | 跳過 SSL 憑證驗證 | false |
| `caCert` | CA 憑證路徑 | 無 |
| `tlsClientCert` | 用戶端憑證路徑 | 無 |
| `tlsClientKey` | 用戶端私鑰路徑 | 無 |
