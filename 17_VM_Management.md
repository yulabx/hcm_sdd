# 17 VM Management VM 管理

## 1. 功能目的

VM Management 用於管理 HCM 中已配置資源底下的 VM。使用者可以依 Cloud、Pool、Project、System、Allocation 查詢 VM，並在 provider 支援時建立 VM、修改 VM 基本資料、刪除 HCM VM 資料、執行開機與關機。

本功能承接 Allocation Management 的結果。也就是說，VM 建立不是從空白雲端任意建立，而是必須落在已啟用的 Pool 與已生效的 Allocation 或 Dedicated Assignment 之下，並受該 Pool / Allocation 的 CPU、Memory、Disk、Subnet 與 provider 能力限制。

VM Management 會使用 Cloud Settings 與 Provider Plugin 已同步回 HCM 的 Pool、Subnet、Security Group、Image、Template、Flavor 與 VM 清單。真正呼叫 provider 建立 VM、開機、關機或追蹤狀態時，必須依 `02_Provider_Plugin_Model.md` 的標準掛載點執行；不同 provider 的畫面欄位、外部 API / SDK 上下行、欄位轉換與限制，集中描述在 `provider_plugins/*.md`。

VM Management 不負責建立 Project、System 或 Allocation。若使用者尚未有可用額度，需先經 Apply Wizard 與 Allocation Management 完成配置。

## 2. 使用者與權限

| 角色 | 可執行動作 | 說明 |
| --- | --- | --- |
| 一般使用者 | 查詢自己專案可見 VM、檢視 VM 詳細資料 | 可見範圍依使用者所屬 Project / System 決定 |
| 專案管理者 | 查詢專案 VM、建立 VM、修改 VM 基本資料、開關機 | 是否可操作依組織授權與 provider 能力決定 |
| HCM 管理者 | 查詢全部 VM、建立/修改/刪除 VM、開關機 | 可跨 Cloud、Pool、Project、System 管理 |


## 3. 前置資料與業務規則

| 前置資料 | 用途 | 來源 |
| --- | --- | --- |
| Cloud Connection | 判斷 VM 所屬 provider 與可用能力 | Cloud Settings |
| Pool | VM 建立與查詢的資源範圍 | Cloud Initialization / Pool Settings |
| Allocation / Dedicated Assignment | 判斷使用者可用額度與歸屬 | Allocation Management |
| Subnet | VM 建立時可選網路 | Cloud Initialization / Subnet Settings / Allocation |
| Security Group | VM 建立時可選安全群組 | Provider Plugin，同步支援者才有 |
| VM Catalog | VM 建立時可選 Image、Template、Flavor | Provider Plugin 同步 |
| VM Inventory | 既有 VM 清單與狀態 | Provider Plugin 同步或 VM 操作回寫 |

主要業務規則：

- 只有 Active Pool 可作為 VM 建立或查詢的主要範圍。
- Shared Pool 建立 VM 時需選擇已生效的 Allocation；VM 使用該 Allocation 的配額與可用 Subnet。
- Dedicated Pool 建立 VM 時使用該 Dedicated Assignment 所屬 Project / System；整個 Pool 可供該系統使用。
- VM 建立前需檢查 CPU、Memory、Disk 是否超出可用額度。
- Provider 不支援 VM 建立時，建立入口應不開放，或僅提供同步 VM 的檢視；實際規則依 provider plugin 文件。
- Provider 不支援 VM 開機/關機時，Start / Stop 操作應不開放或不可執行；實際規則依 provider plugin 文件。
- Provider 特有欄位，例如 Image、Template、Flavor、Security Group、IP 模式、Namespace，不在本文件展開，需由 provider plugin 文件定義。

## 4. 畫面與 Event 規格

### 4.1 VM Management 主畫面

#### 畫面目的

提供 VM 管理入口。左側以 Pool 為主維度查詢與篩選，右側顯示選定 Pool 的 VM 與可操作項目。

