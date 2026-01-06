# Phase 5.3: Repository使用和依賴注入

## 🎯 本Phase學習目標
- 深入理解FastAPI的依賴注入系統
- 學會使用 `Depends()` 管理依賴關係
- 掌握可重用依賴的設計模式
- 體驗現代框架的控制反轉概念

---

## 📋 具體任務描述
使用FastAPI的 `Depends()` 機制整合repository到service layer中，建立可測試、可維護的依賴注入架構。

## 🔧 實作要求

### ✅ 必須完成的項目
1. **建立依賴函數**: 定義repository和service的依賴函數
2. **使用Depends裝飾器**: 在endpoint中注入依賴
3. **測試依賴注入**: 確認依賴正確創建和使用
4. **理解生命週期**: 觀察依賴物件的創建時機

### 📝 依賴注入實作
```python
# app/dependencies.py (新建立)
from app.repos.user_repo import UserRepository, InMemoryUserRepository
from app.services.auth_service import AuthService

def get_user_repository() -> UserRepository:
    """提供UserRepository依賴"""
    return InMemoryUserRepository()

def get_auth_service(
    user_repo: UserRepository = Depends(get_user_repository)
) -> AuthService:
    """提供AuthService依賴，注入UserRepository"""
    return AuthService(user_repo)

# app/api/auth.py 中使用
from fastapi import Depends
from app.dependencies import get_auth_service
from app.services.auth_service import AuthService

@router.post("/login")
async def login(
    request: LoginRequest,
    auth_service: AuthService = Depends(get_auth_service)
):
    """使用依賴注入的登入endpoint"""
    result = await auth_service.login(request)
    # 處理結果...
```

### 🎯 檔案結構
```
app/
├── dependencies.py    # 新建立
├── api/
│   └── auth.py       # 修改
├── services/
│   └── auth_service.py
└── repos/
    └── user_repo.py
```

---

## 📚 學習重點

### 🔍 核心概念
1. **Depends()函數**: FastAPI的依賴注入核心機制
2. **依賴層次**: 依賴可以有自己的依賴 (sub-dependencies)
3. **自動解析**: FastAPI自動解析和提供依賴
4. **單例模式**: 同一個請求中依賴會被重複使用

### 🌟 FastAPI依賴注入的強大功能
這是FastAPI最出色的特色之一：

#### **自動依賴解析**
- FastAPI自動分析函數簽名
- 遞迴解析所有依賴關係
- 自動創建和注入物件

#### **依賴快取**
- 同一請求中的相同依賴只創建一次
- 提升效能和一致性
- 避免重複初始化

#### **可測試性**
```python
# 測試時可以輕鬆替換依賴
def get_test_user_repo():
    return MockUserRepository()

app.dependency_overrides[get_user_repository] = get_test_user_repo
```

#### **彈性設計**
- 可以注入類別、函數、常數
- 支援async和sync依賴
- 可以有選擇性依賴 (Optional)

### 💡 進階依賴注入模式
```python
# 1. 全域依賴 (所有endpoints都會執行)
app = FastAPI(dependencies=[Depends(verify_api_key)])

# 2. Router層級依賴
router = APIRouter(dependencies=[Depends(verify_user_permission)])

# 3. 條件依賴
def get_current_user(token: str = Depends(oauth2_scheme)):
    # 驗證token並回傳用戶
    pass

# 4. 類別作為依賴
class DatabaseSession:
    def __init__(self):
        self.session = create_session()

    def __enter__(self):
        return self.session

    def __exit__(self, *args):
        self.session.close()
```

---

## 🔍 Code Review重點

### ✅ 依賴設計檢查
- [ ] 依賴函數命名清楚 (get_xxx)
- [ ] 依賴層次合理，不過度複雜
- [ ] 回傳型別明確定義
- [ ] 依賴函數獨立可測試

### 🏗️ 架構檢查
- [ ] dependencies.py組織清楚
- [ ] 避免循環依賴
- [ ] 依賴職責單一
- [ ] 易於mock和測試

---

## 📝 實驗任務

### 🧪 依賴注入實驗
1. **觀察依賴創建**:
   ```python
   def get_user_repo():
       print("Creating UserRepository")
       return InMemoryUserRepository()
   ```
   多次調用endpoint，觀察print次數

2. **依賴快取驗證**:
   在同一個endpoint中注入同樣依賴兩次，確認是同一個實例

3. **依賴覆蓋測試**:
   ```python
   def override_dependency():
       return MockUserRepository()

   app.dependency_overrides[get_user_repository] = override_dependency
   ```

---

## 📖 學習筆記區

### 🤔 我的理解
```
Depends()的工作原理：

依賴注入 vs 直接創建物件的差異：

為什麼依賴會被快取：

如何設計好的依賴結構：
```

### 🔬 實驗發現
```
依賴創建的時機：

同一請求中依賴重用的情況：

dependency_overrides的作用：
```

### 💡 架構思考
```
這種依賴注入對測試的幫助：

如何避免依賴地獄：

在大型專案中的應用策略：
```

---

## 🚀 下一步預告
**Phase 6.1** 將學習密碼安全處理，你會發現依賴注入讓安全功能的整合變得非常簡潔！

---

## ⏰ 執行記錄
- **開始時間**: ___________
- **完成時間**: ___________
- **依賴注入理解程度** (1-5): ___________