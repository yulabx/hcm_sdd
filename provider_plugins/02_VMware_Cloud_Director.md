# 02 VMware Cloud Director Provider Plugin

## 1. Provider 基本資訊

VMware Cloud Director Provider Plugin 用於將 VCD tenant 可見的 VDC、Org VDC Network、vApp Template 與 VM 轉成 HCM 標準資料，供 Cloud Settings、Apply、Allocation 與 VM Management 查詢使用。

HCM 中的 Cloud / Provider ID 可以是業務命名；Driver ID 固定為 `vmware-cloud-director`，用來套用本文件定義的 Provider Plugin 規格。

| 項目 | 說明 |
| --- | --- |
| Provider 名稱 | VMware Cloud Director |
| Provider 類型 | 私有雲 / 代管雲 |
| HCM Provider Driver | `vmware-cloud-director` |
| 主要資源概念 | VDC、Org VDC Network、vApp Template、VM |
| HCM 主要對應資料 | Cloud Connection、Pool、Subnet、Template Catalog、VM |
| 主要使用功能 | Cloud Settings、VM Management |

## 2. 共同規範勾稽

### 2.1 掛載點對照表

| 02 規範掛載點 | VCD 是否支援 | VCD 文件章節 | 標準輸入 | 標準輸出 | 外部 API 章節 | 影響的 HCM 資料 |
| --- | --- | --- | --- | --- | --- | --- |
| Provider 授權 | 支援 | Auth 與連線設定、Provider 能力支援矩陣 | Connection Auth Input | Auth Result | 授權/登入 | Cloud Connection |
| 同步資源池 | 支援 | VDC / Pool 映射 | Sync Pools Input | Pool Sync Result | 同步 VDC / Pool | Pool |
| 同步網路 | 支援 | Network / Subnet 映射 | Sync Network Input | Network Sync Result | 同步 Network | Subnet |
| 同步 VM 規格來源 | 支援 | Template Catalog 映射 | Sync VM Catalog Input | VM Catalog Result | 同步 vApp Template | Pool VM Catalog |
| 同步 VM 清單 | 支援 | VM 映射、狀態映射 | Sync VM Inventory Input | VM Inventory Result | 同步 VM 清單 | VM |
| Allocation 附加資源 | 不支援 | Provider 能力支援矩陣 | Allocation Extension Input | Allocation Extension Result | 不支援 | 無 |
| 建立 VM | 不支援 | Provider 能力支援矩陣 | Create VM Input | Create VM Result | 不支援 | 無 |
| VM 開機 | 不支援 | Provider 能力支援矩陣 | VM Power Input | VM Power Result | 不支援 | 無 |
| VM 關機 | 不支援 | Provider 能力支援矩陣 | VM Power Input | VM Power Result | 不支援 | 無 |
| VM 狀態追蹤 | 不支援 | Provider 能力支援矩陣 | VM Status Input | VM Status Result | 不支援 | 無 |

### 2.2 標準輸入/輸出落地表

