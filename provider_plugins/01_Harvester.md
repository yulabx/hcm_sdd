# 01 Harvester Provider Plugin

## 1. Provider 基本資訊

Harvester Provider Plugin 用於將 Harvester / KubeVirt 環境中的叢集資源轉成 HCM 標準資料，並支援後續 Allocation Management 與 VM Management 使用。

Harvester 在 HCM 中的定位偏向私有雲 Provider。HCM 會把一個 Harvester Cluster 視為可管理的雲端連線範圍，將 Cluster 內的節點容量、網路、Image 與 VM 同步成 HCM 的 Pool、Subnet、Image Catalog 與 VM。

HCM 中的 Cloud / Provider ID 可以是業務命名，例如 `cht-harvester`；Driver ID 則固定為 `harvester`，用來套用本文件定義的 Provider Plugin 規格。

| 項目 | 說明 |
| --- | --- |
| Provider 名稱 | Harvester |
| Provider 類型 | 私有雲 |
| HCM Provider Driver | `harvester` |
| 主要資源概念 | Cluster、Namespace、Node、NetworkAttachmentDefinition、VirtualMachineImage、VirtualMachine |
| HCM 主要對應資料 | Cloud Connection、Pool、Subnet、Image Catalog、Allocation、VM |
| 主要使用功能 | Cloud Settings、Allocation Management、VM Management |

## 2. 共同規範勾稽

本章把 Harvester Provider Plugin 對回 `02_Provider_Plugin_Model.md` 的共同掛載點與標準輸入/輸出。讀者可先用本章確認 Harvester 對應哪個 02 規範掛載點，再追到 API 章節看實際 Method、URL、上行與下行。

### 2.1 掛載點對照表

| 02 規範掛載點 | Harvester 是否支援 | Harvester 文件章節 | 標準輸入 | 標準輸出 | 外部介接章節 | 影響的 HCM 資料 |
| --- | --- | --- | --- | --- | --- | --- |
| Provider 授權 | 支援 | Auth 與連線設定 | Connection Auth Input | Auth Result | 授權/登入 | Cloud Connection |
| 同步資源池 | 支援 | 外部 API 上下行與範例 | Sync Pools Input | Pool Sync Result | 同步 Pool | Pool |
| 同步網路 | 支援 | 外部 API 上下行與範例 | Sync Network Input | Network Sync Result | 同步 Network | Subnet |
| 同步 VM 規格來源 | 支援 | Image Catalog 映射、功能畫面差異 | Sync VM Catalog Input | VM Catalog Result | 同步 Image | Pool Image Catalog |
| 同步 VM 清單 | 支援 | VM 映射、VM 狀態映射 | Sync VM Inventory Input | VM Inventory Result | 同步 VM 清單 | VM |
| Allocation 附加資源 | 支援 | 功能畫面差異、Namespace 狀態映射 | Allocation Extension Input | Allocation Extension Result | Allocation 附加資源：Namespace | Allocation |
| 建立 VM | 支援 | 功能畫面差異、VM 映射 | Create VM Input | Create VM Result | 建立 VM | VM |
| VM 開機 | 支援 | VM 狀態映射 | VM Power Input | VM Power Result | VM 開機 / 關機 | VM |
| VM 關機 | 支援 | VM 狀態映射 | VM Power Input | VM Power Result | VM 開機 / 關機 | VM |
| VM 狀態追蹤 | 支援 | VM 映射、VM 狀態映射 | VM Status Input | VM Status Result | VM 狀態追蹤 | VM |

### 2.2 標準輸入/輸出落地表

| 02 標準資料 | Harvester 對應欄位/概念 | Provider 文件章節 | 轉換或限制 |
| --- | --- | --- | --- |
| Connection Auth Input | Base URL、API Token、Namespace Filter、TLS Skip Verify | Auth 與連線設定 | API Token 以 Bearer Token 使用；Namespace Filter 作為同步/操作範圍 |
| Auth Result | API 可呼叫狀態 | 授權/登入 | Harvester 無獨立登入 API，授權結果由後續 API 呼叫成功與否判斷 |
| Sync Pools Input | Cloud Connection、Base URL、Token | 同步 Pool | Harvester 以 Cluster 形成 Pool，不依 VDC/VPC 拆分 |
| Pool Sync Result | Cluster / Node / Longhorn 容量 | Pool 映射、同步 Pool | Node CPU/Memory 與 Longhorn storage 轉成 HCM Pool 容量 |
| Sync Network Input | Cloud Connection、Cluster scope | 同步 Network | 取得 Cluster 可用 NetworkAttachmentDefinition |
| Network Sync Result | NetworkAttachmentDefinition、IPPool | Network / Subnet 映射 | 轉成 HCM Subnet；Harvester 不產生 Security Group |
| Sync VM Catalog Input | Cloud Connection、Pool | 同步 Image | Harvester 使用 Image，不使用 Template/Flavor 作為主要來源 |
| VM Catalog Result | VirtualMachineImage | Image Catalog 映射 | `namespace/name` 成為 Image id；storageClassName 建立 VM 時需使用 |
| Sync VM Inventory Input | Cloud Connection、Namespace Filter | 同步 VM 清單 | 若有 Namespace Filter，只保留指定 namespace 的 VM |
| VM Inventory Result | VirtualMachine、VirtualMachineInstance | VM 映射、同步 VM 清單 | VM/VMI 合併後轉成 HCM VM 狀態、IP、NIC、Disk |
| Allocation Extension Input | Allocation、Project、System、namespace | Allocation 附加資源：Namespace | namespace 可由 HCM 建議，也可由管理者輸入；在 Allocation Configure / Execute 生效時建立或更新 |
| Allocation Extension Result | Namespace 建立或更新結果 | Namespace 狀態映射 | 成功轉 `ready`，失敗轉 `error` 並回寫可讀訊息 |
| Create VM Input | VM name、已生效 Allocation namespace、Image id、CPU、Memory、Disk、Subnet、Static IP | 建立 VM | VM 建立只使用 Allocation 已建立的 namespace，不在 VM 建立時才建立 namespace；會拆成 PVC、Secret、VirtualMachine 多個 API |
| Create VM Result | namespace/name、初始狀態、IP | 建立 VM | 建立後初始狀態為 `provisioning`，後續由狀態追蹤更新 |
| VM Power Input / Result | namespace/name、start/stop | VM 開機 / 關機 | 操作成功先回 `starting` 或 `stopping` |
| VM Status Input / Result | namespace/name、VM/VMI 狀態 | VM 狀態追蹤、VM 狀態映射 | KubeVirt printableStatus 轉成 HCM 標準 VM 狀態 |

