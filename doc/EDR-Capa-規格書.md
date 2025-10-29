# EDR Capa 支援完整規格書

## 文件資訊

| 項目 | 內容 |
|------|------|
| 文件版本 | 1.0.0 |
| 建立日期 | 2025-10-29 |
| 最後更新 | 2025-10-29 |
| 文件狀態 | 草稿 |
| 負責單位 | EDR 開發團隊 |

## 目錄

1. [專案概述](#專案概述)
2. [系統架構](#系統架構)
3. [功能需求](#功能需求)
4. [技術規格](#技術規格)
5. [效能需求](#效能需求)
6. [安全需求](#安全需求)
7. [開發計畫](#開發計畫)
8. [測試策略](#測試策略)
9. [部署規劃](#部署規劃)
10. [維護與支援](#維護與支援)

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

### 模組架構

#### 模組 1：規則解析器 (Rule Parser)
**職責**：將 YAML 格式的 capa 規則安全地解析為記憶體中的強型別物件

**輸入**：
- YAML 格式的規則檔案
- 規則更新通知

**輸出**：
- 結構化的規則物件樹
- 解析錯誤報告

**關鍵元件**：
- YAML 解析器
- 規則驗證器
- 物件建構器

#### 模組 2：事件抽象層 (Event Abstraction Layer)
**職責**：標準化異質 EDR 事件為統一格式

**輸入**：
- 原始 EDR 事件（API Hook、程序事件、記憶體事件等）

**輸出**：
- 標準化事件物件

**關鍵元件**：
- 事件正規化器
- 事件佇列管理
- 事件快取機制

#### 模組 3：匹配引擎 (Matching Engine)
**職責**：高效評估規則特徵樹

**輸入**：
- 標準化事件
- 解析後的規則
- 評估上下文

**輸出**：
- 匹配結果
- 觸發路徑

**關鍵元件**：
- 特徵評估器
- 邏輯節點處理器
- 快取管理器

#### 模組 4：整合層 (Integration Layer)
**職責**：串接 EDR 主系統與 Capa 引擎

**輸入**：
- EDR 系統事件
- 系統配置

**輸出**：
- 結構化告警
- 效能指標

**關鍵元件**：
- 工作流程管理
- 告警生成器
- 效能監控

---

## 功能需求

### FR-001：規則管理

#### FR-001-1：規則載入
- **描述**：系統啟動時自動載入 capa 規則庫
- **優先級**：P0（關鍵）
- **需求**：
  - 支援從本地檔案系統載入規則
  - 支援從遠端伺服器下載規則
  - 支援規則熱更新（無需重啟）
  - 載入失敗時提供詳細錯誤訊息

#### FR-001-2：規則驗證
- **描述**：確保載入的規則格式正確且有效
- **優先級**：P0（關鍵）
- **需求**：
  - 驗證 YAML 格式正確性
  - 驗證必要欄位完整性
  - 驗證規則邏輯一致性
  - 檢查規則相依性（無循環依賴）

#### FR-001-3：規則過濾
- **描述**：根據配置選擇性載入規則
- **優先級**：P1（重要）
- **需求**：
  - 支援按命名空間過濾
  - 支援按 ATT&CK 技術過濾
  - 支援按作業系統過濾
  - 支援按嚴重性過濾

### FR-002：事件處理

#### FR-002-1：事件接收
- **描述**：從 EDR 系統接收各類事件
- **優先級**：P0（關鍵）
- **需求**：
  - 支援 API 呼叫事件
  - 支援程序建立/終止事件
  - 支援記憶體操作事件
  - 支援檔案系統事件
  - 支援登錄檔操作事件
  - 支援網路事件

#### FR-002-2：事件標準化
- **描述**：將原始事件轉換為標準格式
- **優先級**：P0（關鍵）
- **需求**：
  - 統一時間戳格式（UTC）
  - 標準化程序識別（PID、TID）
  - 統一資料型別
  - 保留原始事件資料

#### FR-002-3：事件佇列
- **描述**：管理待處理事件佇列
- **優先級**：P0（關鍵）
- **需求**：
  - 支援高併發事件寫入
  - 實作背壓機制（防止記憶體溢位）
  - 支援事件優先級
  - 事件遺失時記錄日誌

### FR-003：規則匹配

#### FR-003-1：基本匹配
- **描述**：評估事件是否符合規則條件
- **優先級**：P0（關鍵）
- **支援的特徵類型**：
  - ✅ API 呼叫（api）
  - ✅ 字串匹配（string/substring）
  - ✅ 正規表示式（regex）
  - ✅ 位元組序列（bytes）
  - ✅ 數值常數（number）
  - ✅ 作業系統（os）
  - ✅ 架構（arch）
  - ✅ 檔案格式（format）

#### FR-003-2：進階匹配
- **描述**：支援複雜的匹配邏輯
- **優先級**：P1（重要）
- **支援的特徵類型**：
  - ⚠️ 特徵（characteristic）- 部分支援
  - ⚠️ 匯入/匯出（import/export）- 需 PE 解析
  - ⚠️ COM GUID（com）- 需 COM 監控
  - ⚠️ .NET 類別/命名空間（class/namespace）- 需 CLR 監控
  - ❌ 助記符/運算元（mnemonic/operand）- 需反組譯器

#### FR-003-3：規則相依性
- **描述**：支援規則間的相依匹配
- **優先級**：P1（重要）
- **需求**：
  - 支援 `match` 語句
  - 自動解析規則執行順序
  - 快取規則匹配結果
  - 偵測並拒絕循環依賴

### FR-004：告警管理

#### FR-004-1：告警生成
- **描述**：規則匹配成功時生成結構化告警
- **優先級**：P0（關鍵）
- **告警內容**：
  - 觸發規則的完整元資料
  - 目標程序資訊
  - 觸發時間戳
  - 匹配的特徵路徑
  - ATT&CK/MBC 映射
  - 嚴重性評分

#### FR-004-2：告警去重
- **描述**：避免重複告警淹沒系統
- **優先級**：P1（重要）
- **需求**：
  - 同一程序/規則組合在時間視窗內僅告警一次
  - 可配置去重視窗（預設 5 分鐘）
  - 記錄重複次數

#### FR-004-3：告警傳送
- **描述**：將告警傳送至後端系統
- **優先級**：P0（關鍵）
- **需求**：
  - 支援非同步傳送
  - 傳送失敗時重試（最多 3 次）
  - 本地快取未送達告警
  - 支援批次傳送

---

## 技術規格

### 資料結構定義

#### 規則結構（Rule）

```cpp
class Rule {
public:
    Meta meta;                    // 規則元資料
    FeatureNode* features;        // 特徵樹根節點

    // 方法
    bool evaluate(EvaluationContext& ctx) const;
    std::string toJson() const;
};
```

#### 元資料結構（Meta）

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

    // 核心評估方法
    virtual bool evaluate(EvaluationContext& ctx) = 0;

    // 輔助方法
    virtual std::string getType() const = 0;
    virtual std::string toString() const = 0;
};
```

#### 邏輯節點

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

// NOT 節點
class NotNode : public FeatureNode {
private:
    std::unique_ptr<FeatureNode> child;
public:
    bool evaluate(EvaluationContext& ctx) override {
        return !child->evaluate(ctx);
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

    // 方法
    std::string toJson() const;
    std::string getSeverity() const;    // 根據 ATT&CK/MBC 計算嚴重性
};
```

### 演算法規格

#### 規則評估演算法

```
演算法：EvaluateRule
輸入：規則 R，評估上下文 C
輸出：布林值（匹配/不匹配）

1. 若 C.evaluationCache 包含 R.features，回傳快取結果
2. 設定 C.currentRule = R
3. result = R.features.evaluate(C)
4. C.evaluationCache[R.features] = result
5. 若 result == true：
   a. C.cacheRuleResult(R.meta.name, true)
   b. 生成告警 GenerateAlert(R, C)
6. 回傳 result
```

#### 記憶體掃描最佳化演算法

```
演算法：LazyMemoryScan
輸入：程序 PID，搜尋模式 P
輸出：匹配結果

1. 若 memoryCache[PID] 已存在，搜尋快取並回傳
2. 提交非同步任務：
   a. regions = EnumerateMemoryRegions(PID)
   b. 對每個 region：
      - 若 region.isExecutable == false 且 region.protection 包含 "READ"：
        - content = ReadMemory(PID, region.baseAddress, region.size)
        - memoryCache[PID].add({region, content})
3. 等待任務完成（設定超時 5 秒）
4. 在 memoryCache[PID] 中搜尋模式 P
5. 回傳匹配結果
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
    bool enableDisassembly = false;          // 啟用反組譯（高成本）
    std::string alertEndpoint;               // 告警傳送端點
    LogLevel logLevel = LogLevel::INFO;      // 日誌級別
};
```

---

## 效能需求

### NFR-001：延遲需求

| 指標 | 目標值 | 最大可接受值 |
|------|--------|--------------|
| 單一事件處理延遲（P95） | < 10ms | < 50ms |
| 規則評估延遲（P95） | < 5ms | < 20ms |
| 告警生成延遲 | < 1ms | < 5ms |
| 規則載入時間（1000 條規則） | < 5s | < 15s |

### NFR-002：吞吐量需求

| 指標 | 目標值 |
|------|--------|
| 每秒處理事件數（單核心） | ≥ 5,000 |
| 每秒處理事件數（4 核心） | ≥ 15,000 |
| 規則匹配併發數 | ≥ 100 |

### NFR-003：資源使用

| 資源 | 正常負載 | 高負載 | 最大限制 |
|------|----------|--------|----------|
| CPU 使用率 | < 10% | < 30% | 50% |
| 記憶體使用 | < 500MB | < 1GB | 2GB |
| 磁碟 I/O | < 10MB/s | < 50MB/s | 100MB/s |

### NFR-004：可擴展性

- 支援載入 ≥ 5,000 條規則而不顯著降低效能
- 支援監控 ≥ 1,000 個並行程序
- 支援 ≥ 10,000 events/sec 的事件速率

---

## 安全需求

### SEC-001：規則安全

1. **規則完整性驗證**
   - 所有規則檔案必須經過簽章驗證
   - 支援 SHA256 雜湊校驗
   - 拒絕載入未簽署或簽章無效的規則

2. **規則隔離**
   - 規則評估在受限環境中執行
   - 防止規則邏輯導致記憶體洩漏
   - 實作超時機制防止無限迴圈

### SEC-002：資料安全

1. **記憶體保護**
   - 掃描的程序記憶體不得寫入磁碟（除非加密）
   - 評估完成後立即清除記憶體快取
   - 敏感資料（如密碼）自動遮罩

2. **告警資料保護**
   - 告警傳輸使用 TLS 1.3 加密
   - 本地快取告警使用 AES-256 加密
   - 實作存取控制防止未授權讀取

### SEC-003：權限控制

1. **最小權限原則**
   - 引擎以非特權使用者執行（必要時提權）
   - 僅請求必要的系統權限
   - 記憶體掃描使用 SeDebugPrivilege（Windows）

---

## 開發計畫

### 階段 1：基礎建設（預估 4 週）

**目標**：建立核心架構和基礎元件

| 任務 | 負責人 | 預估工時 | 依賴 |
|------|--------|----------|------|
| 選擇並整合 YAML 函式庫 | 開發組 | 3 天 | - |
| 定義核心資料結構 | 架構組 | 5 天 | - |
| 實作規則解析器 | 開發組 | 8 天 | YAML 函式庫 |
| 建立事件抽象層 | 開發組 | 5 天 | - |
| 撰寫單元測試 | QA 組 | 5 天 | 解析器完成 |

**交付成果**：
- ✅ 可執行的規則解析器
- ✅ 標準化事件結構定義
- ✅ 單元測試覆蓋率 > 80%

### 階段 2：匹配引擎開發（預估 6 週）

**目標**：實作核心匹配邏輯

| 任務 | 負責人 | 預估工時 | 依賴 |
|------|--------|----------|------|
| 設計評估上下文 | 架構組 | 3 天 | - |
| 實作邏輯節點評估器 | 開發組 | 5 天 | 評估上下文 |
| 實作基本陳述節點（API、字串等） | 開發組 | 10 天 | 邏輯節點 |
| 實作進階陳述節點（PE、COM 等） | 開發組 | 8 天 | 基本陳述 |
| 實作快取機制 | 開發組 | 5 天 | - |
| 撰寫整合測試 | QA 組 | 5 天 | 匹配引擎完成 |

**交付成果**：
- ✅ 功能完整的匹配引擎
- ✅ 支援 80% 以上的 capa 特徵類型
- ✅ 整合測試覆蓋率 > 70%

### 階段 3：EDR 整合（預估 4 週）

**目標**：與 EDR 系統深度整合

| 任務 | 負責人 | 預估工時 | 依賴 |
|------|--------|----------|------|
| 設計整合工作流程 | 架構組 | 3 天 | - |
| 實作事件接收器 | 開發組 | 5 天 | EDR SDK |
| 實作告警生成器 | 開發組 | 5 天 | 匹配引擎 |
| 實作告警傳送器 | 開發組 | 5 天 | 告警生成器 |
| 實作效能監控 | 開發組 | 4 天 | - |
| 端對端測試 | QA 組 | 8 天 | 整合完成 |

**交付成果**：
- ✅ 完整的 EDR 整合
- ✅ 告警能正確傳送至後端
- ✅ 端對端測試通過

### 階段 4：最佳化與壓力測試（預估 3 週）

**目標**：確保生產環境可用性

| 任務 | 負責人 | 預估工時 | 依賴 |
|------|--------|----------|------|
| 實作短路評估最佳化 | 開發組 | 3 天 | - |
| 實作事件預過濾器 | 開發組 | 5 天 | - |
| 實作非同步記憶體掃描 | 開發組 | 5 天 | - |
| 效能基準測試 | QA 組 | 3 天 | - |
| 壓力測試（10,000 events/sec） | QA 組 | 5 天 | 最佳化完成 |
| 效能調校 | 開發組 | 4 天 | 壓力測試結果 |

**交付成果**：
- ✅ 滿足所有效能需求
- ✅ 壓力測試報告
- ✅ 效能調校文件

### 階段 5：文件與部署（預估 2 週）

**目標**：準備生產部署

| 任務 | 負責人 | 預估工時 | 依賴 |
|------|--------|----------|------|
| 撰寫技術文件 | 文件組 | 5 天 | - |
| 撰寫操作手冊 | 文件組 | 3 天 | - |
| 建立部署腳本 | DevOps 組 | 3 天 | - |
| 內部培訓 | 全員 | 2 天 | 文件完成 |
| Beta 測試 | QA 組 | 3 天 | 部署完成 |

**交付成果**：
- ✅ 完整技術文件
- ✅ 操作手冊
- ✅ 部署腳本
- ✅ Beta 測試報告

---

## 測試策略

### 單元測試

**目標**：確保每個元件獨立運作正常

**範圍**：
- 規則解析器（所有特徵類型）
- 邏輯節點（AND/OR/NOT/COUNT）
- 陳述節點（API/String/Regex/Bytes 等）
- 評估上下文
- 快取機制

**工具**：
- C++: Google Test + Google Mock
- Python: pytest

**覆蓋率要求**：≥ 80%

### 整合測試

**目標**：驗證模組間協作

**測試案例**：
1. 規則解析 → 匹配引擎 → 告警生成（端對端流程）
2. 事件標準化 → 評估上下文 → 規則評估
3. 規則相依性解析與執行
4. 快取機制正確性

**工具**：
- 自訂整合測試框架
- 模擬 EDR 事件生成器

**覆蓋率要求**：≥ 70%

### 效能測試

**測試類型**：

1. **基準測試（Benchmark）**
   - 單一規則評估時間
   - 事件處理延遲
   - 記憶體掃描速度

2. **負載測試（Load Test）**
   - 1,000 events/sec 持續 1 小時
   - 5,000 events/sec 持續 10 分鐘

3. **壓力測試（Stress Test）**
   - 10,000 events/sec 持續 5 分鐘
   - 15,000 events/sec 持續 1 分鐘

4. **浸泡測試（Soak Test）**
   - 正常負載持續 24 小時
   - 監控記憶體洩漏

**工具**：
- Apache JMeter（事件生成）
- Valgrind（記憶體分析）
- perf（CPU 分析）

### 安全測試

**測試類型**：

1. **輸入驗證測試**
   - 惡意構造的 YAML 規則
   - 極大/極小數值
   - 特殊字元注入

2. **權限測試**
   - 驗證最小權限執行
   - 權限提升檢測

3. **資料洩漏測試**
   - 記憶體殘留檢查
   - 日誌敏感資料檢查

**工具**：
- OWASP ZAP（漏洞掃描）
- Valgrind（記憶體檢查）

### 兼容性測試

**測試平台**：
- Windows 10/11（x64）
- Windows Server 2019/2022（x64）
- （未來）Linux（Ubuntu 20.04/22.04）

**測試場景**：
- 不同 CPU 架構（Intel/AMD）
- 不同記憶體配置（8GB/16GB/32GB）
- 不同磁碟類型（HDD/SSD/NVMe）

---

## 部署規劃

### 部署架構

```
┌─────────────────────────────────────────┐
│            管理控制台                    │
│    (規則管理、配置管理、告警查看)        │
└─────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│            後端伺服器                    │
│  - 規則儲存庫                            │
│  - 告警接收器                            │
│  - 分析引擎                              │
└─────────────────────────────────────────┘
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
┌──────────────────┐  ┌──────────────────┐
│   EDR Agent 1    │  │   EDR Agent N    │
│  + Capa 引擎     │  │  + Capa 引擎     │
└──────────────────┘  └──────────────────┘
```

### 部署步驟

#### 第 1 步：準備環境

1. 檢查系統需求：
   - 作業系統：Windows 10+ 或 Server 2019+
   - 記憶體：≥ 4GB 可用
   - 磁碟空間：≥ 2GB
   - 網路：能連接後端伺服器

2. 安裝依賴套件：
   - Visual C++ Redistributable 2022
   - .NET Framework 4.8（若使用 .NET 功能）

#### 第 2 步：安裝 Capa 引擎

1. 複製引擎檔案至 EDR 安裝目錄：
   ```
   C:\Program Files\EDR\
   ├── capa-engine.dll
   ├── capa-config.json
   └── rules\
       └── (規則檔案)
   ```

2. 註冊服務：
   ```powershell
   sc create CapaEngine binPath= "C:\Program Files\EDR\capa-service.exe"
   sc config CapaEngine start= auto
   ```

#### 第 3 步：配置引擎

編輯 `capa-config.json`：

```json
{
  "engine": {
    "maxConcurrentEvaluations": 100,
    "eventQueueSize": 10000,
    "memoryScanTimeout": 5000,
    "alertDeduplicationWindow": 300000,
    "enableMemoryScanning": true,
    "enableDisassembly": false
  },
  "rules": {
    "localPath": "C:\\Program Files\\EDR\\rules",
    "remoteUrl": "https://edr-backend.example.com/rules",
    "updateInterval": 3600,
    "filters": {
      "namespaces": ["anti-analysis", "communication", "data-manipulation"],
      "minSeverity": "medium"
    }
  },
  "alerts": {
    "endpoint": "https://edr-backend.example.com/alerts",
    "batchSize": 10,
    "retryAttempts": 3,
    "localCachePath": "C:\\ProgramData\\EDR\\alert-cache"
  },
  "logging": {
    "level": "INFO",
    "path": "C:\\ProgramData\\EDR\\logs\\capa-engine.log",
    "maxSizeMB": 100,
    "maxFiles": 10
  }
}
```

#### 第 4 步：啟動服務

```powershell
net start CapaEngine
```

#### 第 5 步：驗證部署

1. 檢查服務狀態：
   ```powershell
   sc query CapaEngine
   ```

2. 檢查日誌：
   ```powershell
   Get-Content "C:\ProgramData\EDR\logs\capa-engine.log" -Tail 50
   ```

3. 觸發測試告警：
   ```powershell
   # 執行已知惡意行為（在測試環境中）
   rundll32.exe shell32.dll,Control_RunDLL
   ```

### 滾動更新策略

1. **準備階段**
   - 在測試環境驗證新版本
   - 準備回滾計畫

2. **部署階段**
   - 選擇 10% 端點進行金絲雀部署
   - 監控 24 小時，無異常則繼續
   - 逐步擴展至 50%、100%

3. **驗證階段**
   - 檢查告警數量變化
   - 檢查效能指標
   - 收集使用者反饋

4. **回滾機制**
   - 保留舊版本檔案
   - 提供一鍵回滾腳本
   - 回滾後自動驗證

---

## 維護與支援

### 日常維護

#### 規則更新
- **頻率**：每週一次
- **流程**：
  1. 從官方儲存庫拉取最新規則
  2. 在測試環境驗證
  3. 推送至生產環境
  4. 監控告警變化

#### 日誌輪轉
- **策略**：
  - 單一日誌檔案最大 100MB
  - 保留最近 10 個檔案
  - 壓縮歸檔舊日誌

#### 效能監控
- **監控指標**：
  - CPU 使用率（每 5 分鐘）
  - 記憶體使用（每 5 分鐘）
  - 事件處理延遲（即時）
  - 告警數量（每小時）

- **告警閾值**：
  - CPU > 50% 持續 10 分鐘 → 警告
  - 記憶體 > 1.5GB → 警告
  - 處理延遲 > 100ms P95 → 警告

### 故障排除

#### 常見問題

**問題 1：引擎啟動失敗**

*症狀*：服務無法啟動，事件檢視器顯示錯誤

*可能原因*：
- 配置檔案格式錯誤
- 規則檔案損毀
- 記憶體不足

*排查步驟*：
1. 檢查 `capa-config.json` 語法
2. 驗證規則檔案完整性
3. 檢查可用記憶體

**問題 2：記憶體使用過高**

*症狀*：記憶體持續增長，最終耗盡

*可能原因*：
- 記憶體洩漏
- 快取未正確清理
- 事件佇列積壓

*排查步驟*：
1. 檢查事件佇列大小
2. 檢視記憶體快取統計
3. 使用 Valgrind 分析記憶體洩漏

**問題 3：告警數量異常**

*症狀*：告警數量突然暴增或歸零

*可能原因*：
- 規則配置變更
- 去重視窗設置不當
- 後端連接問題

*排查步驟*：
1. 檢查最近的規則更新
2. 驗證告警傳送端點可達
3. 檢視去重設定

### 升級計畫

#### 小版本升級（例如 1.1 → 1.2）

**準備工作**：
- 閱讀變更日誌
- 備份配置檔案
- 在測試環境驗證

**升級步驟**：
1. 停止服務
2. 替換引擎檔案
3. 合併配置檔案變更
4. 啟動服務
5. 驗證功能

**預估停機時間**：< 5 分鐘

#### 大版本升級（例如 1.x → 2.0）

**準備工作**：
- 詳細審閱升級指南
- 測試環境完整測試
- 準備回滾計畫
- 通知使用者

**升級步驟**：
1. 執行資料庫遷移（若需要）
2. 停止服務
3. 備份整個安裝目錄
4. 安裝新版本
5. 遷移配置檔案
6. 啟動服務
7. 執行驗證測試

**預估停機時間**：< 30 分鐘

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
| TLS | Transport Layer Security，傳輸層安全協議 |

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
- v1.0.0 (2025-10-29): 初始版本