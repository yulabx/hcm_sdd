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

| 02 規範掛載點 | Harvester 是否支援 | Harvester 文件章節 | 標準輸入 | 標準輸出 | 外部 API 章節 | 影響的 HCM 資料 |
| --- | --- | --- | --- | --- | --- | --- |
| Provider 授權 | 支援 | Auth 與連線設定、Provider 能力支援矩陣 | Connection Auth Input | Auth Result | 授權/登入 | Cloud Connection |
| 同步資源池 | 支援 | Pool 映射、Provider 能力支援矩陣 | Sync Pools Input | Pool Sync Result | 同步 Pool | Pool |
| 同步網路 | 支援 | Network / Subnet 映射、Provider 能力支援矩陣 | Sync Network Input | Network Sync Result | 同步 Network | Subnet |
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

## 4. Provider 能力支援矩陣

| 能力 | 是否支援 | Provider 資料來源 | 產生/更新的 HCM 資料 |
| --- | --- | --- | --- |
| 授權/登入 | 支援 | API Token | Cloud Connection 授權狀態 |
| 同步資源池 | 支援 | Cluster / Node / Longhorn 容量 | Pool |
| 同步網路 | 支援 | NetworkAttachmentDefinition | Subnet |
| 同步 VM 規格來源 | 支援 | VirtualMachineImage | Pool Image Catalog |
| 同步安全群組 | 不支援 | Harvester 無對應 AWS Security Group 概念 | 無 |
| 同步 VM 清單 | 支援 | KubeVirt VirtualMachine / VirtualMachineInstance | VM |
| 建立 Allocation 附加資源 | 支援 | Kubernetes Namespace | Allocation namespace 與 namespace 狀態 |
| 建立 VM | 支援 | KubeVirt VirtualMachine、PVC、Secret | VM |
| VM 開機/關機 | 支援 | KubeVirt subresource start / stop | VM 狀態 |
| VM 狀態追蹤 | 支援 | VirtualMachine / VirtualMachineInstance | VM 狀態與 IP |

## 5. Provider 到 HCM 欄位映射

### 5.1 Pool 映射

Harvester 的 HCM Pool 通常代表一個 Cluster 可被 HCM 管理的整體容量範圍。若未來要把 Node Pool、StorageClass 或 Namespace 拆成多個 Pool，需在本章補充拆分規則。

| Provider 欄位/概念 | Provider 意義 | HCM 欄位 | HCM 意義 | 轉換規則 | 範例 |
| --- | --- | --- | --- | --- | --- |
| Cluster Endpoint | Harvester Cluster 來源 | Pool cloud_ref | Provider 來源識別 | 以連線或 Cluster 識別形成 Pool 來源 | `harvester-tpe` |
| Node CPU | Cluster 可用 CPU | Pool CPU total | HCM 可配置 CPU | 轉成 CPU Core | `64 Core` |
| Node Memory | Cluster 記憶體 | Pool Memory total | HCM 可配置 Memory | 轉成 GB | `512 GB` |
| Longhorn Storage | Cluster 儲存容量 | Pool Disk total | HCM 可配置 Disk | 轉成 TB | `20 TB` |
| Cloud Connection | 來源連線 | Pool ref.cloud_connection_id | 後續追蹤同步來源 | 保留 HCM Connection ID | `harvester-tpe-01` |

### 5.2 Network / Subnet 映射

Harvester 的網路來源以 NetworkAttachmentDefinition 為主，HCM 將其轉為 Subnet 供 VM 建立時選擇。