## 3. Auth 與連線設定

Harvester 連線以 Cluster API Endpoint 與 API Token 為主。若 Harvester 使用自簽憑證，Cloud Connection 可提供跳過 TLS 驗證的業務選項，讓管理者能先完成串接。

授權畫面的顯示規則與事件行為，統一定義於 `4.1 Provider 授權差異區詳情`。

### 3.1 Connection 表單欄位

| 欄位 | 業務意義 | 必填 | 範例 | 備註 |
| --- | --- | --- | --- | --- |
| Cloud | 選擇 Harvester Provider | 是 | Harvester |
| Label | HCM 內顯示的連線名稱 | 是 | Taipei Harvester |
| Base URL | Harvester API Endpoint | 是 | `https://harvester.example.com` | 對應 Harvester / Rancher API 入口 |
| Namespace Filter | 限制同步或操作的 Namespace 範圍 | 否 | `prod-app` | 多筆時代表只同步指定 Namespace 的 VM |
| TLS Skip Verify | 是否跳過 TLS 驗證 | 否 | 是 / 否 | 自簽憑證環境可使用 |
| Auth Type | 授權方式 | 是 | API Token | Harvester 使用 Token |
| API Token | 存取 Harvester API 的 Token | 是 | `<masked>` | 顯示時必須遮罩 |

### 3.2 授權差異表

| 授權項目 | 說明 |
| --- | --- |
| 授權型態 | 表單式授權 |
| Connection 表單欄位 | Base URL、Namespace Filter、TLS Skip Verify、API Token |
| 是否顯示授權差異區 | 否 |
| 授權差異區欄位 | 無 |
| 授權動作 | 使用者儲存 Connection 後，後續同步或 VM 操作以 API Token 存取 Harvester |
| 授權結果 | HCM 可取得 Cluster 資源、Image、VM 與執行 VM 操作 |
| 敏感資訊遮罩 | API Token 必須遮罩 |

## 4. 功能畫面差異

Harvester 的差異集中在 Cloud Settings 的連線/同步、Allocation Management 的 namespace，以及 VM Management 的 Image 與網路欄位。Harvester 不支援 AWS Security Group；若 HCM 畫面有安全群組區塊，Harvester 不提供可同步或可選的 provider 原生安全群組。

| 功能 | 畫面/流程位置 | 畫面差異 | 欄位/選項差異 | 可操作能力 | 業務限制/提示 |
| --- | --- | --- | --- | --- | --- |
| Cloud Settings | Connection 表單 | 顯示 Cluster 設定語意 | Base URL、Namespace Filter、TLS Skip Verify、API Token | 可建立/更新連線 | Token 必須遮罩；自簽憑證可跳過 TLS |
| Cloud Settings | 同步資料 Wizard | Template Step 顯示為 Image | 同步 VirtualMachineImage | 可同步 Pool、Network、Image、VM | Security Group 不適用 |
| Allocation Management | Shared Allocation 表單 | 顯示 Provider Extension 區塊 | Namespace 欄位與 namespace 狀態 | 可填寫或使用建議 namespace；Configure / Execute 生效時建立或更新 namespace | namespace 需符合 Kubernetes 命名規則 |
| VM Management | VM 建立表單 | 顯示 Image 選項 | Image 來源為 Pool Image Catalog | 可建立 VM | Image 若為 legacy 需提示是否可用 |
| VM Management | Network / Security Group | 不顯示 Harvester 原生 Security Group 選項 | Network 來自 Harvester Subnet | 可選 IP 模式與網路 | Harvester 不支援 AWS Security Group |
| VM Management | VM 操作 | 支援 Start / Stop / 狀態追蹤 | instanceId 需帶 namespace/name | 可開機、關機、查詢狀態 | 操作結果以 KubeVirt 狀態回寫 HCM |

### 4.1 Provider 授權差異區詳情

| Auth Type | 顯示欄位/區塊 | 主要操作 | 畫面輸出 | 備註 |
| --- | --- | --- | --- | --- |
| `api_token` | 不顯示授權差異區 | 儲存 Connection 後直接進同步或 VM 操作 | 無額外授權畫面 | Token 可用性由後續 API 結果判斷 |

#### 4.1.1 api_token

**畫面示意圖**

