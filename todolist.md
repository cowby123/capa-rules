# EDR 支援 Capa 規則：詳細實作計畫

本文件提供一份詳細的、可操作的待辦事項清單，旨在指導開發者或 AI 在 EDR (Endpoint Detection and Response) 產品中實作對 capa 規則的支援。

**核心思想：** 將 capa 的靜態分析特徵轉譯為對 EDR 動態事件流的即時匹配。

---

## 模組 1: 規則解析器 (Rule Parser)

**目標：** 將 YAML 格式的 capa 規則檔案安全地解析為記憶體中強型別的物件。

- [ ] **1.1. 選擇並整合 YAML 函式庫**
    - 任務：為專案選擇一個成熟、安全的 YAML 解析函式庫 (例如 C++ 的 `yaml-cpp`, Rust 的 `serde_yaml`, Python 的 `PyYAML`) 並整合到建置系統中。

- [ ] **1.2. 定義核心資料結構**
    - 任務：以所選語言定義能完整表達 capa 規則的資料結構 (class/struct)。
    - **`Rule` 結構**:
        ```
        class Rule {
            Meta meta;
            FeatureNode features;
        }
        ```
    - **`Meta` 結構**:
        ```
        class Meta {
            string name;
            string namespace;
            list<string> authors;
            map<string, string> scopes; // e.g., {"static": "process", "dynamic": "process"}
            list<string> attck;
            list<string> mbc;
            list<string> references;
            list<string> examples;
            string description;
        }
        ```

- [ ] **1.3. 定義特徵節點 (FeatureNode) 的多態結構**
    - 任務：使用繼承或介面 (interface) 來定義特徵樹的節點，區分為「邏輯節點」和「陳述節點」。
    - **基礎介面 `FeatureNode`**:
        ```
        interface FeatureNode {
            // 每個節點都需要一個被評估的方法
            bool evaluate(EvaluationContext context);
        }
        ```
    - **邏輯節點 (Logic Nodes)**:
        - `AndNode(children: list<FeatureNode>)`
        - `OrNode(children: list<FeatureNode>)`
        - `NotNode(child: FeatureNode)`
        - `SomeNode(count: int, children: list<FeatureNode>)`
        - `OptionalNode(child: FeatureNode)`
    - **陳述節點 (Statement Nodes)**:
        - `ApiStatement(functionName: string)`
        - `StringStatement(value: string)`
        - `RegexStatement(pattern: string)`
        - `BytesStatement(value: bytes)`
        - `NumberStatement(value: int)`
        - `CharacteristicStatement(type: string)`
        - `OsStatement(os: string)`
        - `ArchStatement(arch: string)`
        - `MatchStatement(ruleName: string)`

- [ ] **1.4. 實作解析器邏輯 (`RuleParser`)**
    - 任務：建立一個 `RuleParser` 類別，負責將 YAML 內容轉換為 `Rule` 物件。
    - **主要函式**:
        ```
        class RuleParser {
            // 遞迴函式，將 YAML 節點轉換為 FeatureNode 物件樹
            private FeatureNode parseFeature(yaml_node) { ... }

            // 主函式
            public Rule parse(string yamlContent) {
                // 1. 解析 meta 區塊到 Meta 物件
                // 2. 呼叫 parseFeature 遞迴解析 features 區塊
                // 3. 組裝成 Rule 物件並回傳
            }
        }
        ```

- [ ] **1.5. 撰寫單元測試**
    - 任務：針對 `RuleParser` 撰寫完整的單元測試，確保：
        - 所有邏輯節點和陳述節點都能被正確解析。
        - 巢狀的複雜規則可以被正確建構成樹。
        - 格式錯誤或包含無效特徵的規則會拋出預期的例外。

---

## 模組 2: EDR 事件抽象層 (Event Abstraction Layer)

**目標：** 將 EDR agent 收集到的異質原始事件，標準化為可供匹配引擎使用的格式。

- [ ] **2.1. 定義標準化事件結構**
    - 任務：定義一系列代表端點行為的標準化事件結構。
        - `ApiCallEvent(pid: int, timestamp: datetime, function_name: string, arguments: list, return_value: any)`
        - `ProcessMemoryRegion(pid: int, base_address: long, size: long, content: bytes)`
        - `SystemInfo(os: string, arch: string)`