| Provider 欄位/概念 | Provider 意義 | HCM 欄位 | HCM 意義 | 轉換規則 | 範例 |
| --- | --- | --- | --- | --- | --- |
| metadata.namespace / metadata.name | Harvester network 識別 | Subnet cloud_ref | Provider 來源識別 | 組成 `namespace/name` | `default/vlan100` |
| network name | 網路名稱 | Subnet name | HCM 顯示名稱 | 加上類型與 CIDR 提示 | `[APP] vlan100 (10.1.0.0/24)` |
| route annotation | 路由資訊 | Subnet cidr / gateway | HCM 網段資訊 | 從 annotation 取 CIDR 與 gateway | `10.1.0.0/24` |
| HCM Pool | 網路可用範圍 | Subnet pool_ids | 可被哪些 Pool 使用 | 依同步來源連到 Harvester Pool | `pool-harvester-tpe` |
| 管理者補填類型 | HCM 業務分類 | Subnet type | APP / DB / Management 等類型 | 同步後由管理者確認 | `app` |

### 5.3 Image Catalog 映射

Harvester 使用 VirtualMachineImage 作為 VM 建立來源。HCM 顯示為 Image Catalog，而不是 Template。

| Provider 欄位/概念 | Provider 意義 | HCM 欄位 | HCM 意義 | 轉換規則 | 範例 |
| --- | --- | --- | --- | --- | --- |
| metadata.namespace / metadata.name | Image 識別 | Image id | HCM Image 選項識別 | 組成 `namespace/name` | `harvester-public/ubuntu-22` |
| spec.displayName | Image 顯示名稱 | Image label | 使用者可讀名稱 | 優先使用 displayName | `Ubuntu 22.04` |
| status.storageClassName | Image 所在 StorageClass | storageClassName | 建立 VM root disk 時參考 | 保留原值 | `longhorn` |
| label image-type | Image 類型 | imageType / is_legacy | 判斷是否為 ISO 或可直接用於磁碟 | ISO 或無 StorageClass 視為 legacy | `raw` |

### 5.4 VM 映射

| Provider 欄位/概念 | Provider 意義 | HCM 欄位 | HCM 意義 | 轉換規則 | 範例 |
| --- | --- | --- | --- | --- | --- |
| metadata.uid 或 namespace/name | VM 來源識別 | VM provider_ref / cloud_ref | Provider VM 識別 | 優先保留 UID，操作時需可回推 namespace/name | `default/web-01` |
| metadata.namespace | VM 所在 Namespace | VM namespace | Allocation / VM 所屬隔離範圍 | 保留 namespace | `erp-prod` |
| metadata.name | VM 名稱 | VM name / hostname | HCM VM 名稱 | 原樣保留 | `web-01` |
| spec domain cpu | VM CPU | VM CPU | VM 規格 | 轉成 Core | `4` |
| spec domain memory | VM Memory | VM Memory | VM 規格 | 轉成 GB | `16 GB` |
| volumes / disks | VM Disk | VM Disk | VM 磁碟容量 | 轉成 GB | `100 GB` |
| VMI network interfaces | VM IP | VM IP | VM 連線資訊 | 取主要 IP | `10.1.0.10` |

## 6. 功能畫面差異

Harvester 的差異集中在 Cloud Settings 的連線/同步、Allocation Management 的 namespace，以及 VM Management 的 Image 與網路欄位。Harvester 不支援 AWS Security Group；若 HCM 畫面有安全群組區塊，Harvester 不提供可同步或可選的 provider 原生安全群組。

