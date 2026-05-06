# 03 AWS Provider Plugin

## 1. Provider 基本資訊

AWS Provider Plugin 用於將 AWS EC2 / VPC 資源轉成 HCM 標準資料，並支援在 VM Management 建立、追蹤、開機與關機 EC2 Instance。

HCM 中的 Cloud / Provider ID 可以是業務命名；Driver ID 固定為 `aws`，用來套用本文件定義的 Provider Plugin 規格。

| 項目 | 說明 |
| --- | --- |
| Provider 名稱 | AWS |
| Provider 類型 | 公有雲 |
| HCM Provider Driver | `aws` |
| 主要資源概念 | Region、VPC、Subnet、Security Group、AMI、EC2 Instance |
| HCM 主要對應資料 | Cloud Connection、Pool、Subnet、Security Group、Image Catalog、VM |
| 主要使用功能 | Cloud Settings、VM Management |

## 2. 共同規範勾稽

### 2.1 掛載點對照表

| 02 規範掛載點 | AWS 是否支援 | AWS 文件章節 | 標準輸入 | 標準輸出 | 外部 API 章節 | 影響的 HCM 資料 |
| --- | --- | --- | --- | --- | --- | --- |
| Provider 授權 | 支援 | Auth 與連線設定 | Connection Auth Input | Auth Result | 授權/登入 | Cloud Connection |
| 同步資源池 | 支援 | VPC / Pool 映射 | Sync Pools Input | Pool Sync Result | 同步 VPC / Pool | Pool |
| 同步網路 | 支援 | Network / Subnet / Security Group 映射 | Sync Network Input | Network Sync Result | 同步 Network / Security Group | Subnet、Security Group |
| 同步 VM 規格來源 | 支援 | Image Catalog 映射 | Sync VM Catalog Input | VM Catalog Result | 同步 AMI / Image | Pool VM Catalog |
| 同步 VM 清單 | 支援 | VM 映射、狀態映射 | Sync VM Inventory Input | VM Inventory Result | 同步 EC2 VM 清單 | VM |
| Allocation 附加資源 | 不支援 | Provider 能力支援矩陣 | Allocation Extension Input | Allocation Extension Result | 不支援 | 無 |
| 建立 VM | 支援 | 功能畫面差異、VM 映射 | Create VM Input | Create VM Result | 建立 VM | VM |
| VM 開機 | 支援 | 狀態映射 | VM Power Input | VM Power Result | VM 開機 / 關機 | VM |
| VM 關機 | 支援 | 狀態映射 | VM Power Input | VM Power Result | VM 開機 / 關機 | VM |
| VM 狀態追蹤 | 支援 | VM 映射、狀態映射 | VM Status Input | VM Status Result | VM 狀態追蹤 | VM |

### 2.2 標準輸入/輸出落地表

| 02 標準資料 | AWS 對應欄位/概念 | Provider 文件章節 | 轉換或限制 |
| --- | --- | --- | --- |
| Connection Auth Input | Access Key ID、Secret Access Key、Region、Session Token | Auth 與連線設定 | Secret / Session Token 必須遮罩 |
| Auth Result | AWS credential 可用狀態、Region | 授權/登入 | 以 DescribeRegions 驗證 region 與 credential |
| Sync Pools Input | Region、Cloud Connection | 同步 VPC / Pool | AWS 以 VPC 作為 HCM Pool |
| Pool Sync Result | VPC | VPC / Pool 映射 | AWS VPC 不直接回 CPU/Memory/Disk 容量，容量欄位可為空 |
| Sync Network Input | VPC id | 同步 Network / Security Group | 同步 Subnet 與 Security Group |
| Network Sync Result | DescribeSubnets、DescribeSecurityGroups | Network / Subnet / Security Group 映射 | AWS 支援 Security Group |
| Sync VM Catalog Input | Region、AMI owner/filter | 同步 AMI / Image | 取 self AMI 與 quick start public AMI |
| VM Catalog Result | AMI | Image Catalog 映射 | Image id 為 AMI ID，保留 rootDeviceName |
| Sync VM Inventory Input | VPC id | 同步 EC2 VM 清單 | DescribeInstances 依 VPC 過濾 |
| VM Inventory Result | EC2 Instance、InstanceType | VM 映射 | DescribeInstanceTypes 補 vCPU / Memory |
| Allocation Extension Input / Result | 不適用 | Provider 能力支援矩陣 | AWS Allocation 不建立外部附加資源 |
| Create VM Input | AMI、Instance Type、Subnet、Security Group、Disk、Tags | 建立 VM | 轉成 RunInstances；Security Group 僅 AWS 支援 |
| Create VM Result | InstanceId、state、PrivateIpAddress | 建立 VM | 建立後進入 provisioning，後續 poll 更新狀態與 IP |
| VM Power Input / Result | InstanceId、start/stop | VM 開機 / 關機 | StartInstances / StopInstances |
| VM Status Input / Result | InstanceId | VM 狀態追蹤 | DescribeInstances 取得 state、IP、NIC |

