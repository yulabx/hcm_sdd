# 16 Allocation Management 配置管理

## 1. 功能目的

Allocation Management 用於管理 HCM 中已生效與待處理的資源配置。它承接 Apply Wizard 送出的 Allocation Request，讓管理者將申請轉成正式 Allocation，並維護 Shared Pool 配額、Dedicated Pool 指派與申請完成狀態。

本功能的核心目的有三個：

- 查看目前已生效的 Shared Allocation 與 Dedicated Assignment。
- 處理待分配申請，將 Shared Pool 申請補齊配額、Subnet 與必要附加資訊。
- 在申請條件完成後，將 Request 標記為完成，使其成為後續 VM Management 可使用的資源額度。

Allocation Management 可能涉及 Provider Plugin，但僅限 Provider 在「分配完成前」需要建立或準備附屬資源時，例如 Harvester namespace。不同 Provider 的附屬資源、輸入欄位、外部 API 上下行與轉換規則，應放在 Provider Plugin 文件中描述。

## 2. 使用者與權限

| 角色 | 可執行動作 | 說明 |
| --- | --- | --- |
| HCM 管理者 | 檢視所有配置、建立與修改 Shared Allocation、建立與解除 Dedicated Assignment、完成 Request | Allocation Management 的主要使用者 |
| 專案管理者 | 查詢自己專案的 Allocation 狀態 | 原則上不直接進行資源分配，除非業務規則授權 |
| 一般使用者 | 不建議進入本功能 | 透過 Apply History 查看自己的申請狀態即可 |


## 3. 畫面與 Event 規格

### 3.1 Allocation Management 主畫面

#### 畫面目的

提供正式配置與待處理申請的切換入口，並用 KPI 顯示目前資源配置概況。

```text
┌────────────────────────────────────────────────────────────┐
│ Allocation Management                                      │
│ [Pools] [Requests 3]                                       │
├────────────────────────────────────────────────────────────┤
│ Shared Allocations | Dedicated Assignments | Pending | Risk │
├────────────────────────────────────────────────────────────┤
│ Current Tab Content                                        │
└────────────────────────────────────────────────────────────┘
```

**畫面資料**

| 資料 | 用途 | 來源 |
| --- | --- | --- |
| 目前頁籤 | 判斷顯示正式配置或待處理申請 | 畫面狀態 |
| Shared Allocation 數量 | 呈現已生效的共享配額筆數 | HCM Allocation |
| Dedicated Assignment 數量 | 呈現已生效的專用資源池指派數量 | HCM Allocation |
| Pending Request 數量 | 呈現待處理申請數量 | HCM Allocation Request |
| 高風險 Shared Pool 數量 | 提醒共享資源池使用量偏高 | HCM Pool 與 Allocation |

**Event**

| UI 元件 | Event | 業務動作 | 異動/查詢資料 | HCM Backend API | 畫面結果 |
| --- | --- | --- | --- | --- | --- |
| Pools 頁籤 | 點擊 | 查看已生效配置 | 查詢 Allocation、Pool、Project、System | `GET /api/allocations` 和 `GET /api/pools` | 顯示 Shared Allocation 與 Dedicated Assignment |
| Requests 頁籤 | 點擊 | 查看待處理申請 | 查詢 Allocation Request | `GET /api/allocation-requests` | 顯示 Pending Request 清單 |
| KPI 區 | 顯示 | 提供資源配置概況 | 彙總 HCM 本地資料 | `GET /api/allocations`、`GET /api/allocation-requests` 和 `GET /api/pools` | 顯示配置數、待處理數與風險數 |

### 3.2 Pools 頁籤

#### 畫面目的

讓管理者依資源池查看已生效的配置。Shared Pool 以配額方式管理；Dedicated Pool 以整池指派方式管理。

```text
┌────────────────────────────────────────────────────────────┐
│ Pools                                                      │
├────────────────────────────────────────────────────────────┤
│ Shared Allocation                                          │
│ ┌────────────────────────────────────────────────────────┐ │
│ │ Pool S1  Cloud / Site / Env             [Add]          │ │
│ │ CPU bar | Memory bar | Disk bar                         │ │
│ │ System | Project | Env | CPU | Mem | Disk | Subnet |... │ │
│ └────────────────────────────────────────────────────────┘ │
│                                                            │
│ Dedicated Assignment                         [新增指派]    │
│ ┌────────────────────────────────────────────────────────┐ │
│ │ Pool D1  Cloud / Site / Env                            │ │
│ │ System A / Project A / dedicated        [解除指派]      │ │
│ └────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────┘
```