| 功能 | 畫面/流程位置 | 畫面差異 | 欄位/選項差異 | 可操作能力 | 業務限制/提示 |
| --- | --- | --- | --- | --- | --- |
| Cloud Settings | Connection 表單 | 顯示 Cluster 設定語意 | Base URL、Namespace Filter、TLS Skip Verify、API Token | 可建立/更新連線 | Token 必須遮罩；自簽憑證可跳過 TLS |
| Cloud Settings | 同步資料 Wizard | Template Step 顯示為 Image | 同步 VirtualMachineImage | 可同步 Pool、Network、Image、VM | Security Group 不適用 |
| Allocation Management | Shared Allocation 表單 | 顯示 Provider Extension 區塊 | Namespace 欄位與 namespace 狀態 | 可填寫或使用建議 namespace；Configure / Execute 生效時建立或更新 namespace | namespace 需符合 Kubernetes 命名規則 |
| VM Management | VM 建立表單 | 顯示 Image 選項 | Image 來源為 Pool Image Catalog | 可建立 VM | Image 若為 legacy 需提示是否可用 |
| VM Management | Network / Security Group | 不顯示 Harvester 原生 Security Group 選項 | Network 來自 Harvester Subnet | 可選 IP 模式與網路 | Harvester 不支援 AWS Security Group |
| VM Management | VM 操作 | 支援 Start / Stop / 狀態追蹤 | instanceId 需帶 namespace/name | 可開機、關機、查詢狀態 | 操作結果以 KubeVirt 狀態回寫 HCM |

## 7. 外部 API 上下行與範例

本章列出 Harvester Provider Plugin 需要呼叫的外部 API。URL 皆以 Connection 表單的 `Base URL` 為前綴，例如 `{baseUrl} = https://harvester.example.com`。所有 request 皆需帶入授權 header：

| Header | 值 | 說明 |
| --- | --- | --- |
| Authorization | `Bearer <masked>` | Harvester API Token，一律遮罩 |
| Content-Type | `application/json` | POST / PUT request 使用 |

### 7.1 授權/登入

Harvester 不需要額外登入換 token。HCM 使用 Connection 表單保存的 API Token 直接呼叫後續 API。

| 項目 | 說明 |
| --- | --- |
| Method | 不適用，無獨立登入 API |
| URL | 不適用 |
| 上行 | 後續 API request header 帶 `Authorization: Bearer <masked>` |
| 下行 | 由各資源 API 回傳 200/201/202 或錯誤狀態 |
| HCM 取用 | Token 是否可用由同步或操作 API 的結果判斷 |

### 7.2 同步 Pool

同步 Pool 時，HCM 會先以 Harvester Cluster 形成一個 Pool 來源，再查 Kubernetes Node 與 Longhorn 容量，轉成 HCM Pool 的 CPU、Memory、Disk。

#### 7.2.1 API 清單

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

#### 7.2.2 上行範例

```http
GET {baseUrl}/api/v1/nodes
Authorization: Bearer <masked>
```

#### 7.2.3 下行摘要範例

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

#### 7.2.4 HCM 轉換

| HCM 欄位 | 來源 API | 來源欄位 | 換算公式 |
| --- | --- | --- | --- |
| `cpu_total` | nodes | `status.allocatable.cpu`（無則 `capacity.cpu`） | `sum(parseCpuQuantity(cpu))` |
| `cpu_provisioned` | nodes | `annotations['management.cattle.io/pod-requests'].cpu` | `sum(parseCpuQuantity(podRequests.cpu))` |
| `mem_total_gb` | nodes | `status.allocatable.memory`（無則 `capacity.memory`） | `sum(parseMemoryToGb(memory))` |
| `mem_provisioned_gb` | nodes | `annotations['management.cattle.io/pod-requests'].memory` | `sum(parseMemoryToGb(podRequests.memory))` |
| `disk_total_tb` | Longhorn nodes + overcommit settings | `storageMaximum`、`storageReserved`、`overcommit` | `sum( max(0, storageMaximum - storageReserved) × (overcommit/100) )` 轉 TB |
| `disk_provisioned_tb` | Longhorn nodes | `storageScheduled` | `sum(storageScheduled)` 轉 TB |

**Overcommit 規則：**
- overcommit 只影響 `disk_total_tb`，不影響 `disk_provisioned_tb`
- 若 settings API 全部 fallback 失敗，預設 overcommit = 200

**呼叫策略：**
- nodes、Longhorn nodes、overcommit settings 三類 API 並行呼叫
- Longhorn nodes 與 overcommit settings 各自依序 fallback，第一個成功即採用

