# 規則格式

capa 使用一組規則來識別程式中的功能。這些規則即使對於逆向工程新手來說也很容易編寫。透過編寫規則，你可以擴展 capa 能夠識別的功能。在某些方面，capa 規則混合了 OpenIOC、Yara 和 YAML 格式。

以下是 capa 使用的一個範例規則：

```yaml
rule:
  meta:
    name: create TCP socket
    namespace: communication/socket/tcp
    authors:
      - william.ballenthin@mandiant.com
      - joakim@intezer.com
      - anushka.virgaonkar@mandiant.com
    scopes:
      static: basic block
      dynamic: call
    mbc:
      - Communication::Socket Communication::Create TCP Socket [C0001.011]
    examples:
      - Practical Malware Analysis Lab 01-01.dll_:0x10001010
  features:
    - or:
      - and:
        - number: 6 = IPPROTO_TCP
        - number: 1 = SOCK_STREAM
        - number: 2 = AF_INET
        - or:
          - api: ws2_32.socket
          - api: ws2_32.WSASocket
          - api: socket
      - property/read: System.Net.Sockets.TcpClient::Client
```

本文件定義了你在編寫 capa 規則時可以使用的可用結構和功能。我們將從高層結構開始，然後深入探討 capa 支援的邏輯結構和功能。

