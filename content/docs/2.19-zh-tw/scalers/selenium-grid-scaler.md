+++
title = "Selenium Grid Scaler"
availability = "v2.4+"
maintainer = "Volvo Cars, SeleniumHQ"
category = "Testing"
description = "根據會話佇列中等待的請求數量擴縮 Selenium 瀏覽器節點"
go_file = "selenium_grid_scaler"
+++

## TL;DR

Selenium Grid 擴縮器根據會話佇列中的請求數量和 Grid 的最大會話數自動擴縮瀏覽器節點。每個待處理請求會建立一個瀏覽器節點，除以可並行執行的最大會話數。

---

## 翻譯 (Translation)

### 觸發器規格

此規格描述基於會話佇列中的請求數量和 Grid 最大會話數擴縮瀏覽器節點的 `selenium-grid` 觸發器。

擴縮器為會話佇列中的每個待處理請求建立一個瀏覽器節點，除以可並行執行的最大會話數。您需要為 Selenium Grid 中要支援的每個瀏覽器功能建立一個觸發器。

```yaml
triggers:
  - type: selenium-grid
    metadata:
      url: 'http://selenium-hub:4444/graphql'
      browserName: ''
      browserVersion: ''
      platformName: ''
      unsafeSsl: false
      activationThreshold: 0
      nodeMaxSessions: 1
      enableManagedDownloads: true
      capabilities: ''
```

**參數列表：**

- `url` - Selenium Grid 的 Graphql URL。如果端點需要驗證，您可以使用 `TriggerAuthentication` 提供憑證。
- `browserName` - 瀏覽器名稱，通常在瀏覽器功能中傳遞。（可選）
- `sessionBrowserName` - 活動會話時的瀏覽器名稱，僅在佇列和活動會話之間 `browserName` 變更時設定。（可選）
- `browserVersion` - 瀏覽器版本，通常在瀏覽器功能中傳遞。（可選）
- `unsafeSsl` - 透過 HTTPS 連線時跳過憑證驗證。（值：`true`、`false`，預設：`false`，可選）
- `activationThreshold` - 啟動擴縮器的目標值。（預設：`0`，可選）
- `platformName` - 瀏覽器平台名稱。（可選）
- `nodeMaxSessions` - 可在節點上並行執行的最大會話數。更新此參數以與節點配置 `--max-sessions`（`SE_NODE_MAX_SESSIONS`）對齊，以獲得正確的擴縮行為。（預設：`1`，可選）
- `enableManagedDownloads` - 設定為啟用節點自動管理給定會話在節點上下載的檔案。（預設：`true`，可選）
- `capabilities` - 新增更多自訂功能以匹配特定節點。應為 JSON 字串。（可選）

**觸發器驗證**
- `username` - GraphQL 端點基本驗證的使用者名稱。（可選）
- `password` - GraphQL 端點基本驗證的密碼。（可選）
- `authType` - 驗證類型。如果 Selenium Grid 在具有其他驗證類型的 Ingress 代理後面，可以設定為 `Bearer` 或 `OAuth2`。（可選）
- `accessToken` - 存取令牌。當設定 `authType` 時必填。（可選）

---

## 實作範例 (Practical Example)

Chrome 瀏覽器的 Selenium Grid 擴縮器：

```yaml
kind: Deployment
metadata:
  name: selenium-node-chrome
  labels:
    deploymentName: selenium-node-chrome
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: selenium-node-chrome
        image: selenium/node-chrome:latest
        ports:
        - containerPort: 5555
        env:
        - name: SE_NODE_BROWSER_VERSION
          value: ''
        - name: SE_NODE_PLATFORM_NAME
          value: 'Linux'
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: selenium-grid-scaledobject-chrome
  namespace: keda
  labels:
    deploymentName: selenium-node-chrome
spec:
  maxReplicaCount: 8
  scaleTargetRef:
    name: selenium-node-chrome
  triggers:
    - type: selenium-grid
      metadata:
        url: 'http://selenium-hub:4444/graphql'
        browserName: 'chrome'
        platformName: 'Linux'
        unsafeSsl: 'true'
```

Python 綁定中的請求：

```python
options = ChromeOptions()
options.set_capability('platformName', 'Linux')
driver = webdriver.Remote(options=options, command_executor=SELENIUM_GRID_URL)
```

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `url` | Selenium Grid GraphQL URL | 無（必填）|
| `browserName` | 瀏覽器名稱 | 無 |
| `browserVersion` | 瀏覽器版本 | 無 |
| `platformName` | 平台名稱 | 無 |
| `nodeMaxSessions` | 節點最大會話數 | 1 |
| `activationThreshold` | 啟動閾值 | 0 |
| `unsafeSsl` | 跳過 SSL 驗證 | false |
| `enableManagedDownloads` | 啟用受管理下載 | true |

| 瀏覽器 | browserName | sessionBrowserName |
|--------|-------------|-------------------|
| Chrome | chrome | - |
| Firefox | firefox | - |
| Edge | MicrosoftEdge | msedge |