### 7.3 同步 Network

同步 Network 時，HCM 讀取 Harvester NetworkAttachmentDefinition 與 IPPool，轉為 HCM Subnet。

#### 7.3.1 API 清單

| 順序 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 1 | GET | `{baseUrl}/apis/k8s.cni.cncf.io/v1/network-attachment-definitions` | 取得 Harvester 可用網路 | 無 body | `items[].metadata.namespace`、`name`、`annotations`、`labels`、`spec.config` |
| 2 | GET | `{baseUrl}/apis/network.harvesterhci.io/v1beta1/ippools` | 取得網路對應 IPPool | 無 body | `items[].metadata.name`、`spec.ipv4Config.subnet`、`gateway` |

#### 7.3.2 上行範例

```http
GET {baseUrl}/apis/k8s.cni.cncf.io/v1/network-attachment-definitions
Authorization: Bearer <masked>
```

#### 7.3.3 下行摘要範例

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

#### 7.3.4 HCM 轉換

| Provider 來源 | HCM 欄位 | 轉換規則 |
| --- | --- | --- |
| metadata.namespace + metadata.name | Subnet cloud_ref | 轉成 `namespace/name` |
| route annotation cidr | Subnet cidr | 優先取 route annotation |
| IPPool ipv4Config.subnet | Subnet cidr | route annotation 無 CIDR 時使用 |
| spec.config.ipam.subnet | Subnet cidr | 前兩者無資料時使用 |
| route / IPPool / ipam gateway | Subnet gateway | 依同樣優先序取值 |
| network.harvesterhci.io/type / vlan-id | Subnet name 補充資訊 | 顯示 VLAN 或 Bridge |

### 7.4 同步 Image

同步 Image 時，HCM 查詢 Harvester VirtualMachineImage，寫入 Pool 的 Image Catalog。

#### 7.4.1 API 清單

| 順序 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 1 | GET | `{baseUrl}/apis/harvesterhci.io/v1beta1/virtualmachineimages` | 取得 VM 建立可用 Image | 無 body | `items[].metadata.namespace`、`metadata.name`、`metadata.labels`、`spec.displayName`、`status.storageClassName` |
| 2 | GET | `{baseUrl}/apis/harvesterhci.io/v1beta1/namespaces/{imageNamespace}/virtualmachineimages/{imageName}` | 建立 VM 前補查單一 Image 的 StorageClass | 無 body | `status.storageClassName` |

#### 7.4.2 上行範例

```http
GET {baseUrl}/apis/harvesterhci.io/v1beta1/virtualmachineimages
Authorization: Bearer <masked>
```

#### 7.4.3 下行摘要範例

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

#### 7.4.4 HCM 轉換

| Provider 來源 | HCM 欄位 | 轉換規則 |
| --- | --- | --- |
| metadata.namespace + metadata.name | Image id | 轉成 `namespace/name` |
| spec.displayName | Image label | 優先使用；無值時用 metadata.name |
| status.storageClassName | Image storageClassName | 原值保留，建立 root disk PVC 時使用 |
| harvesterhci.io/image-type | Image imageType / is_legacy | ISO 或無 StorageClass 標記為 legacy |

### 7.5 Allocation 附加資源：Namespace

Allocation Management 建立 Harvester Allocation 時，Provider Plugin 需確認 namespace 存在。若不存在，建立 namespace；若已存在，補上 HCM 管理 label。

#### 7.5.1 API 清單

| 順序 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 1 | GET | `{baseUrl}/api/v1/namespaces/{namespace}` | 檢查 namespace 是否存在 | 無 body | `metadata.name`、`metadata.labels`、`metadata.resourceVersion` |
| 2 | POST | `{baseUrl}/api/v1/namespaces` | namespace 不存在時建立 | Namespace payload | 建立結果 |
| 3 | GET | `{baseUrl}/api/v1/namespaces/{namespace}` | POST race 或既有 namespace 時取得最新版 | 無 body | `metadata.resourceVersion`、`labels`、`annotations` |
| 4 | PUT | `{baseUrl}/api/v1/namespaces/{namespace}` | namespace 已存在時補 HCM label | Namespace payload，含 resourceVersion | 更新結果 |