## 3. Auth 與連線設定

AWS 連線以 Access Key 為主，Region 為必要欄位。AWS Provider 不使用 Basic、Token 或 Service Account 授權差異區。

| 欄位 | 業務意義 | 必填 | 範例 | 備註 |
| --- | --- | --- | --- | --- |
| Cloud | 選擇 AWS Provider | 是 | AWS |
| Label | HCM 內顯示的連線名稱 | 是 | AWS Tokyo Prod |
| Region | AWS Region | 是 | `ap-northeast-1` | 決定 EC2 endpoint |
| Access Key ID | AWS access key | 是 | `AKIA...` | 顯示時可局部遮罩 |
| Secret Access Key | AWS secret | 是 | `<masked>` | 必須遮罩 |
| Session Token | 暫時 credential token | 否 | `<masked>` | 使用臨時憑證時填寫 |
| Base URL | EC2 endpoint override | 否 | `https://ec2.ap-northeast-1.amazonaws.com` | 一般情境可由 Region 推得 |

## 4. Provider 能力支援矩陣

| 能力 | 是否支援 | Provider 資料來源 | 產生/更新的 HCM 資料 |
| --- | --- | --- | --- |
| 授權/登入 | 支援 | AWS Credential + Region | Cloud Connection 授權狀態 |
| 同步資源池 | 支援 | VPC | Pool |
| 同步網路 | 支援 | Subnet | Subnet |
| 同步安全群組 | 支援 | Security Group | Security Group |
| 同步 VM 規格來源 | 支援 | AMI | Pool Image Catalog |
| 同步 VM 清單 | 支援 | EC2 Instance | VM |
| 建立 Allocation 附加資源 | 不支援 | 無 | 無 |
| 建立 VM | 支援 | RunInstances | VM |
| VM 開機/關機 | 支援 | StartInstances / StopInstances | VM 狀態 |
| VM 狀態追蹤 | 支援 | DescribeInstances | VM 狀態與 IP |

## 5. Provider 到 HCM 欄位映射

### 5.1 VPC / Pool 映射

| Provider 欄位/概念 | Provider 意義 | HCM 欄位 | HCM 意義 | 轉換規則 | 範例 |
| --- | --- | --- | --- | --- | --- |
| VpcId | VPC 識別 | Pool cloud_ref | Provider 來源識別 | 原樣保留 | `vpc-0123456789abcdef0` |
| Name tag | VPC 名稱 | Pool name | HCM 顯示名稱 | 優先取 Name tag，否則取 VpcId | `prod-vpc` |
| CidrBlock | VPC CIDR | Pool description | 顯示 VPC 範圍 | 放入描述 | `AWS VPC 10.0.0.0/16` |
| Region | AWS 區域 | Pool site / region | 區域分類 | 由 Connection region 帶入 | `ap-northeast-1` |
| Capacity | VPC 無固定容量 | Pool capacity | 容量資訊 | CPU/Memory/Disk 可為空，配額由 HCM 管理 | `null` |

### 5.2 Network / Subnet / Security Group 映射

| Provider 欄位/概念 | Provider 意義 | HCM 欄位 | HCM 意義 | 轉換規則 | 範例 |
| --- | --- | --- | --- | --- | --- |
| SubnetId | Subnet 識別 | Subnet cloud_ref | Provider 來源識別 | 原樣保留 | `subnet-001` |
| Name tag | Subnet 名稱 | Subnet name | HCM 顯示名稱 | 優先取 Name tag | `app-subnet-a` |
| CidrBlock | Subnet CIDR | Subnet cidr | 網段 | 原樣保留 | `10.0.1.0/24` |
| State | Subnet 狀態 | Subnet status | 是否可用 | available -> active | `active` |
| GroupId | Security Group 識別 | Security Group id | VM 建立可選安全群組 | 原樣保留 | `sg-001` |
| GroupName / Description | Security Group 資訊 | Security Group name / description | 顯示與辨識 | 原樣保留 | `web-sg` |

### 5.3 Image Catalog 映射