**畫面資料**

| 資料 | 用途 | 來源 |
| --- | --- | --- |
| Shared Pool 清單 | 顯示可配置共享配額的資源池 | HCM Pool |
| Shared Allocation 清單 | 顯示每個 Shared Pool 已分配給哪些專案與系統 | HCM Allocation |
| Dedicated Pool 清單 | 顯示可指派或已指派的專用資源池 | HCM Pool |
| Dedicated Assignment 清單 | 顯示 Dedicated Pool 已指派給哪些專案與系統 | HCM Allocation |
| Pool 容量與使用量 | 顯示 CPU、Memory、Disk 使用狀況 | HCM Pool 與 Allocation |
| Cloud / Site / Env | 協助管理者判斷資源位置與用途 | HCM Cloud、Pool |

**Event**

| UI 元件 | Event | 業務動作 | 異動/查詢資料 | HCM Backend API | 畫面結果 |
| --- | --- | --- | --- | --- | --- |
| Add Allocation | 點擊 | 為 Shared Pool 新增正式配額 | 查詢 Pool、Project、System、Subnet | `GET /api/pools`、`GET /api/projects`、`GET /api/systems`、`GET /api/subnets` | 開啟 Shared Allocation 表單 |
| Edit | 點擊 | 修改既有 Shared Allocation | 查詢既有 Allocation | `GET /api/allocations` | 開啟 Shared Allocation 表單 |
| Remove | 點擊 | 解除 Shared Allocation | 刪除或停用 Allocation | `DELETE /api/allocations/:id` | Allocation 從表格移除 |
| 新增指派 | 點擊 | 手動建立 Dedicated Pool 指派 | 查詢可用 Dedicated Pool、Project、System | `GET /api/pools`、`GET /api/projects` 和 `GET /api/systems` | 開啟 Dedicated 指派表單 |
| 解除指派 | 點擊 | 解除 Dedicated Pool 與系統的關係 | 刪除或停用 Allocation | `DELETE /api/allocations/:id` | Dedicated Assignment 從清單移除 |

### 3.3 Shared Allocation 表單

#### 畫面目的

建立或修改 Shared Pool 的正式配額。管理者需指定專案、系統、CPU、Memory、Disk、可用 Subnet，並視 Provider 差異填寫附加欄位。

```text
┌────────────────────────────────────────────────────────────┐
│ Create / Edit Shared Allocation                            │
├────────────────────────────────────────────────────────────┤
│ Pool Capacity: CPU / Memory / Disk free                    │
│                                                            │
│ Project [v]        System [v]                              │
│                                                            │
│ Quota                                                     │
│ CPU [__]  Memory [__]  Disk [__]                           │
│                                                            │
│ Subnet                                                     │
│ [ ] subnet-a  [ ] subnet-b                                 │
│                                                            │
│ Provider Extension                                         │
│ Namespace [____________]  Status: pending                  │
│                                                            │
│                                      [Cancel] [Save]       │
└────────────────────────────────────────────────────────────┘
```

**畫面資料**

| 資料 | 用途 | 來源 |
| --- | --- | --- |
| Pool 容量摘要 | 協助管理者判斷剩餘可配置額度 | HCM Pool 與 Allocation |
| 專案清單 | 指定 Allocation 歸屬專案 | HCM Project |
| 系統清單 | 指定 Allocation 歸屬系統 | HCM System |
| CPU / Memory / Disk | 設定 Shared Allocation 配額 | 使用者輸入 |
| Subnet 清單 | 指定此 Allocation 可使用的網段 | HCM Subnet |
| Provider Extension 欄位 | 補充 Provider 特有的配置資訊 | Provider Plugin 規格 |

**Event**