#### 7.5.2 POST 上行範例

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

#### 7.5.3 PUT 上行範例

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

#### 7.5.4 HCM 轉換

| Provider 結果 | HCM 欄位 | 轉換規則 |
| --- | --- | --- |
| GET/POST/PUT 成功 | Allocation namespace_status | `ready` |
| namespace 名稱 | Allocation namespace | 原值保留 |
| Provider 錯誤訊息 | Allocation namespace_error | 儲存可讀錯誤摘要 |

### 7.6 建立 VM

建立 Harvester VM 不是單一 API。HCM 需依序查重、準備 root disk PVC、準備 cloud-init Secret，最後建立 KubeVirt VirtualMachine。

#### 7.6.1 API 清單

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

#### 7.6.2 HCM 標準輸入

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

#### 7.6.3 建立 PVC 上行範例

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

#### 7.6.4 建立 Secret 上行範例

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

`networkdata` 只有使用者指定 Static IP 時才需要；若使用 DHCP 或 pod network，可不帶此欄位。

#### 7.6.5 建立 VirtualMachine 上行範例

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

#### 7.6.6 下行與 HCM 取用

Harvester 建立 VM 成功後，HCM 不需要保存完整 response；需形成以下標準結果：

```json
{
  "instanceId": "erp-web/web-01",
  "state": "provisioning",
  "privateIpAddress": "10.1.0.10"
}
```

| Provider 結果 | HCM 欄位 | 說明 |
| --- | --- | --- |
| namespace/name | VM provider instance id | 後續開關機、poll 狀態使用 |
| 建立成功 | VM status | 初始為 `provisioning` |
| static IP | VM IP | 若使用者有指定 IP，先寫入 HCM；後續 poll 再更新 |

### 7.7 同步 VM 清單

#### 7.7.1 API 清單

| 順序 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 1 | GET | `{baseUrl}/apis/kubevirt.io/v1/virtualmachines` | 取得所有 VM | 無 body | `items[].metadata`、`spec.template.spec`、`status` |
| 2 | GET | `{baseUrl}/apis/kubevirt.io/v1/virtualmachineinstances` | 取得執行中 VM instance 與 IP | 無 body | `items[].metadata`、`status.interfaces` |

若 Connection 設定了 Namespace Filter，HCM 在取得 VM 後只保留指定 namespace 的 VM。

#### 7.7.2 上行範例

```http
GET {baseUrl}/apis/kubevirt.io/v1/virtualmachines
Authorization: Bearer <masked>
```

#### 7.7.3 下行摘要範例

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

### 7.8 VM 狀態追蹤

#### 7.8.1 API 清單

| 順序 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 1 | GET | `{baseUrl}/apis/kubevirt.io/v1/namespaces/{namespace}/virtualmachines/{vmName}` | 查詢 VM 定義與狀態 | 無 body | `status.printableStatus`、`spec.running` |
| 2 | GET | `{baseUrl}/apis/kubevirt.io/v1/namespaces/{namespace}/virtualmachineinstances/{vmName}` | 查詢執行中 instance 與 IP | 無 body | `status.interfaces[].ipAddress` |

#### 7.8.2 HCM 取用

| Provider 結果 | HCM 欄位 | 說明 |
| --- | --- | --- |
| printableStatus / running | VM status | 轉成 HCM 標準狀態 |
| VMI interfaces | VM IP / NIC | 顯示 VM 目前 IP |

### 7.9 VM 開機 / 關機

#### 7.9.1 API 清單

