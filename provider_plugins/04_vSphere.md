# 04 vSphere Provider Plugin

## 1. Provider 基本資訊

vSphere Provider Plugin 用於透過既有 vSphere 介接 API 取得 Cluster、Network、Template 與 VM inventory，轉成 HCM 標準資料供 Cloud Settings 與 VM Management 查詢使用。

HCM 中的 Cloud / Provider ID 可以是業務命名；Driver ID 固定為 `vsphere`，用來套用本文件定義的 Provider Plugin 規格。

| 項目 | 說明 |
| --- | --- |
| Provider 名稱 | vSphere |
| Provider 類型 | 私有雲 |
| HCM Provider Driver | `vsphere` |
| 主要資源概念 | Cluster、Resource Pool、Network / Portgroup、Template、VM |
| HCM 主要對應資料 | Cloud Connection、Pool、Subnet、Template Catalog、VM |
| 主要使用功能 | Cloud Settings、VM Management |

## 2. 共同規範勾稽

### 2.1 掛載點對照表

| 02 規範掛載點 | vSphere 是否支援 | vSphere 文件章節 | 標準輸入 | 標準輸出 | 外部 API 章節 | 影響的 HCM 資料 |
| --- | --- | --- | --- | --- | --- | --- |
| Provider 授權 | 支援 | Auth 與連線設定 | Connection Auth Input | Auth Result | 授權/登入 | Cloud Connection |
| 同步資源池 | 支援 | Cluster / Pool 映射 | Sync Pools Input | Pool Sync Result | 同步 Cluster / Pool | Pool |
| 同步網路 | 支援 | Network / Subnet 映射 | Sync Network Input | Network Sync Result | 同步 Network | Subnet |
| 同步 VM 規格來源 | 支援 | Template Catalog 映射 | Sync VM Catalog Input | VM Catalog Result | 同步 Template | Pool VM Catalog |
| 同步 VM 清單 | 支援 | VM 映射、狀態映射 | Sync VM Inventory Input | VM Inventory Result | 同步 VM 清單 | VM |
| Allocation 附加資源 | 不支援 | Provider 能力支援矩陣 | Allocation Extension Input | Allocation Extension Result | 不支援 | 無 |
| 建立 VM | 不支援 | Provider 能力支援矩陣 | Create VM Input | Create VM Result | 不支援 | 無 |
| VM 開機 | 不支援 | Provider 能力支援矩陣 | VM Power Input | VM Power Result | 不支援 | 無 |
| VM 關機 | 不支援 | Provider 能力支援矩陣 | VM Power Input | VM Power Result | 不支援 | 無 |
| VM 狀態追蹤 | 不支援 | Provider 能力支援矩陣 | VM Status Input | VM Status Result | 不支援 | 無 |

### 2.2 標準輸入/輸出落地表

| 02 標準資料 | vSphere 對應欄位/概念 | Provider 文件章節 | 轉換或限制 |
| --- | --- | --- | --- |
| Connection Auth Input | Base URL、Token、Pool Filter | Auth 與連線設定 | Token 以 Bearer Token 呼叫介接 API |
| Auth Result | 介接 API health 成功狀態 | 授權/登入 | GET `/health` 成功代表可連線 |
| Sync Pools Input | Cloud Connection、cluster scope | 同步 Cluster / Pool | vSphere 以 Cluster 作為 HCM Pool |
| Pool Sync Result | `/clusters` records | Cluster / Pool 映射 | CPU 原始單位為 MHz；Memory / Storage 為 GiB |
| Sync Network Input | Cloud Connection、cluster_ref | 同步 Network | 可依 cluster_refs 過濾 |
| Network Sync Result | `/networks` records | Network / Subnet 映射 | Network name 可推導 CIDR，不足時需人工補齊 |
| Sync VM Catalog Input | Cloud Connection、cluster_ref | 同步 Template | Template 目前不依 pool 過濾 |
| VM Catalog Result | `/templates` records | Template Catalog 映射 | CPU、Memory、Disk 作為 VM 建立預設資訊 |
| Sync VM Inventory Input | Cloud Connection、cluster_ref | 同步 VM 清單 | `/vms` records 依 cluster_ref 過濾 |
| VM Inventory Result | VM inventory record | VM 映射 | power_state 轉 HCM 狀態 |
| Allocation Extension Input / Result | 不適用 | Provider 能力支援矩陣 | vSphere Allocation 使用 HCM 共通 Project/System/Quota/Subnet |
| Create VM Input / Result | 不適用 | Provider 能力支援矩陣 | 目前專案未接 vSphere 建立 VM 能力 |
| VM Power Input / Result | 不適用 | Provider 能力支援矩陣 | 目前專案未接 vSphere 開關機能力 |
| VM Status Input / Result | 不適用 | Provider 能力支援矩陣 | 目前專案未接單台 VM 狀態追蹤能力 |

