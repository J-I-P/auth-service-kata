# Phase 0.1: 第一個FastAPI應用程式

## 🎯 本Phase學習目標
- 理解FastAPI應用程式的最基本架構
- 學會建立FastAPI instance
- 實作人生第一個HTTP endpoint
- 了解Python decorator在路由系統中的魔法

---

## 📋 具體任務描述
在 `app/main.py` 中建立一個最簡單的FastAPI應用程式，實作 `GET /ping` endpoint，回傳固定的JSON: `{"message": "pong"}`

## 🔧 實作要求

### ✅ 必須完成的項目
1. **Import FastAPI**: 正確import FastAPI類別
2. **建立app實例**: 建立FastAPI應用程式物件
3. **實作路由**: 使用`@app.get()`裝飾器
4. **回傳JSON**: 函數回傳dictionary格式的資料
5. **程式可執行**: 檔案import時不會出錯

### 📝 程式碼結構預期
```python
# 你需要寫類似這樣的結構，但請自己實作：

# 1. import 語句

# 2. 建立 FastAPI app

# 3. 定義路由函數

# 4. (main.py就這些內容就夠了)
```

### ❌ 本Phase不需要做的
- 不用寫啟動程式碼 (`if __name__ == "__main__"`)
- 不用處理錯誤情況
- 不用考慮其他HTTP方法
- 不用新增其他endpoint

---

## 📚 學習重點

### 🔍 核心概念
1. **FastAPI Class**:
   - 這是整個應用程式的核心
   - 負責處理HTTP請求和路由

2. **Decorator Pattern**:
   - `@app.get()` 是Python decorator
   - 它將普通函數"升級"為HTTP handler

3. **路徑定義**:
   - `"/ping"` 定義了URL路徑
   - 使用者訪問 `http://server/ping` 會觸發這個函數

4. **自動JSON序列化**:
   - FastAPI會自動將Python dict轉為JSON回應
   - 不需要手動處理JSON格式

### 💡 深度思考
- 為什麼用decorator而不是直接調用函數？
- FastAPI如何知道這個函數要處理GET請求？
- 回傳dict跟回傳JSON string有什麼差異？

---

## 🔍 Code Review重點

完成後我會檢查以下項目：

### ✅ 正確性檢查
- [ ] 是否正確import FastAPI
- [ ] 是否建立FastAPI app實例
- [ ] 是否使用正確的decorator語法
- [ ] 是否回傳正確的資料格式

### 🎨 程式風格檢查
- [ ] 變數命名是否有意義（通常app實例叫做`app`）
- [ ] 函數命名是否描述其功能
- [ ] 是否有適當的空行分隔邏輯區塊
- [ ] import語句是否放在檔案最上方

### 🧠 概念理解檢查
- [ ] 是否理解decorator的作用
- [ ] 是否理解FastAPI的基本工作原理
- [ ] 是否知道為什麼回傳dict就變成JSON

---

## 📝 測試驗證

### 🏃‍♂️ 執行測試
完成後，你的程式應該能夠：
```bash
# 1. 檔案可以被Python成功import (不會出錯)
python -c "from app.main import app; print('Success!')"

# 2. 下個phase我們會學如何實際啟動server測試
```

### 🎯 成功標準
- 程式碼可以執行不出錯
- 有明確的`GET /ping`路由定義
- 回傳格式正確：`{"message": "pong"}`

---

## 🧠 延伸思考問題

### 💭 概念題
1. 為什麼FastAPI選擇用decorator來定義路由？
2. 如果我們回傳`"pong"`字串而不是`{"message": "pong"}`會怎樣？
3. 除了`@app.get()`，你猜還有哪些HTTP方法的decorator？

### 🔬 實驗題
1. 試試看回傳不同的資料型別（string, list, 數字）
2. 試試看定義不同的路徑（如`/hello`）
3. 試試看路由函數取不同的名字

### 🌟 FastAPI獨有特色預告
完成這個基本endpoint後，你會發現FastAPI的神奇之處：
- **自動文檔生成**: 下個phase啟動server後，訪問 `http://localhost:8000/docs` 會看到自動生成的Swagger UI
- **型別驗證**: 基於Python type hints的自動資料驗證（後續phases會深入）
- **效能優化**: 內建async/await支援，高效能的API響應
- **開發體驗**: 自動補全、錯誤提示等現代化開發體驗

這就是為什麼FastAPI被稱為「現代化的Python Web框架」！

---

## 📖 學習筆記區

### 🤔 我的理解
```
寫下你對以下概念的理解：

1. FastAPI是什麼？


2. Decorator在這裡的作用是什麼？


3. 為什麼回傳dict會自動變成JSON？

```

### 😅 遇到的問題
```
記錄實作過程中遇到的問題和解決方式：

問題1:
解決:

問題2:
解決:
```

### 💡 新發現
```
記錄任何意外的發現或有趣的觀察：


```

---

## 🚀 下一步預告
**Phase 0.2** 將學習：
- 設定專案依賴 (pyproject.toml)
- 安裝FastAPI和uvicorn
- 理解Python專案的依賴管理

**為什麼重要**: 真實的專案需要依賴管理，而且我們需要安裝uvicorn才能實際運行你剛寫的程式碼！

---

## ⏰ 執行記錄
- **開始時間**: ___________
- **完成時間**: ___________
- **花費時間**: ___________
- **困難程度** (1-5): ___________
- **理解程度** (1-5): ___________

---

## ✨ 完成了嗎？
當你完成這個Phase後，把你的程式碼貼給我，我會進行詳細的code review，然後我們就可以進入Phase 0.2了！

記住：**這只是個開始，但每個大師都是從Hello World開始的！** 🌟