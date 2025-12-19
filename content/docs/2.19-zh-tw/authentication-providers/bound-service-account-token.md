+++
title = "繫結服務帳戶令牌"
+++

## TL;DR

繫結服務帳戶令牌（Bound Service Account Token）允許 KEDA 代表指定的 Kubernetes 服務帳戶請求和使用令牌。需要適當的 RBAC 權限設定。

---

## 翻譯 (Translation)

您可以透過定義 Kubernetes 服務帳戶的 `serviceAccountName` 來將一個或多個服務帳戶令牌拉取到觸發器中。

```yaml
boundServiceAccountToken:                   # 可選。
  - parameter: connectionString             # 必填 - 由擴縮觸發器定義
    serviceAccountName: my-service-account  # 必填。
```

**假設：** 除非另有指定，否則 `namespace` 與 ScaledObject 中 `scaleTargetRef.name` 參照的資源位於相同的命名空間。

## KEDA 請求服務帳戶令牌的權限

預設情況下，KEDA Operator 沒有從任意服務帳戶請求服務帳戶令牌所需的權限。這是為了防止權限提升，避免惡意行為者使用 KEDA 代表叢集中的任何服務帳戶請求令牌。

若要允許 KEDA 從服務帳戶請求令牌，您必須使用 RBAC 授予 `keda-operator` 服務帳戶必要的權限。這可以透過在服務帳戶的命名空間中建立 `Role` 和 `RoleBinding` 來完成，以允許 `keda-operator` 服務帳戶對命名空間內的 `serviceaccounts/token` 子資源具有 `create` 權限。

以下是授予 KEDA Operator 從命名空間內的 `my-service-account` 服務帳戶請求和使用令牌所需權限的最小 `Role` 和 `RoleBinding` 範例。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: keda-operator-token-creator
  namespace: my-namespace # 替換為服務帳戶的命名空間
rules:
- apiGroups:
  - ""
  resources:
  - serviceaccounts/token
  verbs:
  - create
  resourceNames:
  - my-service-account # 替換為服務帳戶的名稱
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: keda-operator-token-creator-binding
  namespace: my-namespace # 替換為服務帳戶的命名空間
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: keda-operator-token-creator
subjects:
- kind: ServiceAccount
  name: keda-operator
  namespace: keda # 假設 keda-operator 服務帳戶在 keda 命名空間
```

在叢集中套用類似的限制性權限後，您可以建立參照服務帳戶名稱的 `TriggerAuthentication` 資源，允許 KEDA 代表您的擴縮器請求和使用令牌。請注意，如果擴縮器將驗證委託給 Kubernetes API，服務帳戶還必須具有執行擴縮器所需操作（如查詢指標或管理資源）的必要 Kubernetes API 權限。

### 在 keda-charts 中使用

如果您使用 Helm Charts 部署 KEDA，可以透過在 `values.yaml` 檔案中設定 `boundServiceAccountToken` 欄位來提供 KEDA 請求令牌的服務帳戶的命名空間名稱。例如：

```yaml
# values.yaml
permissions:
  operator:
    restrict:
      serviceAccountTokenCreationRoles:
      - name: myServiceAccount
        namespace: myServiceAccountNamespace
```

這將在 `myServiceAccountNamespace` 命名空間中建立必要的 `Role` 和 `RoleBinding`：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: keda-operator-token-creator-myServiceAccount
  namespace: myServiceAccountNamespace
rules:
- apiGroups:
  - ""
  resources:
  - serviceaccounts/token
  verbs:
  - create
  resourceNames:
  - myServiceAccount
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: keda-operator-token-creator-binding-myServiceAccount
  namespace: myServiceAccountNamespace
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: keda-operator-token-creator-myServiceAccount
subjects:
- kind: ServiceAccount
  name: keda-operator
  namespace: keda
```

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `parameter` | 觸發器使用的參數名稱 | 無（必填）|
| `serviceAccountName` | Kubernetes 服務帳戶名稱 | 無（必填）|

| RBAC 設定 | 說明 |
|----------|------|
| Role | 授予 `serviceaccounts/token` 的 `create` 權限 |
| RoleBinding | 將 Role 繫結到 `keda-operator` 服務帳戶 |
| 命名空間 | Role 和 RoleBinding 必須在服務帳戶的命名空間中 |

| Helm 設定 | 說明 |
|----------|------|
| `permissions.operator.restrict.serviceAccountTokenCreationRoles` | 自動建立 RBAC 資源的服務帳戶清單 |