| UI 元件 | Event | 業務動作 | 異動/查詢資料 | HCM Backend API | 畫面結果 |
| --- | --- | --- | --- | --- | --- |
| Project | 選擇 | 指定配置歸屬專案 | 查詢 HCM Project 並更新表單 | `GET /api/projects` | 更新可選系統 |
| System | 選擇 | 指定配置歸屬系統 | 查詢 HCM System 並更新表單 | `GET /api/systems` | 顯示選定系統 |
| CPU | 輸入 | 設定 CPU 配額 | 更新表單資料 | 無，前端狀態 | 顯示配額數值 |
| Memory | 輸入 | 設定記憶體配額 | 更新表單資料 | 無，前端狀態 | 顯示配額數值 |
| Disk | 輸入 | 設定磁碟配額 | 更新表單資料 | 無，前端狀態 | 顯示配額數值 |
| Subnet checkbox | 勾選/取消 | 指定此 Allocation 可用網段 | 更新表單資料 | 無，前端狀態 | 更新已選 Subnet |
| Provider Extension 欄位 | 輸入/選擇 | 補充 Provider 特有配置資訊 | 更新表單資料 | 無，前端狀態 | 顯示 Provider 差異資料 |
| Cancel | 點擊 | 放棄本次設定 | 不異動資料 | 無，前端狀態 | 關閉表單 |
| Save | 點擊 | 建立或更新 Shared Allocation | 新增或更新 HCM Allocation | `POST /api/allocations` 或 `PUT /api/allocations/:id` | 表格更新，必要時更新 Request mount 狀態 |

### 3.4 Dedicated 指派表單

#### 畫面目的

讓管理者將尚未指派的 Dedicated Pool 指派給既有專案與系統。Dedicated 指派代表整個專用資源池交由該系統使用，不在此表單填寫 CPU、Memory、Disk 配額。

```text
┌──────────────────────────────────────────────┐
│ 新增 Dedicated 指派                          │
├──────────────────────────────────────────────┤
│ Dedicated Pool [v]                           │
│ Project        [v]                           │
│ System         [v]                           │
│                                              │
│                         [取消] [建立指派]    │
└──────────────────────────────────────────────┘
```

**畫面資料**

| 資料 | 用途 | 來源 |
| --- | --- | --- |
| 可用 Dedicated Pool | 選擇尚未被指派的專用資源池 | HCM Pool 與 Allocation |
| 專案清單 | 指定 Dedicated Pool 歸屬專案 | HCM Project |
| 系統清單 | 指定 Dedicated Pool 歸屬系統 | HCM System |

**Event**

| UI 元件 | Event | 業務動作 | 異動/查詢資料 | HCM Backend API | 畫面結果 |
| --- | --- | --- | --- | --- | --- |
| Dedicated Pool | 選擇 | 指定要指派的專用資源池 | 查詢 HCM Pool | `GET /api/pools` | 更新表單 |
| Project | 選擇 | 指定配置歸屬專案 | 查詢 HCM Project 並更新表單 | `GET /api/projects` | 更新可選系統 |
| System | 選擇 | 指定配置歸屬系統 | 查詢 HCM System 並更新表單 | `GET /api/systems` | 顯示選定系統 |
| 取消 | 點擊 | 放棄本次指派 | 不異動資料 | 無，前端狀態 | 關閉表單 |
| 建立指派 | 點擊 | 建立 Dedicated Allocation | 新增 HCM Allocation | `POST /api/allocations` | Dedicated Assignment 清單更新 |

### 3.5 Requests 頁籤

#### 畫面目的

顯示 Apply Wizard 送出的待處理申請，讓管理者逐筆處理 Shared Pool 配額，並在所有必要配置完成後完成整張 Request。

```text
┌────────────────────────────────────────────────────────────┐
│ Pending Requests                                           │
├────────────────────────────────────────────────────────────┤
│ REQ-0001  Pending                                          │
│ Project: Project A / Applicant / Date                      │
│                                                            │
│ System A                                                   │
│ ┌────────────────────────────────────────────────────────┐ │
│ │ Pool S1  Shared  CPU 4 / Mem 16GB      [Configure]     │ │
│ │ Pool D1  Dedicated                    Auto handled     │ │
│ └────────────────────────────────────────────────────────┘ │
│                                                            │
│ Shared completed 0 / 1                    [Execute]        │
└────────────────────────────────────────────────────────────┘
```

**畫面資料**

| 資料 | 用途 | 來源 |
| --- | --- | --- |
| Pending Request 清單 | 顯示待處理申請 | HCM Allocation Request |
| 申請人與建立日期 | 協助管理者判斷申請來源 | HCM Allocation Request |
| 專案與系統 | 顯示申請歸屬與配置對象 | HCM Allocation Request |
| Mount 清單 | 顯示每個系統要求的 Pool | HCM Allocation Request |
| Shared mount 完成數 | 判斷 Request 是否可完成 | HCM Allocation Request |

**Event**