```text
┌────────────────────────────────────────────────────────────┐
│ VM Management                                              │
├──────────────────────────┬─────────────────────────────────┤
│ Pool Search / Filters    │ Selected Pool Detail            │
│ [Search _________]       │ Pool Name / Cloud / Env          │
│ [Dedicated] [Shared]     │ CPU | Memory | Disk              │
│ Cloud [v] Project [v]    │                                 │
│ System [v]               │ VM Sections / VM Table           │
│                          │                                 │
│ Pool List                │                         [+ Add]  │
│ ┌──────────────────────┐ │                                 │
│ │ Pool A D/S Cloud     │ │                                 │
│ │ VM count / Usage bar │ │                                 │
│ └──────────────────────┘ │                                 │
└──────────────────────────┴─────────────────────────────────┘
```

**畫面資料**

| 資料 | 用途 | 來源 |
| --- | --- | --- |
| Pool 清單 | 作為 VM 查詢與建立的主要範圍 | HCM Pool |
| VM 清單 | 顯示目前 VM 狀態、規格、IP 與操作 | HCM VM |
| Allocation 清單 | 判斷 Shared Pool 可用額度與 VM 歸屬 | HCM Allocation |
| Project / System | 篩選與顯示 VM 業務歸屬 | HCM Project、System |
| 使用者角色 | 判斷可見範圍與可操作項目 | HCM User |

**Event**

| UI 元件 | Event | 業務動作 | 異動/查詢資料 | HCM Backend API | 畫面結果 |
| --- | --- | --- | --- | --- | --- |
| 進入頁面 | 載入 | 查詢可見 Pool、VM、Allocation | 查詢 HCM Pool、VM、Allocation | `GET /api/pools` 和 `GET /api/vms` | 顯示 Pool 清單與預設 Pool Detail |
| Pool Card | 點擊 | 切換目前管理的 Pool | 讀取選定 Pool、VM、Allocation | `GET /api/pools` 和 `GET /api/vms` | 右側顯示該 Pool 的 VM 資訊 |
| VM 狀態自動更新 | Provider 回寫或同步 | 更新 VM 狀態、IP、NIC 等資訊 | 更新 HCM VM | `GET /api/vms/events` | VM 狀態與清單即時更新 |

### 4.2 Pool 查詢與篩選區

#### 畫面目的

讓使用者快速縮小 Pool 範圍，找到要管理 VM 的資源池。

```text
┌──────────────────────────┐
│ Search [______________]  │
│ Type [Dedicated][Shared] │
│ Cloud [v]                │
│ Project [v]              │
│ System [v]               │
├──────────────────────────┤
│ Pool List                │
│ Pool A / Cloud / Env     │
│ CPU bar / Mem bar / Disk │
│ VM 12                    │
└──────────────────────────┘
```

**畫面資料**

| 資料 | 用途 | 來源 |
| --- | --- | --- |
| 搜尋關鍵字 | 依 Pool 名稱或識別篩選 | 使用者輸入 |
| Pool Type | 篩選 Dedicated 或 Shared Pool | 使用者輸入、HCM Pool |
| Cloud 條件 | 篩選指定 Cloud Connection | HCM Cloud |
| Project / System 條件 | 篩選與使用者或業務歸屬相關的 Pool | HCM Project、System、Allocation |
| Pool 使用量 | 協助判斷資源是否足夠 | HCM Pool、Allocation、VM |

**Event**

| UI 元件 | Event | 業務動作 | 異動/查詢資料 | HCM Backend API | 畫面結果 |
| --- | --- | --- | --- | --- | --- |
| Search | 輸入 | 依關鍵字篩選 Pool | 查詢或過濾 HCM Pool | `GET /api/pools` | Pool 清單縮小 |
| Type Filter | 點擊 | 篩選 Dedicated / Shared | 讀取 Pool type | `GET /api/pools` | Pool 清單更新 |
| Cloud Filter | 選擇 | 篩選指定 Cloud | 讀取 HCM Cloud / Pool | `GET /api/pools` 和 `GET /api/cloud-connections` | Pool 清單更新 |
| Project Filter | 選擇 | 篩選指定 Project 可使用的 Pool | 讀取 Allocation 與 Project | `GET /api/allocations` 和 `GET /api/projects` | Pool 清單更新 |
| System Filter | 選擇 | 篩選指定 System 可使用的 Pool | 讀取 Allocation 與 System | `GET /api/allocations` 和 `GET /api/systems` | Pool 清單更新 |

