# 03 HCM Backend API Contract

## 1. 文件目的

本文件定義 Frontend 與 HCM Backend 之間的 HTTP API 合約，並標示每個 API 是否會進入 Provider Plugin 掛點。

本文件只描述 HCM Backend API，不重寫 Provider 外部 API 或 `02_Provider_Plugin_Model.md` 的標準輸入/輸出。

## 2. 文件邊界

| 範圍 | 本文件是否負責 | 說明 |
|---|---|---|
| Frontend -> Backend endpoint | 是 | 定義 path、method、request/response 類型 |
| 是否進 Provider Plugin | 是 | 標示掛點名稱或 `無` |
| Provider Plugin 標準 input/output | 否 | 請參考 `02_Provider_Plugin_Model.md` |
| Provider 外部 API / SDK | 否 | 請參考 `provider_plugins/*.md` |
| 業務資料詞彙與生命週期 | 否 | 請參考 `01_Business_Data_Dictionary.md` |

## 3. 型別來源

- 後端路由：`backend/src/routes/collections.ts`
- 目前 API 以 document-style collection CRUD 為主，payload 以集合資料型別為準。

## 4. Cloud Settings APIs

| HCM Backend API | HTTP Request | HTTP Response | Provider Plugin 掛點 | 備註 |
|---|---|---|---|---|
| `GET /api/cloud-providers` | none | `CloudProvider[]` | 無 | 依 `order` 排序 |
| `POST /api/cloud-providers` | `CloudProvider` | `CloudProvider` | 無 | 本地資料 |
| `PUT /api/cloud-providers/:id` | `CloudProvider` | `CloudProvider` | 無 | 本地資料 |
| `DELETE /api/cloud-providers/:id` | none | deleted object | 無 | 本地資料 |
| `GET /api/cloud-connections` | none | `CloudConnection[]` | 無 | auth 回傳遮罩 |
| `POST /api/cloud-connections` | `CloudConnection` | `CloudConnection` | 無 | 僅建立 connection |
| `PUT /api/cloud-connections/:id` | `CloudConnection` | `CloudConnection` | 無 | 併入 secret 保留規則 |
| `DELETE /api/cloud-connections/:id` | none | deleted object | 無 | 本地資料 |
| `POST /api/cloud-connections/:id/service-account/start` | `{ client_id? }` | `CloudConnection` | Provider 授權 | 僅 service_account driver |
| `POST /api/cloud-connections/:id/service-account/poll` | none | `CloudConnection` | Provider 授權 | 輪詢外部授權完成 |
| `POST /api/cloud-connections/:id/sync` | `{ phase: "pools" }` | `SyncAcceptedResponse` | 同步資源池 | 非同步，202 accepted |
| `POST /api/cloud-connections/:id/sync` | `{ phase: "network" }` | `SyncAcceptedResponse` | 同步網路 | 非同步，202 accepted |
| `POST /api/cloud-connections/:id/sync` | `{ phase: "templates" }` | `SyncAcceptedResponse` | 同步 VM 規格來源 | 非同步，202 accepted |
| `POST /api/cloud-connections/:id/sync` | `{ phase: "vms" }` | `SyncAcceptedResponse` | 同步 VM 清單 | 非同步，202 accepted |
| `GET /api/cloud-connections/:id/sync-status` | none | `SyncStatusResponse` | 無 | 查同步進度與錯誤 |

## 5. Apply Wizard APIs