| 02 標準資料 | VCD 對應欄位/概念 | Provider 文件章節 | 轉換或限制 |
| --- | --- | --- | --- |
| Connection Auth Input | Base URL、Basic、API Token、Service Account | Auth 與連線設定 | Service Account 需 tenant name、client id、device authorization 與 refresh token |
| Auth Result | Access Token、授權狀態、必要時 refresh token 更新 | 授權/登入 | Service Account 可能先回 pending，再由 poll 完成 |
| Sync Pools Input | Cloud Connection、VCD tenant scope | 同步 VDC / Pool | VCD 以 VDC 作為 HCM Pool |
| Pool Sync Result | VDC ComputeCapacity、StorageProfile | VDC / Pool 映射 | CPU 原始單位常為 MHz；Memory / Storage 轉成 GB/TB |
| Sync Network Input | Cloud Connection、VDC pool id | 同步 Network | 讀取 Org VDC Network；若 owner 為 VDC 則可依 pool 過濾 |
| Network Sync Result | Org VDC Network subnet values | Network / Subnet 映射 | Gateway + prefixLength 推導 CIDR |
| Sync VM Catalog Input | Cloud Connection、VDC id | 同步 vApp Template | 以 Query API 的 vAppTemplate record 作為 VM 建立來源選項 |
| VM Catalog Result | vApp Template | Template Catalog 映射 | HCM 顯示 template、default CPU、Memory、Disk |
| Sync VM Inventory Input | Cloud Connection、VDC id | 同步 VM 清單 | 以 Query API 列 VM，再讀 VM detail 補 NIC / Disk |
| VM Inventory Result | VMRecord + VM detail XML | VM 映射 | VM 狀態保留 VCD status，HCM 顯示時轉標準狀態 |
| Allocation Extension Input / Result | 不適用 | Provider 能力支援矩陣 | VCD Allocation 使用 HCM 共通 Project/System/Quota/Subnet |
| Create VM Input / Result | 不適用 | Provider 能力支援矩陣 | 目前專案未接 VCD 建立 VM 能力 |
| VM Power Input / Result | 不適用 | Provider 能力支援矩陣 | 目前專案未接 VCD 開關機能力 |
| VM Status Input / Result | 不適用 | Provider 能力支援矩陣 | 目前專案未接 VCD 單台 VM 狀態追蹤能力 |

## 3. Auth 與連線設定

VCD Connection 可使用三種授權方式：Basic、Token、Service Account。Service Account 會有外部互動授權流程，需在 Cloud Settings 的 Provider 授權差異區呈現 user code 與 verification URI。

| 欄位 | 業務意義 | 必填 | 範例 | 備註 |
| --- | --- | --- | --- | --- |
| Cloud | 選擇 VCD Provider | 是 | VCD |
| Label | HCM 內顯示的連線名稱 | 是 | VCD Taipei |
| Base URL | VCD API Endpoint | 是 | `https://vcd.example.com` | 不含尾端 slash |
| Auth Type | 授權方式 | 是 | basic / token / service_account | 依租戶授權方式選擇 |
| Username / Password | Basic 授權資料 | Basic 時必填 | `<masked>` | password 需遮罩 |
| API Token | 既有 token | Token 時必填 | `<masked>` | token 需遮罩 |
| Tenant Name | Service Account 所屬 tenant | Service Account 時必填 | `tenant-a` | 用於 tenant OAuth URL |
| Client ID | Service Account client id | Service Account 時必填 | `hcm-client` | 啟動 device authorization 使用 |
| Refresh Token | Service Account refresh token | 完成授權後必填 | `<masked>` | 可由授權流程取得 |

## 4. Provider 能力支援矩陣

| 能力 | 是否支援 | Provider 資料來源 | 產生/更新的 HCM 資料 |
| --- | --- | --- | --- |
| 授權/登入 | 支援 | CloudAPI session、Tenant OAuth | Cloud Connection 授權狀態 |
| 同步資源池 | 支援 | VDC | Pool |
| 同步網路 | 支援 | Org VDC Network | Subnet |
| 同步安全群組 | 不支援 | VCD 目前未納入 HCM Security Group 同步 | 無 |
| 同步 VM 規格來源 | 支援 | vApp Template | Pool VM Catalog |
| 同步 VM 清單 | 支援 | VM Query + VM detail XML | VM |
| 建立 Allocation 附加資源 | 不支援 | 無 | 無 |
| 建立 VM | 不支援 | 目前專案未接 VCD instantiate vApp | 無 |
| VM 開機/關機 | 不支援 | 目前專案未接 VCD power action | 無 |
| VM 狀態追蹤 | 不支援 | 目前專案未接單台 VM poll | 無 |

## 5. Provider 到 HCM 欄位映射

### 5.1 VDC / Pool 映射

