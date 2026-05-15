# 02 Provider Plugin 規範

## 1. 功能目的

Provider Plugin Model 定義 HCM 哪些業務節點可以掛 provider plugin，以及每個掛點必須遵守的標準輸入、標準輸出、外部 API 上下行描述方式與欄位轉換規範。

本文件只定義共同規範，不描述單一 provider 的實際 API 細節。AWS、VMware Cloud Director、Harvester、vSphere 等 provider 的實際差異，需寫在 `provider_plugins/*.md`。

本文件中的「掛載點」是 HCM Backend 內部 Provider Plugin 能力，不是 Frontend 呼叫的 HTTP endpoint。Frontend 與 Backend 的 HTTP API 請參考 `03_HCM_Backend_API_Contract.md`。

## 2. 設計原則

| 原則 | 說明 |
|---|---|
| 功能流程不寫死 provider | 功能文件只描述業務主流程；provider 差異由 plugin 文件補充 |
| 輸入/輸出一致 | 同一個掛點不論 provider 為何，都要回到 HCM 標準資料 |
| API 上下行可追溯 | 每個 provider 文件需說明 HCM 對 provider 送出什麼、provider 回傳什麼 |
| 轉換規則明確 | provider 欄位如何轉成 HCM 欄位必須用表格描述 |
| 畫面差異集中 | provider 對畫面欄位、按鈕、能力限制的影響集中寫在 provider 文件 |
| 敏感資訊遮罩 | password、token、secret、access key 等不得在文件範例中明文出現 |

## 3. Plugin 掛載點總表

| 掛載點 | 業務目的 | 觸發來源 | 標準輸入 | 標準輸出 | 影響的 HCM 資料 |
|---|---|---|---|---|---|
| Provider 授權 | 建立或更新雲端連線授權 | Cloud Settings | Connection Auth Input | Auth Result | Cloud Connection |
| 同步資源池 | 取得 provider 可管理資源池與容量 | Cloud Initialization | Sync Pools Input | Pool Sync Result | Pool |
| 同步網路 | 取得網路與安全群組資料 | Cloud Initialization | Sync Network Input | Network Sync Result | Subnet、Security Group |
| 同步 VM 規格來源 | 取得 template、image、flavor、disk type 等建立 VM 選項 | Cloud Initialization | Sync VM Catalog Input | VM Catalog Result | Pool VM Catalog |
| 同步 VM 清單 | 取得 provider 既有 VM 與狀態 | Cloud Initialization | Sync VM Inventory Input | VM Inventory Result | VM |
| 建立 VM | 依 HCM VM 建立需求在 provider 建立 VM | VM Management | Create VM Input | Create VM Result | VM |
| VM 開機 | 啟動 provider VM | VM Management | VM Power Input | VM Power Result | VM |
| VM 關機 | 停止 provider VM | VM Management | VM Power Input | VM Power Result | VM |
| VM 狀態追蹤 | 查詢 provider VM 最新狀態與 IP | VM Management | VM Status Input | VM Status Result | VM |
| Allocation 附加資源 | Allocation Configure / Execute 生效時在 provider 端建立附屬資源 | Allocation Management | Allocation Extension Input | Allocation Extension Result | Allocation |

### 3.1 Provider 文件勾稽規則

每份 `provider_plugins/*.md` 必須能從本章掛載點追到該 provider 的實際差異。Provider 文件不得只描述 API，也不得只描述能力；需明確回答「共同規範的哪個掛載點，在此 provider 中如何實現」。

Provider 文件需在前段加入「共同規範勾稽」章節，至少包含兩張表：

**掛載點對照表**

| 02 規範掛載點 | Provider 是否支援 | Provider 文件章節 | 標準輸入 | 標準輸出 | 外部 API 章節 | 影響的 HCM 資料 |
|---|---|---|---|---|---|---|

填寫規則：

- `02 規範掛載點` 必須使用本章總表中的名稱。
- `標準輸入`、`標準輸出` 必須使用第 4 章定義的名稱。
- `Provider 文件章節` 指向 provider 文件中描述此能力的章節，例如能力矩陣、畫面差異或欄位映射。
- `外部 API 章節` 指向 provider 文件中列出 Method、URL、上行、下行的章節；若 provider 不支援，寫 `不支援`。
- `影響的 HCM 資料` 必須回到 HCM 標準資料，例如 Cloud Connection、Pool、Subnet、VM Catalog、VM、Allocation。

**標準輸入/輸出落地表**