### 4.3 Pool Detail 與資源摘要

#### 畫面目的

顯示目前選定 Pool 的基本資訊、容量與使用狀態，並作為新增 VM 的入口。

```text
┌────────────────────────────────────────────────────────────┐
│ Pool A                         [D/S] Cloud / Site / Env    │
│ Pool ID / Provider / Location                              │
├────────────────────────────────────────────────────────────┤
│ CPU used/free      Memory used/free      Disk used/free    │
│ [########----]     [######------]        [####--------]    │
└────────────────────────────────────────────────────────────┘
```

**畫面資料**

| 資料 | 用途 | 來源 |
| --- | --- | --- |
| Pool 名稱與類型 | 告知目前管理範圍 | HCM Pool |
| Cloud / Site / Env | 協助判斷 VM 位置與用途 | HCM Cloud、Pool |
| CPU / Memory / Disk 使用量 | 建立 VM 前判斷可用容量 | HCM Pool、Allocation、VM |
| Provider 能力摘要 | 判斷建立與操作入口是否可用 | Provider Plugin 能力 |

**Event**

| UI 元件 | Event | 業務動作 | 異動/查詢資料 | HCM Backend API | 畫面結果 |
| --- | --- | --- | --- | --- | --- |
| Resource Bar | 顯示 | 呈現 Pool 或 Allocation 使用狀態 | 彙總 HCM Pool、Allocation、VM | `GET /api/pools`、`GET /api/allocations` 和 `GET /api/vms` | 顯示使用量與剩餘量 |
| Add VM | 點擊 | 開始在目前 Pool 建立 VM | 查詢 Pool、Allocation、Catalog、Subnet | `GET /api/pools`、`GET /api/allocations`、`GET /api/templates` 和 `GET /api/subnets` | 開啟 VM 建立表單 |

### 4.4 Dedicated Pool VM 區塊

#### 畫面目的

Dedicated Pool 以整個 Pool 指派給單一 Project / System 為主，因此 VM 以單一清單顯示。

```text
┌────────────────────────────────────────────────────────────┐
│ Dedicated Pool VM                              [+ Add VM]  │
├────────────────────────────────────────────────────────────┤
│ Name / Purpose | Status | Spec | IP / Subnet | Actions     │
│ ap-01          | run    | 2C4G | 10.0.1.10   | View Edit...│
└────────────────────────────────────────────────────────────┘
```

**畫面資料**

| 資料 | 用途 | 來源 |
| --- | --- | --- |
| Dedicated Assignment | 判斷 VM 歸屬 Project / System | HCM Allocation |
| VM 清單 | 顯示此 Pool 下所有 VM | HCM VM |
| VM 操作能力 | 判斷 View / Edit / Delete / Start / Stop 是否可用 | HCM 權限與 Provider Plugin |

**Event**

| UI 元件 | Event | 業務動作 | 異動/查詢資料 | HCM Backend API | 畫面結果 |
| --- | --- | --- | --- | --- | --- |
| Add VM | 點擊 | 在 Dedicated Pool 建立 VM | 查詢 Pool、Catalog、Subnet | `GET /api/pools`; 創建時 `POST /api/vms` | 開啟 VM 建立表單 |
| VM Row | 點擊或 View | 検視 VM 詳細資料 | 查詢 HCM VM | `GET /api/vms` | 開啟 VM 検視表單 |
| Edit | 點擊 | 修改 VM 在 HCM 的基本資料 | 查詢 HCM VM | `GET /api/vms` 和 `PUT /api/vms/:id` | 開啟 VM 編輯表單 |
| Delete | 點擊 | 準備刪除 VM 資料 | 查詢 HCM VM | `GET /api/vms` 和 `DELETE /api/vms/:id` | 開啟刪除確認 |
| Start / Stop | 點擊 | 準備變更 VM 電源狀態 | 查詢 HCM VM | `GET /api/vms`；確認後 `POST /api/vms/:id/start` 或 `POST /api/vms/:id/stop` | 開啟操作確認 |

