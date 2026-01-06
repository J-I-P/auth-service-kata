# Phase 7.1: AuthService類別設計

## 🎯 本Phase學習目標
- 理解Service Layer的設計概念
- 學會分離業務邏輯和資料存取
- 設計可測試的服務類別
- 建立依賴注入的基礎

---

## 📋 具體任務描述
在 `app/services/auth_service.py` 中設計AuthService類別，定義登入驗證的業務邏輯介面。

## 🔧 實作要求

### ✅ 必須完成的項目
1. **建立services目錄**: 在app/下建立services資料夾
2. **設計AuthService類別**: 定義主要的業務邏輯介面
3. **依賴注入設計**: 透過建構函數接收repository
4. **方法設計**: login方法的基本結構

### 📝 Service設計結構
```python
# app/services/auth_service.py
from typing import Optional
from app.repos.user_repo import UserRepository
from app.schemas.auth import LoginRequest, LoginResponse

class AuthService:
    def __init__(self, user_repo: UserRepository):
        self.user_repo = user_repo

    async def login(self, request: LoginRequest) -> Optional[LoginResponse]:
        """
        登入業務邏輯
        1. 根據username查找使用者
        2. 驗證密碼
        3. 生成token
        4. 回傳LoginResponse或None
        """
        # 目前只是設計，下個phase實作
        pass

    async def _verify_password(self, plain_password: str, hashed_password: str) -> bool:
        """驗證密碼的私有方法"""
        # 目前只是設計
        pass

    def _generate_token(self, username: str) -> str:
        """生成token的私有方法"""
        # 目前只是設計
        pass
```

---

## 📚 學習重點

### 🔍 核心概念
1. **Service Layer Pattern**:
   - 封裝業務邏輯
   - 協調多個repository
   - 處理業務規則

2. **依賴注入**:
   - 透過建構函數注入dependencies
   - 提高可測試性
   - 降低耦合度

3. **單一職責原則**:
   - AuthService只處理認證邏輯
   - 不直接處理HTTP請求
   - 不直接存取資料庫

### 💡 設計原則
- 業務邏輯應該獨立於框架
- 服務應該容易單元測試
- 私有方法處理內部邏輯
- 公開方法是業務操作

---

## 🔍 Code Review重點

### ✅ 設計檢查
- [ ] 類別職責清楚單一
- [ ] 依賴注入正確實作
- [ ] 方法命名表達業務意圖
- [ ] 私有方法適當使用

### 🏗️ 架構檢查
- [ ] 不直接相依於FastAPI
- [ ] 使用抽象的repository介面
- [ ] 型別提示完整

---

## 📝 設計思考

### 🤔 業務流程分析
```
登入流程拆解：
1. 接收登入請求
2. 查找使用者
3. 驗證密碼
4. 生成token
5. 回傳結果

每個步驟的責任歸屬：
- AuthService:
- UserRepository:
- PasswordUtil:
```

### 🧪 測試考量
```
如何測試這個service：
- Mock UserRepository
- 準備測試資料
- 驗證各種情況

需要測試的案例：
- 成功登入
- 使用者不存在
- 密碼錯誤
```

---

## 📖 學習筆記區

### 🤔 我的理解
```
Service Layer的作用：

依賴注入的好處：

為什麼要分離業務邏輯：
```

### 💭 設計疑問
```
還有哪些方法可能需要：

私有方法和公開方法的選擇：

錯誤處理應該在哪一層：
```

---

## 🚀 下一步預告
**Phase 7.2** 將實作login方法的具體邏輯，整合password驗證和token生成。

---

## ⏰ 執行記錄
- **開始時間**: ___________
- **完成時間**: ___________