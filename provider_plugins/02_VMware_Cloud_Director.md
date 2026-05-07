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

| 02 規範掛載點 | VCD 是否支援 | 標準輸入 | 標準輸出 | 外部介接章節 | 影響的 HCM 資料 |
| --- | --- | --- | --- | --- | --- |
| Provider 授權 | 支援 | Connection Auth Input | Auth Result | 授權/登入 | Cloud Connection |
| 同步資源池 | 支援 | Sync Pools Input | Pool Sync Result | 同步 VDC / Pool | Pool |
| 同步網路 | 支援 | Sync Network Input | Network Sync Result | 同步 Network | Subnet |
| 同步 VM 規格來源 | 支援 | Sync VM Catalog Input | VM Catalog Result | 同步 vApp Template | Pool VM Catalog |
| 同步 VM 清單 | 支援 | Sync VM Inventory Input | VM Inventory Result | 同步 VM 清單 | VM |
| Allocation 附加資源 | 不支援 | Allocation Extension Input | Allocation Extension Result | 不支援 | 無 |
| 建立 VM | 不支援 | Create VM Input | Create VM Result | 不支援 | 無 |
| VM 開機 | 不支援 | VM Power Input | VM Power Result | 不支援 | 無 |
| VM 關機 | 不支援 | VM Power Input | VM Power Result | 不支援 | 無 |
| VM 狀態追蹤 | 不支援 | VM Status Input | VM Status Result | 不支援 | 無 |

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
| Allocation Extension Input / Result | 不適用 | 功能畫面差異 | VCD Allocation 使用 HCM 共通 Project/System/Quota/Subnet |
| Create VM Input / Result | 不適用 | 功能畫面差異 | 目前專案未接 VCD 建立 VM 能力 |
| VM Power Input / Result | 不適用 | 功能畫面差異 | 目前專案未接 VCD 開關機能力 |
| VM Status Input / Result | 不適用 | 功能畫面差異 | 目前專案未接 VCD 單台 VM 狀態追蹤能力 |

## 3. Auth 與連線設定

VCD Connection 可使用三種授權方式：Basic、Token、Service Account。Service Account 會有外部互動授權流程，需在 Cloud Settings 的 Provider 授權差異區呈現 user code 與 verification URI。

授權畫面的顯示規則、按鈕行為與狀態轉換，統一定義於 `4.1 Provider 授權差異區詳情`。

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

## 4. 功能畫面差異

| 功能 | 畫面/流程位置 | 畫面差異 | 欄位/選項差異 | 可操作能力 | 業務限制/提示 |
| --- | --- | --- | --- | --- | --- |
| Cloud Settings | Connection 表單 | 顯示 VCD tenant 連線設定 | Base URL、Basic/Token/Service Account | 可建立/更新連線 | Service Account 可能需外部授權 |
| Cloud Settings | Provider 授權差異區 | Service Account 顯示 user code / verification URI | 授權狀態 pending / authorized / error | 可啟動與檢查授權 | token / password 必須遮罩 |
| Cloud Settings | 同步資料 Wizard | Pool 為 VDC；Template 為 vApp Template | 同步 Pool、Network、Template、VM | 可同步資料 | Security Group 不適用 |
| Allocation Management | Shared Allocation 表單 | 使用共通欄位 | Project、System、Quota、Subnet | 不建立 Provider 附加資源 | 不顯示 Provider Extension |
| VM Management | VM 清單 | 顯示同步回來的 VCD VM | VCD status、IP、Template 來源 | 目前以檢視同步資料為主 | 目前不支援由 HCM 建立/開關 VCD VM |

### 4.1 Provider 授權差異區詳情

本節補齊 `12_Cloud_Settings.md` 的 `3.4 Provider 授權差異區` 在 VCD Provider 的具體畫面規格。Cloud Settings 只定義共通入口；VCD 的欄位、狀態與操作以本節為準。

