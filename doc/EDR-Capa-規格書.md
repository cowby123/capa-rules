# EDR Capa 支援功能規格書

## 文件資訊

| 項目 | 內容 |
|------|------|
| 文件版本 | 1.1.0 |
| 建立日期 | 2025-10-29 |
| 最後更新 | 2025-10-29 |
| 文件狀態 | 草稿 |
| 負責單位 | EDR 開發團隊 |

## 目錄

1. [專案概述](#專案概述)
2. [系統架構](#系統架構)
3. [核心功能列表](#核心功能列表)
4. [技術規格](#技術規格)
5. [效能需求](#效能需求)

---

## 專案概述

### 專案目標

本專案旨在將 [Mandiant capa](https://github.com/mandiant/capa) 規則引擎整合至 EDR (Endpoint Detection and Response) 系統中，實現：

1. **即時威脅偵測**：將靜態分析規則轉換為動態行為監控
2. **即時威脅阻擋**：偵測到惡意行為時立即阻止程序執行
3. **自動化響應**：根據規則嚴重性自動執行防禦動作
4. **標準化報告**：提供符合 ATT&CK 和 MBC 框架的威脅情報
5. **可擴展架構**：支援自訂規則和持續更新

### 核心概念

**即時動態防禦**：capa 原本用於靜態分析可執行檔案，本專案將其規則邏輯轉譯為對 EDR 動態事件流的即時匹配與阻擋。當偵測到符合惡意規則的行為時，系統會立即採取防禦措施，包括：
- 終止惡意程序
- 阻止危險 API 呼叫
- 隔離可疑檔案
- 回滾惡意變更

### 專案範圍

#### 包含範圍
- 規則解析與驗證
- 事件標準化與處理
- 即時匹配引擎
- **即時阻擋機制**（新增）
- **響應動作執行器**（新增）
- 告警生成與管理
- 效能監控與最佳化

#### 不包含範圍
- EDR 基礎設施建置
- 網路流量分析

---

## 系統架構

### 整體架構圖

```
┌─────────────────────────────────────────────────────────────┐
│                        EDR 主系統                            │
│              (Kernel Driver + User Agent)                   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                 Capa 即時防禦引擎整合層                      │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ 規則解析器   │  │ 事件抽象層   │  │ 匹配引擎     │      │
│  │ (Parser)     │  │ (Abstraction)│  │ (Matcher)    │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                              │
│  ┌──────────────────────────────────────────────────┐      │
│  │          評估上下文管理 (Context Manager)         │      │
│  └──────────────────────────────────────────────────┘      │
│                                                              │
│  ┌──────────────────────────────────────────────────┐      │
│  │     ⚡ 即時決策引擎 (Real-time Decision Engine)   │      │
│  │   - 威脅等級評估                                  │      │
│  │   - 阻擋策略選擇                                  │      │
│  │   - 誤報風險評估                                  │      │
│  └──────────────────────────────────────────────────┘      │
│                                                              │
│  ┌──────────────────────────────────────────────────┐      │
│  │     🛡️ 響應動作執行器 (Response Executor)        │      │
│  │   - API 呼叫攔截/阻擋                             │      │
│  │   - 程序終止                                      │      │
│  │   - 檔案隔離                                      │      │
│  │   - 記憶體清除                                    │      │
│  └──────────────────────────────────────────────────┘      │
│                                                              │
│  ┌──────────────────────────────────────────────────┐      │
│  │          告警生成器 (Alert Generator)             │      │
│  └──────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   EDR 後端處理系統                           │
│  (告警儲存、分析、視覺化、響應審計)                          │
└─────────────────────────────────────────────────────────────┘
```

### 即時阻擋資料流程

```
事件源 → 事件標準化 → 評估上下文 → 規則匹配
                                      ↓
                            ┌─────────┴─────────┐
                            ↓                   ↓
                      匹配成功              匹配失敗
                            ↓                   ↓
                     威脅等級評估           允許執行
                            ↓
                ┌───────────┴───────────┐
                ↓                       ↓
         高危/關鍵威脅              中/低危威脅
                ↓                       ↓
          立即阻擋動作            告警 + 監控
          (終止/攔截)              (可選阻擋)
                ↓                       ↓
          生成阻擋告警            生成偵測告警
                ↓                       ↓
          ─────────┴───────────────────┘
                        ↓
                  後端記錄與審計
```

---

## 核心功能列表

### 模組 1：規則解析器

#### 1.1 YAML 解析功能
**功能描述**：將 YAML 格式的 capa 規則檔案解析為記憶體中的結構化物件

**核心能力**：
- 解析 YAML 格式規則檔案
- 驗證規則格式正確性
- 建構規則物件樹
- 錯誤訊息回報

**支援的規則元素**：
- `meta` 區塊（名稱、命名空間、作者、描述等）
- `features` 區塊（邏輯運算式和特徵陳述）
- 巢狀邏輯結構（and、or、not、count）

#### 1.2 規則驗證功能
**功能描述**：確保載入的規則有效且可執行

**驗證項目**：
- 必要欄位完整性檢查
- 規則邏輯一致性驗證
- 規則相依性檢查（避免循環依賴）
- 不支援特徵類型識別

#### 1.3 規則載入管理
**功能描述**：管理規則的載入、更新和卸載

**核心能力**：
- 從本地檔案系統載入規則
- 從遠端伺服器下載規則
- 支援規則熱更新（無需重啟服務）
- 按命名空間/作業系統/嚴重性過濾規則

---

### 模組 2：事件抽象層

#### 2.1 事件標準化
**功能描述**：將異質 EDR 事件轉換為統一格式

**支援的事件類型**：
- **API 呼叫事件**：函數名稱、參數、回傳值
- **程序事件**：建立、終止、資訊變更
- **記憶體事件**：配置、保護變更、內容讀取
- **檔案系統事件**：讀取、寫入、刪除、屬性變更
- **登錄檔事件**：鍵值讀取、寫入、刪除
- **網路事件**：連線建立、資料傳輸

**標準化內容**：
- 統一時間戳格式（UTC 毫秒級）
- 標準化程序識別（PID、TID、PPID）
- 統一資料型別表示
- 保留原始事件資料供追溯

#### 2.2 事件佇列管理
**功能描述**：管理待處理事件的佇列和流控

**核心能力**：
- 高併發事件寫入支援
- 背壓機制（防止記憶體溢位）
- 事件優先級處理
- 事件遺失記錄

#### 2.3 事件快取機制
**功能描述**：快取近期事件以支援規則評估

**核心能力**：
- 按程序分組快取事件
- LRU（Least Recently Used）淘汰策略
- 可配置快取大小和時間視窗
- 快取命中率監控

---

### 模組 3：匹配引擎

#### 3.1 評估上下文管理
**功能描述**：為每個程序維護評估所需的上下文資訊

**上下文內容**：
- API 呼叫歷史記錄
- 程序記憶體區域資訊
- 系統環境資訊（OS、架構）
- 其他規則的匹配結果
- 評估快取

**核心方法**：
- `getApiCallHistory(pid)`：取得 API 呼叫記錄
- `getProcessMemory(pid)`：取得程序記憶體內容（懶加載）
- `getSystemInfo()`：取得系統資訊
- `getRuleResult(ruleName)`：取得規則匹配結果

#### 3.2 邏輯節點評估
**功能描述**：實作規則邏輯運算的評估

**支援的邏輯節點**：
- **AND 節點**：所有子節點都為真才匹配（支援短路評估）
- **OR 節點**：任一子節點為真即匹配（支援短路評估）
- **NOT 節點**：子節點為假才匹配
- **COUNT 節點**：符合條件的子節點數量滿足要求
  - `N or more`：至少 N 個子節點為真
  - `N or fewer`：最多 N 個子節點為真
  - `(min, max)`：子節點為真的數量在範圍內
- **OPTIONAL 節點**：`0 or more` 的別名

#### 3.3 基本特徵評估
**功能描述**：評估基本的規則特徵陳述

**支援的特徵類型**：

##### API 呼叫（api）
- 匹配指定的 API 函數呼叫
- 自動處理 A/W 後綴（ANSI/Unicode 版本）
- 支援帶/不帶模組名稱的匹配
- 範例：`api: CreateFile` 匹配 `CreateFileA` 和 `CreateFileW`

##### 字串匹配（string/substring）
- 在程序記憶體中搜尋字串
- 支援精確匹配和子字串匹配
- 支援正規表示式匹配
- 支援 case-insensitive 選項
- 觸發懶加載記憶體掃描

##### 位元組序列（bytes）
- 在程序記憶體中搜尋位元組序列
- 支援最多 0x100 位元組的序列
- 用於搜尋特定的二進位模式（如 GUID、常數）

##### 數值常數（number）
- 在記憶體或 API 參數中搜尋數值
- 支援十六進位和十進位格式
- 支援不同位元組順序（endianness）
- 範例：`number: 0x40 = PAGE_EXECUTE_READWRITE`

##### 作業系統（os）
- 匹配程序執行的作業系統
- 支援：windows、linux、macos 等
- 從系統資訊快取中取得

##### 架構（arch）
- 匹配 CPU 架構
- 支援：i386（32 位元）、amd64（64 位元）
- 從系統資訊快取中取得

##### 檔案格式（format）
- 匹配可執行檔格式
- 支援：pe、elf、dotnet
- 透過解析程序映像檔標頭判斷

#### 3.4 進階特徵評估
**功能描述**：評估進階的規則特徵陳述

**支援的特徵類型**：

##### 匯入/匯出（import/export）
- **Import**：匹配 PE 檔案的匯入表
  - 支援 DLL 名稱匹配
  - 支援序數（ordinal）匯入
  - 範例：`import: kernel32.WinExec`
- **Export**：匹配 PE 檔案的匯出表
  - 支援轉送匯出（forwarded export）
  - 範例：`export: InstallA`

##### PE 區段（section）
- 匹配 PE 檔案的區段名稱
- 範例：`section: .rsrc`

##### 特徵（characteristic）
- 將 capa 的 characteristic 映射至 EDR 行為特徵
- 支援的映射（範例）：
  - `packed`：高熵值記憶體區段
  - `embedded pe`：記憶體中的 PE 檔案
  - `loop`：重複執行的程式碼模式
  - `nzxor`：非歸零 XOR 操作

##### 規則相依（match）
- 匹配其他規則的執行結果
- 支援命名空間匹配
- 自動解析規則執行順序
- 偵測並拒絕循環依賴
- 範例：`match: create process`

##### COM GUID（com）【未來支援】
- 匹配 COM 物件的 GUID
- 需要監控 `CoCreateInstance` 等 COM API
- 範例：`com/class: InternetExplorer`

##### .NET 類別/命名空間（class/namespace）【未來支援】
- 匹配 .NET 類別和命名空間
- 需要 CLR 監控能力
- 範例：`class: System.IO.File`

#### 3.5 快取與最佳化
**功能描述**：提升評估效能的機制

**最佳化策略**：
- **評估快取**：相同特徵節點不重複評估
- **短路評估**：AND/OR 節點儘早確定結果
- **懶加載記憶體掃描**：僅在需要時才讀取程序記憶體
- **規則結果快取**：已評估的規則結果可重複使用
- **事件預過濾**：不符合任何規則的事件快速跳過

---

### 模組 4：即時決策與阻擋引擎

#### 4.1 威脅等級評估
**功能描述**：當規則匹配成功時，評估威脅的嚴重程度和風險等級

**評估維度**：
- **規則嚴重性**：基於 ATT&CK/MBC 映射的威脅等級
  - 關鍵（Critical）：直接威脅系統安全
  - 高危（High）：可能造成重大損害
  - 中危（Medium）：需要監控的可疑行為
  - 低危（Low）：異常但風險較低的行為

- **行為危險度**：分析行為本身的危險性
  - 檔案加密行為
  - 記憶體注入
  - 權限提升
  - 資料外洩
  - 系統破壞

- **上下文分析**：考慮程序的整體行為模式
  - 程序來源是否可信
  - 行為序列是否符合攻擊鏈
  - 是否存在多個可疑行為

- **誤報風險評估**：降低誤殺合法程式的機率
  - 白名單檢查（簽章、雜湊、路徑）
  - 行為序列合理性分析
  - 歷史信譽評分

#### 4.2 阻擋策略選擇
**功能描述**：根據威脅等級選擇適當的響應策略

**阻擋策略**：

1. **立即終止（Kill）**
   - **觸發條件**：關鍵威脅 + 高危行為 + 低誤報風險
   - **動作**：立即終止程序
   - **適用場景**：
     - 勒索軟體加密行為
     - 記憶體注入到系統程序
     - 批量檔案刪除
     - 清除系統還原點

2. **API 攔截（Block）**
   - **觸發條件**：高危威脅 + 特定危險 API
   - **動作**：阻止特定 API 呼叫但不終止程序
   - **適用場景**：
     - 嘗試停用防毒軟體
     - 嘗試修改關鍵登錄檔
     - 嘗試建立服務
     - 嘗試讀取敏感資料

3. **程序掛起（Suspend）**
   - **觸發條件**：高危威脅 + 中等誤報風險
   - **動作**：暫停程序執行，等待分析或使用者決策
   - **適用場景**：
     - 可疑但未確認的威脅
     - 需要進一步分析的行為
     - 可能誤報的高危行為

4. **監控模式（Monitor）**
   - **觸發條件**：中/低危威脅
   - **動作**：允許執行但加強監控
   - **適用場景**：
     - 異常但可能合法的行為
     - 學習模式下的新規則
     - 低風險探測行為

5. **檔案隔離（Quarantine）**
   - **觸發條件**：高危威脅 + 檔案相關行為
   - **動作**：移動檔案至隔離區
   - **適用場景**：
     - 投放惡意檔案
     - 建立可執行檔
     - 修改系統檔案

#### 4.3 決策引擎實作
**功能描述**：實作決策邏輯和策略管理

**決策流程**：
```
規則匹配成功
    ↓
提取規則元資料（ATT&CK、MBC、severity）
    ↓
分析行為上下文（程序資訊、行為序列）
    ↓
計算威脅分數（0-100）
    ↓
查詢白名單/黑名單
    ↓
評估誤報風險
    ↓
選擇阻擋策略
    ↓
檢查策略執行前提條件
    ↓
執行響應動作
```

**配置選項**：
- 全域阻擋開關（啟用/停用）
- 按規則類型配置策略
- 按程序路徑配置策略
- 白名單管理
- 威脅分數閾值設定

---

### 模組 5：響應動作執行器

#### 5.1 API 攔截與阻擋
**功能描述**：在 API 呼叫層面攔截並阻止危險操作

**實作方式**：
- **Kernel Hook**：在核心層攔截系統呼叫
- **User-mode Hook**：在使用者層攔截 API 呼叫
- **返回值偽造**：讓惡意程式以為 API 呼叫成功

**支援的 API 類別**：
- **程序操作**：CreateProcess、OpenProcess、TerminateProcess
- **檔案操作**：CreateFile、WriteFile、DeleteFile
- **登錄檔操作**：RegCreateKey、RegSetValue、RegDeleteKey
- **網路操作**：socket、connect、send
- **記憶體操作**：VirtualAllocEx、WriteProcessMemory
- **服務操作**：CreateService、OpenSCManager

**阻擋模式**：
- **同步阻擋**：立即回傳錯誤碼（如 ACCESS_DENIED）
- **非同步阻擋**：延遲回傳或超時
- **欺騙模式**：回傳假成功但不執行

#### 5.2 程序終止
**功能描述**：終止惡意程序及其子程序

**終止策略**：
- **優雅終止**：嘗試正常關閉（TerminateProcess）
- **強制終止**：核心層終止（ZwTerminateProcess）
- **程序樹終止**：終止所有子程序
- **防護機制**：防止惡意程序重新啟動

**安全措施**：
- 防止終止關鍵系統程序（csrss.exe、smss.exe 等）
- 防止終止自身和 EDR Agent
- 記錄終止操作供審計

#### 5.3 檔案隔離
**功能描述**：將可疑檔案移動至隔離區

**隔離流程**：
1. 建立檔案快照（備份）
2. 加密檔案內容
3. 移動至隔離目錄
4. 更新檔案索引
5. 記錄隔離資訊

**隔離目錄**：
- 路徑：`C:\ProgramData\EDR\Quarantine\`
- 權限：僅 SYSTEM 可存取
- 加密：AES-256 加密
- 保留時間：可配置（預設 30 天）

**還原功能**：
- 支援從隔離區還原檔案
- 需要管理員權限
- 記錄還原操作

#### 5.4 記憶體清除
**功能描述**：清除程序記憶體中的惡意程式碼

**清除操作**：
- **shellcode 清除**：覆寫記憶體中的 shellcode
- **注入 DLL 卸載**：卸載注入的惡意 DLL
- **記憶體頁面保護**：修改記憶體頁面權限

#### 5.5 系統還原
**功能描述**：回滾惡意變更

**支援的還原操作**：
- **登錄檔還原**：恢復被修改的登錄檔項
- **檔案還原**：恢復被刪除或修改的檔案
- **服務還原**：恢復服務配置
- **排程任務還原**：移除惡意排程任務

**實作機制**：
- 維護系統變更日誌
- 建立還原點
- 支援選擇性還原

#### 5.6 網路隔離
**功能描述**：切斷惡意程序的網路連線

**隔離方式**：
- **防火牆規則**：動態新增防火牆阻擋規則
- **連線終止**：強制關閉現有連線
- **DNS 攔截**：阻止 DNS 解析

---

### 模組 6：告警生成與管理

#### 6.1 告警生成
**功能描述**：當規則匹配成功時生成結構化告警

**告警內容**：
- 觸發規則的完整元資料（名稱、命名空間、描述）
- 目標程序資訊（PID、路徑、命令列、雜湊值）
- 觸發時間戳（UTC）
- 匹配的特徵路徑（用於解釋告警原因）
- ATT&CK 技術映射
- MBC 行為映射
- 嚴重性評分

#### 6.2 告警去重
**功能描述**：避免重複告警淹沒系統

**去重策略**：
- 同一程序/規則組合在時間視窗內僅告警一次
- 可配置去重視窗（預設 5 分鐘）
- 記錄重複次數作為額外資訊
- 支援告警聚合

#### 6.3 告警傳送
**功能描述**：將告警傳送至後端系統

**傳送機制**：
- 支援 HTTP/HTTPS POST
- 非同步傳送（不阻塞主流程）
- 傳送失敗時自動重試（可配置次數）
- 本地快取未成功傳送的告警
- 支援批次傳送以提升效率
- TLS 1.3 加密傳輸

---

### 模組 7：效能監控

#### 7.1 效能指標收集
**功能描述**：收集引擎運行的效能數據

**監控指標**：
- **CPU 使用率**：引擎佔用的 CPU 百分比
- **記憶體使用**：引擎使用的記憶體量
- **事件處理延遲**：從事件到達到處理完成的時間
- **規則評估延遲**：單一規則評估耗時
- **告警生成速率**：每秒生成的告警數量
- **快取命中率**：各級快取的命中率
- **佇列深度**：待處理事件佇列長度

#### 7.2 效能報告與告警
**功能描述**：輸出效能數據並在異常時告警

**輸出方式**：
- 定期寫入日誌檔案
- 支援 Prometheus 格式匯出
- 提供 HTTP 健康檢查端點
- 效能異常時發送告警（例如：延遲過高、記憶體洩漏）

---

### 模組 8：整合與配置

#### 8.1 EDR 事件源整合
**功能描述**：與 EDR 系統的事件源進行整合

**整合方式**：
- **核心驅動整合**：接收來自核心驅動的事件（程序、檔案、登錄檔、網路）
- **ETW (Event Tracing for Windows) 整合**：訂閱 Windows ETW 事件
- **Hook 引擎整合**：接收 API Hook 產生的事件
- **事件過濾**：在源頭過濾不相關事件，減少處理負擔

**事件類型對應**：
```
EDR 事件類型           →  Capa 事件類型
ProcessCreate          →  process/create
ProcessTerminate       →  process/terminate
FileCreate             →  file/create
FileWrite              →  file/write
RegistrySetValue       →  registry/set
NetworkConnect         →  network/connect
ThreadCreate           →  thread/create
MemoryAllocate         →  memory/allocate
```

#### 8.2 配置管理
**功能描述**：提供靈活的配置選項

**配置檔案格式**：JSON 或 YAML

**主要配置項**：
```yaml
engine:
  # 規則設定
  rules:
    directory: "C:/EDR/capa-rules"
    auto_reload: true
    reload_interval: 300  # 秒
    enabled_namespaces:
      - "malware"
      - "anti-analysis"
      - "persistence"

  # 效能設定
  performance:
    max_concurrent_evaluations: 100
    event_queue_size: 10000
    memory_scan_timeout: 5000
    cache_size_mb: 512

  # 阻擋設定
  blocking:
    enabled: true
    mode: "automatic"  # automatic, manual, learning
    default_action: "monitor"  # kill, block, suspend, monitor, quarantine

  # 告警設定
  alert:
    endpoint: "https://siem.company.com/api/alerts"
    batch_size: 100
    send_interval: 10  # 秒
    deduplication_window: 300  # 秒
    retry_count: 3

  # 日誌設定
  logging:
    level: "INFO"  # DEBUG, INFO, WARN, ERROR
    file: "C:/EDR/logs/capa-engine.log"
    max_size_mb: 100
    max_files: 10
```

#### 8.3 規則管理
**功能描述**：提供規則的生命週期管理

**功能清單**：
- **規則載入**：從本地目錄或遠端伺服器載入規則
- **規則驗證**：檢查規則語法和語義正確性
- **規則熱更新**：不停機更新規則集
- **規則版本控制**：追蹤規則版本和變更歷史
- **規則選擇性啟用**：根據命名空間或標籤啟用/停用規則
- **規則統計**：記錄每條規則的匹配次數和效能數據

#### 8.4 白名單與例外管理
**功能描述**：管理已知良性程式和例外情況

**白名單類型**：
- **程序路徑白名單**：信任的程式路徑（支援萬用字元）
- **簽章白名單**：信任的程式碼簽章
- **規則例外**：針對特定規則的例外清單
- **組織例外**：組織特定的合法行為例外

**配置範例**：
```yaml
whitelist:
  processes:
    - path: "C:/Windows/System32/*"
      signature: "Microsoft Corporation"
    - path: "C:/Program Files/CompanyApp/*"
      signature: "Company Name"

  rule_exceptions:
    - rule: "create-service"
      processes:
        - "C:/Admin/ServiceManager.exe"
    - rule: "registry-persistence"
      processes:
        - "C:/Setup/Installer.exe"
```

---

### 模組 9：引擎 API

#### 9.1 初始化與生命週期管理
**功能描述**：提供引擎初始化和關閉的 API

**API 定義**：
```cpp
class CapaEngine {
public:
    // 建構函數，載入配置
    CapaEngine(const std::string& configPath);

    // 初始化引擎（載入規則、建立執行緒池等）
    bool initialize();

    // 啟動引擎
    bool start();

    // 停止引擎（完成待處理事件後停止）
    void stop();

    // 強制關閉（立即終止，可能丟失事件）
    void forceShutdown();

    // 取得引擎狀態
    EngineStatus getStatus() const;
};

enum class EngineStatus {
    UNINITIALIZED,  // 未初始化
    INITIALIZING,   // 初始化中
    RUNNING,        // 運行中
    STOPPING,       // 停止中
    STOPPED,        // 已停止
    ERROR           // 錯誤狀態
};
```

#### 9.2 事件處理 API
**功能描述**：提供事件提交和處理的 API

**API 定義**：
```cpp
class CapaEngine {
public:
    // 同步處理單一事件（阻塞直到處理完成）
    MatchResult processEventSync(const Event& event);

    // 非同步處理單一事件（立即返回）
    void processEventAsync(const Event& event,
                          std::function<void(MatchResult)> callback = nullptr);

    // 批次處理事件
    std::vector<MatchResult> processBatch(const std::vector<Event>& events);

    // 取得事件佇列狀態
    QueueStatus getQueueStatus() const;
};

struct MatchResult {
    bool matched;                           // 是否匹配規則
    std::vector<std::string> matchedRules;  // 匹配的規則清單
    std::string threatLevel;                // 威脅等級
    std::string recommendedAction;          // 建議動作
    std::map<std::string, std::string> metadata;  // 額外元資料
};
```

#### 9.3 規則管理 API
**功能描述**：提供規則載入和管理的 API

**API 定義**：
```cpp
class CapaEngine {
public:
    // 載入單一規則檔案
    bool loadRule(const std::string& filePath);

    // 載入規則目錄
    bool loadRulesFromDirectory(const std::string& dirPath,
                               const RuleFilter& filter = RuleFilter::all());

    // 從遠端載入規則
    bool loadRulesFromRemote(const std::string& url,
                            const std::string& authToken = "");

    // 卸載特定規則
    bool unloadRule(const std::string& ruleName);

    // 啟用/停用規則
    bool enableRule(const std::string& ruleName, bool enable);

    // 取得已載入規則清單
    std::vector<RuleInfo> getLoadedRules() const;

    // 取得規則統計資訊
    RuleStatistics getRuleStatistics(const std::string& ruleName) const;
};

struct RuleInfo {
    std::string name;
    std::string namespace_;
    std::string description;
    bool enabled;
    int matchCount;
    double avgEvaluationTime;
};
```

#### 9.4 配置管理 API
**功能描述**：提供執行時配置修改的 API

**API 定義**：
```cpp
class CapaEngine {
public:
    // 更新配置（部分配置需要重啟生效）
    bool updateConfig(const EngineConfig& newConfig);

    // 取得當前配置
    EngineConfig getConfig() const;

    // 設定阻擋模式
    bool setBlockingMode(BlockingMode mode);

    // 設定日誌級別
    bool setLogLevel(LogLevel level);

    // 重新載入配置檔案
    bool reloadConfigFromFile();
};

enum class BlockingMode {
    DISABLED,    // 停用阻擋
    LEARNING,    // 學習模式（僅記錄）
    AUTOMATIC    // 自動阻擋
};
```

#### 9.5 統計與監控 API
**功能描述**：提供引擎統計和監控資訊的 API

**API 定義**：
```cpp
class CapaEngine {
public:
    // 取得引擎統計資訊
    EngineStatistics getStatistics() const;

    // 重置統計計數器
    void resetStatistics();

    // 取得效能指標
    PerformanceMetrics getPerformanceMetrics() const;

    // 匯出 Prometheus 格式指標
    std::string exportPrometheusMetrics() const;
};

struct EngineStatistics {
    uint64_t totalEventsProcessed;      // 處理的事件總數
    uint64_t totalMatches;              // 匹配總數
    uint64_t totalBlocks;               // 阻擋總數
    double avgProcessingTime;           // 平均處理時間（毫秒）
    double currentEventsPerSecond;      // 當前事件處理速率
    size_t queueDepth;                  // 佇列深度
    size_t activeEvaluations;           // 進行中的評估數
};

struct PerformanceMetrics {
    double cpuUsagePercent;             // CPU 使用率
    size_t memoryUsageBytes;            // 記憶體使用量
    double cacheHitRate;                // 快取命中率
    double p50Latency;                  // P50 延遲（毫秒）
    double p95Latency;                  // P95 延遲（毫秒）
    double p99Latency;                  // P99 延遲（毫秒）
};
```

#### 9.6 告警訂閱 API
**功能描述**：提供告警訂閱和回調的 API

**API 定義**：
```cpp
class CapaEngine {
public:
    // 註冊告警回調函數
    CallbackHandle registerAlertCallback(
        std::function<void(const CapaMatchAlert&)> callback
    );

    // 取消註冊回調
    void unregisterAlertCallback(CallbackHandle handle);

    // 設定告警過濾器
    void setAlertFilter(const AlertFilter& filter);
};

struct CapaMatchAlert {
    std::string alertId;                    // 告警 ID
    std::string ruleName;                   // 規則名稱
    std::string ruleNamespace;              // 規則命名空間
    ProcessInfo processInfo;                // 程序資訊
    std::string threatLevel;                // 威脅等級
    std::string recommendedAction;          // 建議動作
    std::vector<std::string> attackTechniques;  // ATT&CK 技術
    std::vector<std::string> mbcBehaviors;      // MBC 行為
    std::string timestamp;                  // 時間戳
    std::map<std::string, std::string> metadata;  // 額外資訊
};
```

#### 9.7 白名單管理 API
**功能描述**：提供執行時白名單管理的 API

**API 定義**：
```cpp
class CapaEngine {
public:
    // 添加程序路徑到白名單
    bool addProcessToWhitelist(const std::string& processPath);

    // 從白名單移除程序
    bool removeProcessFromWhitelist(const std::string& processPath);

    // 添加規則例外
    bool addRuleException(const std::string& ruleName,
                         const std::string& processPath);

    // 移除規則例外
    bool removeRuleException(const std::string& ruleName,
                            const std::string& processPath);

    // 取得白名單清單
    std::vector<std::string> getWhitelist() const;

    // 檢查程序是否在白名單中
    bool isWhitelisted(const std::string& processPath) const;
};
```

---

## 技術規格

### 資料結構定義

#### 規則物件（Rule）

```cpp
class Rule {
public:
    Meta meta;                    // 規則元資料
    FeatureNode* features;        // 特徵樹根節點

    bool evaluate(EvaluationContext& ctx) const;
    std::string toJson() const;
};
```

#### 元資料（Meta）

```cpp
class Meta {
public:
    std::string name;                      // 規則名稱
    std::string namespace_;                // 命名空間
    std::vector<std::string> authors;      // 作者列表
    std::map<std::string, std::string> scopes;  // 適用範疇
    std::vector<std::string> attck;        // ATT&CK 映射
    std::vector<std::string> mbc;          // MBC 映射
    std::vector<std::string> references;   // 參考資料
    std::vector<std::string> examples;     // 範例樣本
    std::string description;               // 描述
    bool lib;                              // 是否為函式庫規則
};
```

#### 特徵節點基類（FeatureNode）

```cpp
class FeatureNode {
public:
    virtual ~FeatureNode() = default;
    virtual bool evaluate(EvaluationContext& ctx) = 0;
    virtual std::string getType() const = 0;
    virtual std::string toString() const = 0;
};
```

#### 邏輯節點範例

```cpp
// AND 節點
class AndNode : public FeatureNode {
private:
    std::vector<std::unique_ptr<FeatureNode>> children;
public:
    bool evaluate(EvaluationContext& ctx) override {
        for (const auto& child : children) {
            if (!child->evaluate(ctx)) {
                return false;  // 短路評估
            }
        }
        return true;
    }
};

// OR 節點
class OrNode : public FeatureNode {
private:
    std::vector<std::unique_ptr<FeatureNode>> children;
public:
    bool evaluate(EvaluationContext& ctx) override {
        for (const auto& child : children) {
            if (child->evaluate(ctx)) {
                return true;  // 短路評估
            }
        }
        return false;
    }
};

// COUNT 節點（N or more）
class CountNode : public FeatureNode {
private:
    int minCount;
    std::vector<std::unique_ptr<FeatureNode>> children;
public:
    bool evaluate(EvaluationContext& ctx) override {
        int matchCount = 0;
        for (const auto& child : children) {
            if (child->evaluate(ctx)) {
                matchCount++;
                if (matchCount >= minCount) {
                    return true;  // 短路評估
                }
            }
        }
        return matchCount >= minCount;
    }
};
```

#### 陳述節點範例

```cpp
// API 陳述
class ApiStatement : public FeatureNode {
private:
    std::string apiName;
public:
    bool evaluate(EvaluationContext& ctx) override {
        auto apiCalls = ctx.getApiCallHistory(ctx.getCurrentPid());
        for (const auto& call : apiCalls) {
            // 匹配精確名稱或帶 A/W 後綴的版本
            if (call.functionName == apiName ||
                call.functionName == apiName + "A" ||
                call.functionName == apiName + "W") {
                return true;
            }
        }
        return false;
    }
};

// 字串陳述
class StringStatement : public FeatureNode {
private:
    std::string value;
    bool isRegex;
    bool caseSensitive;
public:
    bool evaluate(EvaluationContext& ctx) override {
        // 觸發記憶體掃描（懶加載）
        auto memoryRegions = ctx.getProcessMemory(ctx.getCurrentPid());

        if (isRegex) {
            std::regex pattern(value, caseSensitive ?
                std::regex::ECMAScript :
                std::regex::icase);
            for (const auto& region : memoryRegions) {
                if (std::regex_search(region.content, pattern)) {
                    return true;
                }
            }
        } else {
            for (const auto& region : memoryRegions) {
                if (region.content.find(value) != std::string::npos) {
                    return true;
                }
            }
        }
        return false;
    }
};
```

#### 評估上下文（EvaluationContext）

```cpp
class EvaluationContext {
private:
    int currentPid;
    std::map<int, std::vector<ApiCallEvent>> apiCallCache;
    std::map<int, std::vector<ProcessMemoryRegion>> memoryCache;
    SystemInfo systemInfo;
    std::map<std::string, bool> ruleResults;
    std::map<FeatureNode*, bool> evaluationCache;

public:
    // 核心查詢方法
    std::vector<ApiCallEvent> getApiCallHistory(int pid);
    std::vector<ProcessMemoryRegion> getProcessMemory(int pid);
    SystemInfo getSystemInfo();
    bool getRuleResult(const std::string& ruleName);

    // 快取管理
    void clearCache();
    void cacheRuleResult(const std::string& ruleName, bool result);

    // 輔助方法
    int getCurrentPid() const { return currentPid; }
    void setCurrentPid(int pid) { currentPid = pid; }
};
```

#### 標準化事件結構

```cpp
// API 呼叫事件
struct ApiCallEvent {
    int pid;
    int tid;
    uint64_t timestamp;        // 毫秒級 Unix 時間戳
    std::string functionName;
    std::vector<std::string> arguments;
    std::any returnValue;
    std::string moduleName;    // 例如 "kernel32.dll"
};

// 程序記憶體區域
struct ProcessMemoryRegion {
    int pid;
    uint64_t baseAddress;
    size_t size;
    std::string content;       // 實際記憶體內容
    std::string protection;    // 例如 "PAGE_EXECUTE_READWRITE"
    bool isExecutable;
};

// 系統資訊
struct SystemInfo {
    std::string os;            // "windows", "linux", "macos"
    std::string arch;          // "i386", "amd64"
    std::string osVersion;     // 例如 "Windows 10 Build 19041"
    std::string hostname;
};

// 程序資訊
struct ProcessInfo {
    int pid;
    int ppid;
    std::string imagePath;
    std::string commandLine;
    std::string userName;
    uint64_t startTime;
    std::string imageHash;     // SHA256
};
```

#### 告警結構（CapaMatchAlert）

```cpp
class CapaMatchAlert {
public:
    Meta matchedRuleMeta;               // 觸發規則元資料
    ProcessInfo processInfo;            // 程序資訊
    uint64_t timestamp;                 // 告警時間戳
    std::vector<std::string> triggeringPath;  // 觸發路徑
    std::map<std::string, std::string> additionalContext;  // 額外上下文

    std::string toJson() const;
    std::string getSeverity() const;    // 根據 ATT&CK/MBC 計算嚴重性
};
```

### API 介面

#### 規則載入 API

```cpp
class RuleLoader {
public:
    // 從檔案載入單一規則
    std::unique_ptr<Rule> loadFromFile(const std::string& filePath);

    // 從目錄批次載入規則
    std::vector<std::unique_ptr<Rule>> loadFromDirectory(
        const std::string& dirPath,
        const RuleFilter& filter = RuleFilter::all()
    );

    // 從遠端伺服器載入規則
    std::vector<std::unique_ptr<Rule>> loadFromRemote(
        const std::string& url,
        const std::string& authToken = ""
    );

    // 驗證規則有效性
    ValidationResult validate(const Rule& rule);
};
```

#### 匹配引擎 API

```cpp
class MatchingEngine {
public:
    // 初始化引擎
    MatchingEngine(const EngineConfig& config);

    // 載入規則集
    void loadRules(std::vector<std::unique_ptr<Rule>> rules);

    // 處理單一事件
    std::vector<CapaMatchAlert> processEvent(const Event& event);

    // 處理事件批次
    std::vector<CapaMatchAlert> processBatch(const std::vector<Event>& events);

    // 取得引擎狀態
    EngineStats getStats() const;

    // 熱更新規則
    void updateRules(std::vector<std::unique_ptr<Rule>> newRules);

    // 停止引擎
    void shutdown();
};
```

#### 配置介面

```cpp
struct EngineConfig {
    int maxConcurrentEvaluations = 100;      // 最大並行評估數
    int eventQueueSize = 10000;              // 事件佇列大小
    int memoryScanTimeout = 5000;            // 記憶體掃描超時（毫秒）
    int alertDeduplicationWindow = 300000;   // 去重視窗（毫秒）
    bool enableMemoryScanning = true;        // 啟用記憶體掃描
    std::string alertEndpoint;               // 告警傳送端點
    LogLevel logLevel = LogLevel::INFO;      // 日誌級別
};
```

---

## 效能需求

### 延遲需求

| 指標 | 目標值 | 最大可接受值 |
|------|--------|--------------|
| 單一事件處理延遲（P95） | < 10ms | < 50ms |
| 規則評估延遲（P95） | < 5ms | < 20ms |
| 告警生成延遲 | < 1ms | < 5ms |
| 規則載入時間（1000 條規則） | < 5s | < 15s |

### 吞吐量需求

| 指標 | 目標值 |
|------|--------|
| 每秒處理事件數（單核心） | ≥ 5,000 |
| 每秒處理事件數（4 核心） | ≥ 15,000 |
| 規則匹配併發數 | ≥ 100 |

### 資源使用

| 資源 | 正常負載 | 高負載 | 最大限制 |
|------|----------|--------|----------|
| CPU 使用率 | < 10% | < 30% | 50% |
| 記憶體使用 | < 500MB | < 1GB | 2GB |
| 磁碟 I/O | < 10MB/s | < 50MB/s | 100MB/s |

### 可擴展性

- 支援載入 ≥ 5,000 條規則而不顯著降低效能
- 支援監控 ≥ 1,000 個並行程序
- 支援 ≥ 10,000 events/sec 的事件速率

---

## 附錄

### 附錄 A：支援的 Capa 特徵清單

| 特徵類型 | 優先級 | 實作狀態 | 備註 |
|---------|--------|----------|------|
| api | P0 | ✅ 已實作 | 核心功能 |
| string | P0 | ✅ 已實作 | 支援正規表示式 |
| substring | P0 | ✅ 已實作 | - |
| bytes | P0 | ✅ 已實作 | - |
| number | P0 | ✅ 已實作 | - |
| os | P0 | ✅ 已實作 | - |
| arch | P0 | ✅ 已實作 | - |
| format | P1 | ✅ 已實作 | 僅支援 PE |
| characteristic | P1 | ⚠️ 部分實作 | 需映射至 EDR 事件 |
| import | P1 | ⚠️ 部分實作 | 需 PE 解析 |
| export | P2 | ⚠️ 部分實作 | 需 PE 解析 |
| section | P2 | ⚠️ 部分實作 | 需 PE 解析 |
| match | P1 | ✅ 已實作 | 規則相依性 |
| com | P2 | ⏳ 規劃中 | 需 COM 監控 |
| class | P2 | ⏳ 規劃中 | 需 CLR 監控 |
| namespace | P2 | ⏳ 規劃中 | 需 CLR 監控 |
| mnemonic | P3 | ❌ 不支援 | 成本過高 |
| operand | P3 | ❌ 不支援 | 成本過高 |

### 附錄 B：詞彙表

| 術語 | 定義 |
|------|------|
| ATT&CK | MITRE ATT&CK 框架，描述攻擊者戰術和技術 |
| MBC | Malware Behavior Catalog，惡意軟體行為目錄 |
| EDR | Endpoint Detection and Response，端點偵測與響應 |
| Capa | Mandiant 開發的惡意軟體能力識別工具 |
| PE | Portable Executable，Windows 可執行檔格式 |
| CLR | Common Language Runtime，.NET 執行時 |
| API Hook | 攔截 API 呼叫的技術 |
| P95 | 第 95 百分位數，表示 95% 的樣本低於此值 |
| LRU | Least Recently Used，最近最少使用淘汰策略 |

### 附錄 C：參考資料

1. **Capa 官方文件**
   - GitHub: https://github.com/mandiant/capa
   - 規則格式: https://github.com/mandiant/capa-rules/blob/master/doc/format.md

2. **MITRE ATT&CK**
   - 官網: https://attack.mitre.org/
   - 技術矩陣: https://attack.mitre.org/matrices/enterprise/

3. **MBC**
   - GitHub: https://github.com/MBCProject/mbc-markdown

4. **技術標準**
   - PE 格式規範: Microsoft PE and COFF Specification
   - YAML 1.2: https://yaml.org/spec/1.2/spec.html

---

**文件結束**

*版本歷史*：
- v1.1.0 (2025-10-29): 簡化版本，聚焦功能描述
- v1.0.0 (2025-10-29): 初始版本