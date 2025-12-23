
## TL;DR

本文件列出 KEDA 發出的所有 Kubernetes 事件類型，包括 ScaledObject/ScaledJob 的就緒狀態、驗證失敗、刪除事件，以及擴縮器和目標的啟動/停用事件。

---

## 翻譯 (Translation)

KEDA 發出以下 [Kubernetes 事件](https://kubernetes.io/docs/reference/kubernetes-api/cluster-resources/event-v1/)：

| 事件 | 類型 | 說明 |
|------|------|------|
| `ScaledObjectReady` | `Normal` | ScaledObject 首次就緒，或物件之前的就緒條件狀態為 `Unknown` 或 `False` 時 |
| `ScaledJobReady` | `Normal` | ScaledJob 首次就緒，或物件之前的就緒條件狀態為 `Unknown` 或 `False` 時 |
| `ScaledObjectCheckFailed` | `Warning` | ScaledObject 的檢查驗證失敗時 |
| `ScaledJobCheckFailed` | `Warning` | ScaledJob 的檢查驗證失敗時 |
| `ScaledObjectDeleted` | `Normal` | ScaledObject 被刪除並從 KEDA 監控中移除時 |
| `ScaledJobDeleted` | `Normal` | ScaledJob 被刪除並從 KEDA 監控中移除時 |
| `KEDAScalersStarted` | `Normal` | ScaledObject 或 ScaledJob 的擴縮器監控迴圈已啟動時 |
| `KEDAScalersStopped` | `Normal` | ScaledObject 或 ScaledJob 的擴縮器監控迴圈已停止時 |
| `KEDAScalerFailed` | `Warning` | 擴縮器無法建立或檢查其事件來源時 |
| `KEDAScalerInfo` | `Normal` | 擴縮器包含已棄用的欄位時 |
| `KEDAScalerInfo` | `Warning` | 擴縮器包含非預期的參數時（預設停用，使用 `KEDA_CHECK_UNEXPECTED_SCALERS_PARAMS` 啟用）|
| `KEDAScaleTargetActivated` | `Normal` | ScaledObject 的擴縮目標（Deployment、StatefulSet 等）被擴展到 1 時，由 {scalers1;scalers2;...} 觸發 |
| `KEDAScaleTargetDeactivated` | `Normal` | ScaledObject 的擴縮目標（Deployment、StatefulSet 等）被縮減到 0 時 |
| `KEDAScaleTargetActivationFailed` | `Warning` | KEDA 無法將 ScaledObject 的擴縮目標擴展到 1 時 |
| `KEDAScaleTargetDeactivationFailed` | `Warning` | KEDA 無法將 ScaledObject 的擴縮目標縮減到 0 時 |
| `KEDAJobsCreated` | `Normal` | KEDA 為 ScaledJob 建立 Job 時 |
| `TriggerAuthenticationAdded` | `Normal` | 新增新的 TriggerAuthentication 時 |
| `TriggerAuthenticationDeleted` | `Normal` | TriggerAuthentication 被刪除時 |
| `ClusterTriggerAuthenticationAdded` | `Normal` | 新增新的 ClusterTriggerAuthentication 時 |
| `ClusterTriggerAuthenticationDeleted` | `Normal` | ClusterTriggerAuthentication 被刪除時 |

---

## 快速參考 (Quick Reference)

| 事件類型 | 說明 |
|----------|------|
| `Normal` | 正常操作事件 |
| `Warning` | 警告或錯誤事件 |

| 指令 | 說明 |
|------|------|
| `kubectl get events --field-selector involvedObject.kind=ScaledObject` | 查看 ScaledObject 相關事件 |
| `kubectl get events -n <namespace>` | 查看命名空間中的事件 |
| `kubectl describe scaledobject <name>` | 查看 ScaledObject 詳細資訊（包含事件）|
