+++
title = "快速入門"
description = "KEDA 新手使用者指南"
weight = 1
+++

## TL;DR

KEDA（Kubernetes Event-driven Autoscaler）是一個事件驅動的自動擴縮容元件，能讓 Kubernetes 根據外部事件來源（如訊息佇列、資料庫、HTTP 請求等）動態調整 Pod 數量。本文件提供不同角色的使用者入門指引。

---

## 翻譯 (Translation)

歡迎來到 **KEDA**（Kubernetes 事件驅動自動擴縮容器）的技術文件。

請使用左側導覽列來深入了解 KEDA 的架構（Architecture），以及如何部署和使用 KEDA。

### 我該從哪裡開始？

請根據您的角色選擇對應的文件：

| 角色                        | 文件說明                                                                                                                                                     |
|-----------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 使用者 (User)               | 本文件適合想要部署 KEDA 來擴縮 Kubernetes 的使用者。                                                                                                         |
| 核心貢獻者 (Core Contributor) | 如要貢獻 KEDA 核心專案，請參閱 [KEDA GitHub 儲存庫](https://github.com/kedacore/keda)。                                                                      |
| 文件貢獻者 (Documentation Contributor) | 如要新增或貢獻此文件，或在本機建置並預覽文件，請參閱 [keda-docs GitHub 儲存庫](https://github.com/kedacore/keda-docs)。                                      |
| 其他貢獻者 (Other Contributor) | 請參閱 [GitHub 上的 KEDA 專案](https://github.com/kedacore/)，以取得其他 KEDA 儲存庫，包括專案治理、測試及外部擴縮器（External Scalers）。                   |

---

## 說明 (Explanation)

### 什麼是 KEDA？

KEDA 是 Kubernetes 原生的事件驅動自動擴縮器（Event-driven Autoscaler）。傳統的 Kubernetes HPA（Horizontal Pod Autoscaler，水平 Pod 自動擴縮器）只能根據 CPU 或記憶體使用率來擴縮，而 KEDA 則可以根據各種外部事件來源進行擴縮，例如：

- 訊息佇列（Message Queue）中的訊息數量
- 資料庫中的記錄數
- HTTP 請求數
- 排程時間（Cron）

### KEDA 的核心元件

1. **Scaler（擴縮器）**：連接外部事件來源並取得指標（Metrics）
2. **ScaledObject（擴縮物件）**：定義如何擴縮 Deployment 或 StatefulSet
3. **ScaledJob（擴縮任務）**：定義如何擴縮 Kubernetes Job

---

## 實作範例 (Practical Example)

```bash
# 安裝 KEDA（使用 Helm）
helm repo add kedacore https://kedacore.github.io/charts  # 新增 KEDA Helm 儲存庫
helm repo update                                           # 更新儲存庫索引
helm install keda kedacore/keda --namespace keda --create-namespace  # 在 keda 命名空間安裝 KEDA

# 驗證安裝
kubectl get pods -n keda  # 列出 keda 命名空間中的所有 Pod
```

---

## 常見錯誤與提示 (Common Mistakes & Tips)

| 常見錯誤 | 正確做法 |
|----------|----------|
| 未建立 keda 命名空間就安裝 | 使用 `--create-namespace` 參數自動建立命名空間 |
| 忘記更新 Helm 儲存庫 | 安裝前先執行 `helm repo update` |
| 直接刪除 KEDA 但未處理 ScaledObject | 先刪除所有 ScaledObject 再移除 KEDA |
| Kubernetes 版本過舊 | KEDA 需要 Kubernetes 1.30 或更高版本 |

### 提示

- 建議在獨立的 `keda` 命名空間中安裝 KEDA，方便管理
- 可使用 `kubectl get scaledobject` 查看所有擴縮物件的狀態
- 如遇問題，可查看 KEDA Operator 的日誌：`kubectl logs -n keda -l app=keda-operator`

---

## 快速參考 (Quick Reference)

| 指令 | 說明 |
|------|------|
| `helm repo add kedacore https://kedacore.github.io/charts` | 新增 KEDA Helm 儲存庫 |
| `helm install keda kedacore/keda -n keda --create-namespace` | 安裝 KEDA |
| `kubectl get pods -n keda` | 查看 KEDA Pod 狀態 |
| `kubectl get scaledobject` | 列出所有 ScaledObject |
| `kubectl get scaledjob` | 列出所有 ScaledJob |
| `helm uninstall keda -n keda` | 解除安裝 KEDA |