### 4.5 Shared Pool VM 區塊

#### 畫面目的

Shared Pool 以 Allocation 為額度邊界，因此 VM 需依 Allocation 分組，讓使用者清楚知道每台 VM 消耗哪一份 Project / System 配額。

```text
┌────────────────────────────────────────────────────────────┐
│ Shared Pool                                                │
├────────────────────────────────────────────────────────────┤
│ Allocation: Project A / System A       Free CPU/Mem/Disk   │
│                                            [+ Add VM]      │
│ Name / Status / Spec / IP / Actions                       │
│                                                            │
│ Allocation: Project B / System B       Free CPU/Mem/Disk   │
│                                            [+ Add VM]      │
│ Name / Status / Spec / IP / Actions                       │
│                                                            │
│ Unassigned VM                                              │
│ Name / Status / Spec / IP                                  │
└────────────────────────────────────────────────────────────┘
```

**畫面資料**

| 資料 | 用途 | 來源 |
| --- | --- | --- |
| Shared Allocation 清單 | 分組顯示 VM 與可用額度 | HCM Allocation |
| 每個 Allocation 的 VM | 呈現各 Project / System 使用狀況 | HCM VM |
| 未歸屬 VM | 顯示尚未對應 Allocation 的 VM | HCM VM 同步資料 |
| Allocation 可用 Subnet | 建立 VM 時限制可選網路 | HCM Allocation、Subnet |

**Event**

| UI 元件 | Event | 業務動作 | 異動/查詢資料 | HCM Backend API | 畫面結果 |
| --- | --- | --- | --- | --- | --- |
| Allocation Section | 顯示 | 依 Allocation 分組顯示 VM | 查詢 HCM Allocation、VM | `GET /api/allocations` 和 `GET /api/vms` | 顯示該 Allocation 的 VM 與剩餘額度 |
| Add VM | 點擊 | 在指定 Allocation 下建立 VM | 查詢 Allocation、Catalog、Subnet | `GET /api/allocations`; 創建時 `POST /api/vms` | 開啟 VM 建立表單並帶入 Allocation |
| Unassigned VM | 顯示 | 呈現同步回來但尚未歸屬 Allocation 的 VM | 查詢 HCM VM | `GET /api/vms` | 顯示為待整理資料 |
| VM Actions | 點擊 | 檢視、修改、刪除或開關機 | 查詢 HCM VM | `GET /api/vms`、`PUT /api/vms/:id`、`DELETE /api/vms/:id`、`POST /api/vms/:id/start`、`POST /api/vms/:id/stop` | 開啟對應表單或確認 |

### 4.6 VM 建立 / 編輯 / 檢視表單

#### 畫面目的

讓使用者建立 VM 或維護 VM 的 HCM 基本資料。建立 VM 時，表單欄位會依 provider 能力與 Pool / Allocation 條件呈現。

```text
┌────────────────────────────────────────────────────────────┐
│ Add / Edit / View VM                                       │
├────────────────────────────────────────────────────────────┤
│ Pool / Allocation / Project / System                       │
│ VM Name [____________]  Hostname [____________]             │
│ Purpose [____________]  Env [v]                            │
│                                                            │
│ Spec Source                                                │
│ Template/Image/Flavor [v]                                  │
│ CPU [__]  Memory [__]  Disk [__]                           │
│                                                            │
│ Network                                                    │
│ Subnet [v]  IP Mode [Auto/Static]  IP [____________]        │
│ Security Group [provider dependent]                        │
│                                                            │
│ Initial Status [Running] [Stopped]                         │
│ Note [____________________________________________]         │
│                                                            │
│ Quota Check: CPU / Memory / Disk remaining                 │
│                                      [Cancel] [Save]       │
└────────────────────────────────────────────────────────────┘
```

**畫面資料**

