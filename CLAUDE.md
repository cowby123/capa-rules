# Claude 專案設定

## 語言設定

**重要：本專案的所有 Claude 互動都必須使用繁體中文**

### 規則
- 所有問答對話使用繁體中文
- 程式碼註解使用繁體中文
- 提交訊息（commit messages）使用繁體中文
- 文件說明使用繁體中文
- 錯誤訊息和日誌使用繁體中文

### 範例

#### 程式碼註解
```python
# 正確：檢查檔案是否存在
def check_file_exists(path):
    # 使用 os.path 模組來驗證路徑
    return os.path.exists(path)

# 錯誤：Check if file exists
def check_file_exists(path):
    # Use os.path module to validate path
    return os.path.exists(path)
```

#### 提交訊息
```
正確：新增使用者驗證功能
錯誤：Add user authentication feature
```

#### 互動對話
```
使用者：這個函數做什麼？
Claude：這個函數負責檢查檔案是否存在於指定路徑。

而非：
User: What does this function do?
Claude: This function checks if a file exists at the specified path.
```

---

**最後更新：2025-10-28**
