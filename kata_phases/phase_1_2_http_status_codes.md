# Phase 1.2: HTTP狀態碼實作

## 🎯 本Phase學習目標
- 理解HTTP狀態碼的意義和分類
- 學會在FastAPI中回傳不同狀態碼
- 實作錯誤處理的基礎概念
- 建立RESTful API的狀態回應習慣

---

## 📋 具體任務描述
實作三個新的endpoints來展示不同的HTTP狀態碼：200 (成功), 400 (客戶端錯誤), 404 (找不到資源)。

## 🔧 實作要求

### ✅ 必須完成的項目
1. **GET /status/ok**: 回傳200狀態碼和成功訊息
2. **GET /status/bad-request**: 回傳400狀態碼和錯誤訊息
3. **GET /status/not-found**: 回傳404狀態碼和找不到訊息
4. **保留之前功能**: ping和echo endpoints繼續正常運作
5. **使用FastAPI的HTTPException**: 正確拋出HTTP錯誤

### 📝 實作結構提示
```python
from fastapi import FastAPI, HTTPException

# 200 - 成功回應
@app.get("/status/ok")
async def status_ok():
    return {"status": "success", "message": "Everything is working fine"}

# 400 - 客戶端錯誤
@app.get("/status/bad-request")
async def status_bad_request():
    # 使用 HTTPException 回傳400
    pass

# 404 - 資源不存在
@app.get("/status/not-found")
async def status_not_found():
    # 使用 HTTPException 回傳404
    pass
```

### 🎯 預期回應
1. `GET /status/ok` → 200
   ```json
   {"status": "success", "message": "Everything is working fine"}
   ```

2. `GET /status/bad-request` → 400
   ```json
   {"detail": "This is a simulated bad request error"}
   ```

3. `GET /status/not-found` → 404
   ```json
   {"detail": "The requested resource was not found"}
   ```

---

## 📚 學習重點

### 🔍 核心概念
1. **HTTP狀態碼分類**:
   - 2xx: 成功 (200 OK, 201 Created)
   - 4xx: 客戶端錯誤 (400 Bad Request, 404 Not Found, 401 Unauthorized)
   - 5xx: 服務器錯誤 (500 Internal Server Error)

2. **HTTPException**:
   - FastAPI處理錯誤的標準方式
   - 自動設定正確的狀態碼和格式

3. **錯誤回應格式**:
   - FastAPI的預設錯誤格式 `{"detail": "message"}`
   - 一致性的重要性

### 💡 深度思考
- 什麼情況下應該使用不同的狀態碼？
- 為什麼要有統一的錯誤格式？
- 前端應用如何根據狀態碼處理回應？

---

## 🔍 Code Review重點

### ✅ 功能檢查
- [ ] 三個新endpoint都能正常工作
- [ ] 狀態碼正確 (200, 400, 404)
- [ ] 錯誤訊息清晰明確
- [ ] HTTPException使用正確

### 🎨 程式品質
- [ ] import語句正確
- [ ] 錯誤訊息有意義
- [ ] 函數命名清楚

---

## 📝 測試驗證

### 🧪 測試所有endpoints
```bash
# 測試200狀態碼
curl -w "%{http_code}\n" http://localhost:8000/status/ok

# 測試400狀態碼
curl -w "%{http_code}\n" http://localhost:8000/status/bad-request

# 測試404狀態碼
curl -w "%{http_code}\n" http://localhost:8000/status/not-found

# 確認自動文檔更新
# 訪問 http://localhost:8000/docs
```

### 📊 預期結果
每個endpoint應該回傳對應的HTTP狀態碼和適當的JSON回應。

---

## 📖 學習筆記區

### 🤔 我的理解
```
1. 常用HTTP狀態碼含義：
   - 200:
   - 400:
   - 401:
   - 404:
   - 500:

2. HTTPException的作用：


3. 為什麼需要標準的錯誤格式：

```

### 🔬 實驗發現
```
測試過程中的發現：

在瀏覽器 vs curl 的表現差異：

自動文檔中的變化：
```

---

## 🚀 下一步預告
**Phase 1.3** 將學習URL參數處理，包括路徑參數和查詢參數。

---

## ⏰ 執行記錄
- **開始時間**: ___________
- **完成時間**: ___________