| 資料 | 用途 | 來源 |
| --- | --- | --- |
| Pool / Allocation | 決定 VM 建立範圍、歸屬與額度 | HCM Pool、Allocation |
| VM 名稱、Hostname、用途、Env | VM 業務識別與管理資訊 | 使用者輸入 |
| Template / Image / Flavor | 決定 VM 建立來源與預設規格 | Provider Plugin 同步 VM Catalog |
| CPU / Memory / Disk | VM 規格與額度消耗 | 使用者輸入或 Catalog 預設 |
| Subnet / IP | VM 網路設定 | HCM Subnet、Allocation |
| Security Group | VM 網路安全設定 | Provider Plugin，支援者才顯示 |
| Initial Status | 建立完成後期望狀態 | 使用者輸入、Provider 能力 |
| Quota Check | 避免超出 Pool / Allocation 可用額度 | HCM Pool、Allocation、VM |

**Event**

| UI 元件 | Event | 業務動作 | 異動/查詢資料 | HCM Backend API | 畫面結果 |
| --- | --- | --- | --- | --- | --- |
| Allocation | 選擇 | 指定 Shared Pool VM 使用哪份額度 | 查詢 HCM Allocation | `GET /api/allocations` | 更新可用配額與可選 Subnet |
| VM Name | 輸入 | 設定 VM 顯示名稱與建立名稱 | 更新表單資料 | 無，前端狀態 | 顯示名稱 |
| Hostname | 輸入 | 設定 VM 主機名稱 | 更新表單資料 | 無，前端狀態 | 顯示 hostname |
| Template / Image / Flavor | 選擇 | 指定 VM 建立來源 | 查詢 HCM VM Catalog | `GET /api/templates` | 帶入預設 CPU / Memory / Disk 或規格摘要 |
| CPU / Memory / Disk | 輸入 | 設定 VM 規格與額度消耗 | 更新表單資料並檢查配額 | 無，前端狀態 | 顯示剩餘額度與是否超額 |
| Add / Remove Disk | 點擊 | 調整 VM 磁碟需求 | 更新表單資料並檢查配額 | 無，前端狀態 | 磁碟清單與總容量更新 |
| Subnet | 選擇 | 指定 VM 使用網段 | 查詢 HCM Subnet | `GET /api/subnets` | 更新 IP 與網路資訊 |
| IP Mode | 選擇 | 指定自動或固定 IP | 更新表單資料 | 無，前端狀態 | 顯示或停用 IP 欄位 |
| Security Group | 選擇/顯示 | 指定或呈現 VM 安全群組 | 查詢 HCM Security Group | `GET /api/security-groups` | 安全群組資料更新 |
| Initial Status | 選擇 | 指定建立後期望狀態 | 更新表單資料 | 無，前端狀態 | 顯示期望狀態 |
| Cancel | 點擊 | 放棄本次建立或修改 | 不異動資料 | 無，前端狀態 | 關閉表單 |
| Save | 點擊 | 建立 VM 或更新 VM 基本資料 | 新增或更新 HCM VM | `POST /api/vms` 或 `PUT /api/vms/:id` | 成功後 VM 清單更新 |

### 4.7 VM 開機 / 關機確認

#### 畫面目的

在使用者變更 VM 電源狀態前確認操作目標，避免誤開機或誤關機。

```text
┌──────────────────────────────────────────────┐
│ Confirm VM Power Action                      │
├──────────────────────────────────────────────┤
│ VM: ap-prod-01                               │
│ Current Status: running                      │
│ Action: Stop                                 │
│                                              │
│                         [Cancel] [Confirm]   │
└──────────────────────────────────────────────┘
```

**畫面資料**

| 資料 | 用途 | 來源 |
| --- | --- | --- |
| VM 摘要 | 確認操作對象 | HCM VM |
| 目前狀態 | 判斷可執行 Start 或 Stop | HCM VM |
| Provider VM 識別 | Provider 操作目標 | HCM VM `vm.ref.id` |
| 目標狀態 | 追蹤操作完成後狀態 | 操作類型 |

**Event**