| 動作 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 開機 | PUT | `{baseUrl}/apis/subresources.kubevirt.io/v1/namespaces/{namespace}/virtualmachines/{vmName}/start` | 啟動 VM | 無 body | 呼叫成功與否 |
| 關機 | PUT | `{baseUrl}/apis/subresources.kubevirt.io/v1/namespaces/{namespace}/virtualmachines/{vmName}/stop` | 停止 VM | 無 body | 呼叫成功與否 |

#### 7.9.2 上行範例

```http
PUT {baseUrl}/apis/subresources.kubevirt.io/v1/namespaces/erp-web/virtualmachines/web-01/start
Authorization: Bearer <masked>
```

```http
PUT {baseUrl}/apis/subresources.kubevirt.io/v1/namespaces/erp-web/virtualmachines/web-01/stop
Authorization: Bearer <masked>
```

#### 7.9.3 HCM 取用

| Provider 結果 | HCM 欄位 | 說明 |
| --- | --- | --- |
| start 呼叫成功 | VM status | 先更新為 `starting`，後續由 poll 校正 |
| stop 呼叫成功 | VM status | 先更新為 `stopping`，後續由 poll 校正 |

## 8. 單位換算與狀態映射

### 8.1 單位換算

| Provider 資料 | HCM 資料 | 換算規則 | 範例 |
| --- | --- | --- | --- |
| CPU Core | CPU Core | 原則上 1:1 | `4` -> `4 Core` |
| Memory bytes / Ki / Mi / Gi | Memory GB | 轉成 GB 顯示 | `17179869184 bytes` -> `16 GB` |
| Storage bytes / Gi | Disk TB 或 GB | Pool 容量轉 TB；VM Disk 轉 GB | `1024 Gi` -> `1 TB` |
| Image storageClassName | Image metadata | 原值保留 | `longhorn` |

### 8.2 VM 狀態映射

| Harvester / KubeVirt 狀態 | HCM 標準狀態 | 說明 |
| --- | --- | --- |
| `Running` / `running` | `running` | VM 已執行 |
| `Starting` | `starting` | VM 開機中 |
| `Stopping` | `stopping` | VM 關機中 |
| `Stopped` / `PowerOff` | `stopped` | VM 已停止 |
| 建立中或狀態未定 | `provisioning` | HCM 尚未取得穩定狀態 |

### 8.3 Namespace 狀態映射

| Provider 結果 | HCM Allocation namespace_status | 說明 |
| --- | --- | --- |
| 尚未建立或等待處理 | `pending` | Allocation 已填 namespace，但 Provider 尚未確認 |
| Namespace 存在或建立成功 | `ready` | VM 建立可使用此 namespace |
| 建立或更新失敗 | `error` | 需顯示 namespace_error 供管理者處理 |

## 9. 限制與待確認

| 項目 | 說明 | 建議確認方向 |
| --- | --- | --- |
| Pool 拆分粒度 | 目前以 Cluster 作為主要 Pool 概念 | 是否需依 Node Pool、StorageClass 或 Namespace 拆分 |
| Namespace 建立時點 | 可在 Allocation 建立時處理，也可延到 VM 建立前處理 | 建議由業務決定；若要讓 VM Management 更穩定，建議 Allocation 時先建立 |
| Namespace 命名規則 | 需符合 Kubernetes DNS-1123 | 建議使用 project-system 形式並保留人工調整 |
| Security Group | Harvester 無 AWS Security Group 概念 | 若需要網路安全政策，需另定義對應資源 |
| Image legacy 判斷 | ISO 或無 storageClassName 可能不適合直接作 root disk | 建議在 VM 建立畫面提示 |
| VM 刪除 | 本文件描述建立、開關機與狀態追蹤 | VM 刪除時是否清除 PVC / Secret 需另行確認 |
| 靜態 IP | Harvester network 是否支援靜態 IP 取決於環境設定 | Provider 文件先標示為可選能力，實際支援需與平台確認 |