| Provider 欄位/概念 | Provider 意義 | HCM 欄位 | HCM 意義 | 轉換規則 | 範例 |
| --- | --- | --- | --- | --- | --- |
| ImageId | AMI 識別 | Image id | VM 建立來源 | 原樣保留 | `ami-001` |
| Name / Name tag | AMI 名稱 | Image label | 使用者可讀名稱 | self AMI 取 tag/name；public AMI 加 OS 前綴 | `Ubuntu 22.04 LTS (...)` |
| RootDeviceName | Root disk device | rootDeviceName | 建立 VM block device 參考 | 原樣保留 | `/dev/sda1` |
| CreationDate | AMI 建立時間 | is_latest | 是否為最新 | quick start 取最新一筆 | `true` |

### 5.4 VM 映射

| Provider 欄位/概念 | Provider 意義 | HCM 欄位 | HCM 意義 | 轉換規則 | 範例 |
| --- | --- | --- | --- | --- | --- |
| InstanceId | EC2 Instance 識別 | VM cloud_ref | Provider VM 識別 | 原樣保留 | `i-001` |
| Name tag | VM 名稱 | VM name / hostname | HCM 顯示名稱 | 優先取 Name tag | `app-01` |
| State.Name | EC2 狀態 | VM status | HCM 狀態 | running/stopped/pending/stopping 對應標準狀態 | `running` |
| InstanceType | EC2 規格 | VM flavor | Instance Type | 原樣保留 | `t3.medium` |
| DescribeInstanceTypes | vCPU / Memory | VM CPU / Memory | 規格顯示 | DefaultVCpus、Memory MiB 轉 GB | `2 vCPU / 4 GB` |
| PrivateIpAddress | 私有 IP | VM IP | 主要 IP | 原樣保留 | `10.0.1.10` |
| NetworkInterfaces | NIC | VM NIC / SG | 網卡、IP、安全群組 | 解析 SubnetId、Groups | `sg-001` |
| BlockDeviceMappings | Disk | VM Disk | 磁碟 | 目前 VM 同步未展開 BlockDeviceMappings，`diskGb` 為空、`disks` 為空；建立 VM 時才把 HCM disk 需求轉成 BlockDeviceMappings | `100 GB` |

## 6. 功能畫面差異

| 功能 | 畫面/流程位置 | 畫面差異 | 欄位/選項差異 | 可操作能力 | 業務限制/提示 |
| --- | --- | --- | --- | --- | --- |
| Cloud Settings | Connection 表單 | 顯示 AWS credential 欄位 | Region、Access Key、Secret、Session Token | 可建立/更新連線 | Secret 必須遮罩 |
| Cloud Settings | 同步資料 Wizard | Pool 為 VPC；Image 為 AMI | 同步 VPC、Subnet、Security Group、AMI、EC2 | 可同步資料 | VPC 無固定 CPU/Memory/Disk 容量 |
| Allocation Management | Shared Allocation 表單 | 使用共通欄位 | Project、System、Quota、Subnet | 不建立 Provider 附加資源 | 不顯示 Provider Extension |
| VM Management | VM 建立表單 | 顯示 Image、Instance Type、Subnet、Security Group | AWS 支援 Security Group 多選 | 可建立 EC2 | Security Group 僅 AWS 支援 |
| VM Management | VM 操作 | 支援 Start / Stop / 狀態追蹤 | instanceId 為 EC2 InstanceId | 可開機、關機、查詢狀態 | 建立後 IP 可能需 poll 才完整 |

## 7. 外部介接上下行與範例

AWS Provider 現行開發以 AWS SDK 介接 EC2，不由 HCM 自行組 AWS Query API URL。HCM 會將 Connection 的 Access Key、Secret、Region 與 Session Token 組成 SDK credential context，再呼叫 EC2 SDK Command。

本章分成兩塊：

- `7.1 現行 SDK 介接規格`：開發與驗收以此為準。
- `7.2 等價 AWS Query API 對照`：保留 Method / URL / Action 對照，方便理解外部 API 上下行，但不是現行開發的主要介接方式。

### 7.1 現行 SDK 介接規格

#### 7.1.1 SDK Credential Context

| Connection 欄位 | SDK 對應 | 必填 | 備註 |
| --- | --- | --- | --- |
| Region | EC2Client region | 是 | 例如 `ap-northeast-1` |
| Access Key ID | credentials.accessKeyId | 是 | 顯示時可局部遮罩 |
| Secret Access Key | credentials.secretAccessKey | 是 | 必須遮罩 |
| Session Token | credentials.sessionToken | 否 | 臨時憑證使用 |
| Base URL | EC2Client endpoint override | 否 | 程式會在 Base URL 有值時帶入 SDK `endpoint`；一般由 Region 推得 |

AWS 不需要登入換 token。HCM 以 Access Key / Secret / Region 建立 AWS credential context，並用 `DescribeRegionsCommand` 驗證 credential。SDK runtime headers 會帶入 `x-codex-aws-access-key-id`、`x-codex-aws-secret-access-key`、`x-codex-aws-region`，若有 session token 則加上 `x-codex-aws-session-token`。