```text
+--------------------------------------------------------------+
| Cloud Settings / 編輯 Connection                              |
+--------------------------------------------------------------+
| Provider: Harvester                                            |
| Label: Taipei Harvester                                        |
| Base URL: https://harvester.example.com                        |
| Namespace Filter: prod-app                                     |
| TLS Skip Verify: Yes                                           |
| Auth Type: api_token                                           |
| API Token: ********                                            |
+--------------------------------------------------------------+
| Provider 授權差異區：不顯示                                      |
| 說明：儲存後直接使用 Bearer Token 呼叫 Harvester API            |
+--------------------------------------------------------------+
```

| Event/動作 | 觸發 | 成功結果 | 失敗結果 |
| --- | --- | --- | --- |
| 儲存 Connection | 點擊儲存 | 進入可同步與可操作狀態 | 顯示連線建立失敗訊息 |
| 執行同步或 VM 操作 | Wizard 同步或 VM 動作觸發 API 呼叫 | API 回應成功，授權視為可用 | 顯示授權失敗或 token 無效訊息 |

## 5. 外部介接上下行與範例

本章列出 Harvester Provider Plugin 需要呼叫的外部 API。URL 皆以 Connection 表單的 `Base URL` 為前綴，例如 `{baseUrl} = https://harvester.example.com`。所有 request 皆需帶入授權 header：

| Header | 值 | 說明 |
| --- | --- | --- |
| Authorization | `Bearer <masked>` | Harvester API Token，一律遮罩 |
| Content-Type | `application/json` | POST / PUT request 使用 |

### 5.1 授權/登入

Harvester 不需要額外登入換 token。HCM 使用 Connection 表單保存的 API Token 直接呼叫後續 API。

| 項目 | 說明 |
| --- | --- |
| Method | 不適用，無獨立登入 API |
| URL | 不適用 |
| 上行 | 後續 API request header 帶 `Authorization: Bearer <masked>` |
| 下行 | 由各資源 API 回傳 200/201/202 或錯誤狀態 |
| HCM 取用 | Token 是否可用由同步或操作 API 的結果判斷 |

### 5.2 同步 Pool

同步 Pool 時，HCM 會先以 Harvester Cluster 形成一個 Pool 來源，再查 Kubernetes Node 與 Longhorn 容量，轉成 HCM Pool 的 CPU、Memory、Disk。

#### 5.2.1 API 清單

| 順序 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 1 | GET | `{baseUrl}/api/v1/nodes` | 取得 Cluster Node CPU / Memory | 無 body | `items[].status.allocatable.cpu`、`items[].status.allocatable.memory`、`items[].metadata.annotations["management.cattle.io/pod-requests"]` |
| 2a | GET | `{baseUrl}/apis/longhorn.io/v1beta2/nodes` | 取得 Longhorn disk 容量 | 無 body | `items[].status.diskStatus.*.storageMaximum`、`storageScheduled` |
| 2b | GET | `{baseUrl}/apis/longhorn.io/v1beta2/namespaces/longhorn-system/nodes` | 2a 無資料時的替代 URL | 無 body | 同 2a |
| 2c | GET | `{baseUrl}/apis/longhorn.io/v1beta1/nodes` | v1beta2 不可用時的替代 URL | 無 body | 同 2a |
| 2d | GET | `{baseUrl}/apis/longhorn.io/v1beta1/namespaces/longhorn-system/nodes` | v1beta1 namespace 版本替代 URL | 無 body | 同 2a |
| 3a | GET | `{baseUrl}/apis/harvesterhci.io/v1beta1/settings/overcommit-config` | 取得 Harvester overcommit 設定 | 無 body | storage overcommit percentage |
| 3b | GET | `{baseUrl}/v1/harvester/settings/overcommit-config` | 3a 無資料時的替代 URL | 無 body | storage overcommit percentage |
| 3c | GET | `{baseUrl}/apis/longhorn.io/v1beta2/namespaces/longhorn-system/settings/storage-over-provisioning-percentage` | 取得 Longhorn over provisioning 設定 | 無 body | `value` |
| 3d | GET | `{baseUrl}/apis/longhorn.io/v1beta1/namespaces/longhorn-system/settings/storage-over-provisioning-percentage` | v1beta1 替代 URL | 無 body | `value` |

#### 5.2.2 上下行範例

**上行 Request 範例：**

```http
GET {baseUrl}/api/v1/nodes
Authorization: Bearer <masked>
```

**下行 Response 範例：**

```json
{
  "items": [
    {
      "metadata": {
        "name": "node-01",
        "annotations": {
          "management.cattle.io/pod-requests": "{\"cpu\":\"12000m\",\"memory\":\"64Gi\"}"
        }
      },
      "status": {
        "allocatable": {
          "cpu": "32",
          "memory": "256Gi"
        }
      }
    }
  ]
}
```

#### 5.2.3 掛載點輸出

