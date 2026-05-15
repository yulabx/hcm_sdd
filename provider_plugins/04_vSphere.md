# 04 vSphere Provider Plugin

## 1. Provider 基本資訊

vSphere Provider Plugin 用於透過既有 vSphere 介接 API 取得 Cluster、Network、Template 與 VM inventory，轉成 HCM 標準資料供 Cloud Settings 與 VM Management 查詢使用。

HCM 中的 Cloud / Provider ID 可以 be 業務命名；Driver ID 固定為 `vsphere`，用來套用本文件定義的 Provider Plugin 規格。

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

| 02 規範掛載點 | vSphere 是否支援 | vSphere 文件章節 | 標準輸入 | 標準輸出 | 外部介接章節 | 影響的 HCM 資料 |
| --- | --- | --- | --- | --- | --- | --- |
| Provider 授權 | 支援 | Auth 與連線設定 | Connection Auth Input | Auth Result | 5.1 授權/登入 | Cloud Connection |
| 同步資源池 | 支援 | Cluster / Pool 映射 | Sync Pools Input | Pool Sync Result | 5.2 同步 Cluster / Pool | Pool |
| 同步網路 | 支援 | Network / Subnet 映射 | Sync Network Input | Network Sync Result | 5.3 同步 Network | Subnet |
| 同步 VM 規格來源 | 支援 | Template Catalog 映射 | Sync VM Catalog Input | VM Catalog Result | 5.4 同步 Template | Pool VM Catalog |
| 同步 VM 清單 | 支援 | VM 映射、狀態映射 | Sync VM Inventory Input | VM Inventory Result | 5.5 同步 VM 清單 | VM |
| Allocation 附加資源 | 不支援 | 功能畫面差異 | Allocation Extension Input | Allocation Extension Result | 不支援 | 無 |
| 建立 VM | 不支援 | 功能畫面差異 | Create VM Input | Create VM Result | 不支援 | 無 |
| VM 開機 | 不支援 | 功能畫面差異 | VM Power Input | VM Power Result | 不支援 | 無 |
| VM 關機 | 不支援 | 功能畫面差異 | VM Power Input | VM Power Result | 不支援 | 無 |
| VM 狀態追蹤 | 不支援 | 功能畫面差異 | VM Status Input | VM Status Result | 不支援 | 無 |

### 2.2 標準輸入/輸出落地表

| 02 標準資料 | vSphere 對應欄位/概念 | Provider 文件章節 | 轉換或限制 |
| --- | --- | --- | --- |
| Connection Auth Input | Base URL、Token、Pool Filter | Auth 與連線設定 | Token 以 Bearer Token 呼叫介接 API |
| Auth Result | 介接 API health 成功狀態 | 授權/登入 | GET `/health` 成功代表可連線 |
| Sync Pools Input | Cloud Connection、cluster scope | 同步 Cluster / Pool | vSphere 以 Cluster 作為 HCM Pool |
| Pool Sync Result | `/clusters` records | Cluster / Pool 映射 | 需對齊持久層基礎單位 (Bytes/Millicores) |
| Sync Network Input | Cloud Connection、cluster_ref | 同步 Network | 可依 cluster_refs 過濾 |
| Network Sync Result | `/networks` records | Network / Subnet 映射 | Network name 可推導 CIDR，不足時需人工補齊 |
| Sync VM Catalog Input | Cloud Connection、cluster_ref | 同步 Template | Template 目前不依 pool 過濾 |
| VM Catalog Result | `/templates` records | Template Catalog 映射 | 需對齊基礎單位 (Bytes/Millicores) |
| Sync VM Inventory Input | Cloud Connection、cluster_ref | 同步 VM 清單 | `/vms` records 依 cluster_ref 過濾 |
| VM Inventory Result | VM inventory record | VM 映射 | 規格需對齊持久層基礎單位 (Bytes/Millicores) |
| Allocation Extension Input / Result | 不適用 | 功能畫面差異 | vSphere Allocation 使用 HCM 共通 Project/System/Quota/Subnet |
| Create VM Input / Result | 不適用 | 功能畫面差異 | 建立 VM 走既有 Terraform 流程，HCM 不直接呼叫 vSphere 建立 API |
| VM Power Input / Result | 不適用 | 功能畫面差異 | 目前專案未接 vSphere 開關機能力 |
| VM Status Input / Result | 不適用 | 功能畫面差異 | 目前專案未接單台 VM 狀態追蹤能力 |