## 3. Auth 與連線設定

vSphere Connection 指向 HCM 可呼叫的 vSphere 介接 API，而不是讓前端直接連 vCenter。介接 API 內部可透過 pyVmomi 或其他既有機制讀取 vSphere inventory。

| 欄位 | 業務意義 | 必填 | 範例 | 備註 |
| --- | --- | --- | --- | --- |
| Cloud | 選擇 vSphere Provider | 是 | vSphere |
| Label | HCM 內顯示的連線名稱 | 是 | vSphere Taipei |
| Base URL | vSphere 介接 API Endpoint | 是 | `https://vsphere-api.example.com` | 可填 root 或 health/clusters 等 endpoint，HCM 會視為 root |
| Pool Filter | 限制同步 Cluster / Resource Pool 範圍 | 否 | `domain-c1` | 多 cluster 情境可使用 |
| Auth Type | 授權方式 | 是 | Token | vSphere 使用 Token |
| Token | 介接 API token | 是 | `<masked>` | 必須遮罩 |

## 4. Provider 能力支援矩陣

| 能力 | 是否支援 | Provider 資料來源 | 產生/更新的 HCM 資料 |
| --- | --- | --- | --- |
| 授權/登入 | 支援 | 介接 API `/health` | Cloud Connection 授權狀態 |
| 同步資源池 | 支援 | `/clusters` | Pool |
| 同步網路 | 支援 | `/networks` | Subnet |
| 同步安全群組 | 不支援 | vSphere 目前未納入 HCM Security Group 同步 | 無 |
| 同步 VM 規格來源 | 支援 | `/templates` | Pool VM Catalog |
| 同步 VM 清單 | 支援 | `/vms` | VM |
| 建立 Allocation 附加資源 | 不支援 | 無 | 無 |
| 建立 VM | 不支援 | 目前專案未接 vSphere 建立 VM API | 無 |
| VM 開機/關機 | 不支援 | 目前專案未接 vSphere power API | 無 |
| VM 狀態追蹤 | 不支援 | 目前專案未接單台 VM poll | 無 |

## 5. Provider 到 HCM 欄位映射

### 5.1 Cluster / Pool 映射

| Provider 欄位/概念 | Provider 意義 | HCM 欄位 | HCM 意義 | 轉換規則 | 範例 |
| --- | --- | --- | --- | --- | --- |
| cluster_ref | Cluster 識別 | Pool cloud_ref | Provider 來源識別 | 原樣保留 | `domain-c1` |
| name | Cluster 名稱 | Pool name | HCM 顯示名稱 | 原樣帶入 | `Cluster-Prod` |
| total_cpu_mhz | CPU 總量 | Pool CPU total | 可配置 CPU | 原始 MHz 保留，同步後顯示可換算 Core | `80000 MHz` |
| used_cpu_mhz | CPU 已用 | Pool CPU used | 已使用 CPU | 無欄位時以 0 表示 | `24000 MHz` |
| total_memory_gib | Memory 總量 | Pool Memory total | GB | GiB 視為 GB 顯示 | `512 GB` |
| used_memory_gib | Memory 已用 | Pool Memory used | GB | GiB 視為 GB 顯示 | `200 GB` |
| total_storage_gib / free_storage_gib | Storage 容量 | Pool Disk | GB / TB | 若 total 為 0 代表來源未提供完整值 | `1024 GB free` |

### 5.2 Network / Subnet 映射

| Provider 欄位/概念 | Provider 意義 | HCM 欄位 | HCM 意義 | 轉換規則 | 範例 |
| --- | --- | --- | --- | --- | --- |
| network_ref | Network / Portgroup 識別 | Subnet cloud_ref | Provider 來源識別 | 原樣保留 | `network-100` |
| name | Network 名稱 | Subnet name | HCM 顯示名稱 | 原樣帶入 | `192.168.1.X` |
| name pattern | 可推導網段名稱 | Subnet cidr / gateway | 網段資訊 | `192.168.1.X` -> `192.168.1.0/24`，gateway -> `.254` | `192.168.1.0/24` |
| cluster_refs | 可用 Cluster | Subnet pool relation | 可用 Pool | 對應 HCM Pool cloud_ref | `domain-c1` |
| cluster_names | Cluster 顯示名稱 | sync_meta | 同步摘要 | 保留供管理者確認 | `Cluster-Prod` |