| 標準輸出欄位（Pool Sync Result） | Provider 欄位 | 重要備註 |
| --- | --- | --- |
| `provider_pool_id` | Cluster 名稱 | 必填；Harvester 以整個 Cluster 作為一個 Pool |
| `cpu_total` | `status.allocatable.cpu`（無則 `capacity.cpu`） | 來源 nodes；`sum(parseCpuQuantity(cpu))`；單位 Core |
| `cpu_provisioned` | `annotations['management.cattle.io/pod-requests'].cpu` | 來源 nodes；`sum(parseCpuQuantity(podRequests.cpu))`；已分配給 Pod 的 CPU |
| `memory_total_gb` | `status.allocatable.memory`（無則 `capacity.memory`） | 來源 nodes；`sum(parseMemoryToGb(memory))`；由 Ki/Mi/Gi 換算為 GB |
| `memory_provisioned_gb` | `annotations['management.cattle.io/pod-requests'].memory` | 來源 nodes；`sum(parseMemoryToGb(podRequests.memory))`；已分配給 Pod 的 Memory |
| `disk_total_gb` | `storageMaximum`、`storageReserved`、overcommit% | 來源 Longhorn nodes + overcommit settings；`sum(max(0, storageMaximum - storageReserved) x (overcommit/100))` 轉 GB；overcommit 預設 200 |
| `disk_provisioned_gb` | `storageScheduled` | 來源 Longhorn nodes；`sum(storageScheduled)` 轉 GB；不受 overcommit 影響 |
| `ref.id` | Cluster 名稱 | 重複同步識別既有 Pool |
| `ref.name` | Cluster 名稱 | 備援識別 |
| `ref.cloud_connection_id` | Connection ID | 隔離不同連線的 Pool |
| `ref.sync_meta` | 來自 Harvester pool 同步摘要（如 `cpu_total_cores`、`memory_total_gb`、`storage_total_gb`、`storage_allocated_gb`） | 會寫入；僅 `cpu_mhz_per_core` / `cpu_total_mhz` / `cpu_used_mhz` 不寫入（或清為 undefined） |

**注意事項：**
- nodes、Longhorn nodes、overcommit settings 三類 API 並行呼叫
- Longhorn nodes 與 overcommit settings 各自依序 fallback，第一個成功即採用
- 若 overcommit settings API 全部 fallback 失敗，預設 overcommit = 200

### 5.3 同步 Network

同步 Network 時，HCM 讀取 Harvester NetworkAttachmentDefinition 與 IPPool，轉為 HCM Subnet。

#### 5.3.1 API 清單

| 順序 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 1 | GET | `{baseUrl}/apis/k8s.cni.cncf.io/v1/network-attachment-definitions` | 取得 Harvester 可用網路 | 無 body | `items[].metadata.namespace`、`name`、`annotations`、`labels`、`spec.config` |
| 2 | GET | `{baseUrl}/apis/network.harvesterhci.io/v1beta1/ippools` | 取得網路對應 IPPool | 無 body | `items[].metadata.name`、`spec.ipv4Config.subnet`、`gateway` |

#### 5.3.2 上下行範例

**上行 Request 範例：**

```http
GET {baseUrl}/apis/k8s.cni.cncf.io/v1/network-attachment-definitions
Authorization: Bearer <masked>
```

**下行 Response 範例：**

```json
{
  "items": [
    {
      "metadata": {
        "namespace": "default",
        "name": "vlan100",
        "labels": {
          "network.harvesterhci.io/type": "L2VlanNetwork",
          "network.harvesterhci.io/vlan-id": "100"
        },
        "annotations": {
          "network.harvesterhci.io/route": "{\"cidr\":\"10.1.0.0/24\",\"gateway\":\"10.1.0.1\"}"
        }
      },
      "spec": {
        "config": "{\"ipam\":{\"subnet\":\"10.1.0.0/24\",\"gateway\":\"10.1.0.1\"}}"
      }
    }
  ]
}
```

#### 5.3.3 掛載點輸出

| 標準輸出欄位（Network Sync Result） | Provider 欄位 | 重要備註 |
| --- | --- | --- |
| `provider_network_id` | metadata.namespace + metadata.name | 必填；格式 `namespace/name` 作為唯一識別 |
| `name` | metadata.name | 必填；網路名稱；可由 `network.harvesterhci.io/type` / `vlan-id` 補充顯示資訊 |
| `cidr` | route annotation / IPPool / spec.config | 依優先序 route annotation -> IPPool -> spec.config；無 CIDR 時可留空 |
| `gateway` | route / IPPool / ipam gateway | 依優先序 route -> IPPool -> ipam；無值時推導或留空 |
| `vlan_id` | network.harvesterhci.io/vlan-id label | VLAN 網路時填入 |
| `owner_pool_ids` | Cluster 層級 | Harvester 不區分 Pool，所有 Network 可被全 Cluster 使用 |
| `ref.id` | `network_id`（= metadata.name，僅 name，不含 namespace） | 重複同步識別既有 Subnet；namespace/name 在 `cloud_ref`，`ref.id` 只存 name |
| `ref.name` | metadata.name | 備援識別 |
| `ref.subnet_idx` | NetworkAttachmentDefinition 陣列 index（0-based） | 第一筆為 0；多網路時遞增 |
| `ref.owner_ref` | 不寫入（undefined） | Harvester driver 不設定 owner_ref；pool 歸屬由 pool_ref_ids 決定 |
| `ref.owner_type` | `unknown` | Harvester driver 回傳 unknown；pool 歸屬走 `targetPoolRefIds = [vdcRef.id]` 路徑 |
| `ref.pool_ref_ids` | Pool ref.id（= poolIdFromBaseUrl） | 追蹤 Subnet 隸屬 Pool |
| `ref.cloud_connection_id` | Connection ID | 隔離不同連線的 Subnet |

### 5.4 同步 Image

同步 Image 時，HCM 查詢 Harvester VirtualMachineImage，寫入 Pool 的 Image Catalog。

#### 5.4.1 API 清單

| 順序 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 1 | GET | `{baseUrl}/apis/harvesterhci.io/v1beta1/virtualmachineimages` | 取得 VM 建立可用 Image | 無 body | `items[].metadata.namespace`、`metadata.name`、`metadata.labels`、`spec.displayName`、`status.storageClassName` |
| 2 | GET | `{baseUrl}/apis/harvesterhci.io/v1beta1/namespaces/{imageNamespace}/virtualmachineimages/{imageName}` | 建立 VM 前補查單一 Image 的 StorageClass | 無 body | `status.storageClassName` |