## 3. Auth 與連線設定

vSphere Connection 指向 HCM 可呼叫的 vSphere 介接 API，而不是讓前端直接連 vCenter。介接 API 內部可透過 pyVmomi 或其他既有機制讀取 vSphere inventory。

授權畫面的顯示規則與事件行為，統一定義於 `4.1 Provider 授權差異區詳情`。

| 欄位 | 業務意義 | 必填 | 範例 | 備註 |
| --- | --- | --- | --- | --- |
| Cloud | 選擇 vSphere Provider | 是 | vSphere |
| Label | HCM 內顯示的連線名稱 | 是 | vSphere Taipei |
| Base URL | vSphere 介接 API Endpoint | 是 | `https://vsphere-api.example.com` | 可填 root 或 health/clusters 等 endpoint，HCM 會視為 root |
| Pool Filter | 限制同步 Cluster / Resource Pool 範圍 | 否 | `domain-c1` | 多 cluster 情境可使用 |
| Auth Type | 授權方式 | 是 | Token | vSphere 使用 Token |
| Token | 介接 API token | 是 | `<masked>` | 必須遮罩 |

## 4. 功能畫面差異

| 功能 | 畫面/流程位置 | 畫面差異 | 欄位/選項差異 | 可操作能力 | 業務限制/提示 |
| --- | --- | --- | --- | --- | --- |
| Cloud Settings | Connection 表單 | 顯示 vSphere 介接 API 設定 | Base URL、Pool Filter、Token | 可建立/更新連線 | Token 必須遮罩 |
| Cloud Settings | 同步資料 Wizard | Pool 為 Cluster；Template 為 vSphere Template | 同步 Cluster、Network、Template、VM | 可同步資料 | Network CIDR 可能需人工補齊 |
| Allocation Management | Shared Allocation 表單 | 使用共通欄位 | Project、System、Quota、Subnet | 不建立 Provider 附加資源 | 不顯示 Provider Extension |
| VM Management | VM 清單 | 顯示同步回來的 vSphere VM | power_state、IP、Template 來源 | 目前以檢視同步資料為主 | 建立 VM 走既有 Terraform 流程；HCM 不直接提供 vSphere 建立/開關機 |

### 4.1 Provider 授權差異區詳情

| Auth Type | 顯示欄位/區塊 | 主要操作 | 畫面輸出 | 備註 |
| --- | --- | --- | --- | --- |
| `token` | 不顯示授權差異區 | 儲存 Connection 後直接進同步 | 無額外授權畫面 | Token 可用性由 `/health` 回應判斷 |

#### 4.1.1 token

**畫面示意圖**

```text
+--------------------------------------------------------------+
| Cloud Settings / 編輯 Connection                              |
+--------------------------------------------------------------+
| Provider: vSphere                                              |
| Label: vSphere Taipei                                          |
| Base URL: https://vsphere-api.example.com                     |
| Pool Filter: domain-c1 (optional)                             |
| Auth Type: token                                               |
| Token: ********                                                |
+--------------------------------------------------------------+
| Provider 授權差異區：不顯示                                      |
| 說明：儲存後直接以 Bearer Token 呼叫 vSphere 介接 API           |
+--------------------------------------------------------------+
```

| Event/動作 | 觸發 | 成功結果 | 失敗結果 |
| --- | --- | --- | --- |
| 儲存 Connection | 點擊儲存 | 進入可同步狀態 | 顯示連線建立失敗訊息 |
| 執行授權驗證 | 同步前觸發 `GET /health` | 健康檢查成功，進入後續流程 | 顯示 token 無效或 API 不可用訊息 |

## 5. 外部介接上下行與範例

本章列出 vSphere Provider Plugin 需要呼叫的外部 API。URL 皆以 Connection 表單的 `Base URL` 清理後作為 root，例如 `{baseUrl} = https://vsphere-api.example.com`。所有 request 皆需帶入授權 header：

| Header | 值 | 說明 |
| --- | --- | --- |
| Authorization | `Bearer <masked>` | vSphere 介接 API token，一律遮罩 |
| Content-Type | `application/json` | JSON API 使用 |

