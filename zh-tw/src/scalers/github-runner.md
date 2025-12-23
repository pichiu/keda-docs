
## TL;DR

GitHub Runner 擴縮器根據 GitHub Actions 中排隊的工作流程作業數量自動擴縮自託管執行器。支援 Personal Access Token 和 GitHub App 驗證方式。

---

## 翻譯 (Translation)

### 觸發器規格

此規格描述基於 GitHub Actions 中排隊作業進行擴縮的 `github-runner` 觸發器。

```yaml
triggers:
  - type: github-runner
    metadata:
      githubApiURL: "https://api.github.com"
      owner: "{owner}"
      runnerScope: "{runnerScope}"
      repos: "{repos}"
      labels: "{labels}"
      noDefaultLabels: "{noDefaultLabels}"
      targetWorkflowQueueLength: "1"
      applicationID: "{applicationID}"
      installationID: "{installationID}"
    authenticationRef:
      name: personalAccessToken or appKey triggerAuthentication Reference
```

**參數列表：**

- `githubApiURL` - GitHub API 的 URL，預設為 https://api.github.com。如果您有自己的 GitHub Appliance，才需要修改此項。（可選）
- `owner` - GitHub 儲存庫的擁有者，或擁有儲存庫的組織。（必填）
- `runnerScope` - 執行器的範圍，可以是 "org"、"ent" 或 "repo"。（必填）
- `repos` - 要擴縮的儲存庫清單，以逗號分隔。（可選）
- `labels` - 要擴縮的執行器標籤清單，以逗號分隔。（可選）
- `noDefaultLabels` - 不在預設執行器標籤（"self-hosted"、"linux"、"x64"）上擴縮。（值：`true`、`false`，預設："false"，可選）
- `enableEtags` - 啟用 etag 標頭以向 GitHub API 發出條件請求。（值：`true`、`false`，預設："false"，可選）
- `targetWorkflowQueueLength` - 要擴縮的排隊作業目標數量。（可選，預設：1）
- `applicationID` - GitHub App 的應用程式 ID。（可選，如果設定了 installationID 則必填）
- `installationID` - GitHub App 安裝到組織或儲存庫後的安裝 ID。（可選，如果設定了 applicationID 則必填）

### 驗證參數

您可以使用 Personal Access Token 或 GitHub App 私鑰透過 `TriggerAuthentication` 設定與 GitHub 進行驗證。

**令牌或金鑰驗證：**

- `personalAccessToken` - 來自您使用者的 GitHub Personal Access Token（PAT）。（可選，如果不使用 GitHub App 則必填）
- `appKey` - GitHub App 的私鑰。這是您建立 GitHub App 時下載的 `.pem` 檔案的內容。（可選，如果設定了 applicationID 則必填）

### 設定 GitHub App

您可以使用 GitHub App 與 GitHub 進行驗證。如果您想要更安全的驗證方法和更高的速率限制，這很有用。

1. 在您的組織或儲存庫中建立 GitHub App。
2. 記下應用程式 ID。您將需要它來設定擴縮器。
3. 在您的 GitHub App 上停用 Webhook。
4. 設定您的 GitHub App 的權限：
    - **儲存庫權限**：Actions - 唯讀、Administration - 讀寫、Metadata - 唯讀
    - **組織權限**：Actions - 唯讀、Metadata - 唯讀、Self-hosted Runners - 讀寫
5. 下載 GitHub App 的私鑰。
6. 在您的組織或儲存庫上安裝 GitHub App。
7. 記下安裝 ID。

---

## 實作範例 (Practical Example)

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: github-auth
data:
  personalAccessToken: <encoded personalAccessToken>
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: github-trigger-auth
  namespace: default
spec:
  secretTargetRef:
    - parameter: personalAccessToken
      name: github-auth
      key: personalAccessToken
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: github-runner-scaledobject
  namespace: default
spec:
  scaleTargetRef:
    name: gitrunner-deployment
  minReplicaCount: 1
  maxReplicaCount: 5
  triggers:
  - type: github-runner
    metadata:
      githubApiURL: "https://api.github.com"
      owner: "kedacore"
      runnerScope: "repo"
      repos: "keda,keda-docs"
      labels: "golang,helm"
      targetWorkflowQueueLength: "1"
    authenticationRef:
      name: github-trigger-auth
```

---

## 快速參考 (Quick Reference)

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `githubApiURL` | GitHub API URL | https://api.github.com |
| `owner` | 擁有者/組織 | 無（必填）|
| `runnerScope` | 執行器範圍（org/ent/repo）| 無（必填）|
| `repos` | 儲存庫清單 | 無 |
| `labels` | 執行器標籤 | 無 |
| `targetWorkflowQueueLength` | 目標佇列長度 | 1 |
| `noDefaultLabels` | 不使用預設標籤 | false |
| `enableEtags` | 啟用 Etag 條件請求 | false |

| 驗證方式 | 說明 |
|----------|------|
| Personal Access Token | 使用 GitHub PAT |
| GitHub App | 使用 GitHub App 私鑰（速率限制更高）|
