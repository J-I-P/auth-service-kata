# Phase 8.1: Token生成與FastAPI安全功能

## 🎯 本Phase學習目標
- 實作基本的token生成邏輯
- 了解FastAPI的安全功能框架
- 學習OAuth2標準在FastAPI中的實作
- 體驗現代API的安全設計模式

---

## 📋 具體任務描述
實作簡單但安全的token生成機制，為將來整合JWT或其他認證方案做準備，同時體驗FastAPI的安全功能。

## 🔧 實作要求

### ✅ 必須完成的項目
1. **建立token工具模組**: 實作token生成和管理邏輯
2. **整合到AuthService**: 在登入成功時生成token
3. **學習FastAPI安全**: 了解security模組的基本概念
4. **設計可擴展架構**: 為未來JWT整合預留空間

### 📝 Token生成實作
```python
# app/utils/token.py (新建立)
import uuid
import secrets
from typing import Optional
from datetime import datetime, timedelta

class SimpleTokenManager:
    """簡單的token管理器 - 生產環境應使用JWT"""

    def __init__(self):
        self._tokens = {}  # 記憶體儲存，實際應用需要Redis

    def generate_token(self, username: str, expires_minutes: int = 60) -> str:
        """生成access token"""
        # 使用cryptographically secure的隨機字串
        token = secrets.token_urlsafe(32)

        # 設定過期時間
        expires_at = datetime.utcnow() + timedelta(minutes=expires_minutes)

        # 儲存token資訊
        self._tokens[token] = {
            "username": username,
            "expires_at": expires_at,
            "created_at": datetime.utcnow()
        }

        return token

    def verify_token(self, token: str) -> Optional[str]:
        """驗證token並回傳username"""
        if token not in self._tokens:
            return None

        token_data = self._tokens[token]

        # 檢查是否過期
        if datetime.utcnow() > token_data["expires_at"]:
            del self._tokens[token]  # 清除過期token
            return None

        return token_data["username"]

    def revoke_token(self, token: str) -> bool:
        """撤銷token（登出功能）"""
        if token in self._tokens:
            del self._tokens[token]
            return True
        return False

# app/services/auth_service.py 中整合
from app.utils.token import SimpleTokenManager

class AuthService:
    def __init__(self, user_repo: UserRepository):
        self.user_repo = user_repo
        self.token_manager = SimpleTokenManager()

    async def login(self, request: LoginRequest) -> Optional[LoginResponse]:
        # 驗證使用者和密碼...
        user = await self.user_repo.find_by_username(request.username)
        if not user or not verify_password(request.password, user.hashed_password):
            return None

        # 生成token
        access_token = self.token_manager.generate_token(user.username)

        return LoginResponse(
            access_token=access_token,
            token_type="bearer"
        )
```

### 🎯 FastAPI安全功能預覽
```python
# app/security.py - 為未來擴展準備
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

# FastAPI的安全性scheme
security = HTTPBearer()

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security)
) -> str:
    """從token解析當前使用者 - 未來可擴展為JWT"""
    token = credentials.credentials

    # 這裡可以整合JWT驗證
    # username = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])

    # 目前使用簡單的token驗證
    token_manager = SimpleTokenManager()
    username = token_manager.verify_token(token)

    if not username:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid or expired token",
            headers={"WWW-Authenticate": "Bearer"},
        )

    return username

# 使用範例
@router.get("/protected")
async def protected_endpoint(current_user: str = Depends(get_current_user)):
    return {"message": f"Hello {current_user}!"}
```

---

## 📚 學習重點

### 🔍 核心概念
1. **Token生成策略**: 安全隨機字串 vs JWT vs Session
2. **Token儲存**: 記憶體 vs Redis vs Database
3. **Token驗證**: 過期檢查和安全性驗證
4. **Bearer Token**: HTTP Authorization header標準

### 🌟 FastAPI安全功能框架
FastAPI提供完整的安全生態系：

#### **Security Schemes**
```python
# OAuth2密碼流程
from fastapi.security import OAuth2PasswordBearer
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

# HTTP Bearer
from fastapi.security import HTTPBearer
bearer_scheme = HTTPBearer()

# API Key
from fastapi.security import HTTPAPIKey
api_key_scheme = HTTPAPIKey(name="X-API-Key")

# HTTP Basic
from fastapi.security import HTTPBasic
basic_auth = HTTPBasic()
```

#### **自動文檔整合**
- 安全schemes自動出現在Swagger UI
- "Authorize"按鈕讓使用者輸入credentials
- 自動加入security headers到請求中

#### **依賴注入整合**
```python
# 保護整個router
router = APIRouter(dependencies=[Depends(get_current_user)])

# 保護特定endpoint
@router.get("/admin", dependencies=[Depends(require_admin)])
async def admin_only(): pass
```

### 💡 安全設計考量
```python
# 1. Token過期機制
def generate_token(username: str, expires_minutes: int = 60):
    # 合理的過期時間平衡安全性和使用者體驗

# 2. Secure token生成
import secrets
token = secrets.token_urlsafe(32)  # 比uuid更安全

# 3. Token撤銷機制
def logout(token: str):
    token_manager.revoke_token(token)  # 支援登出

# 4. Rate limiting (未來擴展)
from slowapi import Limiter
limiter = Limiter(key_func=get_remote_address)

@limiter.limit("5/minute")
@router.post("/login")
async def login(): pass
```

---

## 🔍 Code Review重點

### ✅ 安全性檢查
- [ ] 使用cryptographically secure的隨機生成
- [ ] Token有適當的過期機制
- [ ] 敏感資訊不會洩漏在logs中
- [ ] 錯誤訊息不透露系統內部資訊

### 🏗️ 架構檢查
- [ ] Token管理邏輯分離到獨立模組
- [ ] 易於替換為JWT或其他方案
- [ ] 依賴注入設計合理
- [ ] 為水平擴展做準備

---

## 📝 實驗任務

### 🧪 安全性實驗
1. **Token唯一性驗證**:
   ```python
   # 生成1000個token，確認沒有重複
   tokens = [token_manager.generate_token("user") for _ in range(1000)]
   assert len(set(tokens)) == 1000
   ```

2. **過期機制測試**:
   ```python
   # 測試短過期時間
   token = token_manager.generate_token("user", expires_minutes=0.01)
   time.sleep(1)
   assert token_manager.verify_token(token) is None
   ```

3. **FastAPI安全文檔**:
   啟動server，訪問 `/docs`，觀察Swagger UI的"Authorize"按鈕

---

## 📖 學習筆記區

### 🤔 我的理解
```
簡單token vs JWT的差異：

Bearer token的標準格式：

為什麼要有token過期機制：

FastAPI security的設計哲學：
```

### 🔒 安全思考
```
這個簡單token方案的安全風險：

水平擴展時的token同步問題：

如何平衡安全性和使用者體驗：

未來升級到JWT的考量：
```

### 💡 架構設計
```
為什麼要分離token管理邏輯：

如何設計可插拔的認證架構：

生產環境需要考慮的額外因素：
```

---

## 🚀 下一步預告
**Phase 8.2** 將學習token回應格式標準化，**Phase 8.3** 會實作token驗證，為完整的認證流程做準備！

---

## ⏰ 執行記錄
- **開始時間**: ___________
- **完成時間**: ___________
- **對安全性設計的理解** (1-5): ___________