| UI 元件 | Event | 業務動作 | 異動/查詢資料 | HCM Backend API | 畫面結果 |
| --- | --- | --- | --- | --- | --- |
| Configure | 點擊 | 針對 Shared Pool 申請建立正式 Allocation | 查詢 Request、Pool、Project、System、Subnet | `GET /api/allocation-requests`、`GET /api/pools`、`GET /api/projects`、`GET /api/systems`、`GET /api/subnets` | 開啟 Shared Allocation 表單並帶入申請資料 |
| Execute | 點擊 | 完成整張 Request | 更新 Request 狀態，建立必要 Dedicated Allocation | `PUT /api/allocation-requests/:id` 和 `POST /api/allocations` | Request 從待處理清單移除 |
| Request Card | 顯示 | 讓管理者檢查申請內容 | 查詢 HCM Allocation Request | `GET /api/allocation-requests` | 顯示專案、系統、Pool 與完成狀態 |

### 3.6 解除配置確認

#### 畫面目的

在管理者移除 Shared Allocation 或解除 Dedicated Assignment 前，提供確認步驟，避免誤解除已生效資源額度。

```text
┌──────────────────────────────────────────────┐
│ Confirm Remove                               │
├──────────────────────────────────────────────┤
│ System / Project / Quota or Pool             │
│                                              │
│                         [Cancel] [Confirm]   │
└──────────────────────────────────────────────┘
```

**畫面資料**

| 資料 | 用途 | 來源 |
| --- | --- | --- |
| Allocation 摘要 | 讓管理者確認解除對象 | HCM Allocation |
| 配額或 Pool 指派資訊 | 說明解除後影響範圍 | HCM Allocation、Pool |

**Event**

| UI 元件 | Event | 業務動作 | 異動/查詢資料 | HCM Backend API | 畫面結果 |
| --- | --- | --- | --- | --- | --- |
| Cancel | 點擊 | 取消解除 | 不異動資料 | 無，前端狀態 | 關閉確認視窗 |
| Confirm | 點擊 | 解除已生效 Allocation | 刪除或停用 HCM Allocation | `DELETE /api/allocations/:id` | Allocation 清單更新 |

## 4. 主要業務資料異動情境

### 4.1 Shared Allocation 建立

管理者可以從 Pools 頁籤手動建立 Shared Allocation，也可以從 Pending Request 的 Configure 進入表單。建立時需指定專案、系統、配額與可用 Subnet。若申請中包含新專案或新系統，會在 Configure 建立正式 Allocation 時建立或確認 Project/System 主檔。完成後，HCM 新增一筆狀態為生效的 Allocation。

若此 Allocation 來自 Pending Request，對應的 Shared mount 會被標記為已完成。當同一張 Request 的所有 Shared mount 均完成後，Request 才可被 Execute。

若 Provider 需要 Allocation 附加資源，例如 Harvester namespace，會在 Shared Allocation 建立或修改生效時建立或更新。VM Management 後續建立 VM 時只使用已生效 Allocation 的附加資源，不在 VM 建立流程才建立 namespace。

### 4.2 Shared Allocation 修改

管理者可以修改已生效 Shared Allocation 的配額、Subnet 與 Provider Extension 資訊。修改後會影響該 Pool 的使用量統計，也會影響 VM Management 後續可使用的額度與網段。

### 4.3 Shared Allocation 解除

管理者可以解除 Shared Allocation。解除後，該專案與系統不再擁有該 Pool 的共享額度。若該 Allocation 已被 VM 使用，實際業務上需確認 VM 與資源回收流程，本文件先不展開。

### 4.4 Dedicated Assignment 建立

Dedicated Assignment 可以由管理者手動建立，也可以在 Execute Request 時依申請內容自動建立。建立後，指定 Dedicated Pool 會歸屬於某個專案與系統。

Dedicated Assignment 不填寫 CPU、Memory、Disk 配額，因為整個 Pool 即代表該系統可使用的資源範圍。

### 4.5 Dedicated Assignment 解除

管理者可以解除 Dedicated Assignment。解除後，該 Dedicated Pool 回到可指派狀態，可再分配給其他專案與系統。

### 4.6 Request 完成

當 Pending Request 中所有 Shared mount 都已配置完成，管理者可執行 Execute。Execute 會完成以下業務動作：