| HCM Backend API | HTTP Request | HTTP Response | Provider Plugin 掛點 | 備註 |
|---|---|---|---|---|
| `GET /api/projects` | none | `Project[]` | 無 | collection 查詢 |
| `POST /api/projects` | `Project` | `Project` | 無 | collection 建立 |
| `PUT /api/projects/:id` | `Project` | `Project` | 無 | collection 更新 |
| `GET /api/systems` | none | `System[]` | 無 | collection 查詢 |
| `POST /api/systems` | `System` | `System` | 無 | collection 建立 |
| `PUT /api/systems/:id` | `System` | `System` | 無 | collection 更新 |
| `GET /api/pools` | none | `Pool[]` | 無 | 使用已同步資料 |
| `GET /api/allocation-requests` | none | `AllocationRequest[]` | 無 | collection 查詢 |
| `POST /api/allocation-requests` | `AllocationRequest` | `AllocationRequest` | 無 | 建立申請 |
| `PUT /api/allocation-requests/:id` | `AllocationRequest` | `AllocationRequest` | 無 | 更新申請 |
| `GET /api/apply-history` | none | `ApplyHistory[]` | 無 | collection 查詢 |
| `POST /api/apply-history` | `ApplyHistory` | `ApplyHistory` | 無 | 建立歷程 |
| `PUT /api/apply-history/:id` | `ApplyHistory` | `ApplyHistory` | 無 | 更新歷程 |

## 6. Allocation Management APIs

| HCM Backend API | HTTP Request | HTTP Response | Provider Plugin 掛點 | 備註 |
|---|---|---|---|---|
| `GET /api/allocations` | none | `Allocation[]` | 無 | collection 查詢 |
| `POST /api/allocations` | `Allocation` | `Allocation` | Allocation 附加資源（視 driver） | Harvester 可能建立 namespace |
| `PUT /api/allocations/:id` | `Allocation` | `Allocation` | Allocation 附加資源（視 driver） | Harvester 可能更新 namespace |
| `DELETE /api/allocations/:id` | none | deleted object | 無 | collection 刪除 |
| `GET /api/allocation-requests` | none | `AllocationRequest[]` | 無 | collection 查詢 |
| `PUT /api/allocation-requests/:id` | `AllocationRequest` | `AllocationRequest` | 無 | 更新 mount / 狀態 |
| `POST /api/projects` | `Project` | `Project` | 無 | 申請生效時可能建立 |
| `POST /api/systems` | `System` | `System` | 無 | 申請生效時可能建立 |
| `PUT /api/apply-history/:id` | `ApplyHistory` | `ApplyHistory` | 無 | request 完成後回寫 |

## 7. VM Management APIs

| HCM Backend API | HTTP Request | HTTP Response | Provider Plugin 掛點 | 備註 |
|---|---|---|---|---|
| `GET /api/vms` | none | `VM[]` | 無 | collection 查詢 |
| `POST /api/vms` | `VM` | `VM` | 建立 VM（視 driver） | driver 支援時會呼叫 `createVm` |
| `PUT /api/vms/:id` | `VM` | `VM` | 無 | 更新 HCM VM 資料 |
| `DELETE /api/vms/:id` | none | deleted object | 無 | 刪除 HCM VM 資料 |
| `POST /api/vms/:id/start` | none | `VM` | VM 開機 | driver 不支援回 501 |
| `POST /api/vms/:id/stop` | none | `VM` | VM 關機 | driver 不支援回 501 |
| `GET /api/vms/events` | SSE | VM status events | 無 | SSE 狀態流 |

## 8. Common Collection APIs

| Collection | APIs |
|---|---|
| `subnets` | `GET/POST/PUT/DELETE /api/subnets` |
| `security-groups` | `GET/POST/PUT/DELETE /api/security-groups` |
| `pools` | `GET/POST/PUT/DELETE /api/pools`, `PATCH /api/pools/:id/toggle` |
| `users` | `GET/POST/PUT/DELETE /api/users` |

## 9. 非同步與錯誤處理原則

- `POST /api/cloud-connections/:id/sync` 為非同步 API，成功接收回 `202 accepted`。
- 同步執行中的進度與錯誤需透過 `GET /api/cloud-connections/:id/sync-status` 查詢。
- VM 開關機若 driver 不支援，回 `501`。
- 通用錯誤格式以 `{ message }` 回傳。

## 10. 待確認事項

| 項目 | 說明 |
|---|---|
| VM 刪除是否要同步刪 provider VM | 目前 API 只保證刪 HCM VM 記錄 |
| Allocation 刪除是否要回收 provider 附加資源 | 目前未定義標準回收行為 |
| `projects` / `systems` 不提供 delete | 是否需保留現況或補刪除流程待確認 |