#### 7.1.2 SDK Command 總表

| Plugin 能力 | SDK Command | SDK Input 摘要 | SDK Output 取用 | HCM 轉換 |
| --- | --- | --- | --- | --- |
| 授權驗證 | `DescribeRegionsCommand` | `{ RegionNames: [region] }` | Command 成功 | Connection 授權可用 |
| 同步 VPC / Pool | `DescribeVpcsCommand` | 無條件或 `{ VpcIds: [vpcId] }` | `Vpcs[]` | VPC -> Pool |
| 同步 Subnet | `DescribeSubnetsCommand` | `Filters: [{ Name: 'vpc-id', Values: [vpcId] }]` | `Subnets[]` | Subnet -> HCM Subnet |
| 同步 Security Group | `DescribeSecurityGroupsCommand` | `Filters: [{ Name: 'vpc-id', Values: [vpcId] }]` | `SecurityGroups[]` | Security Group -> HCM Security Group |
| 同步 AMI / Image | `DescribeImagesCommand` | `Owners: ['self']`；另以 public AMI owner + name/state filter 取 quick start AMI | `Images[]` | AMI -> Image Catalog |
| 同步 EC2 VM 清單 | `DescribeInstancesCommand` | `Filters: [{ Name: 'vpc-id', Values: [vpcId] }]` | `Reservations[].Instances[]` | EC2 -> VM |
| 補 VM 規格 | `DescribeInstanceTypesCommand` | `InstanceTypes: [instanceType...]` | `InstanceTypes[]` | vCPU / Memory |
| 建立 VM | `RunInstancesCommand` | AMI、Instance Type、NetworkInterfaces、BlockDeviceMappings、Tags | `Instances[0]` | InstanceId / 初始狀態 / IP |
| VM 狀態追蹤 | `DescribeInstancesCommand` | `{ InstanceIds: [instanceId] }` | `Reservations[0].Instances[0]` | VM state / IP / NIC |
| VM 開機 | `StartInstancesCommand` | `{ InstanceIds: [instanceId] }` | Command 成功 | HCM 先顯示 `starting` |
| VM 關機 | `StopInstancesCommand` | `{ InstanceIds: [instanceId] }` | Command 成功 | HCM 先顯示 `stopping` |

#### 7.1.3 SDK 上下行範例

本節的 JSON `Input` 為實際傳入 AWS SDK Command 的內容，不包含額外包裝欄位。Command 名稱寫在小節標題與表格中。

##### 7.1.3.1 同步 VPC / Pool

| 項目 | 說明 |
| --- | --- |
| SDK Command | `DescribeVpcsCommand` |
| 用途 | 列出 VPC，或取得指定 VPC 詳細資料 |
| HCM 取用 | `VpcId`、`Tags`、`CidrBlock` |

**listPools Input**

```json
{}
```

**fetchPool Input**

```json
{
  "VpcIds": ["vpc-0123456789abcdef0"]
}
```

**AWS SDK Output 摘要**

```json
{
  "Vpcs": [
    {
      "VpcId": "vpc-0123456789abcdef0",
      "CidrBlock": "10.0.0.0/16",
      "Tags": [
        { "Key": "Name", "Value": "prod-vpc" }
      ]
    }
  ]
}
```

**HCM 轉換結果摘要**

```json
{
  "id": "vpc-0123456789abcdef0",
  "name": "prod-vpc",
  "href": "vpc-0123456789abcdef0",
  "cloud_ref": "vpc-0123456789abcdef0",
  "description": "AWS VPC 10.0.0.0/16",
  "allocationModel": "vpc",
  "cpu": null,
  "memory": null,
  "storage": null
}
```

##### 7.1.3.2 同步 Subnet

| 項目 | 說明 |
| --- | --- |
| SDK Command | `DescribeSubnetsCommand` |
| 用途 | 取得指定 VPC 下的 Subnet；若未指定 VPC，則查詢可見 Subnet |
| HCM 取用 | `SubnetId`、`CidrBlock`、`State`、`Tags` |

**Input**

```json
{
  "Filters": [
    { "Name": "vpc-id", "Values": ["vpc-0123456789abcdef0"] }
  ]
}
```

**AWS SDK Output 摘要**

```json
{
  "Subnets": [
    {
      "SubnetId": "subnet-001",
      "VpcId": "vpc-0123456789abcdef0",
      "CidrBlock": "10.0.1.0/24",
      "State": "available",
      "Tags": [
        { "Key": "Name", "Value": "app-subnet-a" }
      ]
    }
  ]
}
```

**HCM 轉換結果摘要**