| Provider 欄位/概念 | Provider 意義 | HCM 欄位 | HCM 意義 | 轉換規則 | 範例 |
| --- | --- | --- | --- | --- | --- |
| VDC id | VDC 識別 | Pool cloud_ref | Provider 來源識別 | URN tail 或 VDC id | `vdc-001` |
| VDC name | VDC 名稱 | Pool name | HCM 顯示名稱 | 原樣帶入 | `Prod VDC` |
| ComputeCapacity.Cpu.Limit | CPU 上限 | Pool CPU total | 可配置 CPU | 原始單位多為 MHz，HCM 顯示時依規則換算 Core | `80000 MHz` |
| ComputeCapacity.Cpu.Used | CPU 已用 | Pool CPU used | 已使用 CPU | 同 CPU total 換算 | `24000 MHz` |
| ComputeCapacity.Memory | Memory 容量 | Pool Memory | 記憶體容量 | MB 轉 GB | `512 GB` |
| VdcStorageProfile | Storage 容量 | Pool Disk | 磁碟容量 | MB 轉 GB/TB；unlimited 需標記 | `20 TB` |

### 5.2 Network / Subnet 映射

| Provider 欄位/概念 | Provider 意義 | HCM 欄位 | HCM 意義 | 轉換規則 | 範例 |
| --- | --- | --- | --- | --- | --- |
| Org VDC Network id | Network 識別 | Subnet cloud_ref | Provider 來源識別 | `{networkId}-subnet-{index}` | `urn:vcloud:network:001-subnet-1` |
| name | Network 顯示名稱 | Subnet name | HCM 顯示名稱 | 多 subnet 時加上 subnet 序號 | `APP-NET / subnet 1` |
| gateway + prefixLength | 網段資訊 | Subnet cidr / gateway | HCM 網段 | 由 gateway 與 prefixLength 推導 CIDR | `10.1.0.0/24` |
| ownerRef | 所屬 VDC / VDC Group | Subnet pool relation | 可用 Pool | owner 為 VDC 時可直接連到 Pool | `vdc-001` |
| status | Network 狀態 | Subnet status | 是否可用 | REALIZED -> active；其他 -> inactive | `active` |

### 5.3 Template Catalog 映射

| Provider 欄位/概念 | Provider 意義 | HCM 欄位 | HCM 意義 | 轉換規則 | 範例 |
| --- | --- | --- | --- | --- | --- |
| VAppTemplateRecord href/id | Template 識別 | Template id | VM 建立來源選項 | href tail 作為 cloud_ref | `vappTemplate-001` |
| name | Template 名稱 | Template label | 使用者可讀名稱 | 原樣帶入 | `Ubuntu 22.04` |
| numberOfCpus | 預設 CPU | defaultVcpu | 建立 VM 預設值 | 數字帶入 | `2` |
| memoryAllocationMB | 預設 Memory | defaultRamGb | 建立 VM 預設值 | MB 轉 GB | `4 GB` |
| storageKB | 預設 Disk | defaultDiskGb | 建立 VM 預設值 | KB 轉 GB | `50 GB` |

### 5.4 VM 映射

| Provider 欄位/概念 | Provider 意義 | HCM 欄位 | HCM 意義 | 轉換規則 | 範例 |
| --- | --- | --- | --- | --- | --- |
| VMRecord href/id | VM 識別 | VM cloud_ref | Provider VM 識別 | href tail 作為 id | `vm-001` |
| name | VM 名稱 | VM name / hostname | HCM 顯示名稱 | detail XML 可補 hostname | `app-01` |
| status | VCD VM 狀態 | VM status | HCM 狀態 | provider 文件保留原值並由 HCM 顯示轉換 | `POWERED_ON` |
| numberOfCpus | VM CPU | VM CPU | vCPU | 數字帶入 | `4` |
| memoryMB | VM Memory | VM Memory | GB | MB 轉 GB | `16 GB` |
| totalStorageAllocatedMb / disks | VM Disk | VM Disk | GB | MB 轉 GB；detail XML 可補 disk list | `100 GB` |
| NetworkConnectionSection | VM NIC | VM NIC / IP | 網卡與 IP | 解析 NetworkConnection | `10.1.0.10` |

## 6. 功能畫面差異