- [ ] **2.2. 建立事件轉換器 (Event Normalizer)**
    - 任務：為每種 EDR 監控的原始事件（如 API Hook 的原始資料、行程建立的系統事件）編寫轉換邏輯，將其填入對應的標準化事件結構中。

---

## 模組 3: 匹配引擎 (Matching Engine)

**目標：** 根據 EDR 事件流，高效地評估 capa 規則特徵樹。

- [ ] **3.1. 設計評估上下文 (`EvaluationContext`)**
    - 任務：建立一個在評估期間攜帶必要資訊的上下文物件。
    - **`EvaluationContext` 介面**:
        ```
        class EvaluationContext {
            // 取得指定行程的 API 呼叫紀錄
            list<ApiCallEvent> getApiCallHistory(int pid);

            // 取得指定行程的記憶體內容 (可能是懶加載的)
            list<ProcessMemoryRegion> getProcessMemory(int pid);

            // 取得靜態系統資訊
            SystemInfo getSystemInfo();

            // 查詢另一條規則的匹配結果 (支援 `match` 特徵)
            bool getRuleResult(string ruleName);

            // 用於快取，避免重複計算
            map<FeatureNode, bool> evaluationCache;
        }
        ```

- [ ] **3.2. 實作特徵評估器 (`FeatureEvaluator`)**
    - 任務：利用「訪問者模式 (Visitor Pattern)」為 **所有 1.3 中定義的陳述節點** 實作 `evaluate` 方法，定義其在 EDR 環境下的匹配邏輯。
    - **邏輯節點評估**:
        - `AndNode.evaluate`: 對所有 `children` 呼叫 `evaluate`，只有當全部為 `true` 時才回傳 `true` (實現捷徑計算)。
        - `OrNode.evaluate`: 對 `children` 呼叫 `evaluate`，只要任一為 `true` 就回傳 `true` (實現捷徑計算)。
        - `NotNode.evaluate`: 呼叫 `child.evaluate` 並回傳相反結果。
        - `SomeNode.evaluate`: 評估 `children` 直到有 `count` 個為 `true`。
    - **陳述節點評估 (EDR 匹配邏輯)**:
        - [ ] `ApiStatement.evaluate`:
            - **邏輯**: 查詢 `context.getApiCallHistory()`，檢查是否有 API 呼叫的 `function_name` 精確匹配 `ApiStatement.functionName`。
        - [ ] `StringStatement.evaluate` / `SubstringStatement.evaluate`:
            - **邏輯**: 觸發對目標行程的記憶體掃描 (`context.getProcessMemory()`)。在回傳的記憶體區塊 (`ProcessMemoryRegion`) 中搜索完全匹配或部分匹配的字串。
        - [ ] `RegexStatement.evaluate`:
            - **邏輯**: 同上，但在記憶體中執行 `RegexStatement.pattern` 的正規表示式搜索。
        - [ ] `BytesStatement.evaluate`:
            - **邏輯**: 同上，但在記憶體中搜索 `BytesStatement.value` 指定的位元組序列。
        - [ ] `NumberStatement.evaluate`:
            - **邏輯**: 同上，但在記憶體中搜索 `NumberStatement.value` 的二進位表示。
        - [ ] `OffsetStatement.evaluate`:
            - **邏輯**: **[低優先級/可能不適用]** 此特徵在動態分析中意義有限。可考慮忽略，或定義為「從模組基底位置的偏移」。
        - [ ] `OsStatement.evaluate`:
            - **邏輯**: 查詢 `context.getSystemInfo()` 並比對 `os` 欄位。這應在評估開始時快取。
        - [ ] `ArchStatement.evaluate`:
            - **邏輯**: 查詢 `context.getSystemInfo()` 並比對 `arch` 欄位。
        - [ ] `FormatStatement.evaluate`:
            - **邏輯**: 檢查觸發告警的行程其可執行檔格式 (PE)，對應到 `pe`。
        - [ ] `CharacteristicStatement.evaluate`:
            - **邏輯**: 將此對應到 EDR 的高階行為偵測。例如 `characteristic: packed` 可對應到「偵測到行程記憶體區段有高熵值」。`characteristic: in-memory execution` 可對應到「無檔案支援的記憶體執行」事件。
        - [ ] `ImportStatement.evaluate`:
            - **邏輯**: 在行程啟動時，解析其 PE 結構的引入表。查詢 `context` 中快取的引入表資訊，看是否包含 `ImportStatement.importName`。
        - [ ] `ExportStatement.evaluate` / `SectionStatement.evaluate`:
            - **邏輯**: 類似 `ImportStatement`，解析 PE 結構並快取對應資訊。
        - [ ] `MnemonicStatement.evaluate` / `OperandStatement.evaluate`:
            - **邏輯**: **[高成本/選配功能]** 需要對行程記憶體中的可執行區段進行即時反組譯。只有在規則需要時才觸發。
        - [ ] `ClassStatement.evaluate` / `NamespaceStatement.evaluate`:
            - **邏輯**: **[.NET/JIT 環境]** 需要 EDR 具備 .NET Common Language Runtime (CLR) 的監視能力，從 JIT 編譯事件或載入的組件中提取類別與命名空間資訊。
        - [ ] `ComGuidStatement.evaluate`:
            - **邏輯**: 需要 EDR 監控 COM 物件的建立 (`CoCreateInstance`)，並從中提取 GUID。
        - [ ] `MatchStatement.evaluate`:
            - **邏輯**: 呼叫 `context.getRuleResult(MatchStatement.ruleName)` 來取得依賴規則的結果，必須處理循環依賴。