```json
{
  "cloud_ref": "subnet-001",
  "network_id": "subnet-001",
  "subnet_idx": 1,
  "name": "app-subnet-a",
  "cidr": "10.0.1.0/24",
  "gateway": "",
  "status": "active"
}
```

##### 7.1.3.3 同步 Security Group

| 項目 | 說明 |
| --- | --- |
| SDK Command | `DescribeSecurityGroupsCommand` |
| 用途 | 取得指定 VPC 下的 Security Group |
| HCM 取用 | `GroupId`、`GroupName`、`Description` |

**Input**

```json
{
  "Filters": [
    { "Name": "vpc-id", "Values": ["vpc-0123456789abcdef0"] }
  ]
}
```

**AWS SDK Output 摘要**

```json
{
  "SecurityGroups": [
    {
      "GroupId": "sg-001",
      "GroupName": "web-sg",
      "Description": "Allow web traffic"
    }
  ]
}
```

**HCM 轉換結果摘要**

```json
{
  "id": "sg-001",
  "name": "web-sg",
  "description": "Allow web traffic"
}
```

##### 7.1.3.4 同步 AMI / Image

自有 AMI 會先用 `Owners: ['self']` 查詢；Quick Start AMI 會依固定 owner、name filter 與 `state=available` 查詢，取 `CreationDate` 最新的一筆。目前 quick start 類型包含 Amazon Linux 2023、Ubuntu 22.04 LTS、Windows Server 2022、RHEL 9、SUSE Linux 15、Debian 12。

| 項目 | 說明 |
| --- | --- |
| SDK Command | `DescribeImagesCommand` |
| 用途 | 取得自有 AMI 與 quick start public AMI |
| HCM 取用 | `ImageId`、`Name`、`Tags`、`CreationDate`、`RootDeviceName` |

**自有 AMI Input**

```json
{
  "Owners": ["self"]
}
```

**自有 AMI Output 摘要**

```json
{
  "Images": [
    {
      "ImageId": "ami-self-001",
      "Name": "company-base-ubuntu-22",
      "RootDeviceName": "/dev/sda1",
      "Tags": [
        { "Key": "Name", "Value": "Company Ubuntu 22" }
      ]
    }
  ]
}
```

**Quick Start AMI Input 範例**

```json
{
  "Owners": ["099720109477"],
  "Filters": [
    {
      "Name": "name",
      "Values": ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
    },
    {
      "Name": "state",
      "Values": ["available"]
    }
  ]
}
```

**Quick Start AMI Output 摘要**

```json
{
  "Images": [
    {
      "ImageId": "ami-quickstart-001",
      "Name": "ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-20260101",
      "CreationDate": "2026-01-01T00:00:00.000Z",
      "RootDeviceName": "/dev/sda1"
    }
  ]
}
```

**HCM 轉換結果摘要**

```json
[
  {
    "id": "ami-self-001",
    "label": "Company Ubuntu 22",
    "is_latest": true,
    "is_legacy": false,
    "rootDeviceName": "/dev/sda1"
  },
  {
    "id": "ami-quickstart-001",
    "label": "Ubuntu 22.04 LTS (ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-20260101)",
    "is_latest": true,
    "is_legacy": false,
    "rootDeviceName": "/dev/sda1"
  }
]
```

##### 7.1.3.5 建立 VM

目前建立 VM 不帶 `KeyName`、`UserData`，也不把 `CreateVmNicInput.ip` 轉成 AWS `PrivateIpAddress`。若沒有輸入 disk，`BlockDeviceMappings` 不送出，由 AMI 預設 root disk 決定；若有 disk，第一顆 disk 使用 image 的 `rootDeviceName`，沒有時預設 `/dev/xvda`，後續資料碟依序使用 `/dev/sdb`、`/dev/sdc`。若 NIC 沒有 Security Group，該 NIC 的 `Groups` 不送出。

| 項目 | 說明 |
| --- | --- |
| SDK Command | `RunInstancesCommand` |
| 用途 | 建立 EC2 Instance |
| HCM 取用 | `Instances[0].InstanceId`、`Instances[0].PrivateIpAddress`、`Instances[0].State.Name` |

**Input**

```json
{
  "ImageId": "ami-001",
  "InstanceType": "t3.medium",
  "MinCount": 1,
  "MaxCount": 1,
  "NetworkInterfaces": [
    {
      "DeviceIndex": 0,
      "SubnetId": "subnet-001",
      "Groups": ["sg-001"],
      "DeleteOnTermination": true
    }
  ],
  "BlockDeviceMappings": [
    {
      "DeviceName": "/dev/sda1",
      "Ebs": {
        "VolumeSize": 100,
        "VolumeType": "gp3",
        "DeleteOnTermination": true
      }
    }
  ],
  "TagSpecifications": [
    {
      "ResourceType": "instance",
      "Tags": [
        { "Key": "Name", "Value": "app-01" },
        { "Key": "project_id", "Value": "prj-001" },
        { "Key": "system_id", "Value": "sys-001" }
      ]
    }
  ]
}
```

