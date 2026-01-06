# Phase 1.1: HTTP方法理解

## 🎯 本Phase學習目標
- 理解GET和POST的根本差異
- 學會處理POST請求的body
- 實作第一個接收資料的endpoint
- 理解RESTful API設計原則

---

## 📋 具體任務描述
在main.py中新增一個 `POST /echo` endpoint，接收任何JSON資料並原樣回傳，同時保留原有的 `GET /ping`。

## 🔧 實作要求

### ✅ 必須完成的項目
1. **保留原有功能**: `GET /ping` 繼續正常運作
2. **新增POST endpoint**: 實作 `POST /echo`
3. **接收JSON資料**: endpoint能夠接收任意JSON body
4. **原樣回傳**: 將接收到的資料完整回傳
5. **使用正確decorator**: 使用 `@app.post()` decorator

### 📝 實作提示
```python
# 你需要實作類似這樣的結構：

@app.post("/echo")
async def echo_data(data: ???):  # 這裡需要適當的型別提示
    # 回傳接收到的資料
    return ???
```

### 🎯 測試方法
使用curl或Postman測試：
```bash
curl -X POST http://localhost:8000/echo \
     -H "Content-Type: application/json" \
     -d '{"name": "test", "value": 123}'

# 期望回應：
# {"name": "test", "value": 123}
```

---

## 📚 學習重點

### 🔍 核心概念
1. **HTTP方法差異**:
   - GET: 獲取資源，沒有body，冪等性
   - POST: 提交資料，有body，非冪等

2. **Request Body**:
   - POST可以攜帶資料
   - FastAPI自動解析JSON

3. **型別提示**:
   - 使用適當的型別來描述接收的資料
   - FastAPI會根據型別自動驗證

### 💡 深度思考
- 為什麼GET不適合傳送敏感資料？
- POST的body和URL參數有什麼差異？
- 什麼時候應該用GET，什麼時候用POST？

---

## 🔍 Code Review重點

### ✅ 功能檢查
- [ ] 原有 `GET /ping` 仍然正常
- [ ] 新的 `POST /echo` 可以正常工作
- [ ] 能夠接收並回傳任意JSON資料
- [ ] HTTP方法使用正確

### 🎨 程式品質
- [ ] 使用適當的型別提示
- [ ] 函數命名清楚表達功能
- [ ] 程式碼結構清晰

### 🧪 測試案例
- [ ] 簡單物件: `{"key": "value"}`
- [ ] 複雜物件: `{"user": {"name": "John", "age": 30}}`
- [ ] 陣列: `[1, 2, 3]`
- [ ] 空物件: `{}`

---

## 📝 實驗任務

### 🧪 不同資料型別測試
測試以下不同的JSON資料：

1. **字串資料**:
   ```json
   {"message": "hello world"}
   ```

2. **數字資料**:
   ```json
   {"count": 42, "price": 3.14}
   ```

3. **複雜物件**:
   ```json
   {
     "user": {
       "name": "Alice",
       "preferences": ["coding", "reading"]
     }
   }
   ```

4. **陣列資料**:
   ```json
   ["apple", "banana", "cherry"]
   ```

---

## 📖 學習筆記區

### 🤔 我的理解
```
1. GET vs POST的主要差異：


2. Request Body的作用：


3. FastAPI如何處理JSON：

```

### 🔬 實驗結果
```
測試案例1結果：

測試案例2結果：

測試案例3結果：

意外發現：
```

---

## 🚀 下一步預告
**Phase 1.2** 將學習HTTP狀態碼，實作能夠回傳不同狀態碼的endpoint。

---

## ⏰ 執行記錄
- **開始時間**: ___________
- **完成時間**: ___________