- [ ] **3.3. 管理評估狀態與快取**
    - 任務：為每個受監控的行程維護一個獨立的評估狀態。
    - 實作：在 `EvaluationContext` 中加入 `evaluationCache`，在單次評估流程中，若一個特徵節點已被評估過，直接從快取回傳結果，避免重複的 API 查詢或記憶體掃描。

---

## 模組 4: EDR 整合與工作流程 (Integration & Workflow)

**目標：** 將 capa 引擎無縫嵌入 EDR 的主事件處理流程，並產生有意義的告警。

- [ ] **4.1. 設計主工作流程**
    - **On-Agent-Start**: 從後端或本地載入所有 `scope: process` 的規則，並呼叫 `RuleParser` 解析。
    - **On-Process-Create**: 為新行程建立一個 `EvaluationContext`。
    - **On-Event-Arrival** (如 API 呼叫): 
        1. 將事件正規化後加入對應行程的 `EvaluationContext`。
        2. 觸發對所有規則的 `evaluate` 呼叫。

- [ ] **4.2. 實作懶加載的記憶體掃描**
    - 任務：避免不必要的記憶體掃描以降低效能衝擊。
    - 實作：只有當一條規則的評估路徑進行到需要 `string`, `bytes`, `regex` 等記憶體特徵時，才真正觸發對目標行程的記憶體掃描請求。

- [ ] **4.3. 設計告警生成機制**
    - 任務：當一條規則的根節點 `evaluate` 回傳 `true` 時，生成一個結構化的告警事件。
    - **`CapaMatchAlert` 結構**:
        ```
        class CapaMatchAlert {
            Meta matchedRuleMeta; // 觸發規則的元數據
            ProcessInfo processInfo; // 行程資訊
            // 觸發路徑，用於解釋告警原因
            list<FeatureNode> triggeringPath;
        }
        ```

- [ ] **4.4. 後端告警處理與呈現**
    - 任務：將 `CapaMatchAlert` 事件傳送至 EDR 後端，並在儀表板上以使用者友善的方式呈現，包含 ATT&CK/MBC 資訊、描述和參考連結。

---

## 模組 5: 效能與最佳化

**目標：** 確保引擎高效運作，不影響端點穩定性。

- [ ] **5.1. 實作捷徑評估 (Short-Circuiting)**
    - 任務：在 `AndNode` 和 `OrNode` 的 `evaluate` 實作中，一旦結果確定，應立即回傳，不再評估剩餘的子節點。

- [ ] **5.2. 開發事件預過濾器**
    - 任務：在事件進入引擎前，建立一個快速的雜湊表 (hash set) 儲存所有規則關心的 API 名稱或特徵。不在此列表中的事件可以直接拋棄。

- [ ] **5.3. 實作非同步記憶體掃描**
    - 任務：將耗時的記憶體掃描操作放到獨立的低優先權執行緒中執行，完成後再將結果回饋給主評估流程。

- [ ] **5.4. 設定資源限制**
    - 任務：為 capa 引擎的執行緒設定 CPU 和記憶體使用上限，防止其在極端情況下耗盡系統資源。

- [ ] **5.5. 建立壓力測試場景**
    - 任務：模擬高事件量 (每秒數千事件) 和大量規則 (數千條) 的環境，持續監控引擎的效能指標。