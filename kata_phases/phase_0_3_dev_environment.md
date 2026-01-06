# Phase 0.3: 開發環境啟動

## 🎯 本Phase學習目標
- 學會啟動FastAPI開發服務器
- 理解uvicorn的基本使用
- 體驗熱重載(hot reload)機制
- 第一次測試你的API endpoint

---

## 📋 具體任務描述
使用uvicorn啟動你在Phase 0.1建立的FastAPI應用程式，並透過瀏覽器測試 `/ping` endpoint。

## 🔧 實作要求

### ✅ 必須完成的項目
1. **啟動開發服務器**: 使用uvicorn命令啟動app
2. **測試API**: 在瀏覽器訪問 `http://localhost:8000/ping`
3. **查看自動文檔**: 訪問 `http://localhost:8000/docs`
4. **體驗熱重載**: 修改回應內容並觀察自動重啟
5. **正常關閉服務器**: 學會使用Ctrl+C停止服務

### 📝 啟動命令
```bash
# 基本啟動命令
uvicorn app.main:app

# 開發模式（推薦）
uvicorn app.main:app --reload

# 指定host和port（可選）
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

### 🎯 測試檢查點
1. 服務器成功啟動，顯示類似：
   ```
   INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
   INFO:     Started reloader process
   ```

2. 訪問 `http://localhost:8000/ping` 看到：
   ```json
   {"message": "pong"}
   ```

3. 訪問 `http://localhost:8000/docs` 看到自動生成的API文檔

---

## 📚 學習重點

### 🔍 核心概念
1. **ASGI Server**:
   - uvicorn是ASGI（異步服務器網關介面）實作
   - 負責處理HTTP請求和回應

2. **Hot Reload**:
   - 程式碼變更時自動重啟服務器
   - 提升開發效率

3. **自動文檔生成**:
   - FastAPI自動生成OpenAPI/Swagger文檔
   - `/docs` 提供互動式API文檔

### 💡 深度思考
- 為什麼需要ASGI server？
- Hot reload如何檢測程式碼變更？
- 自動文檔是如何生成的？

---

## 🔍 Code Review重點

### ✅ 執行檢查
- [ ] 服務器能夠成功啟動
- [ ] `/ping` endpoint正常回應
- [ ] 自動文檔可以訪問
- [ ] 熱重載功能正常
- [ ] 能夠正常停止服務器

### 🛠️ 故障排除
常見問題：
- 端口被占用：嘗試不同的port
- 模組找不到：檢查工作目錄和PYTHONPATH
- import錯誤：確認app.main模組存在

---

## 📝 實驗任務

### 🧪 熱重載測試
1. 啟動服務器（使用--reload）
2. 修改 `/ping` 的回應內容，例如改成 `{"message": "pong!", "status": "ok"}`
3. 保存檔案
4. 重新訪問 `/ping`，確認變更生效
5. 還原原始內容

### 🌐 自動文檔探索
1. 訪問 `http://localhost:8000/docs`
2. 找到 `/ping` endpoint
3. 點擊「Try it out」按鈕
4. 執行請求並查看回應

---

## 📖 學習筆記區

### 🤔 我的理解
```
1. uvicorn的作用：


2. Hot reload的好處：


3. 自動文檔的價值：

```

### 🐛 遇到的問題
```
問題1：
解決：

問題2：
解決：
```

---

## 🚀 下一步預告
**Phase 1.1** 將學習HTTP方法的差異，實作POST endpoint來理解GET和POST的不同。

---

## ⏰ 執行記錄
- **開始時間**: ___________
- **完成時間**: ___________
- **第一次成功啟動時間**: ___________