### 5.1 授權/登入

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

### 5.2 同步 Cluster / Pool

#### 5.2.1 API 清單

| 順序 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 1 | GET | `{baseUrl}/clusters` | 取得 Cluster / Resource Pool 清單與容量 | 無 body | `records[].cluster_ref`、`name`、`total_cpu_mhz`、`used_cpu_mhz`、`total_memory_bytes`、`used_memory_bytes`、`total_storage_bytes`、`free_storage_bytes`、`used_storage_bytes`、`provisioned_storage_bytes` |

#### 5.2.2 上下行範例

**上行 Request 範例：**

```http
GET {baseUrl}/clusters
Authorization: Bearer <masked>
Accept: application/json
```

**下行 Response 範例：**

```json
{
  "records": [
    {
      "cluster_ref": "domain-c1",
      "name": "Cluster-Prod",
      "total_cpu_mhz": 80000,
      "used_cpu_mhz": 24000,
      "total_memory_bytes": 549755813888,
      "used_memory_bytes": 137438953472,
      "total_storage_bytes": 2199023255552,
      "free_storage_bytes": 1099511627776,
      "used_storage_bytes": 1099511627776,
      "uncommitted_storage_bytes": 412316860416,
      "provisioned_storage_bytes": 1511828488192
    }
  ]
}
```

#### 5.2.3 掛載點輸出

| 標準輸出欄位（Pool Sync Result） | Provider 欄位 | 重要備註 |
| --- | --- | --- |
| `provider_pool_id` | `cluster_ref` | 必填；vSphere Cluster reference |
| `name` | `name` | 必填；Cluster 名稱 |
| `cpu_total_millicores` | `total_cpu_mhz` | 單位為 MHz；以 `cpu_mhz_per_core` 換算為 Millicores |
| `cpu_used_millicores` | `used_cpu_mhz` | vSphere 實際使用 CPU；單位為 MHz，換算為 Millicores |
| `cpu_provisioned_millicores` | `/vms records[].cpu_count` 加總 | VM 已配置 vCPU 加總；`cpu_count * 1000` |
| `mem_total_bytes` | `total_memory_bytes` | 原始單位即為 **Bytes** |
| `mem_used_bytes` | `used_memory_bytes` | vSphere 實際使用 Memory；介接服務轉出的 **Bytes** |
| `mem_provisioned_bytes` | `/vms records[].memory_bytes` 加總 | VM 已配置 Memory 加總 |
| `disk_total_bytes` | `total_storage_bytes` | 原始單位即為 **Bytes** |
| `disk_used_bytes` | `used_storage_bytes` | vSphere 實際已使用 Storage；若來源缺值，HCM fallback 為 `total_storage_bytes - free_storage_bytes` |
| `disk_provisioned_bytes` | `provisioned_storage_bytes` | VM 已配置 / 已承諾 Storage；可大於 `disk_total_bytes` |
| `ref.id` | `cluster_ref`（如 `domain-c1`） | 重複同步識別既有 Pool |
| `ref.name` | Cluster 名稱 | 備援識別 |
| `ref.cloud_connection_id` | Connection ID | 隔離不同連線的 Pool |
| `ref.sync_meta.cpu_mhz_per_core` | MHz → Core 換算基準 | vSphere 使用 MHz；換算 Core 顯示時需要此值 |
| `ref.sync_meta.cpu_total_mhz` | `total_cpu_mhz` | 保留原始 MHz 值供換算調整 |
| `ref.sync_meta.cpu_used_mhz` | `used_cpu_mhz` | 保留原始 MHz 值供換算調整 |
| `ref.sync_meta.cpu_mhz_per_core` | HCM 手動校正值優先；首次同步預設 `2000` | vSphere 介接 API 不回報 MHz/Core 時，Admin 可在 Cloud Initialization 手動輸入；後續同步不得覆蓋回預設值 |

> UI 資源條顯示規則：Pool bar 支援「已申請」與「使用量」兩種視角。已申請視角使用 `*_provisioned_* / *_total_*`，vSphere 的 CPU / Memory provisioned 由 VM 清單加總，Storage provisioned 使用 `/clusters` 的 `provisioned_storage_bytes`；使用量視角使用 `*_used_* / *_total_*`，來源為 `/clusters` 的 CPU、Memory、Storage used 欄位。vSphere 已申請量可能因 overcommit 大於總量，UI 需保留真實百分比並將 bar 寬度 capped 在 100%。