#### 5.4.2 上下行範例

**上行 Request 範例：**

```http
GET {baseUrl}/apis/harvesterhci.io/v1beta1/virtualmachineimages
Authorization: Bearer <masked>
```

**下行 Response 範例：**

```json
{
  "items": [
    {
      "metadata": {
        "namespace": "harvester-public",
        "name": "ubuntu-22",
        "labels": {
          "harvesterhci.io/image-type": "raw"
        }
      },
      "spec": {
        "displayName": "Ubuntu 22.04"
      },
      "status": {
        "storageClassName": "longhorn"
      }
    }
  ]
}
```

#### 5.4.3 掛載點輸出

| 標準輸出欄位（VM Catalog Result） | Provider 欄位 | 重要備註 |
| --- | --- | --- |
| `template_id` | metadata.namespace + metadata.name | 必填；格式 `namespace/name` 作為唯一識別 |
| `name` | spec.displayName 或 metadata.name | 必填；前端顯示名稱 |
| `storage_class` | status.storageClassName | 建立 VM 的 root disk PVC 時使用 |
| `image_type` | harvesterhci.io/image-type label | ISO / raw 等；無值時由 StorageClass 推導 |

### 5.5 Allocation 附加資源：Namespace

Allocation Management 建立 Harvester Allocation 時，Provider Plugin 需確認 namespace 存在。若不存在，建立 namespace；若已存在，補上 HCM 管理 label。

#### 5.5.1 API 清單

| 順序 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 1 | GET | `{baseUrl}/api/v1/namespaces/{namespace}` | 檢查 namespace 是否存在 | 無 body | `metadata.name`、`metadata.labels`、`metadata.resourceVersion` |
| 2 | POST | `{baseUrl}/api/v1/namespaces` | namespace 不存在時建立 | Namespace payload | 建立結果 |
| 3 | GET | `{baseUrl}/api/v1/namespaces/{namespace}` | POST race 或既有 namespace 時取得最新版 | 無 body | `metadata.resourceVersion`、`labels`、`annotations` |
| 4 | PUT | `{baseUrl}/api/v1/namespaces/{namespace}` | namespace 已存在時補 HCM label | Namespace payload，含 resourceVersion | 更新結果 |

#### 5.5.2 上下行範例

**上行 Request 範例（POST）：**

```http
POST {baseUrl}/api/v1/namespaces
Authorization: Bearer <masked>
Content-Type: application/json
```

```json
{
  "apiVersion": "v1",
  "kind": "Namespace",
  "metadata": {
    "name": "erp-web",
    "labels": {
      "hcm-managed": "true",
      "hcm-project-id": "PRJ-ERP",
      "hcm-system-id": "SYS-WEB"
    }
  }
}
```

**上行 Request 範例（PUT）：**

```http
PUT {baseUrl}/api/v1/namespaces/erp-web
Authorization: Bearer <masked>
Content-Type: application/json
```

```json
{
  "apiVersion": "v1",
  "kind": "Namespace",
  "metadata": {
    "name": "erp-web",
    "resourceVersion": "123456",
    "labels": {
      "existing-label": "keep",
      "hcm-managed": "true",
      "hcm-project-id": "PRJ-ERP",
      "hcm-system-id": "SYS-WEB"
    },
    "annotations": {
      "existing-annotation": "keep"
    }
  }
}
```

**下行 Response 範例：**

```json
{
  "metadata": {
    "name": "erp-web",
    "resourceVersion": "123457",
    "labels": {
      "hcm-managed": "true",
      "hcm-project-id": "PRJ-ERP",
      "hcm-system-id": "SYS-WEB"
    }
  }
}
```

#### 5.5.3 掛載點輸出

| 標準輸出欄位（Allocation Extension Result） | Provider 欄位 | 重要備註 |
| --- | --- | --- |
| `namespace_status` | GET/POST/PUT 成功 | 成功時回寫 `ready` |
| `namespace` | namespace 名稱 | 原值保留 |
| `namespace_error` | Provider 錯誤訊息 | 儲存可讀錯誤摘要 |

### 5.6 建立 VM

建立 Harvester VM 不是單一 API。HCM 需依序查重、準備 root disk PVC、準備 cloud-init Secret，最後建立 KubeVirt VirtualMachine。

#### 5.6.1 API 清單

| 順序 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 1 | GET | `{baseUrl}/apis/kubevirt.io/v1/namespaces/{namespace}/virtualmachines/{vmName}` | 檢查 VM 名稱是否已存在 | 無 body | 是否存在 |
| 2 | GET | `{baseUrl}/apis/harvesterhci.io/v1beta1/namespaces/{imageNamespace}/virtualmachineimages/{imageName}` | 若 Image Catalog 沒有 storageClassName，補查 Image | 無 body | `status.storageClassName` |
| 3 | GET | `{baseUrl}/apis/k8s.cni.cncf.io/v1/network-attachment-definitions` | 使用靜態 IP 時，補查 network CIDR / gateway | 無 body | `metadata`、`spec.config`、route annotation |
| 4 | GET | `{baseUrl}/apis/network.harvesterhci.io/v1beta1/ippools` | 使用靜態 IP 時，補查 IPPool | 無 body | `spec.ipv4Config.subnet`、`gateway` |
| 5 | POST | `{baseUrl}/api/v1/namespaces/{namespace}/persistentvolumeclaims` | 建立 root disk PVC | PVC payload | 建立結果 |
| 6 | POST | `{baseUrl}/api/v1/namespaces/{namespace}/secrets` | 建立 cloud-init Secret | Secret payload | 建立結果 |
| 6a | GET | `{baseUrl}/api/v1/namespaces/{namespace}/secrets/{secretName}` | Secret 已存在時取得 resourceVersion | 無 body | `metadata.resourceVersion` |
| 6b | PUT | `{baseUrl}/api/v1/namespaces/{namespace}/secrets/{secretName}` | Secret 已存在時更新 userdata/networkdata | Secret payload，含 resourceVersion | 更新結果 |
| 7 | POST | `{baseUrl}/apis/kubevirt.io/v1/namespaces/{namespace}/virtualmachines` | 建立 KubeVirt VM | VirtualMachine payload | 建立結果 |