| Auth Type | 顯示欄位/區塊 | 主要操作 | 畫面輸出 | 備註 |
| --- | --- | --- | --- | --- |
| `basic` | 不顯示授權差異區 | 儲存 Connection 後可直接進同步 | 無額外授權畫面 | 授權結果由 `5.1.1 Basic 登入` 實際 API 回應決定 |
| `token` | 不顯示授權差異區 | 儲存 Connection 後可直接進同步 | 無額外授權畫面 | API Token 在 Connection 顯示時需遮罩 |
| `service_account` | 顯示授權差異區 | `啟動授權`、`檢查授權狀態`、`重新授權` | `user_code`、`verification_uri`、`expires_in`、`poll_interval_sec`、`auth_status`、`auth_message` | 授權區塊僅在 service account 顯示 |

#### 4.1.1 basic

**畫面示意圖**

```text
+--------------------------------------------------------------+
| Cloud Settings / 編輯 Connection                              |
+--------------------------------------------------------------+
| Provider: VCD                                                  |
| Label:    VCD Taipei                                           |
| Base URL: https://vcd.example.com                             |
| Auth Type: basic                                               |
| Username: admin@example.com                                    |
| Password: ********                                             |
+--------------------------------------------------------------+
| Provider 授權差異區：不顯示                                      |
| 說明：儲存後直接使用 5.1.1 驗證授權                               |
+--------------------------------------------------------------+
```

| Event/動作 | 觸發 | 成功結果 | 失敗結果 |
| --- | --- | --- | --- |
| 儲存 Connection | 點擊儲存 | 進入可同步狀態 | 顯示授權失敗訊息（依 5.1.1 回應） |

#### 4.1.2 token

**畫面示意圖**

```text
+--------------------------------------------------------------+
| Cloud Settings / 編輯 Connection                              |
+--------------------------------------------------------------+
| Provider: VCD                                                  |
| Label:    VCD Taipei                                           |
| Base URL: https://vcd.example.com                             |
| Auth Type: token                                               |
| API Token: ********                                            |
+--------------------------------------------------------------+
| Provider 授權差異區：不顯示                                      |
| 說明：儲存後直接進同步流程                                       |
+--------------------------------------------------------------+
```

| Event/動作 | 觸發 | 成功結果 | 失敗結果 |
| --- | --- | --- | --- |
| 儲存 Connection | 點擊儲存 | 進入可同步狀態 | 顯示 token 無效或授權失敗訊息 |

#### 4.1.3 service_account

**畫面示意圖**

```text
+--------------------------------------------------------------+
| Cloud Settings / 編輯 Connection                              |
+--------------------------------------------------------------+
| Provider: VCD                                                  |
| Label:    VCD Taipei                                           |
| Base URL: https://vcd.example.com                             |
| Auth Type: service_account                                     |
| Tenant Name: tenant-a                                          |
| Client ID: hcm-client                                          |
| Refresh Token: ********                                        |
+--------------------------------------------------------------+
| Provider 授權差異區                                             |
| Status: pending                                                |
| User Code: ABCD-EFGH                                           |
| Verification URI: https://vcd.example.com/oauth/.../activate  |
| Expires In: 600 sec                                            |
| Poll Interval: 5 sec                                           |
| Message: Waiting user authorization...                         |
| [啟動授權] [檢查授權狀態] [重新授權]                            |
+--------------------------------------------------------------+
```

**流程示意圖（service_account）**

```text
[Connection 儲存完成]
  |
  v
[按下 啟動授權]
  |
  v
[pending]
  - 顯示 user_code / verification_uri
  - 使用者到外部頁面完成授權
  |
  v
[按下 檢查授權狀態]
   |----------------------------|
   | token 取得成功             | token 未完成或失敗
   v                            v
[authorized]                 [pending 或 error]
  - 可進同步                     - 顯示 auth_message
  - 可重新授權                   - 可重試或重新授權
```

| 畫面狀態 | 觸發條件 | 顯示內容 | 可用操作 |
| --- | --- | --- | --- |
| `pending` | 呼叫 `5.1.2 Service Account 啟動授權` 成功 | 顯示 `user_code`、`verification_uri`、到期秒數與輪詢間隔 | 檢查授權狀態、重新授權 |
| `authorized` | 呼叫 `5.1.3 Service Account 取得 / 更新 Token` 成功 | 顯示授權成功訊息與最後更新時間（若有） | 重新授權 |
| `error` | device authorization / token API 失敗或逾時 | 顯示可讀 `auth_message` 錯誤訊息 | 重試、重新授權 |

