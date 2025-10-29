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
2. **自動化分析**：自動識別惡意程式的能力與技術
3. **標準化報告**：提供符合 ATT&CK 和 MBC 框架的威脅情報
4. **可擴展架構**：支援自訂規則和持續更新

### 核心概念

**靜態分析轉動態監控**：capa 原本用於靜態分析可執行檔案，本專案將其規則邏輯轉譯為對 EDR 動態事件流的即時匹配。

### 專案範圍

#### 包含範圍
- 規則解析與驗證
- 事件標準化與處理
- 即時匹配引擎
- 告警生成與管理
- 效能監控與最佳化

#### 不包含範圍
- EDR 基礎設施建置
- 網路流量分析
- 檔案系統即時防護（另有專案負責）

---

## 系統架構

### 整體架構圖

```
┌─────────────────────────────────────────────────────────────┐
│                        EDR 主系統                            │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Capa 引擎整合層                           │
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
│  │          告警生成器 (Alert Generator)             │      │
│  └──────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   EDR 後端處理系統                           │
│  (告警儲存、分析、視覺化、響應)                              │
└─────────────────────────────────────────────────────────────┘
```

### 資料流程

```
事件源 → 事件標準化 → 評估上下文 → 規則匹配 → 告警生成 → 後端處理
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

### 模組 4：告警生成與管理

#### 4.1 告警生成
**功能描述**：當規則匹配成功時生成結構化告警

**告警內容**：
- 觸發規則的完整元資料（名稱、命名空間、描述）
- 目標程序資訊（PID、路徑、命令列、雜湊值）
- 觸發時間戳（UTC）
- 匹配的特徵路徑（用於解釋告警原因）
- ATT&CK 技術映射
- MBC 行為映射
- 嚴重性評分

#### 4.2 告警去重
**功能描述**：避免重複告警淹沒系統

**去重策略**：
- 同一程序/規則組合在時間視窗內僅告警一次
- 可配置去重視窗（預設 5 分鐘）
- 記錄重複次數作為額外資訊
- 支援告警聚合

#### 4.3 告警傳送
**功能描述**：將告警傳送至後端系統

**傳送機制**：
- 支援 HTTP/HTTPS POST
- 非同步傳送（不阻塞主流程）
- 傳送失敗時自動重試（可配置次數）
- 本地快取未成功傳送的告警
- 支援批次傳送以提升效率
- TLS 1.3 加密傳輸

---

### 模組 5：效能監控

#### 5.1 效能指標收集
**功能描述**：收集引擎運行的效能數據

**監控指標**：
- **CPU 使用率**：引擎佔用的 CPU 百分比
- **記憶體使用**：引擎使用的記憶體量
- **事件處理延遲**：從事件到達到處理完成的時間
- **規則評估延遲**：單一規則評估耗時
- **告警生成速率**：每秒生成的告警數量
- **快取命中率**：各級快取的命中率
- **佇列深度**：待處理事件佇列長度

#### 5.2 效能報告與告警
**功能描述**：輸出效能數據並在異常時告警

**輸出方式**：
- 定期寫入日誌檔案
- 支援 Prometheus 格式匯出
- 提供 HTTP 健康檢查端點
- 效能異常時發送告警（例如：延遲過高、記憶體洩漏）

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