| UI 元件 | Event | 業務動作 | 異動/查詢資料 | HCM Backend API | 畫面結果 |
| --- | --- | --- | --- | --- | --- |
| Start | 點擊 | 準備啟動 stopped VM | 查詢 HCM VM | `GET /api/vms` | 開啟確認視窗 |
| Stop | 點擊 | 準備停止 running VM | 查詢 HCM VM | `GET /api/vms` | 開啟確認視窗 |
| Cancel | 點擊 | 放棄本次電源操作 | 不異動資料 | 無，前端狀態 | 關閉確認視窗 |
| Confirm Start | 點擊 | 呼叫 provider 啟動 VM | 更新 HCM VM 狀態為 starting，後續追蹤 running | `POST /api/vms/:id/start` | VM 狀態顯示開機中，完成後更新 |
| Confirm Stop | 點擊 | 呼叫 provider 停止 VM | 更新 HCM VM 狀態為 stopping，後續追蹤 stopped | `POST /api/vms/:id/stop` | VM 狀態顯示關機中，完成後更新 |

### 4.8 VM 刪除確認

#### 畫面目的

在刪除 VM 資料前確認刪除對象與影響範圍。

```text
┌──────────────────────────────────────────────┐
│ Confirm Delete VM                            │
├──────────────────────────────────────────────┤
│ VM: ap-prod-01                               │
│ Pool: Pool A                                 │
│ Project / System: Project A / System A       │
│                                              │
│                         [Cancel] [Delete]    │
└──────────────────────────────────────────────┘
```

**畫面資料**

| 資料 | 用途 | 來源 |
| --- | --- | --- |
| VM 摘要 | 確認刪除對象 | HCM VM |
| Pool / Project / System | 說明刪除後影響的管理範圍 | HCM Pool、Allocation、VM |

**Event**

| UI 元件 | Event | 業務動作 | 異動/查詢資料 | HCM Backend API | 畫面結果 |
| --- | --- | --- | --- | --- | --- |
| Delete | 點擊 | 準備刪除 VM 資料 | 查詢 HCM VM | `GET /api/vms` | 開啟刪除確認 |
| Cancel | 點擊 | 放棄刪除 | 不異動資料 | 無，前端狀態 | 關閉確認視窗 |
| Confirm Delete | 點擊 | 刪除 HCM VM 資料 | 刪除或停用 HCM VM | `DELETE /api/vms/:id` | VM 從清單移除 |

## 5. 主要業務流程

### 5.1 查詢與檢視 VM

1. 使用者進入 VM Management。
2. HCM 依使用者權限載入可見 Pool、Allocation 與 VM。
3. 使用者透過搜尋、Cloud、Project、System 或 Pool Type 篩選。
4. 使用者點選 Pool 後，右側顯示該 Pool 的 VM 清單。
5. 使用者點選 VM 檢視詳細資料。

此流程只查詢 HCM 已保存資料，不直接呼叫 provider。若 Cloud Initialization 或狀態追蹤更新 VM，畫面會反映最新資料。

### 5.2 建立 VM

1. 使用者選擇 Active Pool。
2. 若為 Shared Pool，使用者需指定 Allocation；若為 Dedicated Pool，使用 Pool 的 Dedicated Assignment。
3. HCM 依 Pool provider 顯示可用 VM 建立欄位，例如 Image、Template、Flavor、Subnet、Security Group。
4. 使用者填寫 VM 名稱、規格、網路與初始狀態。
5. HCM 檢查必填欄位與 CPU、Memory、Disk 可用額度。
6. 若 provider 支援建立 VM，HCM 透過 Provider Plugin 建立 VM，並保存 provider 回傳的 VM 識別與初始狀態。
7. 若 provider 建立為非同步，HCM 顯示 provisioning / starting 等進行中狀態，後續由狀態追蹤更新為 running 或 stopped。

### 5.3 修改 VM 基本資料

1. 使用者在 VM 清單點選 Edit。
2. HCM 顯示既有 VM 資料。
3. 使用者修改 HCM 管理資訊，例如用途、備註、顯示名稱或可維護欄位。
4. HCM 更新 VM 資料並回到清單。

此流程以 HCM VM 資料維護為主；是否允許同步修改 provider VM 規格，不在本文件主流程定義，需由 provider plugin 與後續需求補充。

### 5.4 VM 開機 / 關機

