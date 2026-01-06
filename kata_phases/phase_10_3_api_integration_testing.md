# Phase 10.3: API整合測試

## 🎯 本Phase學習目標
- 學會使用FastAPI的TestClient進行API測試
- 掌握整合測試vs單元測試的差異
- 實作完整的API測試場景
- 體驗FastAPI優秀的測試支援

---

## 📋 具體任務描述
使用FastAPI的TestClient為完整的登入API撰寫整合測試，涵蓋成功和失敗的各種場景。

## 🔧 實作要求

### ✅ 必須完成的項目
1. **建立TestClient**: 設定測試環境和client
2. **成功登入測試**: 測試正確的登入流程
3. **失敗場景測試**: 測試各種錯誤情況
4. **依賴覆蓋**: 使用測試專用的dependencies
5. **完整流程驗證**: 從HTTP請求到JSON回應

### 📝 測試實作結構
```python
# tests/test_api_integration.py
import pytest
from fastapi.testclient import TestClient
from app.main import app
from app.dependencies import get_user_repository
from app.repos.user_repo import UserRepository
from app.models.user import User

# 測試專用的repository
class TestUserRepository(UserRepository):
    def __init__(self):
        self._users = {
            "testuser": User(
                username="testuser",
                hashed_password="$2b$12$hashed_password_here"
            )
        }

    async def find_by_username(self, username: str):
        return self._users.get(username)

    async def create_user(self, user: User):
        self._users[user.username] = user
        return user

def get_test_user_repository():
    return TestUserRepository()

# 設定依賴覆蓋
app.dependency_overrides[get_user_repository] = get_test_user_repository

client = TestClient(app)

class TestLoginAPI:
    def test_successful_login(self):
        """測試成功登入"""
        response = client.post(
            "/auth/login",
            json={
                "username": "testuser",
                "password": "password123"
            }
        )

        assert response.status_code == 200
        data = response.json()
        assert "access_token" in data
        assert data["token_type"] == "bearer"

    def test_invalid_username(self):
        """測試不存在的使用者"""
        response = client.post(
            "/auth/login",
            json={
                "username": "nonexistent",
                "password": "password123"
            }
        )

        assert response.status_code == 401
        assert "Invalid username or password" in response.json()["detail"]

    def test_invalid_password(self):
        """測試錯誤密碼"""
        response = client.post(
            "/auth/login",
            json={
                "username": "testuser",
                "password": "wrongpassword"
            }
        )

        assert response.status_code == 401

    def test_validation_errors(self):
        """測試資料驗證錯誤"""
        response = client.post(
            "/auth/login",
            json={
                "username": "ab",  # 太短
                "password": "123"   # 太短
            }
        )

        assert response.status_code == 422
        errors = response.json()["detail"]
        assert len(errors) == 2  # 兩個驗證錯誤

    def test_missing_fields(self):
        """測試缺少必填欄位"""
        response = client.post(
            "/auth/login",
            json={"username": "testuser"}  # 缺少password
        )

        assert response.status_code == 422

    def test_invalid_json(self):
        """測試無效的JSON格式"""
        response = client.post(
            "/auth/login",
            data="invalid json",
            headers={"Content-Type": "application/json"}
        )

        assert response.status_code == 422
```

---

## 📚 學習重點

### 🔍 核心概念
1. **TestClient**: FastAPI內建的測試客戶端
2. **整合測試**: 測試完整的API流程
3. **依賴覆蓋**: 在測試中替換production dependencies
4. **HTTP測試**: 驗證status codes和response格式

### 🌟 FastAPI測試的強大功能
這是FastAPI另一個出色的特色：

#### **TestClient簡潔性**
- 基於Starlette TestClient
- 與requests library相同的API
- 無需實際啟動server
- 同步測試async endpoints

#### **依賴覆蓋機制**
```python
# 輕鬆替換任何依賴
app.dependency_overrides[get_database] = get_test_database
app.dependency_overrides[get_current_user] = get_test_user
```

#### **自動序列化/反序列化**
- 自動處理JSON serialization
- 自動應用Pydantic validation
- 完整模擬真實API行為

#### **完整的HTTP功能**
```python
# 支援所有HTTP功能
response = client.post(
    "/api/endpoint",
    json=data,
    headers={"Authorization": "Bearer token"},
    cookies={"session": "value"},
    files={"upload": ("test.txt", "content")}
)
```

### 💡 測試策略最佳實務
```python
# 1. 測試設定和清理
@pytest.fixture(autouse=True)
def setup_and_teardown():
    # 每個測試前的設定
    app.dependency_overrides[get_user_repo] = get_test_repo
    yield
    # 每個測試後的清理
    app.dependency_overrides.clear()

# 2. 共用測試資料
@pytest.fixture
def sample_user():
    return {
        "username": "testuser",
        "password": "password123"
    }

# 3. 參數化測試
@pytest.mark.parametrize("username,password,expected_status", [
    ("validuser", "validpass", 200),
    ("invaliduser", "validpass", 401),
    ("validuser", "invalidpass", 401),
])
def test_login_scenarios(username, password, expected_status):
    response = client.post("/auth/login", json={
        "username": username,
        "password": password
    })
    assert response.status_code == expected_status
```

---

## 🔍 Code Review重點

### ✅ 測試品質檢查
- [ ] 測試涵蓋主要成功/失敗場景
- [ ] 使用適當的assertion
- [ ] 測試資料清楚且有意義
- [ ] 依賴覆蓋正確設定

### 🧪 測試完整性檢查
- [ ] 測試HTTP status codes
- [ ] 驗證response格式和內容
- [ ] 測試邊界條件
- [ ] 錯誤情況處理完整

---

## 📝 測試場景檢查清單

### ✅ 必測場景
- [ ] 成功登入 (200)
- [ ] 使用者不存在 (401)
- [ ] 密碼錯誤 (401)
- [ ] 欄位驗證錯誤 (422)
- [ ] 缺少必填欄位 (422)
- [ ] 無效JSON格式 (422)

### 🔍 進階場景
- [ ] 空字串輸入
- [ ] 超長輸入
- [ ] 特殊字元處理
- [ ] 並發登入測試

---

## 📖 學習筆記區

### 🤔 我的理解
```
TestClient vs 真實HTTP client的差異：

整合測試 vs 單元測試的使用時機：

依賴覆蓋的工作原理：

為什麼可以同步測試async endpoints：
```

### 🧪 測試心得
```
寫整合測試遇到的困難：

TestClient的便利之處：

測試資料準備的技巧：

如何平衡測試覆蓋率和執行速度：
```

### 💡 測試策略思考
```
什麼情況用整合測試 vs 單元測試：

如何組織大型專案的測試：

CI/CD中的測試執行策略：
```

---

## 🚀 下一步預告
**Phase 11.1** 將學習程式碼品質檢查，包含測試覆蓋率分析！

---

## ⏰ 執行記錄
- **開始時間**: ___________
- **完成時間**: ___________
- **測試通過率**: ___________
- **學到最有用的測試技巧**: ___________