| Event/動作 | API 對應 | 成功後畫面結果 | 失敗後畫面結果 |
| --- | --- | --- | --- |
| 啟動授權 | `POST /oauth/tenant/{tenantName}/device_authorization` | 進入 `pending`，顯示 `user_code` 與 `verification_uri` | 顯示 `error` 與錯誤訊息 |
| 檢查授權狀態 | `POST /oauth/tenant/{tenantName}/token`（device code poll） | 成功時進入 `authorized`，更新 token 摘要 | 維持 `pending` 或轉 `error`（依 OAuth error） |
| 重新授權 | 重新呼叫 device authorization | 重設授權碼與到期時間，狀態回 `pending` | 顯示 `error` 與錯誤訊息 |

安全要求：`password`、`api token`、`refresh token` 任何時候不得明文顯示；畫面僅可顯示遮罩值。

## 5. 外部介接上下行與範例

本章列出 VCD Provider Plugin 需要呼叫的外部 API。URL 皆以 Connection 表單的 `Base URL` 為前綴，例如 `{baseUrl} = https://vcd.example.com`。

### 5.1 授權/登入

#### 5.1.1 Basic 登入

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

| 標準輸出欄位（Auth Result） | Provider 欄位 | 重要備註 |
| --- | --- | --- |
| `access_token` | session response header / token 欄位 | 必填；後續 API Bearer token |
| `auth_status` | HTTP status + token 是否取得 | 成功為 `authorized`，失敗為 `error` |
| `auth_message` | 錯誤訊息（若有） | 失敗時回寫可讀訊息 |

#### 5.1.2 Service Account 啟動授權

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

| 標準輸出欄位（Auth Result） | Provider 欄位 | 重要備註 |
| --- | --- | --- |
| `device_code` | `device_code` | 啟動授權輪詢使用 |
| `user_code` | `user_code` | 顯示於畫面讓使用者輸入 |
| `verification_uri` | `verification_uri` | 顯示於畫面讓使用者開啟 |
| `expires_in` | `expires_in` | 授權碼有效時間 |
| `poll_interval_sec` | `interval` | 輪詢間隔秒數 |
| `auth_status` | 固定回寫 pending | 等待使用者完成授權 |

#### 5.1.3 Service Account 取得 / 更新 Token

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

| 標準輸出欄位（Auth Result） | Provider 欄位 | 重要備註 |
| --- | --- | --- |
| `access_token` | `access_token` | 必填；本次同步授權使用 |
| `refresh_token` | `refresh_token` | 若回傳新值需更新 Connection |
| `auth_status` | token API 成功/失敗 | 成功為 `authorized`，失敗為 `error` |
| `auth_message` | OAuth error response（若有） | 失敗時回寫可讀訊息 |

### 5.2 同步 VDC / Pool

#### 5.2.1 API 清單

| 順序 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 1 | GET | `{baseUrl}/cloudapi/1.0.0/vdcs?page={page}&pageSize=25` | 列出 tenant 可見 VDC | 無 body | `values[].id`、`values[].name` |
| 2 | GET | `{baseUrl}/api/vdc/{vdcId}` | 取得 VDC 容量 | 無 body | `ComputeCapacity`、`VdcStorageProfiles` |
| 3 | GET | VDC storage profile href | 取得 StorageProfile 容量 | 無 body | `Limit`、`StorageUsedMB`、`Units` |

#### 5.2.2 上下行範例

**上行 Request 範例：**

```http
GET {baseUrl}/api/vdc/f40b1346-2451-47ab-946b-7d3d9c77b900
Authorization: Bearer <masked>
Accept: application/*+xml;version=39.1
```

