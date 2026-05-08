# HCM SDD

HCM（Hybrid Cloud Management）Portal 的系統設計文件（SDD）。

本 repo 描述 HCM 的業務概念、資料模型、Provider Plugin 規範、Backend API 合約與各功能畫面的 Event 規格，供前後端開發、QA 與 Provider 整合共同參考。

---

## 從哪裡開始讀

如果你是第一次接觸這份文件，建議依以下順序閱讀：

1. **[00_System_Overview.md](./00_System_Overview.md)**
   系統全貌。了解 HCM 是什麼、有哪些角色、主業務流程長什麼樣子，以及文件之間的呼叫鏈關係。從這裡開始最快建立全局觀。

2. **[01_Business_Data_Dictionary.md](./01_Business_Data_Dictionary.md)**
   業務資料字典。定義整份 SDD 共用的名詞、核心物件欄位、標準狀態與資料關聯。讀完後續文件時遇到不熟悉的術語，回這裡查。

3. **[02_Provider_Plugin_Model.md](./02_Provider_Plugin_Model.md)**
   Provider Plugin 規範。HCM 如何把不同雲端差異封裝成統一掛點，以及每個掛點的標準輸入/輸出定義。若你負責接 Provider，這是必讀文件。

4. **[03_HCM_Backend_API_Contract.md](./03_HCM_Backend_API_Contract.md)**
   Backend API 合約。Frontend 與 Backend 之間的 HTTP endpoint 清單，標示每支 API 是否進入 Provider Plugin 掛點。

5. **功能文件**（依你負責的模組選讀）
   - [10_Resource_Overview.md](./10_Resource_Overview.md) — 資源概念與架構
   - [11_Project_Dimension.md](./11_Project_Dimension.md) — Project 組織維度
   - [12_Cloud_Settings.md](./12_Cloud_Settings.md) — Provider 建立、Connection 設定、資源同步
   - [13_Pool_Settings.md](./13_Pool_Settings.md) — Pool 配置管理
   - [14_Subnet_Settings.md](./14_Subnet_Settings.md) — Subnet 配置管理
   - [15_Apply_Wizard.md](./15_Apply_Wizard.md) — 專案資源申請流程（4 步驟 Wizard）
   - [16_Allocation_Management.md](./16_Allocation_Management.md) — 申請審核與 Allocation 分配
   - [17_VM_Management.md](./17_VM_Management.md) — VM 建立與管理
   - [18_User_and_Role.md](./18_User_and_Role.md) — 使用者與角色管理

6. **[provider_plugins/](./provider_plugins/)**
   各 Provider 的實作差異文件，描述外部 API 上下行、欄位轉換與畫面差異。若你負責特定 Provider 的整合，閱讀對應文件前請先讀完 `02_Provider_Plugin_Model.md`。

---

## 文件分層架構

```
┌─────────────────────────────────────────────────────────────────────┐
│  功能畫面 / Events                                                    │
│  10_Resource · 11_Project · 12_Cloud_Settings · 13_Pool · 14_Subnet │
│  15_Apply_Wizard · 16_Allocation · 17_VM_Mgmt · 18_User_and_Role    │
└────────────────────────────┬────────────────────────────────────────┘
                             │ HTTP API
┌────────────────────────────▼────────────────────────────────────────┐
│  HCM Backend API Contract                                            │
│  03_HCM_Backend_API_Contract.md                                      │
└────────────────────────────┬────────────────────────────────────────┘
                             │ Plugin 掛點
┌────────────────────────────▼────────────────────────────────────────┐
│  Provider Plugin 掛點規範                                             │
│  02_Provider_Plugin_Model.md                                         │
└────────────────────────────┬────────────────────────────────────────┘
                             │ 外部 API / SDK
┌────────────────────────────▼────────────────────────────────────────┐
│  Provider 外部 API 實作                                               │
│  provider_plugins/aws.md · vcd.md · harvester.md · vsphere.md        │
└────────────────────────────┬────────────────────────────────────────┘
                             │ 資料模型
┌────────────────────────────▼────────────────────────────────────────┐
│  業務資料模型                                                          │
│  01_Business_Data_Dictionary.md                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**設計原則**：功能文件只描述業務主流程，不寫死任何 Provider 差異；Provider 差異集中在 `provider_plugins/` 文件；業務名詞與欄位定義集中在資料字典，後續文件不重複定義。

---

## 文件與角色對照

不同角色關注的文件重點不同：

| 角色 | 主要閱讀文件 |
|------|------------|
| 產品 / BA | `00`、`01`、`10`、`11`、功能文件 |
| 前端開發 | `00`、`03`、`10`、`11`、功能文件 |
| 後端開發 | `00`、`01`、`02`、`03` |
| Provider 整合 | `02`、對應 `provider_plugins/` |
| QA | `01`（狀態與資料規則）、`10`（資源概念）、功能文件（Event 規格） |
| 運維 / 系統管理員 | `10`、`11`、`13`、`14`、`18` |

---

## 寫新的 Provider 文件

新增 Provider 時，在 `provider_plugins/` 下建立對應 `.md`，並依 `02_Provider_Plugin_Model.md` 第 3 章的規範填寫：

- 共同規範勾稽表（掛載點對照）
- 標準輸入/輸出落地表（欄位映射）
- Provider 能力矩陣
- 外部 API 上下行摘要
- 功能畫面差異索引