#### 5.6.2 掛載點輸入

| HCM 輸入 | 用途 | Harvester 轉換 |
| --- | --- | --- |
| Allocation namespace | VM 所在 namespace | URL path `{namespace}`、metadata.namespace |
| VM name | VM 名稱 | metadata.name、hostname、PVC/Secret 名稱前綴 |
| Image id | root disk 來源 | PVC annotation `harvesterhci.io/imageId` |
| Image storageClassName | root disk StorageClass | PVC spec.storageClassName |
| CPU | VM CPU | `spec.template.spec.domain.cpu.cores` |
| Memory | VM Memory | `spec.template.spec.domain.memory.guest` 與 resource limits/requests |
| Disk size | root disk 大小 | PVC `resources.requests.storage` |
| Subnet cloud_ref | VM network | `multus.networkName` |
| Static IP | VM 固定 IP | cloud-init networkdata 與 VM annotation |

#### 5.6.3 上下行範例

Harvester 建立 VM 為三步驟流程，需依序執行：
1. 建立 root disk PVC
2. 建立或更新 cloud-init Secret
3. 建立 KubeVirt VirtualMachine

#### Step 1: 建立 root disk PVC

**上行 Request 範例：**

```http
POST {baseUrl}/api/v1/namespaces/erp-web/persistentvolumeclaims
Authorization: Bearer <masked>
Content-Type: application/json
```

```json
{
  "apiVersion": "v1",
  "kind": "PersistentVolumeClaim",
  "metadata": {
    "name": "web-01-rootdisk",
    "namespace": "erp-web",
    "annotations": {
      "harvesterhci.io/imageId": "harvester-public/ubuntu-22"
    }
  },
  "spec": {
    "accessModes": ["ReadWriteMany"],
    "volumeMode": "Block",
    "storageClassName": "longhorn",
    "resources": {
      "requests": {
        "storage": "100Gi"
      }
    }
  }
}
```

**下行 Response 範例：**

```json
{
  "metadata": {
    "name": "web-01-rootdisk",
    "namespace": "erp-web"
  },
  "status": {
    "phase": "Pending"
  }
}
```

#### Step 2: 建立或更新 cloud-init Secret

**上行 Request 範例（POST）：**

```http
POST {baseUrl}/api/v1/namespaces/erp-web/secrets
Authorization: Bearer <masked>
Content-Type: application/json
```

```json
{
  "apiVersion": "v1",
  "kind": "Secret",
  "metadata": {
    "name": "web-01-userdata",
    "namespace": "erp-web"
  },
  "type": "Opaque",
  "stringData": {
    "userdata": "#cloud-config\nhostname: web-01\npackages:\n  - qemu-guest-agent\n",
    "networkdata": "version: 1\nconfig:\n  - type: physical\n    name: enp1s0\n    subnets:\n      - type: static\n        address: 10.1.0.10/24\n        gateway: 10.1.0.1\n"
  }
}
```

**上行 Request 範例（PUT，Secret 已存在）：**

```http
PUT {baseUrl}/api/v1/namespaces/erp-web/secrets/web-01-userdata
Authorization: Bearer <masked>
Content-Type: application/json
```

**下行 Response 範例：**

```json
{
  "metadata": {
    "name": "web-01-userdata",
    "namespace": "erp-web",
    "resourceVersion": "567890"
  }
}
```

`networkdata` 只有使用者指定 Static IP 時才需要；若使用 DHCP 或 pod network，可不帶此欄位。

#### Step 3: 建立 KubeVirt VirtualMachine

**上行 Request 範例：**

```http
POST {baseUrl}/apis/kubevirt.io/v1/namespaces/erp-web/virtualmachines
Authorization: Bearer <masked>
Content-Type: application/json
```

```json
{
  "apiVersion": "kubevirt.io/v1",
  "kind": "VirtualMachine",
  "metadata": {
    "name": "web-01",
    "namespace": "erp-web",
    "labels": {
      "harvesterhci.io/creator": "harvester",
      "harvesterhci.io/vmName": "web-01"
    },
    "annotations": {
      "network.harvesterhci.io/ips": "[\"10.1.0.10/24\"]"
    }
  },
  "spec": {
    "running": true,
    "template": {
      "metadata": {
        "annotations": {
          "harvesterhci.io/diskNames": "[\"web-01-rootdisk\"]",
          "harvesterhci.io/sshNames": "[]"
        },
        "labels": {
          "harvesterhci.io/creator": "harvester",
          "harvesterhci.io/vmName": "web-01"
        }
      },
      "spec": {
        "hostname": "web-01",
        "evictionStrategy": "LiveMigrate",
        "domain": {
          "cpu": { "cores": 4 },
          "memory": { "guest": "16Gi" },
          "resources": {
            "limits": { "memory": "16Gi" },
            "requests": { "memory": "16Gi" }
          },
          "devices": {
            "disks": [
              { "name": "rootdisk", "disk": { "bus": "virtio" }, "bootOrder": 1 },
              { "name": "cloudinitdisk", "disk": { "bus": "virtio" } }
            ],
            "interfaces": [
              { "name": "nic0", "model": "virtio", "bridge": {} }
            ],
            "inputs": [
              { "name": "tablet", "type": "tablet", "bus": "usb" }
            ]
          },
          "machine": { "type": "q35" }
        },
        "networks": [
          { "name": "nic0", "multus": { "networkName": "default/vlan100" } }
        ],
        "volumes": [
          { "name": "rootdisk", "persistentVolumeClaim": { "claimName": "web-01-rootdisk" } },
          {
            "name": "cloudinitdisk",
            "cloudInitNoCloud": {
              "secretRef": { "name": "web-01-userdata" },
              "networkDataSecretRef": { "name": "web-01-userdata" }
            }
          }
        ]
      }
    }
  }
}
```