### 5.3 同步 Network

#### 5.3.1 API 清單

| 順序 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 1 | GET | `{baseUrl}/networks` | 取得 Network / Portgroup | 無 body | `records[].network_ref`、`name`、`cluster_refs`、`cluster_names` |

#### 5.3.2 上下行範例

**上行 Request 範例：**

```http
GET {baseUrl}/networks
Authorization: Bearer <masked>
Accept: application/json
```

**下行 Response 範例：**

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

#### 5.3.3 掛載點輸出

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

### 5.4 同步 Template

#### 5.4.1 API 清單

| 順序 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 1 | GET | `{baseUrl}/templates` | 取得 vSphere Template | 無 body | `records[].vm_ref`、`name`、`cpu_count`、`memory_gib`、`disk_total_gib` |

#### 5.4.2 上下行範例

**上行 Request 範例：**

```http
GET {baseUrl}/templates
Authorization: Bearer <masked>
Accept: application/json
```

**下行 Response 範例：**

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

#### 5.4.3 掛載點輸出

| 標準輸出欄位（VM Catalog Result） | Provider 欄位 | 重要備註 |
| --- | --- | --- |
| `template_id` | `vm_ref` | 必填；vSphere Template reference |
| `name` | `name` | 必填；Template 名稱 |
| `default_cpu` | `cpu_count` | 需對齊基礎單位 (Millicores) |
| `default_memory_bytes` | `memory_bytes` | 介接服務轉出的 **Bytes** |
| `default_disk_bytes` | `disk_total_bytes` | 原始單位即為 **Bytes** |

### 5.5 同步 VM 清單

#### 5.5.1 API 清單

| 順序 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 1 | GET | `{baseUrl}/vms` | 取得 vSphere VM inventory | 無 body | `records[].vm_ref`、`name`、`cluster_ref`、`power_state`、`cpu_count`、`memory_gib`、`disk_total_gib`、`primary_ip`、`ipv4s`、`nics`、`disks` |

#### 5.5.2 上下行範例

**上行 Request 範例：**

```http
GET {baseUrl}/vms
Authorization: Bearer <masked>
Accept: application/json
```

**下行 Response 範例：**

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

#### 5.5.3 掛載點輸出

| 標準輸出欄位（VM Inventory Result） | Provider 欄位 | 重要備註 |
| --- | --- | --- |
| `provider_vm_id` | `records[].vm_ref` | 必填；作為 provider 端 VM 識別，後續追蹤使用 |
| `name` | `records[].name` | 必填；直接作為 VM 顯示名稱 |
| `status` | `records[].power_state` | 必填；需先轉成 HCM 標準狀態（如 `poweredOn` -> `running`） |
| `cpu` | `records[].cpu_count` | 需對齊基礎單位 (Millicores) |
| `ram` | `records[].memory_bytes` | 介接服務轉出的 **Bytes** |
| `disk` | `records[].disk_total_bytes` | 原始單位即為 **Bytes** |
| `ip` | `records[].primary_ip` | 主要 IP |
| `hostname` | `records[].name` 或 guest hostname（若有） | 無獨立 hostname 欄位時可回退名稱 |
| `nics` | `records[].nics` | 解析 network 名稱與 `ipv4s` |
| `disks` | `records[].disks` | 取各 disk 容量 |
| `tags` | vSphere custom attributes（若有） | 無值可留空 |
| `ref.id` | `vm_ref`（如 `vm-200`） | 重複同步識別既有 VM；避免重複建立 |
| `ref.name` | `name` | 備援識別 |
| `ref.cloud_connection_id` | Connection ID | 隔離不同連線的 VM |

## 6. 單位換算與狀態映射

