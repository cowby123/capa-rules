# lib 資料夾說明

本資料夾收錄了各種常用的 capa 行為規則（YAML 格式），這些規則可用於靜態或動態分析二進位檔案時，自動辨識出特定的 API 呼叫、系統行為、演算法特徵等。這些規則多半屬於「基礎積木」型態，可被其他高階規則重複引用。

## 規則分類與簡介

- **記憶體操作**
  - `allocate-memory.yml`：偵測分配記憶體的 API（如 VirtualAlloc、NtAllocateVirtualMemory 等）。
  - `allocate-or-change-rw-memory.yml`：偵測分配或變更可讀寫記憶體區段。
  - `change-memory-protection.yml`：偵測變更記憶體保護屬性的 API（如 VirtualProtect）。
  - `write-process-memory.yml`：偵測寫入其他行程記憶體的 API（如 WriteProcessMemory）。

- **檔案/註冊表/物件操作**
  - `create-or-open-file.yml`：偵測建立或開啟檔案的 API。
  - `create-or-open-registry-key.yml`：偵測建立或開啟註冊表機碼的 API。
  - `create-or-open-section-object.yml`：偵測建立或開啟 section object 的 API。
  - `create-file-compression-interface-context-on-windows.yml`、`create-file-decompression-interface-context-on-windows.yml`：偵測 Windows Cabinet 檔案壓縮/解壓縮介面。

- **程序/執行緒/服務操作**
  - `open-process.yml`：偵測開啟行程的 API。
  - `open-thread.yml`：偵測開啟執行緒的 API。
  - `get-service-handle.yml`：偵測取得服務控制代碼的 API。

- **系統資訊/環境**
  - `get-os-version.yml`：偵測取得作業系統版本的 API 或 PEB 欄位存取。
  - `peb-access.yml`：偵測對 PEB（Process Environment Block）的存取。

- **演算法/資料處理**
  - `calculate-modulo-256-via-x86-assembly.yml`：偵測以 x86 指令計算 256 取餘的特徵。
  - `validate-payment-card-number-using-luhn-algorithm-with-lookup-table.yml`、`validate-payment-card-number-using-luhn-algorithm-with-no-lookup-table.yml`：偵測 Luhn 演算法（信用卡號驗證）的不同實作方式。

- **其他常見行為**
  - `delay-execution.yml`：偵測延遲執行（如 Sleep、WaitForSingleObject 等）。
  - `duplicate-stdin-and-stdout.yml`：偵測複製標準輸入/輸出的行為（如 dup2）。
  - `contain-loop.yml`：偵測程式中存在迴圈、遞迴等特徵。
  - `contain-pusha-popa-sequence.yml`：偵測 x86 指令 pusha/popa（或 pushad/popad）序列。

## 使用說明

這些規則主要供 capa 進行行為分析時作為底層積木，建議不要直接修改。若需自訂高階行為判斷，可在其他規則中引用這些 lib 規則。

---

如需更詳細的規則內容，請直接參考各 yml 檔案。