**下行 Response 範例：**

```json
{
  "metadata": {
    "name": "web-01",
    "namespace": "erp-web"
  },
  "status": {
    "printableStatus": "Starting"
  }
}
```

若使用 pod network 而不是指定 Harvester network，`interfaces` 使用 `masquerade`，`networks` 使用 `pod`：

```json
{
  "interfaces": [
    { "name": "nic0", "model": "virtio", "masquerade": {} }
  ],
  "networks": [
    { "name": "nic0", "pod": {} }
  ]
}
```

#### 5.6.4 掛載點輸出

Harvester 建立 VM 成功後，HCM 不需要保存完整 response；需形成以下標準結果：

```json
{
  "instanceId": "erp-web/web-01",
  "state": "provisioning",
  "privateIpAddress": "10.1.0.10"
}
```

| 標準輸出欄位（Create VM Result） | Provider 欄位 | 重要備註 |
| --- | --- | --- |
| `provider_vm_id` | namespace/name | 後續開關機、poll 狀態使用 |
| `status` | 建立成功 | 初始為 `provisioning` |
| `ip` | static IP | 若使用者有指定 IP，先寫入 HCM；後續 poll 再更新 |

### 5.7 同步 VM 清單

#### 5.7.1 API 清單

| 順序 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 1 | GET | `{baseUrl}/apis/kubevirt.io/v1/virtualmachines` | 取得所有 VM | 無 body | `items[].metadata`、`spec.template.spec`、`status` |
| 2 | GET | `{baseUrl}/apis/kubevirt.io/v1/virtualmachineinstances` | 取得執行中 VM instance 與 IP | 無 body | `items[].metadata`、`status.interfaces` |

若 Connection 設定了 Namespace Filter，HCM 在取得 VM 後只保留指定 namespace 的 VM。

#### 5.7.2 上下行範例

**上行 Request 範例：**

```http
GET {baseUrl}/apis/kubevirt.io/v1/virtualmachines
Authorization: Bearer <masked>
```

**下行 Response 範例：**

```json
{
  "items": [
    {
      "metadata": {
        "namespace": "erp-web",
        "name": "web-01",
        "uid": "8f0b..."
      },
      "status": {
        "printableStatus": "Running"
      },
      "spec": {
        "template": {
          "spec": {
            "domain": {
              "cpu": { "cores": 4 },
              "memory": { "guest": "16Gi" }
            }
          }
        }
      }
    }
  ]
}
```

#### 5.7.3 掛載點輸出

| 標準輸出欄位（VM Inventory Result） | Provider 欄位 | 重要備註 |
| --- | --- | --- |
| `provider_vm_id` | `metadata.namespace` + `/` + `metadata.name` | 必填；作為 provider 端 VM 識別，後續開關機與狀態追蹤使用 |
| `name` | `metadata.name` | 必填；直接作為 VM 顯示名稱 |
| `status` | `status.printableStatus`（VM / VMI） | 必填；需先轉成 HCM 標準狀態 |
| `cpu` | `spec.template.spec.domain.cpu.cores` | Core / vCPU |
| `memory_gb` | `spec.template.spec.domain.memory.guest` | 需由 Gi/Mi 等單位換算為 GB |
| `disk_gb` | `spec.template.spec.domain.devices.disks` + volume/PVC 容量資訊 | 若來源未提供完整容量可留空 |
| `ip` | `status.interfaces[].ipAddress`（VMI） | 主要 IP；通常取第一張主要 NIC |
| `hostname` | VM 名稱或 cloud-init 設定值 | 無獨立欄位時可回退 `name` |
| `nics` | `spec.template.spec.networks` + `status.interfaces[]` | 需合併 network 名稱與 IP |
| `disks` | `spec.template.spec.domain.devices.disks` | provider 支援程度不同 |
| `tags` | `metadata.labels` | 用於歸屬與篩選 |
| `ref.id` | `namespace/name`（= `vm.cloud_ref`） | 重複同步識別既有 VM；避免重複建立 |
| `ref.name` | `metadata.name` | 備援識別 |
| `ref.namespace` | `metadata.namespace` | Harvester 特有；後續開關機 API 需要 namespace |
| `ref.cloud_connection_id` | Connection ID | 隔離不同連線的 VM |

### 5.8 VM 狀態追蹤

#### 5.8.1 API 清單

| 順序 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 1 | GET | `{baseUrl}/apis/kubevirt.io/v1/namespaces/{namespace}/virtualmachines/{vmName}` | 查詢 VM 定義與狀態 | 無 body | `status.printableStatus`、`spec.running` |
| 2 | GET | `{baseUrl}/apis/kubevirt.io/v1/namespaces/{namespace}/virtualmachineinstances/{vmName}` | 查詢執行中 instance 與 IP | 無 body | `status.interfaces[].ipAddress` |

