+++
title = "驗證提供者"
weight = 5
+++

## TL;DR

KEDA 提供多種驗證提供者，用於安全地連接外部事件來源，包括 Kubernetes Secret、環境變數、HashiCorp Vault、Azure Key Vault、AWS Secret Manager、GCP Secret Manager 以及各種雲端平台的 Pod Identity。

---

## 翻譯 (Translation)

KEDA 可用的驗證提供者：

{{< authentication-providers >}}

---

## 說明 (Explanation)

### 驗證提供者類型

| 類型 | 說明 | 適用場景 |
|------|------|----------|
| Secret | 從 Kubernetes Secret 讀取憑證 | 一般用途 |
| ConfigMap | 從 Kubernetes ConfigMap 讀取設定 | 非敏感設定 |
| 環境變數 | 從容器環境變數讀取 | 簡單場景 |
| HashiCorp Vault | 從 Vault 動態取得密鑰 | 企業級密鑰管理 |
| Azure Key Vault | 從 Azure 金鑰保存庫讀取 | Azure 環境 |
| AWS Secret Manager | 從 AWS Secrets Manager 讀取 | AWS 環境 |
| GCP Secret Manager | 從 GCP Secret Manager 讀取 | GCP 環境 |
| Pod Identity | 使用雲端提供者的身分驗證 | 無密鑰驗證 |

---

## 快速參考 (Quick Reference)

| 驗證提供者 | 支援的雲端 | 需要額外設定 |
|------------|------------|--------------|
| Secret | 所有 | 否 |
| Environment Variable | 所有 | 否 |
| Bound Service Account Token | 所有 | 否 |
| Azure Key Vault | Azure | Azure AD 設定 |
| Azure AD Workload Identity | Azure | Workload Identity 設定 |
| AWS Secret Manager | AWS | IAM 角色設定 |
| AWS EKS Pod Identity | AWS | EKS 設定 |
| GCP Secret Manager | GCP | IAM 設定 |
| GCP Workload Identity | GCP | Workload Identity 設定 |
| HashiCorp Vault | 所有 | Vault 伺服器 |