### 目錄
- [規則格式](#規則格式)
  - [yaml](#yaml)
  - [meta 區塊](#meta-區塊)
    - [規則名稱](#規則名稱)
    - [規則命名空間](#規則命名空間)
    - [分析類型](#分析類型)
  - [features 區塊](#features-區塊)
- [提取的功能](#提取的功能)
  - [靜態分析範疇](#靜態分析範疇)
    - [指令功能](#指令功能)
    - [基本區塊功能](#基本區塊功能)
    - [函數功能](#函數功能)
  - [動態分析範疇](#動態分析範疇)
    - [呼叫功能](#呼叫功能)
    - [執行緒功能](#執行緒功能)
    - [程序功能](#程序功能)
  - [通用範疇](#通用範疇)
    - [檔案功能](#檔案功能)
    - [全域功能](#全域功能)
  - [完整功能列表](#完整功能列表)
    - [characteristic（特徵）](#characteristic特徵)
    - [namespace（命名空間）](#namespace命名空間)
    - [class（類別）](#class類別)
    - [api](#api)
    - [property（屬性）](#property屬性)
    - [number（數字）](#number數字)
    - [string 和 substring（字串與子字串）](#string-和-substring字串與子字串)
    - [bytes（位元組）](#bytes位元組)
    - [offset（偏移量）](#offset偏移量)
    - [mnemonic（助記符）](#mnemonic助記符)
    - [operand（運算元）](#operand運算元)
    - [檔案 string 和 substring](#檔案-string-和-substring)
    - [export（匯出）](#export匯出)
    - [import（匯入）](#import匯入)
    - [section（區段）](#section區段)
    - [function-name（函數名稱）](#function-name函數名稱)
    - [os（作業系統）](#os作業系統)
    - [arch（架構）](#arch架構)
    - [format（格式）](#format格式)
  - [計數](#計數)
  - [匹配先前的規則匹配和命名空間](#匹配先前的規則匹配和命名空間)
  - [描述](#描述)
  - [註解](#註解)

## yaml

規則是遵循特定架構的 YAML 檔案。你可以使用任何 YAML 編輯器/語法高亮來協助你。

一旦你有了草稿規則，可以使用 [linter](https://github.com/mandiant/capa/blob/master/scripts/lint.py) 來檢查你的規則是否符合最佳實踐。然後，你應該使用 [formatter](https://github.com/mandiant/capa/blob/master/scripts/capafmt.py) 將規則重新格式化為與所有其他 capa 規則一致的風格。這樣，你就不必在專注於邏輯的同時擔心縮排的寬度。我們在持續整合設定中執行 linter 和 formatter，以確保所有規則都是一致的。

在 YAML 文件中，頂層元素是一個名為 `rule` 的字典，其中包含兩個必需的子字典：`meta` 和 `features`。沒有其他子項。

```yaml
rule:
  meta: ...
  features: ...
```

## meta 區塊

meta 區塊包含識別規則、技術分組以及提供額外文件參考的後設資料。以下是一個範例：

```yaml
meta:
  name: packed with UPX
  namespace: anti-analysis/packer/upx
  authors:
    - william.ballenthin@mandiant.com
  description: the sample appears to be packed with UPX
  scopes:
    static: file
    dynamic: file
  att&ck:
    - Defense Evasion::Obfuscated Files or Information [T1027.002]
  mbc:
    - Anti-Static Analysis::Software Packing
  examples:
    - CD2CBA9E6313E8DF2C1273593E649682
    - Practical Malware Analysis Lab 01-02.exe_:0x0401000
```

以下是常見欄位：

  - `name` 是必需的。此字串應唯一識別規則。更多細節見下方。

  - `namespace` 在規則描述技術時是必需的，並幫助我們將規則分組到不同的類別中。更多細節見下方。

  - `authors` 是規則作者的姓名或帳號列表。

  - `description` 是描述規則意圖或解釋的可選文字。

  - `scopes` 表示在分析靜態或動態分析產物時，規則適用於哪個功能集。有兩個必需的子欄位：`static` 和 `dynamic`。以下是合法值：
    - `scopes.static`：
      - **`instruction`**：匹配在單一指令中找到的功能。這對於識別結構存取或與魔術常數的比較非常有用。
      - **`basic block`**：匹配每個基本區塊中的功能。這用於在規則中實現緊密的局部性（例如函數的參數）。
      - **`function`**：匹配每個函數中的功能。
      - **`file`**：匹配整個檔案中的功能。
    - `scopes.dynamic`：
      - **`call`**：匹配每個追蹤的 API 呼叫位置的功能，例如 API 名稱和參數值。
      - **`span of calls`**：匹配執行緒內 API 呼叫的滑動視窗中的功能。
      - **`thread`**：匹配每個執行緒中的功能。
      - **`process`**：匹配每個程序中的功能。
      - **`file`**：匹配整個檔案的功能，包括可執行檔案功能*以及*整個執行時追蹤。

  - `att&ck` 是規則暗示的 [ATT&CK 框架](https://attack.mitre.org/) 技術的可選列表，例如 `Discovery::Query Registry [T1012]` 或 `Persistence::Create or Modify System Process::Windows Service [T1543.003]`。這些標籤用於在呈現報告時推導樣本的 ATT&CK 映射。

  - `mbc` 是規則暗示的 [惡意軟體行為目錄](https://github.com/MBCProject/mbc-markdown) 技術的可選列表，類似於 ATT&CK 列表。

  - `maec/malware-category` 在規則描述角色時是必需的，例如 `dropper` 或 `backdoor`。

  - `maec/malware-family` 在規則描述惡意軟體家族時是必需的，例如 `PlugX` 或 `Beacon`。

  - `maec/analysis-conclusion` 在規則描述處置時是必需的，例如 `benign` 或 `malicious`。

  - `examples` 是規則應匹配的樣本參考的*必需*列表。linter 會驗證每個規則在規則的 `examples` 列表中引用的每個樣本上正確觸發。這些範例檔案儲存在 [github.com/mandiant/capa-testfiles](https://github.com/mandiant/capa-testfiles) 儲存庫中。`function` 和 `basic block` 範疇規則必須使用格式 `<樣本名稱>:<函數或基本區塊偏移量>` 包含到各自匹配位置的偏移量。

  - `references` 在書籍、文章、部落格文章等中找到的相關資訊列表。

不允許使用其他欄位，linter 會對它們發出警告。

### 規則名稱

`rule.meta.name` 唯一識別一個規則。它可以在其他規則中引用，因此如果你更改規則名稱，請務必搜尋交叉引用。

按照慣例，規則名稱應完成以下句子之一：
  - "程式/函數可能..."
  - "程式被..."

為了聚焦規則名稱，我們嘗試省略冠詞（the/a/an）。例如，偏好 `make HTTP request` 而不是 `make an HTTP request`。

當規則描述實現技術的特定方法時，通常用 "via XYZ" 指定。例如，`make HTTP request via WinInet` 或 `make HTTP request via libcurl`。

當規則描述特定程式語言或執行時，通常用 "in ABC" 指定。

因此，這些是好的規則名稱：
  - (函數可能) "**make HTTP request via WinInet**"
  - (函數可能) "**encrypt data using RC4 via WinCrypt**"
  - (程式被) "**compiled by MSVC**"
  - (程式可能) "**capture screenshot in Go**"

...而這些是不好的規則名稱：
  - "UPX"
  - "encryption with OpenSSL"

### 規則命名空間

規則命名空間幫助我們將相關規則分組在一起。你會注意到規則檔案的檔案系統佈局與它們包含的命名空間相匹配。此外，capa 的輸出按命名空間排序，因此所有 `communication` 匹配都會彼此相鄰呈現。

命名空間是階層式的，因此命名空間的子項編碼了其特定技術。用幾個詞來說，頂層命名空間是：

  - [anti-analysis](https://github.com/mandiant/capa-rules/tree/master/anti-analysis/) - 打包、混淆、反 X 等
  - [collection](https://github.com/mandiant/capa-rules/tree/master/collection/) - 可能被枚舉和收集以供外洩的資料
  - [communication](https://github.com/mandiant/capa-rules/tree/master/communication/) - HTTP、TCP、命令與控制（C2）流量等
  - [compiler](https://github.com/mandiant/capa-rules/tree/master/compiler/) - 偵測建置環境，例如 MSVC、Delphi 或 AutoIT
  - [data-manipulation](https://github.com/mandiant/capa-rules/tree/master/data-manipulation/) - 加密、雜湊等
  - [executable](https://github.com/mandiant/capa-rules/tree/master/executable/) - 可執行檔的特徵，例如 PE 區段或除錯資訊
  - [host-interaction](https://github.com/mandiant/capa-rules/tree/master/host-interaction/) - 存取或操作系統資源，例如程序或登錄檔
  - [impact](https://github.com/mandiant/capa-rules/tree/master/impact/) - 最終目標
  - [internal](https://github.com/mandiant/capa-rules/tree/master/internal/) - capa 內部使用以指導分析
  - [lib](https://github.com/mandiant/capa-rules/tree/master/lib/) - 建立其他規則的建構區塊
  - [linking](https://github.com/mandiant/capa-rules/tree/master/linking/) - 偵測依賴項，例如 OpenSSL 或 Zlib
  - [load-code](https://github.com/mandiant/capa-rules/tree/master/load-code/) - 執行時載入和執行程式碼，例如嵌入的 PE 或 shellcode
  - [malware-family](https://github.com/mandiant/capa-rules/tree/master/malware-family/) - 偵測惡意軟體家族
  - [nursery](https://github.com/mandiant/capa-rules/tree/master/nursery/) - 尚未完善的規則的暫存區
  - [persistence](https://github.com/mandiant/capa-rules/tree/master/persistence/) - 維持存取的各種方式
  - [runtime](https://github.com/mandiant/capa-rules/tree/master/runtime/) - 偵測語言執行時，例如 .NET 平台或 Go
  - [targeting](https://github.com/mandiant/capa-rules/tree/master/targeting/) - 對系統的特殊處理，例如 ATM 機器

我們可以根據需要輕鬆添加更多頂層命名空間。

所有命名空間元件都應該是描述功能概念的名詞，除了可能是最後一個元件。例如，以下是描述與系統硬體互動功能的命名空間子樹：

```
host-interaction/hardware
host-interaction/hardware/storage
host-interaction/hardware/memory
host-interaction/hardware/cpu
host-interaction/hardware/mouse
host-interaction/hardware/keyboard
host-interaction/hardware/keyboard/layout
host-interaction/hardware/cdrom
```

當命名空間有許多常見操作，並且每個操作有許多實現方式時，最後的路徑元件可能是描述操作的動詞。例如，在 Windows 上有*許多*方式執行多個檔案操作，因此命名空間子樹如下所示：

```
rules/host-interaction/file-system
rules/host-interaction/file-system/create
rules/host-interaction/file-system/delete
rules/host-interaction/file-system/write
rules/host-interaction/file-system/copy
rules/host-interaction/file-system/exists
rules/host-interaction/file-system/read
rules/host-interaction/file-system/list
```

命名空間樹的深度不受限制，但我們發現 3-4 個元件通常就足夠了。

### 分析類型

capa 分析在可執行檔案和沙箱（例如 CAPE）擷取的 API 追蹤中找到的功能。我們將這些分析類別稱為「類型」（flavors），並分別使用「靜態分析類型」和「動態分析類型」來指代它們。靜態分析非常適合查看程式的整個邏輯並找到有趣的區域。透過沙箱進行的動態分析有助於繞過打包（在惡意軟體中非常普遍），並能更好地描述程式的實際執行時行為。我們使用 `meta.scopes.$flavor` 鍵來指定規則如何與特定類型互動。

在可能的情況下，我們嘗試編寫在靜態和動態分析類型中都能運作的 capa 規則。例如，以下是在兩種類型中都匹配的規則：

```yml
rule:
  meta:
    name: create mutex
    namespace: host-interaction/mutex
    authors:
      - moritz.raabe@mandiant.com
      - michael.hunhoff@mandiant.com
    scopes:
      static: function
      dynamic: call
  features:
    - or:
      - api: kernel32.CreateMutex
      - api: kernel32.CreateMutexEx
      - api: System.Threading.Mutex::ctor
```

看看 `create mutex` 如何透過檢查反組譯功能（靜態分析）以及執行時 API 追蹤（動態分析）進行推理？

另一方面，某些行為最好由僅在一個範疇中運作的規則來描述。記住，規則的可讀性至關重要，因此避免為了合併規則而使邏輯複雜化。在這種情況下，用 `unsupported` 標記排除的範疇，如下列規則所示：

```yml
rule:
  meta:
    name: check for software breakpoints
    namespace: anti-analysis/anti-debugging/debugger-detection
    authors:
      - michael.hunhoff@mandiant.com
    scopes:
      static: function
      dynamic: unsupported  # 需要助記符功能
  features:
    - and:
      - or:
        - instruction:
          - mnemonic: cmp
          - number: 0xCC = INT3
      - match: contain loop
```

`check for software breakpoints` 在反組譯分析期間運作良好，其中可以匹配低階指令功能，但在動態範疇中不起作用，因為這些功能不可用。因此，我們標記規則 `scopes.dynamic: unsupported`，這樣在處理沙箱追蹤時就不會考慮該規則。

如你將在[提取的功能](#提取的功能)部分看到的，capa 在各種範疇匹配功能，從小（例如 `instruction`）到大（例如 `file`）。在靜態分析中，範疇從 `instruction` 增長到 `basic block`、`function`，然後到 `file`。在動態分析中，範疇從 `call` 增長到 `thread`、`process`，然後到 `file`。

匹配 API 呼叫序列時，靜態範疇通常是 `function`，動態範疇是 `span of calls`。匹配帶參數的單個 API 呼叫時，靜態範疇通常是 `basic block`，動態範疇是 `call`。有一天我們希望在靜態分析類型中直接支援 `call` 範疇。


## features 區塊

本節宣告有關規則匹配所必須存在的功能的邏輯陳述。

有五種可以嵌套的結構表達式：
  - `and` - 所有子表達式都必須匹配
  - `or` - 至少匹配一個子項
  - `not` - 當子表達式不匹配時匹配
  - `N or more` - 至少匹配 `N` 個或更多子項
    - `optional` 是 `0 or more` 的別名，對於記錄相關功能很有用。有關範例，請參見 [write-file.yml](/host-interaction/file-system/write/write-file.yml)。

要為陳述添加上下文，你可以以 `- description: 描述字串` 的形式添加*一個*嵌套描述條目。查看[描述部分](#描述)以獲取更多詳細資訊。

例如，考慮以下規則：

```yaml
      - and:
        - description: CRC-32 演算法的核心
        - mnemonic: shr
        - number: 0xEDB88320
        - number: 8
        - characteristic: nzxor
      - api: RtlComputeCrc32
```

要匹配此規則，函數必須：
  - 包含 `shr` 指令，並且
  - 引用立即常數 `0xEDB88320`，有些人可能認出它與 CRC32 校驗和相關，並且
  - 引用數字 `8`，並且
  - 具有不尋常的功能，在這種情況下，包含非歸零 XOR 指令
如果在函數中僅找到這些功能之一，則規則將不匹配。


# 提取的功能

capa 在多個範疇匹配功能，從小（例如 `instruction`）到大（例如 `file`）。在靜態分析中，範疇從 `instruction` 增長到 `basic block`、`function`，然後到 `file`。在動態分析中，範疇從 `call` 增長到 `thread`、`process`，然後到 `file`：

| 靜態範疇 | 最適合... |
|----------|-----------|
| instruction | 特定的助記符、運算元、常數等組合，用於尋找魔術值 |
| basic block | 緊密相關的指令，例如結構存取或函數呼叫參數 |
| function | API 呼叫、常數等的集合，暗示完整的功能 |
| file | 高層結論，例如加密器、後門或靜態連結某些函式庫 |
| global | 在每個範疇可用的功能，例如架構或作業系統 |

| 動態範疇 | 最適合... |
|----------|-----------|
| call | 單個 API 呼叫及其參數 |
| span of calls | 跨越多個 API 呼叫但少於整個執行緒的行為，整個執行緒可能非常長 |
| thread | 來自多個獨立 span-of-calls 範疇的功能組合（不常見） |
| process | 在（可能是多執行緒的）程式中找到的其他功能組合 |
| file | 高層結論，例如加密器、後門或靜態連結某些函式庫 |
| global | 在每個範疇可用的功能，例如架構或作業系統 |

一般來說，capa 從較低範疇收集功能並將其合併到較高範疇；例如，從單個指令提取的功能會合併到包含這些指令的函數範疇中。這樣，你可以使用針對指令的匹配結果（"常數 X 用於加密演算法 Y"）來識別函數級功能（"加密函數 Z"）。

| 功能 | 靜態範疇 | 動態範疇 |
|------|----------|----------|
| [api](#api) | instruction ↦ basic block ↦ function ↦ file | call ↦ span of calls ↦ thread ↦ process ↦ file |
| [string](#string-和-substring字串與子字串) | instruction ↦ ... | call ↦ ... |
| [bytes](#bytes位元組) | instruction ↦ ... | call ↦ ... |
| [number](#number數字) | instruction ↦ ... | call ↦ ... |
| [characteristic](#characteristic特徵) | instruction ↦ ... | - |
| [mnemonic](#mnemonic助記符) | instruction ↦ ... | - |
| [operand](#operand運算元) | instruction ↦ ... | - |
| [offset](#offset偏移量) | instruction ↦ ... | - |
| [com](#com) | instruction ↦ ... | - |
| [namespace](#namespace命名空間) | instruction ↦ ... | - |
| [class](#class類別) | instruction ↦ ... | - |
| [property](#property屬性) | instruction ↦ ... | - |
| [export](#export匯出) | file | file |
| [import](#import匯入) | file | file |
| [section](#section區段) | file | file |
| [function-name](#function-name函數名稱) | file | - |
| [os](#os作業系統) | global | global |
| [arch](#arch架構) | global | global |
| [format](#format格式) | global | global |

## 靜態分析範疇

### 指令功能

指令功能源自單個指令，例如助記符、字串引用或函數呼叫。以下功能在此範疇及以上範疇相關：

  - [namespace](#namespace命名空間)
  - [class](#class類別)
  - [api](#api)
  - [property](#property屬性)
  - [number](#number數字)
  - [string 和 substring](#string-和-substring字串與子字串)
  - [bytes](#bytes位元組)
  - [com](#com)
  - [offset](#offset偏移量)
  - [mnemonic](#mnemonic助記符)
  - [operand](#operand運算元)

此外，以下 [characteristics](#characteristic特徵) 在此範疇及以上範疇相關：
  - `nzxor`
  - `peb access`
  - `fs access`
  - `gs access`
  - `cross section flow`
  - `indirect call`
  - `call $+5`
  - `unmanaged call`

### 基本區塊功能

基本區塊功能源自同一基本區塊內找到的指令範疇功能的組合。

此外，以下 [characteristics](#characteristic特徵) 在此範疇及以上範疇相關：
  - `tight loop`
  - `stack string`

### 函數功能

函數功能源自同一函數內找到的指令和基本區塊範疇功能的組合。

此外，以下 [characteristics](#characteristic特徵) 在此範疇及以上範疇相關：
  - `loop`
  - `recursive call`
  - `calls from`
  - `calls to`

## 動態分析範疇

### 呼叫功能

呼叫功能從單個沙箱追蹤事件（例如 API 呼叫）收集。它們通常用於匹配 API 名稱和參數（字串或整數常數）。

以下功能在此範疇及以上範疇相關：

  - [api](#api)
  - [number](#number數字)
  - [string 和 substring](#string-和-substring字串與子字串)
  - [bytes](#bytes位元組)

### span-of-calls 功能

「Span of calls」範疇匹配執行緒內 API 呼叫的滑動視窗中的功能。此範疇對於識別跨越多個 API 呼叫的行為很有用，例如 `OpenFile`/`ReadFile`/`CloseFile`，而不必分析整個執行緒，整個執行緒可能非常長。

span-of-calls 範疇不強制呼叫的順序，而是匹配視窗內的一組呼叫。當前視窗大小為 20 個 API 呼叫。選擇這個大小是為了在捕獲跨多個呼叫的邏輯與平衡效能權衡之間取得平衡。

當 span of calls 規則匹配時，它只報告一系列重疊 span 中的第一個匹配，以避免用重複的結果淹沒使用者，例如當程式在緊密迴圈中執行行為時。但是，其他規則可以匹配這些「錘擊」（hammered）的匹配。

沒有 span 特定的功能。

### 執行緒功能

執行緒範疇匹配在同一執行緒內找到的 call 和 span-of-calls 範疇的行為。

雖然不常見，但當規則考慮執行緒內的整個行為集合時，或者至少是非常長的呼叫序列時，這可能很有用。你可能會這樣做以對執行緒的完整活動做出結論，例如「定期注入瀏覽器程序的背景執行緒」。

但是，此範疇容易出現誤報，因為執行緒可能包含大量不保證直接相關的事件。因此，在可能的情況下，偏好使用 span-of-calls 範疇。

沒有執行緒特定的功能。

### 程序功能

程序功能是在同一程序內找到的執行緒範疇功能的組合。這對於匹配在整個程式中找到的行為很有用，即使它是多執行緒的。

沒有程序特定的功能。

## 通用範疇

### 檔案功能

檔案功能源自檔案結構，即 PE 結構或原始檔案資料。

此外，在所有函數（靜態）或所有程序（動態）中找到的所有功能都會收集到檔案範疇中。

此範疇支援以下功能：

  - [string 和 substring](#檔案-string-和-substring)
  - [export](#export匯出)
  - [import](#import匯入)
  - [section](#section區段)
  - [function-name](#function-name函數名稱)
  - [namespace](#namespace命名空間)
  - [class](#class類別)

### 全域功能

全域功能在所有範疇提取。這些功能可能對反組譯和檔案結構解釋都有用，例如目標作業系統或架構。此範疇支援以下功能：

  - [os](#os作業系統)
  - [arch](#arch架構)
  - [format](#format格式)

## 完整功能列表

### characteristic（特徵）

Characteristics 是由分析引擎提取的功能。它們是作者認為有趣的一次性功能。

例如，`characteristic: nzxor` 功能描述非歸零 XOR 指令。

| characteristic | 範疇 | 描述 |
|----------------|------|------|
| `characteristic: embedded pe` | file | （XOR 編碼）嵌入的 PE 檔案。 |
| `characteristic: forwarded export` | file | PE 檔案具有轉送的匯出。 |
| `characteristic: mixed mode` | file | 檔案包含託管和非託管（原生）程式碼，通常在 .NET 中看到 |
| `characteristic: loop` | function | 函數包含迴圈。 |
| `characteristic: recursive call` | function | 函數是遞迴的。 |
| `characteristic: calls from` | function | 此函數有唯一的呼叫。最好這樣使用：`count(characteristic(calls from)): 3 or more` |
| `characteristic: calls to` | function | 此函數被唯一呼叫。最好這樣使用：`count(characteristic(calls to)): 3 or more` |
| `characteristic: tight loop` | basic block, function | 一個基本區塊分支到自身的緊密迴圈。 |
| `characteristic: stack string` | basic block, function | 有一系列看起來像堆疊字串建構的指令。 |
| `characteristic: nzxor` | instruction, basic block, function | 非歸零 XOR 指令 |
| `characteristic: peb access` | instruction, basic block, function | 存取程序環境區塊（PEB），例如透過 fs:[30h]、gs:[60h] |
| `characteristic: fs access` | instruction, basic block, function | 透過 `fs` 區段存取記憶體。 |
| `characteristic: gs access` | instruction, basic block, function | 透過 `gs` 區段存取記憶體。 |
| `characteristic: cross section flow` | instruction, basic block, function | 函數包含對不同區段的呼叫/跳躍。這通常在解包存根中看到。 |
| `characteristic: indirect call` | instruction, basic block, function | 間接呼叫指令；例如，`call edx` 或 `call qword ptr [rsp+78h]`。 |
| `characteristic: call $+5` | instruction, basic block, function | 呼叫剛過當前指令。 |
| `characteristic: unmanaged call` | instruction, basic block, function | 函數包含從託管程式碼到非託管（原生）程式碼的呼叫，通常在 .NET 中看到 |

### namespace（命名空間）
程式邏輯使用的命名空間。

參數是描述命名空間名稱的字串，指定為 `namespace` 或 `namespace.nestednamespace`。

範例：

    namespace: System.IO
    namespace: System.Net

### class（類別）
程式邏輯使用的命名類別。如果可恢復，這必須包括類別的命名空間。

參數是描述類別的字串，指定為 `namespace.class` 或 `namespace.nestednamespace.class`。

範例：

    class: System.IO.File
    class: System.Net.WebResponse

範例規則：[create new application domain in .NET](../host-interaction/memory/create-new-application-domain-in-dotnet.yml)


### api
呼叫命名函數，可能是匯入，也可能是透過函數簽名匹配（如 FLIRT）提取的本地函數（如 `malloc`）。

參數是描述函數名稱的字串，指定為 `functionname`、`module.functionname` 或 `namespace.class::functioname`。

從版本 7 開始，模組（DLL）名稱在匹配期間不使用，因此僅對文件有益。

採用字串參數的 Windows API 函數有兩個 API 版本。例如，`CreateProcessA` 採用 ANSI 字串，而 `CreateProcessW` 採用 Unicode 字串。capa 提取這些 API 功能時帶有和不帶有後綴字元 `A` 或 `W`。這意味著你可以使用基本名稱編寫規則來匹配兩個 API。如果你想匹配特定的 API 版本，可以包含後綴。

.NET 類別和結構實現建構函式（`.ctor`）和靜態建構函式（`.cctor`）方法。capa 將這些建構函式方法分別提取為 `namespace.class::ctor` 和 `namespace.class::cctor`。

範例：

    api: kernel32.CreateFile  # DLL 名稱在匹配期間將被忽略，但作為文件包含是好的
    api: CreateFile  # 匹配 Ansi (CreateFileA) 和 Unicode (CreateFileW) 版本
    api: GetEnvironmentVariableW  # 僅匹配 Unicode 版本
    api: System.IO.File::Delete
    api: System.Net.WebResponse::GetResponseStream
    api: System.Threading.Mutex::ctor # 匹配建立 System.Threading.Mutex 物件

範例規則：[switch active desktop](../host-interaction/gui/switch-active-desktop.yml)

### property（屬性）
程式邏輯使用的類別或結構的成員。如果可恢復，這必須包括成員的類別和命名空間。

參數是描述成員的字串，指定為 `namespace.class::member` 或 `namespace.nestednamespace.class::member`。如果你打算在讀取引用的屬性時發生匹配，你也可以指定 `/read` 存取器，或者如果你打算在寫入引用的屬性時發生匹配，則指定 `/write` 存取器。

範例：

    property/read: System.Environment::OSVersion
    property/write: System.Net.WebRequest::Proxy

範例規則：[enumere GUI resources](../host-interaction/gui/enumerate-gui-resources.yml)

### number（數字）
程式邏輯使用的數字。這不應該是堆疊或結構偏移量。例如，加密常數。

參數是一個數字；如果以 `0x` 為前綴，則為十六進位格式，否則為十進位格式。

為了幫助人類理解數字的含義，例如常數 `0x40` 表示 `PAGE_EXECUTE_READWRITE`，你可以在定義旁邊提供描述。使用內聯語法（首選），在行末添加 ` = 描述字串`。查看[描述部分](#描述)以獲取更多詳細資訊。

範例：

    number: 16
    number: 0x10
    number: 0x40 = PAGE_EXECUTE_READWRITE

注意，capa 將所有數字視為無符號值。負數不是有效的功能值。要匹配負數，你可以指定其二補數表示。例如，32 位元檔案中的 `0xFFFFFFF0`（`-2`）。

如果數字僅在特定架構上相關，請不要猶豫使用如下模式：

```yml
- and:
  - arch: i386
  - number: 4 = size of pointer
```

範例規則：[get disk size](../host-interaction/hardware/storage/get-disk-size.yml)

### string 和 substring（字串與子字串）
程式邏輯引用的字串。這可能是指向 ASCII 或 Unicode 字串的指標。這也可能是混淆的字串，例如堆疊字串。

參數是描述字串的字串。這可以是逐字值或匹配字串的正規表示式。

逐字值必須用雙引號括起來，特殊字元必須轉義。

特殊字元是以下之一：
  - 反斜線，應表示為 `string: "\\"`
  - 換行符或其他非空格空白（例如製表符、CR、LF 等），應表示為 `string: "\n"`
  - 雙引號，應表示為 `string: "\""`

capa 僅匹配逐字字串，例如 `"Mozilla"` 不匹配 `"User-Agent: Mozilla/5.0"`。要匹配帶有前導/尾隨萬用字元的逐字子字串，使用 substring 功能，例如 `substring: "Mozilla"`。對於更複雜的模式，使用下面描述的正規表示式語法。

正規表示式應該用 `/` 字元括起來。預設情況下，capa 使用區分大小寫的匹配，並假設前導和尾隨萬用字元。要執行不區分大小寫的匹配，附加 `i`。要在字串的開頭或結尾錨定正規表示式，使用 `^` 和/或 `$`。例如，`/mozilla/i` 匹配 `"User-Agent: Mozilla/5.0"`。

要為字串添加上下文，使用下面顯示的兩行語法 `...description: 描述字串`。這裡不支援內聯語法。查看[描述部分](#描述)以獲取更多詳細資訊。

範例：

```yaml
- string: "Firefox 64.0"
- string: "Hostname:\t\t\t%s\nIP address:\t\t\t%s\nOS version:\t\t\t%s\n"
- string: "This program cannot be run in DOS mode."
  description: MS-DOS 存根訊息
- string: "{3E5FC7F9-9A51-4367-9063-A120244FBEC7}"
  description: CLSID_CMSTPLUA
- string: /SELECT.*FROM.*WHERE/
  description: SQL WHERE 子句
- string: /Hardware\\Description\\System\\CentralProcessor/i
- substring: "CurrentVersion"
```

注意，正規表示式和子字串匹配成本高昂（`O(features)` 而不是 `O(1)`），因此應謹慎使用。

範例規則：[identify ATM dispenser service provider](../targeting/automated-teller-machine/identify-atm-dispenser-service-provider.yml)

### bytes（位元組）
程式邏輯引用的位元組序列。提供的序列必須從引用位元組的開頭匹配，且不超過 `0x100` 位元組。參數是十六進位位元組序列。為了幫助人類理解位元組序列的含義，你可以提供描述。為此，使用內聯語法，附加你的 ` = 描述字串`。查看[描述部分](#描述)以獲取更多詳細資訊。

下面的範例說明了在呼叫 `CoCreateInstance` 之前將 COM CLSID 推送到堆疊時的位元組匹配。

反組譯：

    push    offset iid_004118d4_IShellLinkA ; riid
    push    1               ; dwClsContext
    push    0               ; pUnkOuter
    push    offset clsid_004118c4_ShellLink ; rclsid
    call    ds:CoCreateInstance

範例規則元素：

    bytes: 01 14 02 00 00 00 00 00 C0 00 00 00 00 00 00 46 = CLSID_ShellLink
    bytes: EE 14 02 00 00 00 00 00 C0 00 00 00 00 00 00 46 = IID_IShellLink

範例規則：[hash data using Whirlpool](../nursery/hash-data-using-whirlpool.yml)

### com
COM 功能表示程式邏輯中使用的元件物件模型（COM）介面和類別。它們有助於識別與 COM 物件、方法、屬性和介面的互動。參數是 COM 類別或介面的名稱。此功能允許你列出人類可讀的名稱，而不是程式中找到的位元組表示。

範例：

```yaml
- com/class: InternetExplorer  # bytes: 01 DF 02 00 00 00 00 00 C0 00 00 00 00 00 00 46 = CLSID_InternetExplorer
- com/interface: IWebBrowser2  # bytes: 61 16 0C D3 AF CD D0 11 8A 3E 00 C0 4F C9 E2 6E = IID_IWebBrowser2
```

規則解析器透過從內部 COM 資料庫獲取 GUID，將 com 功能轉換為其 `bytes` 和 `string` 表示。

上述規則的翻譯表示：

```yaml
- or:
  - string : "0002DF01-0000-0000-C000-000000000046"
    description: CLSID_InternetExplorer as GUID string
  - bytes : 01 DF 02 00 00 00 00 00 C0 00 00 00 00 00 00 46 = CLSID_InternetExplorer as bytes
- or:
  - string: "D30C1661-CDAF-11D0-8A3E-00C04FC9E26E"
    description: IID_IWebBrowser2 as GUID string
  - bytes: 61 16 0C D3 AF CD D0 11 8A 3E 00 C0 4F C9 E2 6E = IID_IWebBrowser2 as bytes
```

注意：自動添加的描述有助於保持一致性並改進文件。

### offset（偏移量）
程式邏輯引用的結構偏移量。這不應該是堆疊偏移量。

參數是一個數字；如果以 `0x` 為前綴，則為十六進位格式，否則為十進位格式。支援負偏移量。偏移量後面可以跟隨可選描述。

如果數字僅與特定架構相關，則可以使用架構變體之一：`number/x32` 或 `number/x64`。

範例：

```yaml
offset: 0xC
offset: 0x14 = PEB.BeingDebugged
offset: -0x4
```

如果偏移量僅在特定架構（例如 32 位元或 64 位元 Intel）上相關，請不要猶豫使用如下模式：

```yml
- and:
  - arch: i386
  - offset: 0xC = offset to linked list head
```

### mnemonic（助記符）

在給定函數中找到的指令助記符。

參數是包含助記符的字串。

範例：

    mnemonic: xor
    mnemonic: shl

範例規則：[check for trap flag exception](../anti-analysis/anti-debugging/debugger-detection/check-for-trap-flag-exception.yml)

### operand（運算元）

特定運算元索引的數字和偏移值。當你想指定資料從來源/目的地的流動時，使用這些功能，例如從結構移動或與常數比較。

範例：

    operand[0].number: 0x10
    operand[1].offset: 0x2C

範例規則：[encrypt data using XTEA](../data-manipulation/encryption/xtea/encrypt-data-using-xtea.yml)


### 檔案 string 和 substring
檔案中存在的 ASCII 或 UTF-16 LE 字串。

參數是描述字串的字串。這可以是逐字值、逐字子字串或匹配字串的正規表示式，並應使用與 [string](#string-和-substring字串與子字串) 功能相同的格式。

範例：

    string: "Z:\\Dev\\dropper\\dropper.pdb"
    string: "[ENTER]"
    string: /.*VBox.*/
    string: /.*Software\\Microsoft\Windows\\CurrentVersion\\Run.*/i
    substring: "CurrentVersion"

注意，正規表示式和子字串匹配成本高昂（`O(features)` 而不是 `O(1)`），因此應謹慎使用。

### export（匯出）

從共享函式庫匯出的例程的名稱。

範例：

    export: InstallA

要指定[轉送的匯出](https://devblogs.microsoft.com/oldnewthing/20060719-24/?p=30473)，使用格式 `<DLL 路徑，小寫>.<符號名稱>`。注意，路徑可以是隱式、相對或絕對的：

    export: "c:/windows/system32/version.GetFileVersionInfoA"
    export: "vresion.GetFileVersionInfoA"

範例規則：[act as password filter DLL](../persistence/authentication-process/act-as-password-filter-dll.yml)

### import（匯入）

從共享函式庫匯入的例程的名稱。這些可以包括在匹配期間檢查的 DLL 名稱。

範例：

    import: kernel32.WinExec
    import: WinExec           # 萬用字元模組名稱
    import: kernel32.#22      # 按序數
    import: System.IO.File::Exists

範例規則：[load NCR ATM library](../targeting/automated-teller-machine/ncr/load-ncr-atm-library.yml)

### function-name（函數名稱）

識別的靜態連結函式庫的名稱，例如透過 FLIRT 恢復的名稱，或從檔案中包含的資訊（例如 .NET 後設資料）提取的名稱。這讓你可以編寫描述第三方函式庫功能的規則，例如「透過 CryptoPP 使用 AES 加密資料」。

範例：

    function-name: "?FillEncTable@Base@Rijndael@CryptoPP@@KAXXZ"
    function-name: Malware.Backdoor::Beacon

範例規則：[execute via .NET startup hook](../runtime/dotnet/execute-via-dotnet-startup-hook.yml)

### section（區段）

結構化檔案中區段的名稱。

範例：

    section: .rsrc


範例規則：[compiled with DMD](../compiler/d/compiled-with-dmd.yml)

### os（作業系統）

樣本執行所在作業系統的名稱。這是透過應用於檔案格式的啟發式確定的（例如，PE 檔案用於 Windows，ELF 檔案中的標頭欄位和註釋部分表示 Linux/*BSD/等）。這讓你可以將僅應在某些平台上找到的邏輯分組，例如 Windows API 僅在 Windows 可執行檔中找到。

範例：

```yml
- or:
  - and:
    description: Windows 特定的 API
    os: windows
    api: CreateFile

  - and:
    description: POSIX 特定的 API
    or:
      - os: linux
      - os: macos
      - ...
    api: fopen
```

有效的作業系統：
  - `windows`
  - `linux`
  - `macos`
  - `hpux`
  - `netbsd`
  - `hurd`
  - `86open`
  - `solaris`
  - `aix`
  - `irix`
  - `freebsd`
  - `tru64`
  - `modesto`
  - `openbsd`
  - `openvms`
  - `nsk`
  - `aros`
  - `fenixos`
  - `cloud`
  - `syllable`
  - `nacl`

注意：你可以透過不指定 `os` 功能或使用 `any` 來匹配任何有效的作業系統，例如 `- os: any`。

範例規則：[discover group policy via gpresult](../collection/group-policy/discover-group-policy-via-gpresult.yml)

### arch（架構）

樣本執行所在 CPU 架構的名稱。這讓你可以將僅應在某些架構上找到的邏輯分組，例如 Intel CPU 的組合語言指令。

有效的架構：
  - `i386` Intel 32 位元
  - `amd64` Intel 64 位元

注意：今天 capa 僅明確支援 Intel 架構（`i386` 和 `amd64`）。因此，大多數規則假設 Intel 指令和助記符。你不必在規則中明確包含此條件：

```yml
- and:
  - mnem: lea
  - or:
    # 此區塊不是必需的！
    - arch: i386
    - arch: amd64
```

但是，如果你有許多架構特定偏移量的群組，這可能很有用，例如：

```yml
- or:
  - and:
    - description: 32 位元結構欄位
    - arch: i386
    - offset: 0x12
    - offset: 0x1C
    - offset: 0x20
  - and:
    - description: 64 位元結構欄位
    - arch: amd64
    - offset: 0x28
    - offset: 0x30
    - offset: 0x40
```

這可能比使用許多 `offset/x32` 或 `offset/x64` 功能更容易理解。

範例規則：[get process heap flags](../host-interaction/process/get-process-heap-flags.yml)

### format（格式）

檔案格式的名稱。

有效格式：
  - `pe`
  - `elf`
  - `dotnet`

範例規則：[access .NET resource](../executable/resource/access-dotnet-resource.yml)

## 計數

許多規則會檢查功能集中功能的選擇組合；但是，某些規則可能會考慮在功能集中看到功能的次數。

這些規則可以表示為：

    count(characteristic(nzxor)): 2           # 完全匹配 count==2
    count(characteristic(nzxor)): 2 or more   # 至少兩次匹配
    count(characteristic(nzxor)): 2 or fewer  # 最多兩次匹配
    count(characteristic(nzxor)): (2, 10)     # 匹配範圍 2<=count<=10 中的任何值

    count(mnemonic(mov)): 3
    count(basic blocks): 4

`count` 支援內聯描述，除了 [strings](#string-和-substring字串與子字串)，透過以下語法：

    count(number(2 = AF_INET/SOCK_DGRAM)): 2

## 匹配先前的規則匹配和命名空間

capa 規則可以指定匹配其他規則匹配或命名空間的邏輯。這允許規則作者將常見的功能模式重構為它們自己的可重用元件。你可以像這樣指定規則匹配表達式：
```yaml
  - and:
      - match: create process
      - match: host-interaction/file-system/write
```
規則由其 `rule.meta.name` 屬性唯一識別；這是應該出現在 `match` 表達式右側的值。

如果匹配期間不存在規則依賴項，capa 將拒絕執行。同樣，你應該確保不要在彼此匹配的規則之間引入循環依賴。

常見的規則模式，例如實現「寫入檔案」的各種方式，可以重構為「函式庫規則」。這些是具有 `rule.meta.lib: True` 的規則。預設情況下，函式庫規則不會作為規則匹配輸出給使用者，但可以被其他規則匹配。當沒有活動規則依賴函式庫規則時，這些函式庫規則將不會被評估 - 保持效能。

## 描述

所有功能和陳述都支援可選描述，這有助於記錄規則並在 capa 的輸出中提供上下文。

對於除 [strings](#string-和-substring字串與子字串) 之外的所有功能，描述可以內聯指定，前面加上 ` = `：` = 描述字串`。例如：

```yaml
- number: 0x5A4D = IMAGE_DOS_SIGNATURE (MZ)
```

內聯語法是首選的。對於 [strings](#string-和-substring字串與子字串)，或者如果描述很長或包含換行符，使用兩行語法。它使用 `description` 標籤，方式如下：`description: 描述字串`。

對於[陳述](#features-區塊)，你可以向陳述添加*一個*嵌套描述條目。

例如：

```yaml
- or:
  - string: "This program cannot be run in DOS mode."
    description: MS-DOS 存根訊息
  - number: 0x5A4D
    description: IMAGE_DOS_SIGNATURE (MZ)
  - and:
    - description: 此 `and` 陳述的文件
    - offset: 0x50 = IMAGE_NT_HEADERS.OptionalHeader.SizeOfImage
    - offset: 0x34 = IMAGE_NT_HEADERS.OptionalHeader.ImageBase
  - and:
    - offset: 0x50 = IMAGE_NT_HEADERS64.OptionalHeader.SizeOfImage
    - offset: 0x30 = IMAGE_NT_HEADERS64.OptionalHeader.ImageBase
```

## 註解

Capa 規則支援內聯/行尾和區塊註解

例如：

```yaml
features:
    # 常數字拼出 ASCII 中的 "expand 32-byte k"（即 4 個字是 "expa"、"nd 3"、"2-by" 和 "te k"）
    - or:
      - description: 金鑰設定的一部分
      - string: "expand 32-byte k = sigma"
      - string: "expand 16-byte k = tau"
      - string: "expand 32-byte kexpand 16-byte k"  # 如果 sigma 和 tau 在連續記憶體中，可能會導致串聯字串
      - and:
        - string: "expa"
        - string: "nd 3"
        - string: "2-by"
        - string: "te k"
      - and:
        - number: 0x61707865 = "apxe"
        - number: 0x3320646E = "3 dn"
        - number: 0x79622D32 = "yb-2"
        - number: 0x6B206574 = "k et"
```