1. 使用者在 VM 清單點選 Start 或 Stop。
2. HCM 顯示確認視窗。
3. 使用者確認後，HCM 透過 Provider Plugin 執行 VM Power 動作。
4. HCM 先將狀態顯示為 starting 或 stopping。
5. Provider 狀態追蹤完成後，VM 狀態更新為 running 或 stopped。

### 5.5 刪除 VM 資料

1. 使用者在 VM 清單點選 Delete。
2. HCM 顯示刪除確認視窗。
3. 使用者確認後，HCM 刪除或停用該筆 VM 管理資料。
4. VM 從 VM Management 清單移除。

本流程目前定義為 HCM VM 資料刪除。是否同步刪除 provider VM、磁碟或相關資源，需另行定義並補入 provider plugin 文件。

## 6. Provider Plugin 掛點

本章只描述 VM Management 何時進入 provider plugin。標準輸入/輸出以 `02_Provider_Plugin_Model.md` 第 4 章為準；各 provider 的 API / SDK 上下行與欄位轉換以 `provider_plugins/*.md` 為準。

| 掛載點 | 觸發畫面 / Event | 業務目的 | 標準輸入 | 標準輸出 | 影響的 HCM 資料 |
| --- | --- | --- | --- | --- | --- |
| 同步 VM 規格來源 | Cloud Initialization 後供 VM 建立表單使用 | 提供 Template、Image、Flavor、Disk Type 等選項 | Sync VM Catalog Input | VM Catalog Result | Pool VM Catalog |
| 同步 VM 清單 | Cloud Initialization 或週期同步後供 VM 清單使用 | 取得 provider 既有 VM 與狀態 | Sync VM Inventory Input | VM Inventory Result | VM |
| 建立 VM | VM 建立表單 Save | 依 HCM VM 建立需求在 provider 建立 VM | Create VM Input | Create VM Result | VM |
| VM 開機 | Confirm Start | 啟動 provider VM | VM Power Input | VM Power Result | VM |
| VM 關機 | Confirm Stop | 停止 provider VM | VM Power Input | VM Power Result | VM |
| VM 狀態追蹤 | 建立 VM、開機、關機後 | 查詢 provider VM 最新狀態、IP 與 NIC | VM Status Input | VM Status Result | VM |

## 7. Provider 差異索引

VM Management 的 provider 差異不寫死在本文件。功能文件只列出受影響的位置，實際差異請查 provider plugin 文件。

| 畫面 / 流程位置 | 受影響內容 | 依賴 Provider 能力 | 差異說明文件 |
| --- | --- | --- | --- |
| VM 建立入口 | 是否允許由 HCM 建立 VM | 建立 VM | `provider_plugins/01_Harvester.md`、`provider_plugins/02_VMware_Cloud_Director.md`、`provider_plugins/03_AWS.md`、`provider_plugins/04_vSphere.md` |
| VM 建立表單 | Image、Template、Flavor、CPU、Memory、Disk 欄位呈現 | 同步 VM 規格來源、建立 VM | 各 provider plugin 的功能畫面差異與 VM Catalog 映射 |
| VM 建立表單 | Subnet、IP Mode、NIC 欄位呈現 | 同步網路、建立 VM | 各 provider plugin 的 Network / Subnet 映射 |
| VM 建立表單 | Security Group 顯示、唯讀或可選 | 同步安全群組、建立 VM | AWS 支援；Harvester、VCD、vSphere 目前不支援，詳見各 provider plugin |
| VM 建立流程 | Allocation 附加資料，例如 Harvester namespace | Allocation 附加資源、建立 VM | `provider_plugins/01_Harvester.md` |
| VM 清單 | provider VM 狀態如何轉成 HCM 狀態 | 同步 VM 清單、VM 狀態追蹤 | 各 provider plugin 的 VM 狀態映射 |
| VM 操作 | Start / Stop 是否可用與操作後狀態 | VM 開機、VM 關機、VM 狀態追蹤 | 各 provider plugin 的能力矩陣與外部 API / SDK 範例 |

目前 provider 支援摘要：