| 項目 | vSphere 單位 / 狀態 | HCM 表示 | 規則 |
| --- | --- | --- | --- |
| CPU | MHz / cpu_count | **Millicores** | Cluster CPU 依每 core MHz 換算；VM CPU 依 cpu_count 乘以 1000 |
| Memory | GiB / Bytes | **Bytes** | 原始 Bytes 直送；GiB 依二進位 ($1024^3$) 換算 |
| Storage | GiB / Bytes | **Bytes** | 原始 Bytes 直送；GiB 依二進位 ($1024^3$) 換算 |
| Network name | `192.168.1.X` | CIDR | 推導為 `192.168.1.0/24` |
| Gateway | CIDR `.0/24` | gateway | 推導 `.254` |
| power_state | poweredOn | RUNNING / running | 執行中 |
| power_state | 其他 | STOPPED / stopped | 非 poweredOn 先視為停止 |

## 7. 限制與待確認

| 項目 | 說明 |
| --- | --- |
| vSphere 建立 VM | 依規格決議改走既有 Terraform 流程；HCM 不直接呼叫 vSphere 建立 VM API |
| vSphere 開關機 | 目前專案未接 vSphere power API，文件不列為支援 |
| Security Group | vSphere 目前不支援 HCM Security Group 同步 |
| Network CIDR | CIDR 目前可能由名稱推導，推導不到時需管理者補齊 |
| Storage total | 介接資料以 `total_storage_bytes` 為準；若來源無法完整計算 datastore capacity，需以介接服務可取得資料呈現 |

## 8. 附錄：vSphere Inventory 介接服務規格 (Integration SDK Specification)

vSphere 介接服務負責將 vCenter 原始的 SOAP 介面轉化為 HCM 易於處理的 JSON REST API。本章節定義其 API 呼叫範例、底層 SDK 介接邏輯與資料映射規格。

### 8.1 介接呼叫範例 (Integration API Examples)

本節展示 HCM Backend 呼叫介接服務取得四類核心資源的標準 JSON 範例。這是開發者對接時的主要參考依據。

#### 8.1.1 同步資源池 (Clusters)
- **Endpoint**: `GET /clusters`
- **Response Record:**
```json
{
  "cluster_ref": "domain-c1",
  "name": "Cluster-Prod",
  "total_cpu_mhz": 80000,
  "used_cpu_mhz": 24000,
  "total_memory_bytes": 549755813888,
  "used_memory_bytes": 137438953472,
  "total_storage_bytes": 2199023255552,
  "free_storage_bytes": 1099511627776,
  "used_storage_bytes": 1099511627776,
  "uncommitted_storage_bytes": 412316860416,
  "provisioned_storage_bytes": 1511828488192
}
```

#### 8.1.2 同步網路 (Networks)
- **Endpoint**: `GET /networks`
- **Response Record:**
```json
{
  "network_ref": "network-100",
  "name": "192.168.1.X",
  "type": "DistributedVirtualPortgroup",
  "vlan": 101,
  "cluster_refs": ["domain-c1"]
}
```

#### 8.1.3 同步虛擬機 (VMs)
- **Endpoint**: `GET /vms`
- **Response Record:**
```json
{
  "vm_ref": "vm-200",
  "name": "web-server-01",
  "power_state": "poweredOn",
  "cpu_count": 4,
  "memory_bytes": 8589934592,
  "primary_ip": "192.168.1.100",
  "disk_total_bytes": 107374182400,
  "nics": [{ "network_name": "192.168.1.X", "ipv4s": ["192.168.1.100"] }]
}
```

#### 8.1.4 同步範本 (Templates)
- **Endpoint**: `GET /templates`
- **Response Record:**
```json
{
  "vm_ref": "vm-450",
  "name": "ubuntu-22-template",
  "cpu_count": 2,
  "memory_bytes": 2147483648,
  "disk_total_bytes": 53687091200
}
```

### 8.2 連線生命週期與會話管理 (Connection & Session - SDK)

介接服務底層透過 `pyVmomi` 的 `SmartConnect` 與 vCenter 建立持久連線。

| 階段 | 實作邏輯 | 說明 |
|---|---|---|
| **身分認證** | **SOAP 帳密認證** | SDK 使用 `VCENTER_USER` 與 `VCENTER_PASSWORD` 向 vCenter 進行身分驗證。 |
| **初始化** | `SmartConnect(host, user, pwd, port, sslContext)` | 建立全域會話物件 (`ServiceInstance`)，作為所有檢索動作的入口。 |
| **安全性** | `ssl._create_unverified_context()` | 若 `VCENTER_INSECURE` 為 true，則忽略 SSL 憑證驗證（適用於測試環境）。 |
| **自動清理** | `atexit.register(Disconnect, si)` | 服務關閉時自動向 vCenter 登出並釋放 Session 資源。 |

