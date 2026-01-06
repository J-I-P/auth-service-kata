# Phase 10.1: 單元測試基礎

## 🎯 本Phase學習目標
- 理解單元測試的概念和重要性
- 學會使用pytest框架
- 實作AuthService的單元測試
- 學會使用mock物件隔離依賴

---

## 📋 具體任務描述
為AuthService撰寫完整的單元測試，涵蓋成功登入和失敗情況。

## 🔧 實作要求

### ✅ 必須完成的項目
1. **安裝測試依賴**: pytest, pytest-asyncio
2. **建立測試檔案**: tests/test_auth_service.py
3. **Mock repository**: 使用unittest.mock模擬資料存取
4. **測試成功案例**: 正確的username/password
5. **測試失敗案例**: 不存在的用戶、錯誤密碼

### 📝 測試結構設計
```python
# tests/test_auth_service.py
import pytest
from unittest.mock import Mock, AsyncMock
from app.services.auth_service import AuthService
from app.schemas.auth import LoginRequest
from app.models.user import User

class TestAuthService:

    @pytest.fixture
    def mock_user_repo(self):
        """建立mock repository"""
        return Mock()

    @pytest.fixture
    def auth_service(self, mock_user_repo):
        """建立auth service with mock repository"""
        return AuthService(mock_user_repo)

    @pytest.mark.asyncio
    async def test_login_success(self, auth_service, mock_user_repo):
        """測試成功登入"""
        # Arrange
        mock_user = User(username="testuser", hashed_password="...")
        mock_user_repo.find_by_username.return_value = mock_user

        request = LoginRequest(username="testuser", password="password123")

        # Act
        result = await auth_service.login(request)

        # Assert
        assert result is not None
        assert result.token_type == "bearer"
        assert result.access_token is not None

    @pytest.mark.asyncio
    async def test_login_user_not_found(self, auth_service, mock_user_repo):
        """測試使用者不存在"""
        # 你需要實作這個測試
        pass

    @pytest.mark.asyncio
    async def test_login_wrong_password(self, auth_service, mock_user_repo):
        """測試密碼錯誤"""
        # 你需要實作這個測試
        pass
```

### 🎯 執行測試
```bash
# 安裝依賴
pip install pytest pytest-asyncio

# 執行測試
pytest tests/test_auth_service.py -v

# 執行特定測試
pytest tests/test_auth_service.py::TestAuthService::test_login_success -v
```

---

## 📚 學習重點

### 🔍 核心概念
1. **單元測試**:
   - 測試獨立的功能單元
   - 快速、可重複、隔離

2. **Mock物件**:
   - 模擬外部依賴
   - 控制測試環境
   - 避免真實資料存取

3. **AAA模式**:
   - Arrange: 準備測試資料
   - Act: 執行被測試的操作
   - Assert: 驗證結果

4. **pytest fixture**:
   - 重複使用的測試設定
   - 依賴注入機制

### 💡 測試策略
- 每個分支都要有測試
- 正常路徑和異常路徑都要測
- 測試要快速且獨立

---

## 🔍 Code Review重點

### ✅ 測試品質檢查
- [ ] 測試案例覆蓋主要情況
- [ ] Mock使用正確
- [ ] Assert檢查完整
- [ ] 測試命名清楚

### 🏗️ 測試結構檢查
- [ ] 使用pytest fixture
- [ ] AAA模式清楚
- [ ] async/await正確使用

---

## 📝 測試案例設計

### ✅ 需要測試的情況
1. **成功登入**:
   - 正確的username和password
   - 回傳有效的LoginResponse

2. **使用者不存在**:
   - repository回傳None
   - auth_service回傳None

3. **密碼錯誤**:
   - 找到使用者但密碼不符
   - auth_service回傳None

### 🧪 Mock設定技巧
```python
# Mock async方法
mock_user_repo.find_by_username = AsyncMock(return_value=mock_user)

# Mock同步方法
mock_user_repo.some_method.return_value = some_value

# 驗證方法被調用
mock_user_repo.find_by_username.assert_called_once_with("testuser")
```

---

## 📖 學習筆記區

### 🤔 我的理解
```
單元測試的好處：

Mock物件的作用：

AAA模式的重要性：
```

### 🧪 測試心得
```
寫測試時遇到的困難：

Mock設定的技巧：

測試讓我發現的問題：
```

---

## 🚀 下一步預告
**Phase 10.2** 將測試Repository layer的功能。

---

## ⏰ 執行記錄
- **開始時間**: ___________
- **完成時間**: ___________
- **第一個通過的測試**: ___________