| 02 標準資料 | Provider 對應欄位/概念 | Provider 文件章節 | 轉換或限制 |
|---|---|---|---|

填寫規則：

- 用來說明第 4 章的標準資料在此 provider 中實際對應到什麼欄位或概念。
- 如果某個標準資料在此 provider 不適用，需寫明 `不適用` 與原因。
- 如果標準資料需要拆成多個 API 或多個 provider 資源，需在 `轉換或限制` 說清楚。

## 4. 標準輸入/輸出定義

### 4.1 Connection Auth Input

| 欄位 | 業務意義 | 必填 | 範例 | 備註 |
|---|---|---|---|---|
| provider | provider 類型 | 是 | AWS、VCD、Harvester、vSphere | 決定使用哪份 provider 文件 |
| connection_id | HCM cloud connection 識別 | 是 | aws-prod-01 | 用於追蹤同步來源 |
| endpoint | provider 連線位置 | 依 provider | https://example.provider | 可為 API endpoint、inventory endpoint |
| auth_type | 授權方式 | 是 | access_key、token、service_account | provider 文件需列出支援方式 |
| credential | 授權資料摘要 | 是 | `<masked>` | 文件不得出現明文 secret |
| scope | 同步或操作範圍 | 否 | region、pool filter、namespace | provider 差異欄位 |

### 4.2 Auth Result

| 欄位 | 業務意義 | 必填 | 範例 | 備註 |
|---|---|---|---|---|
| authorization_status | 授權結果 | 是 | ok、pending、error | service account 類流程可為 pending |
| display_message | 給使用者看的授權狀態 | 否 | 已完成授權 | 不放敏感資訊 |
| next_action | 使用者下一步 | 否 | 請前往驗證網址完成授權 | 依 provider 需要 |
| connection_update | 需回寫 connection 的非敏感資料 | 否 | token 到期時間、授權狀態 | secret 需遮罩 |

### 4.3 Sync Pools Input

| 欄位 | 業務意義 | 必填 | 範例 | 備註 |
|---|---|---|---|---|
| connection | 要同步的 cloud connection | 是 | hicloud-prod | 包含 provider、endpoint、授權摘要 |
| scope | 同步範圍 | 否 | region、pool filter | provider 可依能力使用 |
| existing_hcm_pools | HCM 既有 pool 摘要 | 否 | pool id、provider ref | 用於判斷新增或更新 |

### 4.4 Pool Sync Result

| 欄位 | 業務意義 | 必填 | 範例 | 備註 |
|---|---|---|---|---|
| provider_pool_id | provider 原始資源池識別 | 是 | vdc-001、cluster-01 | 落地到 HCM 時統一寫入 `ref.id` |
| name | 資源池名稱 | 是 | prod-vdc | 可作為 HCM 預設顯示名稱 |
| site | 站點或區域 | 否 | primary、ap-northeast-1 | 沒有時可由管理員補 |
| capacity_cpu | CPU 總量 | 否 | 64000 | 對齊持久層基礎單位 (Millicores) |
| capacity_memory | Memory 總量 | 否 | 549755813888 | 對齊持久層基礎單位 (Bytes) |
| capacity_disk | Disk 總量 | 否 | 2199023255552 | 對齊持久層基礎單位 (Bytes) |
| cpu_provisioned | provider 回報 CPU 已配置量 | 否 | 32000 | 對齊持久層基礎單位 (Millicores) |
| mem_provisioned | provider 回報 Memory 已配置量 | 否 | 274877906944 | 對齊持久層基礎單位 (Bytes) |
| disk_provisioned | provider 回報 Disk 已配置量 | 否 | 1099511627776 | 對齊持久層基礎單位 (Bytes) |
| cpu_used | provider 回報 CPU 實際使用量 | 否 | 16000 | 對齊持久層基礎單位 (Millicores) |
| mem_used | provider 回報 Memory 實際使用量 | 否 | 137438953472 | 對齊持久層基礎單位 (Bytes) |
| disk_used | provider 回報 Disk 實際使用量 | 否 | 549755813888 | 對齊持久層基礎單位 (Bytes) |
| raw_summary | provider 原始資料摘要 | 否 | VDC allocation model | 只保留可閱讀摘要 |

各 provider 可填欄位：