#### 5.8.2 上下行範例

**上行 Request 範例：**

```http
GET {baseUrl}/apis/kubevirt.io/v1/namespaces/erp-web/virtualmachines/web-01
Authorization: Bearer <masked>
```

```http
GET {baseUrl}/apis/kubevirt.io/v1/namespaces/erp-web/virtualmachineinstances/web-01
Authorization: Bearer <masked>
```

**下行 Response 範例：**

```json
{
  "status": {
    "printableStatus": "Running",
    "interfaces": [
      {
        "name": "nic0",
        "ipAddress": "10.1.0.10"
      }
    ]
  }
}
```

#### 5.8.3 掛載點輸出

| 標準輸出欄位（VM Status Result） | Provider 欄位 | 重要備註 |
| --- | --- | --- |
| `status` | `status.printableStatus` / `spec.running` | 轉成 HCM 標準狀態（running/stopped/starting/stopping） |
| `ip` | `status.interfaces[].ipAddress`（VMI） | 主要 IP；通常取第一張主要 NIC |
| `nics` | `status.interfaces[]` | 顯示 VM 目前 NIC 與 IP |

### 5.9 VM 開機 / 關機

#### 5.9.1 API 清單

| 動作 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 開機 | PUT | `{baseUrl}/apis/subresources.kubevirt.io/v1/namespaces/{namespace}/virtualmachines/{vmName}/start` | 啟動 VM | 無 body | 呼叫成功與否 |
| 關機 | PUT | `{baseUrl}/apis/subresources.kubevirt.io/v1/namespaces/{namespace}/virtualmachines/{vmName}/stop` | 停止 VM | 無 body | 呼叫成功與否 |

#### 5.9.2 上下行範例

**上行 Request 範例：**

```http
PUT {baseUrl}/apis/subresources.kubevirt.io/v1/namespaces/erp-web/virtualmachines/web-01/start
Authorization: Bearer <masked>
```

```http
PUT {baseUrl}/apis/subresources.kubevirt.io/v1/namespaces/erp-web/virtualmachines/web-01/stop
Authorization: Bearer <masked>
```

**下行 Response 範例：**

```json
{
  "kind": "Status",
  "status": "Success"
}
```

#### 5.9.3 掛載點輸出

| 標準輸出欄位（VM Power Result） | Provider 欄位 | 重要備註 |
| --- | --- | --- |
| `status` | start/stop 呼叫成功 | start 成功先回 `starting`；stop 成功先回 `stopping`，後續由狀態追蹤校正 |
| `provider_vm_id` | `namespace/vmName` | 後續 VM 狀態追蹤與重試操作使用 |

## 6. 單位換算與狀態映射

### 6.1 單位換算

| Provider 資料 | HCM 資料 | 換算規則 | 範例 |
| --- | --- | --- | --- |
| CPU Core | CPU Core | 原則上 1:1 | `4` -> `4 Core` |
| Memory bytes / Ki / Mi / Gi | Memory GB | 轉成 GB 顯示 | `17179869184 bytes` -> `16 GB` |
| Storage bytes / Gi | Disk TB 或 GB | Pool 容量轉 TB；VM Disk 轉 GB | `1024 Gi` -> `1 TB` |
| Image storageClassName | Image metadata | 原值保留 | `longhorn` |

### 6.2 VM 狀態映射

| Harvester / KubeVirt 狀態 | HCM 標準狀態 | 說明 |
| --- | --- | --- |
| `Running` / `running` | `running` | VM 已執行 |
| `Starting` | `starting` | VM 開機中 |
| `Stopping` | `stopping` | VM 關機中 |
| `Stopped` / `PowerOff` | `stopped` | VM 已停止 |
| 建立中或狀態未定 | `provisioning` | HCM 尚未取得穩定狀態 |

### 6.3 Namespace 狀態映射

| Provider 結果 | HCM Allocation namespace_status | 說明 |
| --- | --- | --- |
| 尚未建立或等待處理 | `pending` | Allocation 已填 namespace，但 Provider 尚未確認 |
| Namespace 存在或建立成功 | `ready` | VM 建立可使用此 namespace |
| 建立或更新失敗 | `error` | 需顯示 namespace_error 供管理者處理 |

## 7. 限制與待確認

| 項目 | 說明 | 建議確認方向 |
| --- | --- | --- |
| Pool 拆分粒度 | 目前以 Cluster 作為主要 Pool 概念 | 是否需依 Node Pool、StorageClass 或 Namespace 拆分 |
| Namespace 建立時點 | 可在 Allocation 建立時處理，也可延到 VM 建立前處理 | 建議由業務決定；若要讓 VM Management 更穩定，建議 Allocation 時先建立 |
| Namespace 命名規則 | 需符合 Kubernetes DNS-1123 | 建議使用 project-system 形式並保留人工調整 |
| Security Group | Harvester 無 AWS Security Group 概念 | 若需要網路安全政策，需另定義對應資源 |
| Image legacy 判斷 | ISO 或無 storageClassName 可能不適合直接作 root disk | 建議在 VM 建立畫面提示 |
| VM 刪除 | 本文件描述建立、開關機與狀態追蹤 | VM 刪除時是否清除 PVC / Secret 需另行確認 |
| 靜態 IP | Harvester network 是否支援靜態 IP 取決於環境設定 | Provider 文件先標示為可選能力，實際支援需與平台確認 |