| 功能 | 畫面/流程位置 | 畫面差異 | 欄位/選項差異 | 可操作能力 | 業務限制/提示 |
| --- | --- | --- | --- | --- | --- |
| Cloud Settings | Connection 表單 | 顯示 VCD tenant 連線設定 | Base URL、Basic/Token/Service Account | 可建立/更新連線 | Service Account 可能需外部授權 |
| Cloud Settings | Provider 授權差異區 | Service Account 顯示 user code / verification URI | 授權狀態 pending / authorized / error | 可啟動與檢查授權 | token / password 必須遮罩 |
| Cloud Settings | 同步資料 Wizard | Pool 為 VDC；Template 為 vApp Template | 同步 Pool、Network、Template、VM | 可同步資料 | Security Group 不適用 |
| Allocation Management | Shared Allocation 表單 | 使用共通欄位 | Project、System、Quota、Subnet | 不建立 Provider 附加資源 | 不顯示 Provider Extension |
| VM Management | VM 清單 | 顯示同步回來的 VCD VM | VCD status、IP、Template 來源 | 目前以檢視同步資料為主 | 目前不支援由 HCM 建立/開關 VCD VM |

## 7. 外部 API 上下行與範例

本章列出 VCD Provider Plugin 需要呼叫的外部 API。URL 皆以 Connection 表單的 `Base URL` 為前綴，例如 `{baseUrl} = https://vcd.example.com`。

### 7.1 授權/登入

#### 7.1.1 Basic 登入

| 項目 | 說明 |
| --- | --- |
| Method | POST |
| URL | `{baseUrl}/cloudapi/1.0.0/sessions` |
| 上行 | Header `Authorization: Basic <masked>`、`Accept: application/json;version=39.1` |
| 下行 | Response header 的 bearer token 或 response body token |
| HCM 取用 | access token 作為後續 API 授權 |

**上行範例**

```http
POST {baseUrl}/cloudapi/1.0.0/sessions
Authorization: Basic <masked>
Accept: application/json;version=39.1
```

#### 7.1.2 Service Account 啟動授權

| 項目 | 說明 |
| --- | --- |
| Method | POST |
| URL | `{baseUrl}/oauth/tenant/{tenantName}/device_authorization` |
| 上行 | form body `client_id={clientId}` |
| 下行 | `device_code`、`user_code`、`verification_uri`、`expires_in`、`interval` |
| HCM 取用 | 回寫 Connection 授權狀態 pending，畫面顯示 user code 與 verification URI |

```http
POST {baseUrl}/oauth/tenant/tenant-a/device_authorization
Content-Type: application/x-www-form-urlencoded

client_id=hcm-client
```

```json
{
  "device_code": "<masked>",
  "user_code": "ABCD-EFGH",
  "verification_uri": "https://vcd.example.com/oauth/tenant/tenant-a/activate",
  "expires_in": 600,
  "interval": 5
}
```

#### 7.1.3 Service Account 取得 / 更新 Token

| 項目 | 說明 |
| --- | --- |
| Method | POST |
| URL | `{baseUrl}/oauth/tenant/{tenantName}/token` |
| 上行 | device code poll 或 refresh token form body |
| 下行 | `access_token`、`refresh_token` |
| HCM 取用 | access token 用於本次同步；若回傳新 refresh token，更新 Connection 授權摘要 |

```http
POST {baseUrl}/oauth/tenant/tenant-a/token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token&refresh_token=<masked>&client_id=hcm-client
```

### 7.2 同步 VDC / Pool

#### 7.2.1 API 清單

| 順序 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 1 | GET | `{baseUrl}/cloudapi/1.0.0/vdcs?page={page}&pageSize=25` | 列出 tenant 可見 VDC | 無 body | `values[].id`、`values[].name` |
| 2 | GET | `{baseUrl}/api/vdc/{vdcId}` | 取得 VDC 容量 | 無 body | `ComputeCapacity`、`VdcStorageProfiles` |
| 3 | GET | VDC storage profile href | 取得 StorageProfile 容量 | 無 body | `Limit`、`StorageUsedMB`、`Units` |

```http
GET {baseUrl}/api/vdc/vdc-001
Authorization: Bearer <masked>
Accept: application/*+xml;version=39.1
```

```xml
<Vdc name="Prod VDC">
  <ComputeCapacity>
    <Cpu><Units>MHz</Units><Limit>80000</Limit><Used>24000</Used></Cpu>
    <Memory><Units>MB</Units><Limit>524288</Limit><Used>131072</Used></Memory>
  </ComputeCapacity>
</Vdc>
```