### 5.3 Template Catalog 映射

| Provider 欄位/概念 | Provider 意義 | HCM 欄位 | HCM 意義 | 轉換規則 | 範例 |
| --- | --- | --- | --- | --- | --- |
| vm_ref | Template 識別 | Template id | VM 建立來源選項 | 原樣保留 | `vm-100` |
| name | Template 名稱 | Template label | 使用者可讀名稱 | 原樣帶入 | `Ubuntu-22.04-template` |
| cpu_count | 預設 CPU | defaultVcpu | 建立 VM 預設值 | 原樣帶入 | `2` |
| memory_gib | 預設 Memory | defaultRamGb | 建立 VM 預設值 | GiB 視為 GB | `4 GB` |
| disk_total_gib | 預設 Disk | defaultDiskGb | 建立 VM 預設值 | GiB 視為 GB | `50 GB` |

### 5.4 VM 映射

| Provider 欄位/概念 | Provider 意義 | HCM 欄位 | HCM 意義 | 轉換規則 | 範例 |
| --- | --- | --- | --- | --- | --- |
| vm_ref | VM 識別 | VM cloud_ref | Provider VM 識別 | 原樣保留 | `vm-200` |
| name | VM 名稱 | VM name / hostname | HCM 顯示名稱 | 原樣帶入 | `web-01` |
| cluster_ref | 所屬 Cluster | VM pool relation | 歸屬 Pool | 對應 Pool cloud_ref | `domain-c1` |
| power_state | 電源狀態 | VM status | HCM 狀態 | poweredOn -> RUNNING；其他 -> STOPPED | `poweredOn` |
| cpu_count | VM CPU | VM CPU | vCPU | 原樣帶入 | `4` |
| memory_gib | VM Memory | VM Memory | GB | GiB 視為 GB | `8 GB` |
| disk_total_gib / disks | VM Disk | VM Disk | GB | GiB 視為 GB | `100 GB` |
| primary_ip / ipv4s | VM IP | VM IP | 主要 IP | 優先取 primary_ip | `192.168.1.100` |
| nics | VM NIC | VM NIC | 網卡與網路 | 解析 network_ref、network_name、ipv4s | `network-100` |

## 6. 功能畫面差異

| 功能 | 畫面/流程位置 | 畫面差異 | 欄位/選項差異 | 可操作能力 | 業務限制/提示 |
| --- | --- | --- | --- | --- | --- |
| Cloud Settings | Connection 表單 | 顯示 vSphere 介接 API 設定 | Base URL、Pool Filter、Token | 可建立/更新連線 | Token 必須遮罩 |
| Cloud Settings | 同步資料 Wizard | Pool 為 Cluster；Template 為 vSphere Template | 同步 Cluster、Network、Template、VM | 可同步資料 | Network CIDR 可能需人工補齊 |
| Allocation Management | Shared Allocation 表單 | 使用共通欄位 | Project、System、Quota、Subnet | 不建立 Provider 附加資源 | 不顯示 Provider Extension |
| VM Management | VM 清單 | 顯示同步回來的 vSphere VM | power_state、IP、Template 來源 | 目前以檢視同步資料為主 | 目前不支援由 HCM 建立/開關 vSphere VM |

## 7. 外部 API 上下行與範例

本章列出 vSphere Provider Plugin 需要呼叫的外部 API。URL 皆以 Connection 表單的 `Base URL` 清理後作為 root，例如 `{baseUrl} = https://vsphere-api.example.com`。所有 request 皆需帶入授權 header：

| Header | 值 | 說明 |
| --- | --- | --- |
| Authorization | `Bearer <masked>` | vSphere 介接 API token，一律遮罩 |
| Content-Type | `application/json` | JSON API 使用 |

### 7.1 授權/登入

| 項目 | 說明 |
| --- | --- |
| Method | GET |
| URL | `{baseUrl}/health` |
| 上行 | Header `Authorization: Bearer <masked>` |
| 下行 | HTTP 200 即視為可用 |
| HCM 取用 | Connection 授權狀態 |

```http
GET {baseUrl}/health
Authorization: Bearer <masked>
```

### 7.2 同步 Cluster / Pool