| Provider | 建議必填 | 容量欄位（有就填） | 使用量/已配置欄位（有就填） | 備註 |
|---|---|---|---|---|
| Harvester | `provider_pool_id`、`name` | `capacity_cpu`、`capacity_memory`、`capacity_disk` | `cpu_provisioned`、`mem_provisioned`、`disk_provisioned` | 需對齊持久層基礎單位 |
| VMware Cloud Director | `provider_pool_id`、`name` | `capacity_cpu`、`capacity_memory`、`capacity_disk` | `cpu_provisioned` 或 `used_cpu`（擇一） | 需對齊持久層基礎單位 |
| AWS | `provider_pool_id`、`name` | 可留空 | 可留空 | VPC 不提供固定容量，以 `raw_summary` 補充 |
| vSphere | `provider_pool_id`、`name` | `capacity_cpu`、`capacity_memory`、`capacity_disk` | `used_cpu` 或 `cpu_provisioned` | 需對齊持久層基礎單位 |

填寫規範：

- 若來源無對應值，可省略該欄位，不需以 0 假填。
- 若容量欄位無法可靠換算，需在 `raw_summary` 註明原因與來源限制。
- HCM UI 的 Pool bar 支援兩種視角：已申請視角使用 `*_provisioned_* / *_total_*`，使用量視角使用 `*_used_* / *_total_*`。Provider plugin 應盡量同時提供 provisioned 與 used；若只能提供其中一種，UI 需以可取得欄位呈現或顯示 0 / N/A。

### 4.5 Sync Network Input

| 欄位 | 業務意義 | 必填 | 範例 | 備註 |
|---|---|---|---|---|
| connection | 要同步的 cloud connection | 是 | aws-prod-01 | 決定 provider |
| pool_ref | provider pool 識別 | 否 | vpc-001、vdc-001 | 有些 provider 以 pool 為網路範圍 |
| scope | 同步範圍 | 否 | region、namespace | provider 差異 |

### 4.6 Network Sync Result

| 欄位 | 業務意義 | 必填 | 範例 | 備註 |
|---|---|---|---|---|
| subnets | provider 網路轉成 HCM subnet 的清單 | 是 | subnet list | 欄位轉換需在 provider 文件說明 |
| security_groups | provider 安全群組清單 | 否 | sg list | provider 不支援時可為空 |
| pool_relations | subnet 與 pool 的關聯 | 否 | subnet-1 -> pool-1 | 可由同步或人工補 |

### 4.7 Sync VM Catalog Input

| 欄位 | 業務意義 | 必填 | 範例 | 備註 |
|---|---|---|---|---|
| connection | 要同步的 cloud connection | 是 | aws-prod-01 | 決定 provider |
| pool_ref | provider pool 識別 | 否 | vdc-001、cluster-01 | provider 需要時填入 |
| existing_catalog | HCM 既有 VM catalog 摘要 | 否 | image list、template list | 用於保留 legacy 選項 |

### 4.8 VM Catalog Result

| 欄位 | 業務意義 | 必填 | 範例 | 備註 |
|---|---|---|---|---|
| templates | template 型 VM 建立來源 | 否 | Ubuntu Template | 包含 `default_cpu`, `default_memory`, `default_disk` (需對齊基礎單位) |
| images | image 型 VM 建立來源 | 否 | AMI、Harvester Image | 包含 `image_id`, `storage_class` |
| flavors | 固定規格選項 | 否 | t3.medium | 包含 `cpu`, `memory` (需對齊基礎單位) |
| disk_types | 磁碟類型選項 | 否 | gp3、ssd | provider 支援時才有 |
| catalog_notes | 建立 VM 時的限制說明 | 否 | disk follows template | 顯示在 provider 差異文件 |

### 4.9 Sync VM Inventory Input

| 欄位 | 業務意義 | 必填 | 範例 | 備註 |
|---|---|---|---|---|
| connection | 要同步的 cloud connection | 是 | hicloud-prod | 決定 provider |
| pool_ref | provider pool 識別 | 是 | vdc-001 | 取得該範圍 VM |
| scope | 額外篩選範圍 | 否 | namespace、tag | provider 差異 |

### 4.10 VM Inventory Result