**AWS SDK Output 摘要**

```json
{
  "Instances": [
    {
      "InstanceId": "i-001",
      "State": { "Name": "pending" },
      "PrivateIpAddress": "10.0.1.10"
    }
  ]
}
```

**HCM 回傳摘要**

```json
{
  "instanceId": "i-001",
  "privateIpAddress": "10.0.1.10",
  "state": "pending"
}
```

##### 7.1.3.6 同步 EC2 VM 清單

VM 清單同步會並行查詢 VPC 內 EC2 與 Subnet，用 SubnetId 對應 Subnet 顯示名稱；再用 VM 清單中的唯一 InstanceType 查 `DescribeInstanceTypesCommand` 補 vCPU 與 Memory。現行同步不解析 EBS disk 容量，`diskGb` 為 `null`、`disks` 為空陣列。

| 項目 | 說明 |
| --- | --- |
| SDK Command | `DescribeInstancesCommand`、`DescribeSubnetsCommand`、`DescribeInstanceTypesCommand` |
| 用途 | 同步 VPC 內 EC2 清單、Subnet 名稱與 Instance Type 規格 |
| HCM 取用 | InstanceId、Tags、PrivateDnsName、ImageId、PrivateIpAddress、InstanceType、State、NetworkInterfaces、InstanceType specs |

**DescribeInstancesCommand Input**

```json
{
  "Filters": [
    { "Name": "vpc-id", "Values": ["vpc-0123456789abcdef0"] }
  ]
}
```

**DescribeSubnetsCommand Input**

```json
{
  "Filters": [
    { "Name": "vpc-id", "Values": ["vpc-0123456789abcdef0"] }
  ]
}
```

**DescribeInstanceTypesCommand Input**

```json
{
  "InstanceTypes": ["t3.medium"]
}
```

**AWS SDK Output 摘要**

```json
{
  "Reservations": [
    {
      "Instances": [
        {
          "InstanceId": "i-001",
          "ImageId": "ami-001",
          "PrivateDnsName": "ip-10-0-1-10.ap-northeast-1.compute.internal",
          "PrivateIpAddress": "10.0.1.10",
          "InstanceType": "t3.medium",
          "State": { "Name": "running" },
          "Tags": [
            { "Key": "Name", "Value": "app-01" }
          ],
          "NetworkInterfaces": [
            {
              "SubnetId": "subnet-001",
              "PrivateIpAddress": "10.0.1.10",
              "Groups": [{ "GroupId": "sg-001" }]
            }
          ]
        }
      ]
    }
  ],
  "Subnets": [
    {
      "SubnetId": "subnet-001",
      "Tags": [
        { "Key": "Name", "Value": "app-subnet-a" }
      ]
    }
  ],
  "InstanceTypes": [
    {
      "InstanceType": "t3.medium",
      "VCpuInfo": { "DefaultVCpus": 2 },
      "MemoryInfo": { "SizeInMiB": 4096 }
    }
  ]
}
```

**HCM 轉換結果摘要**

```json
{
  "cloud_ref": "i-001",
  "href": "i-001",
  "name": "app-01",
  "hostname": "ip-10-0-1-10.ap-northeast-1.compute.internal",
  "imageId": "ami-001",
  "ipAddress": "10.0.1.10",
  "instanceType": "t3.medium",
  "status": "running",
  "vcpu": 2,
  "ramGb": 4,
  "diskGb": null,
  "nics": [
    {
      "name": "app-subnet-a",
      "ip": "10.0.1.10",
      "subnet_id": null,
      "securityGroupIds": ["sg-001"]
    }
  ],
  "disks": []
}
```

##### 7.1.3.7 VM 狀態追蹤

| 項目 | 說明 |
| --- | --- |
| SDK Command | `DescribeInstancesCommand` |
| 用途 | 查詢單台 EC2 最新狀態與 IP |
| HCM 取用 | `State.Name`、`PrivateIpAddress`、`NetworkInterfaces[].PrivateIpAddress` |

**Input**

```json
{
  "InstanceIds": ["i-001"]
}
```

**AWS SDK Output 摘要**

```json
{
  "Reservations": [
    {
      "Instances": [
        {
          "InstanceId": "i-001",
          "State": { "Name": "running" },
          "PrivateIpAddress": "10.0.1.10",
          "NetworkInterfaces": [
            {
              "PrivateIpAddress": "10.0.1.10"
            }
          ]
        }
      ]
    }
  ]
}
```