| 順序 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 1 | GET | `{baseUrl}/clusters` | 取得 Cluster / Resource Pool 清單與容量 | 無 body | `records[].cluster_ref`、`name`、`total_cpu_mhz`、`used_cpu_mhz`、`total_memory_gib`、`used_memory_gib`、`total_storage_gib`、`free_storage_gib` |

```json
{
  "records": [
    {
      "cluster_ref": "domain-c1",
      "name": "Cluster-Prod",
      "total_cpu_mhz": 80000,
      "used_cpu_mhz": 24000,
      "total_memory_gib": 512,
      "used_memory_gib": 200,
      "total_storage_gib": 0,
      "free_storage_gib": 1024
    }
  ]
}
```

| 標準輸出欄位（Pool Sync Result） | Provider 欄位 | 重要備註 |
| --- | --- | --- |
| `provider_pool_id` | `cluster_ref` | 必填；vSphere Cluster reference |
| `name` | `name` | 必填；Cluster 名稱 |
| `cpu_total` | `total_cpu_mhz` | 單位為 MHz；需視 HCM 設定決定是否轉 Core |
| `cpu_provisioned` | `used_cpu_mhz` | 單位為 MHz；已使用的 CPU |
| `memory_total_gb` | `total_memory_gib` | 單位 GiB；視為 GB 顯示 |
| `memory_provisioned_gb` | `used_memory_gib` | 單位 GiB；已使用的 Memory |
| `disk_total_gb` | `total_storage_gib` | 單位 GiB；視為 GB 顯示 |
| `disk_provisioned_gb` | `total_storage_gib - free_storage_gib` | 已使用的 Storage 容量 |
| `ref.id` | `cluster_ref`（如 `domain-c1`） | 重複同步識別既有 Pool |
| `ref.name` | Cluster 名稱 | 備援識別 |
| `ref.cloud_connection_id` | Connection ID | 隔離不同連線的 Pool |
| `ref.sync_meta.cpu_mhz_per_core` | MHz → Core 換算基準 | vSphere 使用 MHz；換算 Core 顯示時需要此值 |
| `ref.sync_meta.cpu_total_mhz` | `total_cpu_mhz` | 保留原始 MHz 值供換算調整 |
| `ref.sync_meta.cpu_used_mhz` | `used_cpu_mhz` | 保留原始 MHz 值供換算調整 |

### 7.3 同步 Network

| 順序 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 1 | GET | `{baseUrl}/networks` | 取得 Network / Portgroup | 無 body | `records[].network_ref`、`name`、`cluster_refs`、`cluster_names` |

```json
{
  "records": [
    {
      "network_ref": "network-100",
      "name": "192.168.1.X",
      "cluster_refs": ["domain-c1"],
      "cluster_names": ["Cluster-Prod"]
    }
  ]
}
```

| 標準輸出欄位（Network Sync Result） | Provider 欄位 | 重要備註 |
| --- | --- | --- |
| `provider_network_id` | `network_ref` | 必填；vSphere Network reference |
| `name` | `name` | 必填；Network 名稱（通常為 IP pattern，如 `192.168.1.X`） |
| `cidr` | `name` 推導 | 由 Network 名稱推導 CIDR；如 `192.168.1.X` 推導 `192.168.1.0/24` |
| `gateway` | `name` 推導 | 由 CIDR 推導 gateway；如 `192.168.1.0/24` 推導 `.254` |
| `owner_pool_ids` | `cluster_refs[]` | 必填；Network 隸屬的 Cluster 清單 |
| `ref.id` | `network_ref`（如 `network-100`） | 重複同步識別既有 Subnet |
| `ref.name` | Network 名稱 | 備援識別 |
| `ref.subnet_idx` | `1`（固定） | vsphere driver 固定回傳 1；vSphere Network 為單一 subnet |
| `ref.owner_ref` | 不寫入（undefined） | vsphere driver 不設定 owner_ref；cluster 歸屬由 pool_ref_ids 決定 |
| `ref.pool_ref_ids` | `cluster_refs[]` | 追蹤 Network 隸屬的所有 Cluster；透過 targetPoolRefIds 路徑寫入 |
| `ref.cloud_connection_id` | Connection ID | 隔離不同連線的 Subnet |

### 7.4 同步 Template

| 順序 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 1 | GET | `{baseUrl}/templates` | 取得 vSphere Template | 無 body | `records[].vm_ref`、`name`、`cpu_count`、`memory_gib`、`disk_total_gib` |

