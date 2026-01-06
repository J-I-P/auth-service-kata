# Phase 2.3: 回應資料序列化

## 🎯 本Phase學習目標
- 學會定義response model
- 理解資料序列化的概念
- 實作type-safe的API回應
- 掌握FastAPI的response_model機制

---

## 📋 具體任務描述
定義response model並在endpoint中使用，確保API回應的格式一致性和型別安全。

## 🔧 實作要求

### ✅ 必須完成的項目
1. **定義UserResponse model**: 不包含敏感資訊的使用者回應格式
2. **使用response_model**: 在endpoint中指定回應模型
3. **測試序列化**: 驗證回應格式正確
4. **處理Optional欄位**: 學會處理可選的回應欄位

### 📝 實作結構提示
```python
# app/schemas/user.py 擴展
class UserResponse(BaseModel):
    id: int
    name: str
    email: str
    # 注意：不包含密碼等敏感資訊

# 在endpoint中使用
@app.post("/users", response_model=UserResponse)
async def create_user(user: User) -> UserResponse:
    # 處理邏輯
    return UserResponse(...)
```

---

## 📚 學習重點

### 🔍 核心概念
1. **Response Model**: 定義API回應的結構
2. **資料序列化**: 物件轉JSON的過程
3. **資訊隱藏**: 不在回應中洩漏敏感資料
4. **Type Safety**: 編譯時期的型別檢查

### 💡 最佳實務
- Request model vs Response model分離
- 永遠不在response中包含密碼
- 使用clear的命名慣例

---

## 📖 學習筆記區

### 🤔 我的理解
```
Response model的好處：

序列化的作用：

為什麼要分離request/response model：
```

---

## ⏰ 執行記錄
- **開始時間**: ___________
- **完成時間**: ___________