### 8.3 資料檢索與映射規格 (Retrieval & Mapping - SDK Logic)

本節定義介接服務如何將 vSphere 原始物件屬性檢索並轉換為上述 API 欄位。

#### 8.3.1 檢索模式
為確保效能，採用 **ContainerView** 模式進行批次檢索：
- **指令**：`content.viewManager.CreateContainerView(rootFolder, [vim.Type], recursive=True)`。
- **目標**：`ClusterComputeResource` (資源池)、`Network` (網路)、`VirtualMachine` (VM/Template)。

#### 8.3.2 Cluster (資源池) 映射
- **關鍵邏輯**：
    - **Storage**：遍歷 `cluster.datastore` 成員，手動加總 `capacity`、`freeSpace`、`uncommitted`，並輸出實際使用與已配置量。

| API 欄位 | SDK 屬性路徑 | 說明 |
|---|---|---|
| `cluster_ref` | `_moId` | vSphere 內部唯一識別碼 |
| `total_cpu_mhz` | `summary.totalCpu` | 叢集總 CPU 頻率 |
| `used_cpu_mhz` | `summary.usageSummary.cpuUsageMHz` | 需啟用 DRS 始可精確統計 |
| `total_memory_bytes` | `summary.totalMemory` | 原始記憶體 Bytes |
| `used_memory_bytes` | `summary.usageSummary.memUsageMB` | 介接服務轉為 Bytes (MiB * 1024^2) |
| `total_storage_bytes` | `cluster.datastore[].summary.capacity` 加總 | Datastore 總容量 Bytes |
| `free_storage_bytes` | `cluster.datastore[].summary.freeSpace` 加總 | Datastore 剩餘容量 Bytes |
| `used_storage_bytes` | `total_storage_bytes - free_storage_bytes` | Datastore 實際已使用容量 Bytes |
| `uncommitted_storage_bytes` | `cluster.datastore[].summary.uncommitted` 加總 | Thin provisioning 尚未實際寫入但已承諾容量 Bytes |
| `provisioned_storage_bytes` | `used_storage_bytes + uncommitted_storage_bytes` | VM 已配置 / 已承諾 Storage Bytes |

#### 8.3.3 Network (網路) 映射
- **關鍵邏輯**：
    - **叢集歸屬**：遍歷 `host` 參照並透過 `host.parent` 向上溯源至 `vim.ClusterComputeResource`。

| API 欄位 | SDK 屬性路徑 | 說明 |
|---|---|---|
| `network_ref` | `_moId` | 網路物件識別碼 |
| `vlan` | `config.defaultPortConfig.vlan.vlanId` | 僅分散式交換機支援 |

#### 8.3.4 VM & Template (虛擬機與範本) 映射
- **關鍵邏輯**：
    - **類型區分**：依據 `config.template` (Boolean) 分流至 `/templates` (True) 或 `/vms` (False)。
    - **IP 取得**：讀取 `guest.ipAddress` (需安裝 VMware Tools)。

| API 欄位 | SDK 屬性路徑 | 說明 |
|---|---|---|
| `vm_ref` | `_moId` | 虛擬機識別碼 |
| `memory_bytes` | `summary.config.memorySizeMB` | 介接服務轉為 Bytes (MiB * 1024^2) |
| `disk_total_bytes` | `VirtualDisk.capacityInBytes` | 總磁碟容量 Bytes (Fallback to capacityInKB) |
| `is_template` | `config.template` | **辨識關鍵**：True 為範本，False 為一般 VM |

### 8.4 業務約束與限制 (Constraints)

- **VMware Tools 依賴**：虛擬機的詳細 IP 資訊高度依賴於 Guest OS 內安裝的 VMware Tools。
- **DRS 依賴**：資源池的「已使用資源」指標需在 vCenter 啟用 DRS 才能精確統計。
- **儲存總量限制**：由於 vSphere 未提供 Cluster 級別的總儲存容量，同步結果為物理加總。