**HCM 回傳摘要**

```json
{
  "state": "running",
  "ip": "10.0.1.10",
  "nics": [
    { "ip": "10.0.1.10" }
  ]
}
```

##### 7.1.3.8 VM 開機 / 關機

| 項目 | 開機 | 關機 |
| --- | --- | --- |
| SDK Command | `StartInstancesCommand` | `StopInstancesCommand` |
| 用途 | 啟動 EC2 Instance | 停止 EC2 Instance |
| HCM 取用 | SDK 成功即可 | SDK 成功即可 |

**StartInstancesCommand Input**

```json
{
  "InstanceIds": ["i-001"]
}
```

**HCM 回傳摘要**

```json
{
  "instanceId": "i-001",
  "newStatus": "starting"
}
```

**StopInstancesCommand Input**

```json
{
  "InstanceIds": ["i-001"]
}
```

**HCM 回傳摘要**

```json
{
  "instanceId": "i-001",
  "newStatus": "stopping"
}
```

### 7.2 等價 AWS Query API 對照

本節保留 AWS Query API 的 Method / URL / Action 對照，用於說明 SDK Command 最終對應的 AWS 外部 API。現行開發以 `7.1 現行 SDK 介接規格` 為準，不要求 HCM 手動組 Query API 或 SigV4 request。

若以 AWS Query API 表示，主要 endpoint 為 `https://ec2.{region}.amazonaws.com/`，或使用 Connection 的 Base URL override。以下以 `{ec2Endpoint}` 表示。所有 request 需以 AWS Signature Version 4 簽章；文件範例以業務欄位呈現，敏感資訊一律遮罩。

#### 7.2.1 授權/登入等價 API

| 項目 | 說明 |
| --- | --- |
| Method | POST |
| URL | `{ec2Endpoint}` |
| 上行 | Query API `Action=DescribeRegions&Version=2016-11-15&RegionName.1={region}`，AWS SigV4 header |
| 下行 | DescribeRegionsResponse |
| HCM 取用 | 呼叫成功代表 credential 與 region 可用 |

#### 7.2.2 同步 VPC / Pool 等價 API

| 順序 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 1 | POST | `{ec2Endpoint}` | 列出 VPC | `Action=DescribeVpcs` | `vpcSet.item[].vpcId`、`cidrBlock`、`tagSet` |
| 2 | POST | `{ec2Endpoint}` | 取得指定 VPC | `Action=DescribeVpcs&VpcId.1={vpcId}` | 同上 |

```http
POST {ec2Endpoint}
Authorization: AWS4-HMAC-SHA256 Credential=<masked>
Content-Type: application/x-www-form-urlencoded

Action=DescribeVpcs&Version=2016-11-15
```

```json
{
  "Vpcs": [
    {
      "VpcId": "vpc-0123456789abcdef0",
      "CidrBlock": "10.0.0.0/16",
      "Tags": [{ "Key": "Name", "Value": "prod-vpc" }]
    }
  ]
}
```

#### 7.2.3 同步 Network / Security Group 等價 API

| 順序 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 1 | POST | `{ec2Endpoint}` | 取得 VPC 下 Subnet | `Action=DescribeSubnets&Filter.1.Name=vpc-id&Filter.1.Value.1={vpcId}` | `SubnetId`、`VpcId`、`CidrBlock`、`State`、`Tags` |
| 2 | POST | `{ec2Endpoint}` | 取得 VPC 下 Security Group | `Action=DescribeSecurityGroups&Filter.1.Name=vpc-id&Filter.1.Value.1={vpcId}` | `GroupId`、`GroupName`、`Description` |

```json
{
  "Subnets": [
    {
      "SubnetId": "subnet-001",
      "VpcId": "vpc-0123456789abcdef0",
      "CidrBlock": "10.0.1.0/24",
      "State": "available"
    }
  ],
  "SecurityGroups": [
    {
      "GroupId": "sg-001",
      "GroupName": "web-sg",
      "Description": "Allow web traffic"
    }
  ]
}
```

#### 7.2.4 同步 AMI / Image 等價 API

| 順序 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 1 | POST | `{ec2Endpoint}` | 取得自有 AMI | `Action=DescribeImages&Owner.1=self` | `ImageId`、`Name`、`RootDeviceName`、`Tags` |
| 2 | POST | `{ec2Endpoint}` | 取得 quick start public AMI | `Action=DescribeImages&Owner.1={owner}&Filter name/state` | `ImageId`、`Name`、`CreationDate`、`RootDeviceName` |

```json
{
  "Images": [
    {
      "ImageId": "ami-001",
      "Name": "ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-20260101",
      "CreationDate": "2026-01-01T00:00:00.000Z",
      "RootDeviceName": "/dev/sda1"
    }
  ]
}
```