### 7.3 同步 Network

| 順序 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 1 | GET | `{baseUrl}/cloudapi/1.0.0/orgVdcNetworks?page={page}&pageSize=25` | 取得 Org VDC Network | 無 body | `values[].id`、`name`、`ownerRef`、`subnets.values[]`、`status` |
| 2 | GET | `{baseUrl}/cloudapi/1.0.0/vdcGroups/{vdcGroupRef}/vdcs?page={page}&pageSize=25` | owner 為 VDC Group 時查成員 VDC | 無 body | `values[].vdcRef.id` |

```json
{
  "values": [
    {
      "id": "urn:vcloud:network:net-001",
      "name": "APP-NET",
      "status": "REALIZED",
      "ownerRef": { "id": "urn:vcloud:vdc:vdc-001" },
      "subnets": {
        "values": [
          { "gateway": "10.1.0.1", "prefixLength": 24 }
        ]
      }
    }
  ]
}
```

### 7.4 同步 vApp Template

| 順序 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 1 | GET | `{baseUrl}/api/query?type=vAppTemplate&page={page}&pageSize=25&format=records&filter=vdc=={vdcId}` | 取得 VDC Template | 無 body | `VAppTemplateRecord.href`、`name`、`numberOfCpus`、`memoryAllocationMB`、`storageKB` |

```xml
<QueryResultRecords total="1">
  <VAppTemplateRecord
    name="Ubuntu-22.04"
    href="https://vcd.example.com/api/vAppTemplate/vappTemplate-001"
    numberOfCpus="2"
    memoryAllocationMB="4096"
    storageKB="52428800" />
</QueryResultRecords>
```

### 7.5 同步 VM 清單

| 順序 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 1 | GET | `{baseUrl}/api/query?type=vm&page={page}&pageSize=25&format=records&filter=vdc=={vdcId}` | 查詢 VM record | 無 body | `VMRecord.href`、`name`、`status`、`ipAddress`、`numberOfCpus`、`memoryMB`、`totalStorageAllocatedMb` |
| 2 | GET | VMRecord href | 取得 VM detail | 無 body | `ComputerName`、`NetworkConnectionSection`、disk `Item` |

```xml
<VMRecord
  name="app-01"
  href="https://vcd.example.com/api/vApp/vm-001"
  status="POWERED_ON"
  ipAddress="10.1.0.10"
  numberOfCpus="4"
  memoryMB="16384"
  totalStorageAllocatedMb="102400" />
```

```xml
<Vm name="app-01">
  <GuestCustomizationSection>
    <ComputerName>app-01</ComputerName>
  </GuestCustomizationSection>
  <NetworkConnectionSection>
    <NetworkConnection network="APP-NET">
      <IpAddress>10.1.0.10</IpAddress>
    </NetworkConnection>
  </NetworkConnectionSection>
</Vm>
```

## 8. 單位換算與狀態映射

| 項目 | VCD 單位 / 狀態 | HCM 表示 | 規則 |
| --- | --- | --- | --- |
| CPU | MHz | Core / vCPU | 依 HCM 顯示設定換算；原始 MHz 保留於同步摘要 |
| Memory | MB | GB | `MB / 1024` |
| Storage | MB / KB | GB / TB | StorageProfile 以 MB，Template storage 以 KB 換算 |
| Network status | REALIZED | active | 其他狀態為 inactive |
| VM status | POWERED_ON 或 status=4 | running | HCM 顯示標準狀態 |
| VM status | POWERED_OFF 或 status=8 | stopped | HCM 顯示標準狀態 |
| VM status | FAILED_CREATION | error | HCM 顯示標準狀態 |

## 9. 限制與待確認

| 項目 | 說明 |
| --- | --- |
| VCD 建立 VM | 目前專案未接 VCD instantiate vApp，文件不列為支援 |
| VCD 開關機 | 目前專案未接 VCD power action，文件不列為支援 |
| Security Group | 目前 VCD 不同步為 HCM Security Group |
| CPU Core 換算 | VCD 原始 CPU 為 MHz，業務顯示 Core 時需確認每 core MHz 基準 |