| Provider | 建立 VM | VM 開機/關機 | 狀態追蹤 | VM Management 主要差異 |
| --- | --- | --- | --- | --- |
| Harvester | 支援 | 支援 | 支援 | 使用 Image；VM 建立使用 Allocation 已生效 namespace；不支援 AWS Security Group |
| VMware Cloud Director | 不支援 | 不支援 | 不支援 | 目前以同步 VM 與檢視為主；Template 資料可作為 catalog 來源 |
| AWS | 支援 | 支援 | 支援 | 使用 AMI、Instance Type、Subnet、Security Group；現行開發以 SDK 為主 |
| vSphere | 不支援 | 不支援 | 不支援 | 目前以同步 VM 與檢視為主；Template 資料可作為 catalog 來源 |

## 8. 狀態與資料異動

### 8.1 VM 狀態

| HCM 狀態 | 業務意義 | 常見來源 |
| --- | --- | --- |
| provisioning | VM 建立中，尚未取得穩定狀態 | 建立 VM 後 |
| running | VM 已啟動 | Provider 同步或狀態追蹤 |
| starting | VM 開機中 | Start 操作後 |
| stopping | VM 關機中 | Stop 操作後 |
| stopped | VM 已停止 | Provider 同步或狀態追蹤 |
| unknown | HCM 暫時無法判斷狀態 | Provider 資料不足或尚未同步 |

### 8.2 主要資料異動

| 動作 | 異動資料 | 說明 |
| --- | --- | --- |
| 建立 VM | 新增 HCM VM | 保存 VM 名稱、Pool、Allocation、規格、網路、provider VM 識別與初始狀態 |
| 修改 VM | 更新 HCM VM | 更新 HCM 管理資訊與可維護欄位 |
| 刪除 VM | 刪除或停用 HCM VM | 從 VM Management 清單移除 |
| 開機 | 更新 HCM VM 狀態 | 先顯示 starting，完成後依 provider 回傳更新 |
| 關機 | 更新 HCM VM 狀態 | 先顯示 stopping，完成後依 provider 回傳更新 |
| Provider 同步 | 新增或更新 HCM VM | 依 provider inventory 更新既有 VM 清單、狀態、IP、NIC |

## 9. 與其他功能的資料流

| 來源功能 | 流入 VM Management 的資料 | 用途 |
| --- | --- | --- |
| Cloud Settings | Cloud Connection、Pool、Subnet、Security Group、VM Catalog、VM Inventory | 決定可管理範圍、建立選項與同步 VM |
| Apply Wizard | Allocation Request | 間接透過 Allocation Management 形成可用額度 |
| Allocation Management | Allocation、Dedicated Assignment、Provider Extension | 決定 VM 建立可用額度、Project / System 歸屬與 provider 附加資料 |
| Provider Plugin | VM Catalog、VM Inventory、VM 操作結果 | 決定欄位差異、建立結果與狀態更新 |

| VM Management 產出資料 | 流向功能 | 用途 |
| --- | --- | --- |
| VM 清單與狀態 | Resource Overview | 顯示資源使用與 VM 狀態 |
| VM 與 Project / System 歸屬 | Project Dimension | 從專案或系統角度查看 VM |
| VM 規格與使用量 | Allocation Management | 協助判斷 Allocation 使用量與剩餘額度 |
| VM 狀態 | VM Management | 持續更新清單與操作狀態 |

## 10. 待確認事項

| 項目 | 說明 |
| --- | --- |
| Provider VM 刪除 | 目前本文件只定義 HCM VM 資料刪除；是否同步刪除 provider VM、Disk、Secret、PVC 等資源需另定義 |
| VM 修改範圍 | 目前以 HCM 管理資訊為主；是否支援 provider VM 規格變更需另定義 |
| Static IP | 不同 provider 與網路環境支援程度不同，實際欄位與限制需看 provider plugin |
| Security Group | AWS 支援，Harvester 不支援；其他 provider 目前不納入 HCM Security Group 同步 |
| Key Pair / User Data | 若業務需要 SSH key、cloud-init 或 userdata，需在 VM 建立表單與 provider plugin 補充 |