**下行 Response 範例（精簡版，取重要欄位）：**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ns2:Vdc xmlns:ns2="http://www.vmware.com/vcloud/v1.5" 
         name="VDC5502279600021" 
         id="urn:vcloud:vdc:f40b1346-2451-47ab-946b-7d3d9c77b900"
         href="https://sddc.hicloud.hinet.net/api/vdc/f40b1346-2451-47ab-946b-7d3d9c77b900"
         status="1">
  <ns2:Description>驗證區</ns2:Description>
  
  <ns2:ComputeCapacity>
    <ns2:Cpu>
      <ns2:Units>MHz</ns2:Units>
      <ns2:Limit>250000</ns2:Limit>
      <ns2:Used>216000</ns2:Used>
    </ns2:Cpu>
    <ns2:Memory>
      <ns2:Units>MB</ns2:Units>
      <ns2:Limit>512000</ns2:Limit>
      <ns2:Used>503808</ns2:Used>
    </ns2:Memory>
  </ns2:ComputeCapacity>

  <ns2:VdcStorageProfiles>
    <ns2:VdcStorageProfile href="https://sddc.hicloud.hinet.net/api/vdcStorageProfile/862e9b33-b6f3-4d7f-85e6-1e3e134f567b"
                           id="urn:vcloud:vdcstorageProfile:862e9b33-b6f3-4d7f-85e6-1e3e134f567b"
                           name="SDDC-SAS-Storage(HN55022796)"/>
    <ns2:VdcStorageProfile href="https://sddc.hicloud.hinet.net/api/vdcStorageProfile/45791192-870a-4b9d-8965-5c4fb42a796c"
                           id="urn:vcloud:vdcstorageProfile:45791192-870a-4b9d-8965-5c4fb42a796c"
                           name="SDDC-SAS-Storage(HN55022796)UAT-1"/>
  </ns2:VdcStorageProfiles>

  <ns2:VCpuInMhz2>2000</ns2:VCpuInMhz2>