#### 7.2.5 建立 VM 等價 API

AWS 建立 VM 以 SDK `RunInstancesCommand` 完成；以下是等價 Query API 對照。HCM Create VM Input 會轉成 AMI、Instance Type、NetworkInterfaces、BlockDeviceMappings 與 Tags。

| 順序 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 1 | POST | `{ec2Endpoint}` | 建立 EC2 Instance | `Action=RunInstances` + VM payload | `instancesSet.item[0].instanceId`、`instanceState.name`、`privateIpAddress` |

```json
{
  "ImageId": "ami-001",
  "InstanceType": "t3.medium",
  "MinCount": 1,
  "MaxCount": 1,
  "NetworkInterfaces": [
    {
      "DeviceIndex": 0,
      "SubnetId": "subnet-001",
      "Groups": ["sg-001"],
      "DeleteOnTermination": true
    }
  ],
  "BlockDeviceMappings": [
    {
      "DeviceName": "/dev/sda1",
      "Ebs": {
        "VolumeSize": 100,
        "VolumeType": "gp3",
        "DeleteOnTermination": true
      }
    }
  ],
  "TagSpecifications": [
    {
      "ResourceType": "instance",
      "Tags": [
        { "Key": "Name", "Value": "app-01" },
        { "Key": "project_id", "Value": "prj-001" },
        { "Key": "system_id", "Value": "sys-001" }
      ]
    }
  ]
}
```

```json
{
  "Instances": [
    {
      "InstanceId": "i-001",
      "State": { "Name": "pending" },
      "PrivateIpAddress": "10.0.1.10"
    }
  ]
}
```

#### 7.2.6 同步 EC2 VM 清單等價 API

| 順序 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 1 | POST | `{ec2Endpoint}` | 查詢 VPC 內 EC2 | `Action=DescribeInstances&Filter.1.Name=vpc-id&Filter.1.Value.1={vpcId}` | `Reservations[].Instances[]` |
| 2 | POST | `{ec2Endpoint}` | 查詢 Instance Type 規格 | `Action=DescribeInstanceTypes&InstanceType.1={type}` | `VCpuInfo.DefaultVCpus`、`MemoryInfo.SizeInMiB` |

#### 7.2.7 VM 狀態追蹤等價 API

| 順序 | Method | URL | 用途 | 上行 | 下行取用欄位 |
| --- | --- | --- | --- | --- | --- |
| 1 | POST | `{ec2Endpoint}` | 查詢單台 EC2 狀態 | `Action=DescribeInstances&InstanceId.1={instanceId}` | `State.Name`、`PrivateIpAddress`、`NetworkInterfaces` |

#### 7.2.8 VM 開機 / 關機等價 API

| 動作 | Method | URL | 上行 | 下行取用 |
| --- | --- | --- | --- | --- |
| 開機 | POST | `{ec2Endpoint}` | `Action=StartInstances&InstanceId.1={instanceId}` | 操作成功後 HCM 先顯示 `starting` |
| 關機 | POST | `{ec2Endpoint}` | `Action=StopInstances&InstanceId.1={instanceId}` | 操作成功後 HCM 先顯示 `stopping` |

## 8. 單位換算與狀態映射

| 項目 | AWS 單位 / 狀態 | HCM 表示 | 規則 |
| --- | --- | --- | --- |
| vCPU | DefaultVCpus | CPU Core | 原樣帶入 |
| Memory | MiB | GB | `MiB / 1024` |
| Disk | GB | GB | 原樣帶入 |
| Instance state | pending | provisioning / starting | 建立或開機中 |
| Instance state | running | running | 執行中 |
| Instance state | stopping | stopping | 關機中 |
| Instance state | stopped | stopped | 已停止 |
| Instance state | terminated | deleted / stopped | 依 HCM 顯示規則 |

## 9. 限制與待確認

| 項目 | 說明 |
| --- | --- |
| AWS Pool 容量 | VPC 不提供固定 CPU/Memory/Disk 容量，HCM 配額需由管理者設定 |
| Security Group | AWS 支援 Security Group；Harvester 不支援，兩者不可混寫 |
| Static IP | 目前建立 VM payload 以 Subnet / SG 為主，是否指定 Private IP 需依產品規則確認 |
| Key Pair / User Data | 目前程式未帶 `KeyName` / `UserData`；若業務需要 SSH key 或 cloud-init，需在 VM Management 文件與 AWS Provider 文件補充欄位 |
| VM Disk 同步 | 目前 VM 清單同步未解析 EC2 EBS volume size，`diskGb` 與 `disks` 可能為空 |
