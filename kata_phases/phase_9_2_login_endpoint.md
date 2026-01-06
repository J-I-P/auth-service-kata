# Phase 9.2: POST /login Endpoint實作

## 🎯 本Phase學習目標
- 整合所有layer實作完整的登入endpoint
- 學會HTTP layer和business layer的連接
- 實作正確的錯誤處理和狀態碼
- 完成authentication service的核心功能

---

## 📋 具體任務描述
在 `app/api/auth.py` 中實作 `POST /login` endpoint，整合AuthService、Repository等所有之前建立的layer。

## 🔧 實作要求

### ✅ 必須完成的項目
1. **實作login endpoint**: POST /login接收LoginRequest
2. **整合AuthService**: 調用business logic
3. **錯誤處理**: 401 for invalid credentials, 422 for validation
4. **成功回應**: 200 with LoginResponse
5. **依賴注入**: 正確建立和注入dependencies

### 📝 Endpoint實作結構
```python
# app/api/auth.py
from fastapi import APIRouter, HTTPException, status
from app.schemas.auth import LoginRequest, LoginResponse
from app.services.auth_service import AuthService
from app.repos.user_repo import InMemoryUserRepository

router = APIRouter(prefix="/auth", tags=["authentication"])

# 依賴注入設計
def get_auth_service():
    user_repo = InMemoryUserRepository()
    return AuthService(user_repo)

@router.post("/login", response_model=LoginResponse)
async def login(request: LoginRequest, auth_service: AuthService = Depends(get_auth_service)):
    """
    使用者登入
    - 驗證帳號密碼
    - 回傳access token
    """
    result = await auth_service.login(request)

    if result is None:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid username or password"
        )

    return result
```

### 🎯 整合到main.py
```python
# app/main.py 中新增
from app.api import auth

app.include_router(auth.router)
```

---

## 📚 學習重點

### 🔍 核心概念
1. **Layer整合**:
   - Router layer → Service layer → Repository layer
   - 每層有清楚的職責分工

2. **依賴注入**:
   - 使用FastAPI的Depends機制
   - 控制物件生命週期

3. **錯誤處理**:
   - 業務錯誤轉為HTTP錯誤
   - 適當的狀態碼選擇

4. **API設計**:
   - RESTful路徑設計
   - 一致的回應格式

### 💡 最佳實務
- 不在router中直接寫業務邏輯
- 錯誤訊息不洩漏系統資訊
- 使用response_model確保回應格式

---

## 🔍 Code Review重點

### ✅ 功能檢查
- [ ] endpoint正常接收LoginRequest
- [ ] 成功登入回傳正確的LoginResponse
- [ ] 錯誤情況回傳401狀態碼
- [ ] 依賴注入正常運作

### 🛡️ 安全檢查
- [ ] 密碼不會在回應中洩漏
- [ ] 錯誤訊息不透露使用者是否存在
- [ ] 使用正確的HTTP狀態碼

### 🏗️ 架構檢查
- [ ] Layer分離清楚
- [ ] 依賴方向正確
- [ ] 可擴展性良好

---

## 📝 測試驗證

### ✅ 成功案例測試
```bash
curl -X POST http://localhost:8000/auth/login \
     -H "Content-Type: application/json" \
     -d '{
       "username": "testuser",
       "password": "password123"
     }'

# 預期回應: 200
# {
#   "access_token": "...",
#   "token_type": "bearer"
# }
```

### ❌ 失敗案例測試
```bash
# 錯誤密碼
curl -X POST http://localhost:8000/auth/login \
     -H "Content-Type: application/json" \
     -d '{
       "username": "testuser",
       "password": "wrongpassword"
     }'

# 預期回應: 401
# {"detail": "Invalid username or password"}
```

### 🔍 驗證項目測試
```bash
# 無效的JSON格式
curl -X POST http://localhost:8000/auth/login \
     -H "Content-Type: application/json" \
     -d '{"username": "ab"}'

# 預期回應: 422 (Validation Error)
```

---

## 📖 學習筆記區

### 🤔 我的理解
```
Layer整合的好處：

依賴注入在這裡的作用：

為什麼錯誤訊息要統一：
```

### 🎯 完成感想
```
整合所有layer的感受：

最困難的部分：

最有成就感的部分：
```

---

## 🎉 恭喜完成核心功能！
這是整個kata最重要的milestone，你已經實作了一個完整的authentication endpoint！

## 🚀 下一步預告
**Phase 10.1** 將開始撰寫測試，驗證你剛實作的功能是否正確運作。

---

## ⏰ 執行記錄
- **開始時間**: ___________
- **完成時間**: ___________
- **第一次成功測試時間**: ___________