</ns2:Vdc>
```

#### 5.2.3 掛載點輸出

| 標準輸出欄位（Pool Sync Result） | Provider 欄位 | 重要備註 |
| --- | --- | --- |
| `provider_pool_id` | `values[].id`（VDC id） | 必填；Pool provider 識別 |
| `name` | `values[].name` | 必填；Pool 顯示名稱 |
| `cpu_total` | `ComputeCapacity.Cpu.Limit` | 來源單位常為 MHz |
| `cpu_used` | `ComputeCapacity.Cpu.Used` | 與 total 同單位 |
| `memory_total_gb` | `ComputeCapacity.Memory.Limit` | `MB / 1024` |
| `memory_used_gb` | `ComputeCapacity.Memory.Used` | `MB / 1024` |
| `disk_total_gb` | `VdcStorageProfiles[].Limit` | 依 `Units` 換算 |
| `disk_used_gb` | `VdcStorageProfiles[].StorageUsedMB` | `MB / 1024` |
| `ref.id` | `values[].id` URN 最後段（VDC UUID） | 重複同步識別既有 Pool；`findPoolByRef` 以此比對 |
| `ref.name` | `values[].name` | 首次同步建立 Pool 名稱的備援來源 |
| `ref.cloud_connection_id` | Connection ID | 隔離不同連線的 Pool；prune 時依此清理過期資料 |
| `ref.description` | VDC 描述（若有） | 保存 provider 原始描述 |
| `ref.sync_meta.cpu_mhz_per_core` | `values[].VCpuInMhz2`（若有） | MHz → Core 換算基準；同步時可帶入預設，無值時再由管理者補填 |
| `ref.sync_meta.cpu_total_mhz` | `ComputeCapacity.Cpu.Limit` 原始 MHz | 保留原始值供未來換算調整 |

### 5.3 同步 Network

#### 5.3.1 API 清單

| 順序 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 1 | GET | `{baseUrl}/cloudapi/1.0.0/orgVdcNetworks?page={page}&pageSize=25` | 取得 Org VDC Network | 無 body | `values[].id`、`name`、`ownerRef`、`subnets.values[]`、`status` |
| 2 | GET | `{baseUrl}/cloudapi/1.0.0/vdcGroups/{vdcGroupRef}/vdcs?page={page}&pageSize=25` | owner 為 VDC Group 時查成員 VDC | 無 body | `values[].vdcRef.id` |

#### 5.3.2 上下行範例

**上行 Request 範例：**

```http
GET {baseUrl}/cloudapi/1.0.0/orgVdcNetworks?page=1&pageSize=25
Authorization: Bearer <masked>
Accept: application/json;version=39.1
```

**下行 Response 範例：**

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

#### 5.3.3 掛載點輸出

| 標準輸出欄位（Network Sync Result） | Provider 欄位 | 重要備註 |
| --- | --- | --- |
| `provider_network_id` | `values[].id` | 必填；Subnet provider 識別來源 |
| `name` | `values[].name` | 必填；Network 顯示名稱 |
| `owner_pool_id` | `ownerRef.id` 或 vdcGroup 成員 `vdcRef.id` | 用於關聯可用 Pool |
| `cidr` | `subnets.values[].gateway` + `prefixLength` | 由 gateway/prefix 組 CIDR |
| `gateway` | `subnets.values[].gateway` | 主要 gateway |
| `status` | `values[].status` | `REALIZED` -> active；其他 inactive |
| `ref.id` | `values[].id` URN 最後段（Network UUID） | 重複同步識別既有 Subnet |
| `ref.name` | `values[].name` | 備援識別 |
| `ref.subnet_idx` | subnet 序號（多 subnet 網路時遞增） | 同一 Network 多個 subnet 的區分 |
| `ref.owner_ref` | `ownerRef.id`（VDC 或 vdcGroup URN） | 判定 Subnet 可被哪些 Pool 使用的依據 |
| `ref.owner_type` | `vdc` 或 `vdcGroup` | 決定 pool_ref_ids 的展開方式 |
| `ref.pool_ref_ids` | VDC UUID 清單（由 ownerRef 解析） | 追蹤 Subnet 隸屬 Pool；prune 時依此清理 |
| `ref.cloud_connection_id` | Connection ID | 隔離不同連線的 Subnet |

### 5.4 同步 vApp Template

#### 5.4.1 API 清單

| 順序 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 1 | GET | `{baseUrl}/api/query?type=vAppTemplate&page={page}&pageSize=25&format=records&filter=vdc=={vdcId}` | 取得 VDC Template | 無 body | `VAppTemplateRecord.href`、`name`、`numberOfCpus`、`memoryAllocationMB`、`storageKB` |

#### 5.4.2 上下行範例

**上行 Request 範例：**

```http
GET {baseUrl}/api/query?type=vAppTemplate&page=1&pageSize=25&format=records&filter=vdc=={vdcId}
Authorization: Bearer <masked>
Accept: application/*+xml;version=39.1
```

**下行 Response 範例：**

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

#### 5.4.3 掛載點輸出

| 標準輸出欄位（VM Catalog Result） | Provider 欄位 | 重要備註 |
| --- | --- | --- |
| `template_id` | `VAppTemplateRecord.href` 最後段 | 必填；Template provider 識別 |
| `name` | `VAppTemplateRecord.name` | 必填；顯示名稱 |
| `default_cpu` | `VAppTemplateRecord.numberOfCpus` | 轉整數 |
| `default_memory_gb` | `VAppTemplateRecord.memoryAllocationMB` | `MB / 1024` |
| `default_disk_gb` | `VAppTemplateRecord.storageKB` | `KB / 1024 / 1024` |

### 5.5 同步 VM 清單

同步 VM 清單採二步驟 API 呼叫：
1. **Step 1** - Query 取得 VM 清單摘要（VMRecord）
2. **Step 2** - 依 VMRecord href 逐筆取得 VM 明細（Vm detail XML）

#### 5.5.1 API 清單

| 順序 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 1 | GET | `{baseUrl}/api/query?type=vm&page={page}&pageSize=25&format=records&filter=vdc=={vdcId}` | 查詢 VM record | 無 body | `VMRecord.href`、`name`、`status`、`ipAddress`、`numberOfCpus`、`memoryMB`、`totalStorageAllocatedMb` |
| 2 | GET | VMRecord href | 取得 VM detail | 無 body | `ComputerName`、`NetworkConnectionSection`、disk `Item` |

#### 5.5.2 上下行範例

#### Step 1：Query VM Records

**用途：** 分頁查詢指定 VDC 的所有 VM record

**上行 Request：** 無 request body；参数由 URL query string 提供

**下行 Response 範例：**

```xml
<QueryResultRecords total="47" pageCount="2" page="1" pageSize="25">

  <VMRecord
    href="https://vcd.example.com/api/vApp/vm-550e8400-e29b-41d4-a716-446655440010"
    name="web-server-01"
    ipAddress="10.100.1.10"
    status="POWERED_ON"
    numberOfCpus="4"
    memoryMB="8192"
    totalStorageAllocatedMb="102400"
    isVAppTemplate="false"
    isDeployed="true"/>

  <VMRecord
    href="https://vcd.example.com/api/vApp/vm-550e8400-e29b-41d4-a716-446655440011"
    name="app-db-01"
    ipAddress="10.100.1.20"
    status="POWERED_ON"
    numberOfCpus="8"
    memoryMB="16384"
    totalStorageAllocatedMb="204800"
    isVAppTemplate="false"
    isDeployed="true"/>

</QueryResultRecords>
```

**重要說明：**
- `<QueryResultRecords>` 為外層包裝，包含分頁資訊（`total`、`pageCount`、`page`、`pageSize`）
- 若 `total > pageSize`，需逐頁迴圈查詢
- HCM 需迭代所有 VMRecord，取得 `href` 後進行 Step 2 詳細查詢

#### Step 2：Fetch VM Detail

**用途：** 逐筆取得 VM 的完整明細，包括 NIC 與 Disk 詳情

**上行 Request：** 無 request body；URL 為上一步 `VMRecord.href`

**下行 Response 範例：**

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

#### 5.5.3 掛載點輸出

| 標準輸出欄位（VM Inventory Result） | Provider 欄位 | 重要備註 |
| --- | --- | --- |
| `provider_vm_id` | `VMRecord.href` 最後段 | 必填；作為 provider 端 VM 識別，後續開關機與追蹤使用 |
| `name` | `VMRecord.name` | 必填；直接作為 VM 顯示名稱 |
| `status` | `VMRecord.status` | 必填；需先轉成 HCM 標準狀態（如 `POWERED_ON` -> `running`） |
| `cpu` | `VMRecord.numberOfCpus` | 轉整數；單位為 Core / vCPU |
| `memory_gb` | `VMRecord.memoryMB` | `MB / 1024` 轉 GB |
| `disk_gb` | `VMRecord.totalStorageAllocatedMb` | `MB / 1024` 轉 GB |
| `ip` | `VMRecord.ipAddress` | 主要 IP；若明細有 NIC IP 可覆蓋 |
| `hostname` | `GuestCustomizationSection.ComputerName` | 明細 API 補值；無值可回退 `name` |
| `nics` | `NetworkConnectionSection.NetworkConnection` | 解析 `network` 與 `IpAddress` |
| `disks` | disk `Item`（`ResourceType=17`） | 依 `AllocationUnits` 換算容量 |
| `tags` | VCD metadata / custom properties（若有） | 無值可留空 |
| `ref.id` | `VMRecord.href` 最後段（VM UUID） | 重複同步識別既有 VM；避免重複建立 |
| `ref.name` | `VMRecord.name` | 備援識別 |
| `ref.cloud_connection_id` | Connection ID | 隔離不同連線的 VM |

## 6. 單位換算與狀態映射

| 項目 | VCD 單位 / 狀態 | HCM 表示 | 規則 |
| --- | --- | --- | --- |
| CPU | MHz | Core / vCPU | 依 HCM 顯示設定換算；原始 MHz 保留於同步摘要 |
| Memory | MB | GB | `MB / 1024` |
| Storage | MB / KB | GB / TB | StorageProfile 以 MB，Template storage 以 KB 換算 |
| Network status | REALIZED | active | 其他狀態為 inactive |
| VM status | POWERED_ON 或 status=4 | running | HCM 顯示標準狀態 |
| VM status | POWERED_OFF 或 status=8 | stopped | HCM 顯示標準狀態 |
| VM status | FAILED_CREATION | error | HCM 顯示標準狀態 |

## 7. 限制與待確認

| 項目 | 說明 |
| --- | --- |
| VCD 建立 VM | 目前專案未接 VCD instantiate vApp，文件不列為支援 |
| VCD 開關機 | 目前專案未接 VCD power action，文件不列為支援 |
| Security Group | 目前 VCD 不同步為 HCM Security Group |
| CPU Core 換算 | VCD 原始 CPU 為 MHz；同步時可優先取回 `VCpuInMhz2` 作為每 core MHz 預設值，若來源缺值再由管理者手動調整 |