| 欄位 | 業務意義 | 必填 | 範例 | 備註 |
|---|---|---|---|---|
| provider_vm_id | provider 原始 VM 識別 | 是 | i-001、vm-123 | 落地到 HCM 時統一寫入 `ref.id` |
| name | VM 名稱 | 是 | ap-prod-01 | 畫面顯示 |
| status | HCM 標準 VM 狀態 | 是 | running | 狀態轉換需寫在 provider 文件 |
| cpu | VM CPU | 否 | 4 | Core / vCPU |
| ram | VM Memory | 否 | 17179869184 | 對齊持久層基礎單位 (Bytes) |
| disk | VM Disk | 否 | 107374182400 | 對齊持久層基礎單位 (Bytes) |
| ip | 主要 IP | 否 | 10.0.1.10 | 可由 NIC 推得 |
| nics | NIC 清單 | 否 | network、ip、security group | provider 支援程度不同 |
| disks | Disk 清單 | 否 | disk name, size (需對齊基礎單位 Bytes) | provider 支援程度不同 |
| tags | Project/System/Env 等標籤 | 否 | project_id、system_id | 用於歸屬 |

### 4.11 Create VM Input

| 欄位 | 業務意義 | 必填 | 範例 | 備註 |
|---|---|---|---|---|
| provider | provider 類型 | 是 | AWS | 決定建立能力 |
| connection | cloud connection | 是 | aws-prod-01 | 用於呼叫 provider |
| pool | HCM pool 與 provider pool ref | 是 | pool-prod-01 | 建立 VM 的資源範圍 |
| allocation | 使用的 allocation | 依流程 | alloc-ap-prod | 用於 quota 與歸屬 |
| vm_name | VM 名稱 | 是 | ap-prod-01 | provider 命名限制需在 provider 文件說明 |
| hostname | 主機名稱 | 否 | ap-prod-01 | provider 支援時帶入 |
| spec_source | template / image / flavor | 是 | AMI、template、flavor | provider 差異最大 |
| cpu | CPU | 依 provider | 4 | Core / vCPU |
| memory_gb | Memory | 依 provider | 16 | GB |
| disks | 磁碟需求 | 依 provider | 100 GB | provider 可不支援調整 |
| nics | 網路需求 | 是 | subnet、ip、security group | 多 NIC 依 provider 能力 |
| tags | Project/System/Env 等標籤 | 否 | project_id、system_id | 用於歸屬 |

### 4.12 Create VM Result

| 欄位 | 業務意義 | 必填 | 範例 | 備註 |
|---|---|---|---|---|
| provider_vm_id | provider 建立出的 VM 識別 | 是 | i-001 | 建立成功後寫入 `ref.id`，供後續追蹤使用 |
| initial_status | 初始狀態 | 是 | provisioning | 建立後通常未立即完成 |
| ip | provider 回傳的初始 IP | 否 | 10.0.1.10 | 可能稍後才取得 |
| follow_up_required | 是否需要後續狀態追蹤 | 否 | 是 | 若 provider 非同步建立 VM |
| display_message | 給使用者看的建立結果 | 否 | VM 建立中 | 不含技術堆疊細節 |

### 4.13 VM Power Input / Result

| 欄位 | 業務意義 | 必填 | 範例 | 備註 |
|---|---|---|---|---|
| provider_vm_id | provider VM 識別 | 是 | i-001 | 來源為 VM 的 `ref.id`，作為開關機目標 |
| action | 操作 | 是 | start、stop | 只允許 provider 支援的操作 |
| expected_status | 操作完成後預期狀態 | 否 | running、stopped | 用於狀態追蹤 |
| result_status | provider 回傳或 HCM 顯示狀態 | 是 | starting、stopping | 可先呈現進行中 |

### 4.14 Allocation Extension Input / Result

| 欄位 | 業務意義 | 必填 | 範例 | 備註 |
|---|---|---|---|---|
| allocation | HCM allocation 資料 | 是 | project、system、pool、quota | 附加資源在 Allocation Configure / Execute 生效時依 allocation 建立 |
| provider_context | provider 需要的上下文 | 依 provider | namespace、cluster | Harvester 等 provider 使用 |
| extension_resource | 建立出的附屬資源 | 否 | namespace | provider 不需要時可無 |
| extension_status | 附加資源狀態 | 否 | ready、error | 功能文件描述呈現方式 |

## 5. 外部 API 上下行文件規範

每份 provider 文件需針對支援的能力提供外部 API 上下行摘要。摘要不用貼完整 API 原文，但必須能回答：

| 問題 | 文件需回答 |
|---|---|
| HCM 送什麼出去 | 上行 request 摘要、必要業務欄位、敏感資訊遮罩 |
| provider 回什麼回來 | 下行 response 摘要、HCM 會取用哪些欄位 |
| HCM 如何轉換 | provider 欄位如何變成 HCM 標準資料 |
| 使用者看到什麼 | 同步、建立或操作完成後畫面會呈現什麼業務結果 |