- 將未處理的 Dedicated mount 轉成 Dedicated Assignment。
- 將 Request 狀態更新為 Completed。
- 更新 Apply History，使申請人可看到申請已完成。
- 若申請中包含新專案或新系統，在 Execute 建立 Dedicated Assignment 或完成 Request 時建立或確認 Project/System 主檔。
- 若 Dedicated Assignment 所屬 Provider 需要附加資源，依 Provider Plugin 規格在 Execute 生效時建立或更新。

## 5. Provider Plugin 掛點

Allocation Management 的共通流程以 HCM 本地資料為主；Provider Plugin 僅在 Provider 需要附加資源或特殊配置條件時介入。

| 掛點 | 觸發功能 | HCM 標準輸入 | Provider 回傳 | HCM 後續處理 |
| --- | --- | --- | --- | --- |
| Allocation 附加資源 | 建立或修改 Allocation | Allocation、Pool、Project、System、Provider Extension 欄位 | 附加資源識別、狀態、錯誤訊息 | 更新 Allocation 的附加資源欄位與狀態 |
| Allocation 解除 | 移除或停用 Allocation | Allocation、Pool、附加資源識別 | 解除結果 | 更新 Allocation 狀態或移除資料 |
| Provider 可配置條件 | 顯示表單或執行 Save 前 | Pool、Project、System、Subnet、Request mount | 可用欄位、限制條件、提示 | 控制 Provider Extension 區塊與業務提示 |

Provider Plugin 文件需說明：

- 哪些 Provider 在 Allocation Management 需要顯示額外欄位。
- 額外欄位屬於表單式輸入、系統建議值，或外部互動結果。
- 建立 Allocation 時是否會呼叫外部 API。
- 外部 API 的上行資料、下行資料與範例。
- Provider 回傳資料如何轉成 HCM Allocation 的標準欄位。

## 6. Provider 差異索引

| 差異項目 | 共通規格 | Provider Plugin 文件需補充 |
| --- | --- | --- |
| 附加資源 | HCM 可保留 Provider Extension 區塊 | Provider 是否需要 namespace、resource group、folder、tenant 等附屬資源 |
| Namespace | 共通文件只視為 Provider Extension | Harvester namespace 的命名、狀態、建立 API 與轉換規則 |
| Subnet 選擇 | Shared Allocation 可勾選 Pool 關聯 Subnet | Provider 是否限制可選 Subnet 或需要額外網路條件 |
| Dedicated 指派 | HCM 以整池指派描述 | Provider 是否需在外部系統建立對應授權或隔離資源 |
| Quota 單位 | HCM 以 CPU Core、Memory GB、Disk TB 表示 | Provider 原始單位與 HCM 單位轉換 |
| 解除配置 | HCM 表示解除 Allocation | Provider 是否需同步解除外部附加資源 |

## 7. 下游資料流

| 產出資料 | 下游功能 | 用途 |
| --- | --- | --- |
| Shared Allocation | VM Management | 決定專案與系統可從哪些 Shared Pool 建立 VM，以及可用配額 |
| Dedicated Assignment | VM Management | 決定專案與系統可使用哪些 Dedicated Pool 建立 VM |
| Allocation 狀態 | Resource Overview | 顯示 Pool 使用量與分配狀況 |
| Allocation 狀態 | Project Dimension | 顯示專案與系統的資源配置狀態 |
| Request 完成狀態 | Apply History | 讓申請人看到申請已完成 |
| Provider 附加資源狀態 | VM Management | 供後續建立 VM 時使用 Provider 需要的附加識別 |

## 8. 待確認事項

| 項目 | 說明 | 建議確認方向 |
| --- | --- | --- |
| Request 是否需要 Reject / Cancel | 目前主流程描述完成申請，未定義退回或取消 | 若業務需要，應補充 Request 退回畫面與狀態 |
| Allocation 解除與既有 VM 關係 | 解除配置可能影響已建立 VM | 建議在 VM Management 或資源回收流程補充 |
| 新專案與新系統生效時點 | Apply Wizard 可能帶入新專案/系統 | 建議與 Apply Wizard 待確認事項一起定義 |
| Provider 附加資源建立時點 | 共通規則為 Allocation Configure / Execute 生效時建立，不等到 VM 建立時才建立 | 各 Provider Plugin 需補充附加資源類型、欄位、狀態與外部 API |
| Dedicated Request 是否可部分完成 | Shared mount 需要先完成，Dedicated mount 目前偏自動處理 | 若業務需要人工審核 Dedicated，需補充待處理狀態 |