```json
{
  "records": [
    {
      "vm_ref": "vm-100",
      "name": "Ubuntu-22.04-template",
      "cpu_count": 2,
      "memory_gib": 4,
      "disk_total_gib": 50
    }
  ]
}
```

| 標準輸出欄位（VM Catalog Result） | Provider 欄位 | 重要備註 |
| --- | --- | --- |
| `template_id` | `vm_ref` | 必填；vSphere Template reference |
| `name` | `name` | 必填；Template 名稱 |
| `default_cpu` | `cpu_count` | Template 預設 vCPU 數 |
| `default_memory_gb` | `memory_gib` | 單位 GiB；視為 GB 顯示 |
| `default_disk_gb` | `disk_total_gib` | 單位 GiB；視為 GB 顯示 |

### 7.5 同步 VM 清單

| 順序 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 1 | GET | `{baseUrl}/vms` | 取得 vSphere VM inventory | 無 body | `records[].vm_ref`、`name`、`cluster_ref`、`power_state`、`cpu_count`、`memory_gib`、`disk_total_gib`、`primary_ip`、`ipv4s`、`nics`、`disks` |

```json
{
  "records": [
    {
      "vm_ref": "vm-200",
      "name": "web-server-01",
      "cluster_ref": "domain-c1",
      "power_state": "poweredOn",
      "cpu_count": 4,
      "memory_gib": 8,
      "disk_total_gib": 100,
      "primary_ip": "192.168.1.100",
      "ipv4s": ["192.168.1.100"],
      "nics": [
        {
          "label": "Network adapter 1",
          "network_ref": "network-100",
          "network_name": "192.168.1.X",
          "ipv4s": ["192.168.1.100"]
        }
      ],
      "disks": [
        {
          "label": "Hard disk 1",
          "capacity_gib": 100
        }
      ]
    }
  ]
}
```

| 標準輸出欄位（VM Inventory Result） | Provider 欄位 | 重要備註 |
| --- | --- | --- |
| `provider_vm_id` | `records[].vm_ref` | 必填；作為 provider 端 VM 識別，後續追蹤使用 |
| `name` | `records[].name` | 必填；直接作為 VM 顯示名稱 |
| `status` | `records[].power_state` | 必填；需先轉成 HCM 標準狀態（如 `poweredOn` -> `running`） |
| `cpu` | `records[].cpu_count` | Core / vCPU |
| `memory_gb` | `records[].memory_gib` | GiB 視為 GB 顯示 |
| `disk_gb` | `records[].disk_total_gib` | GiB 視為 GB 顯示 |
| `ip` | `records[].primary_ip` | 主要 IP |
| `hostname` | `records[].name` 或 guest hostname（若有） | 無獨立 hostname 欄位時可回退名稱 |
| `nics` | `records[].nics` | 解析 network 名稱與 `ipv4s` |
| `disks` | `records[].disks` | 取各 disk 容量 |
| `tags` | vSphere custom attributes（若有） | 無值可留空 |
| `ref.id` | `vm_ref`（如 `vm-200`） | 重複同步識別既有 VM；避免重複建立 |
| `ref.name` | `name` | 備援識別 |
| `ref.cloud_connection_id` | Connection ID | 隔離不同連線的 VM |

## 8. 單位換算與狀態映射

| 項目 | vSphere 單位 / 狀態 | HCM 表示 | 規則 |
| --- | --- | --- | --- |
| CPU | MHz / cpu_count | Core / vCPU | Cluster CPU 保留 MHz；VM CPU 使用 cpu_count |
| Memory | GiB | GB | GiB 視為 GB 顯示 |
| Storage | GiB | GB / TB | GiB 視為 GB，總覽可轉 TB |
| Network name | `192.168.1.X` | CIDR | 推導為 `192.168.1.0/24` |
| Gateway | CIDR `.0/24` | gateway | 推導 `.254` |
| power_state | poweredOn | RUNNING / running | 執行中 |
| power_state | 其他 | STOPPED / stopped | 非 poweredOn 先視為停止 |

## 9. 限制與待確認

| 項目 | 說明 |
| --- | --- |
| vSphere 建立 VM | 目前專案未接 vSphere 建立 VM API，文件不列為支援 |
| vSphere 開關機 | 目前專案未接 vSphere power API，文件不列為支援 |
| Security Group | vSphere 目前不支援 HCM Security Group 同步 |
| Network CIDR | CIDR 目前可能由名稱推導，推導不到時需管理者補齊 |
| Storage total | 介接資料可能未提供完整 total_storage_gib，需以來源可用資料呈現 |