建議格式：

| 項目 | 說明 |
|---|---|
| 使用情境 | 例如同步 VM 清單、建立 VM、啟動 VM |
| 上行 request 摘要 | HCM 對 provider 送出的查詢或操作內容 |
| 下行 response 摘要 | provider 回傳的關鍵資料 |
| HCM 取用欄位 | HCM 實際保留或轉換的欄位 |
| 敏感資訊遮罩 | password、token、secret、access key 以 `<masked>` 表示 |
| 對應標準輸出 | 對應本文件第 4 章哪個標準 result |

## 6. 欄位轉換規範

每份 provider 文件需針對支援的資料類型提供欄位轉換表。

| 資料類型 | 是否必須描述 | 說明 |
|---|---|---|
| Provider 資源池 -> HCM Pool | 是 | 同步資源池支援時必填 |
| Provider 網路 -> HCM Subnet | 是 | 同步網路支援時必填 |
| Provider 安全群組 -> HCM Security Group | provider 支援時 | provider 不支援需註明 |
| Provider template / image / flavor -> HCM VM Catalog | provider 支援時 | 建立 VM 支援時通常必填 |
| Provider VM -> HCM VM | 是 | 同步 VM 清單支援時必填 |
| HCM VM 建立需求 -> Provider VM request | provider 支援時 | 建立 VM 支援時必填 |
| Provider VM 狀態 -> HCM VM 狀態 | provider 支援時 | VM 同步或狀態追蹤支援時必填 |

轉換表格式：

| Provider 欄位 | Provider 意義 | HCM 欄位 | HCM 意義 | 轉換規則 | 範例 |
|---|---|---|---|---|---|

## 7. Provider 能力矩陣格式

各 provider 文件需用同一張能力表描述支援範圍。

| 能力 | 是否支援 | Provider 資料來源 | 產生/更新的 HCM 資料 | 備註 |
|---|---|---|---|---|
| 授權/登入 |  |  | Cloud Connection |  |
| 同步資源池 |  |  | Pool |  |
| 同步網路 |  |  | Subnet |  |
| 同步安全群組 |  |  | Security Group |  |
| 同步 VM 規格來源 |  |  | VM Catalog |  |
| 同步 VM 清單 |  |  | VM |  |
| 建立 VM |  |  | VM |  |
| VM 開機/關機 |  |  | VM 狀態 |  |
| VM 狀態追蹤 |  |  | VM 狀態 / IP |  |
| Allocation 附加資源 |  |  | Allocation |  |

## 8. 功能畫面差異格式

Provider 可能影響 HCM 畫面呈現。功能文件只放索引；差異細節集中寫在各 provider 文件。

| 功能 | 畫面/流程位置 | 畫面差異 | 欄位/選項差異 | 可操作能力 | 業務限制/提示 |
|---|---|---|---|---|---|

常見差異：

| 差異類型 | 說明 |
|---|---|
| 欄位顯示 | 某 provider 需要顯示 namespace、security group、image storage class 等欄位 |
| 欄位唯讀 | 某 provider 的 CPU、Memory、Disk 來自 template 或 flavor，使用者不可編輯 |
| 選項來源 | template、image、flavor、subnet、security group 來源不同 |
| 操作能力 | 建立 VM、啟停 VM、同步安全群組、狀態追蹤可能不同 |
| 限制提示 | provider 不支援某能力時，畫面需告知使用者 |

## 9. 敏感資訊遮罩規範

Provider 文件可描述授權欄位，但不得出現真實敏感資料。

| 資料類型 | 文件呈現方式 |
|---|---|
| password | `<masked>` |
| token | `<masked>` |
| secret access key | `<masked>` |
| refresh token | `<masked>` |
| private key | `<masked>` |
| endpoint | 可保留範例 domain 或使用 `https://provider.example.com` |

## 10. 待確認事項

| 項目 | 說明 |
|---|---|
| VM 刪除是否作為 provider plugin 掛點 | 目前主掛點未列 VM 刪除，需依產品目標確認 |
| VM 修改是否作為 provider plugin 掛點 | 目前主掛點未列 VM 修改，需依產品目標確認 |
| vSphere 是否以直接 provider API 或外部介接服務呈現 | provider 文件需確認實際上下行 |
| vCD 是否支援 HCM 建立 VM | provider 文件需明確標註支援或不支援 |
