# Phase 5.1: Repository抽象介面

## 🎯 本Phase學習目標
- 理解抽象介面的設計原則
- 學會依賴反轉原則(Dependency Inversion)
- 設計可測試的資料存取介面
- 為未來的資料庫切換做準備

---

## 📋 具體任務描述
建立 `UserRepository` 抽象基類，定義使用者資料存取的介面，不實作具體邏輯。

## 🔧 實作要求

### ✅ 必須完成的項目
1. **建立repos目錄**: 在app/下建立repos資料夾
2. **定義抽象介面**: 建立UserRepository抽象基類
3. **定義方法簽名**: find_by_username等核心方法
4. **型別提示**: 完整的參數和回傳值型別

### 📝 檔案結構
```
app/
├── repos/
│   ├── __init__.py        # 空檔案
│   └── user_repo.py       # 新建立
├── models/
│   └── user.py           # User entity
└── main.py
```

### 🎯 Repository介面設計
```python
# app/repos/user_repo.py
from abc import ABC, abstractmethod
from typing import Optional
from app.models.user import User

class UserRepository(ABC):

    @abstractmethod
    async def find_by_username(self, username: str) -> Optional[User]:
        """根據使用者名稱查找使用者"""
        pass

    @abstractmethod
    async def create_user(self, user: User) -> User:
        """建立新使用者"""
        pass
```

---

## 📚 學習重點

### 🔍 核心概念
1. **抽象基類(ABC)**: Python的抽象介面實作方式
2. **依賴反轉**: 高階模組不依賴低階模組
3. **介面隔離**: 只定義必要的方法
4. **可測試性**: 容易建立mock物件

### 💡 設計原則
- 介面應該穩定，不常變動
- 方法命名清楚表達意圖
- 回傳型別明確
- 考慮非同步操作

---

## 🔍 Code Review重點

### ✅ 設計檢查
- [ ] 繼承ABC並使用@abstractmethod
- [ ] 方法簽名清楚明確
- [ ] 型別提示完整
- [ ] docstring描述清楚

### 🏗️ 架構檢查
- [ ] 介面職責單一
- [ ] 不包含實作細節
- [ ] 便於未來擴展

---

## 📖 學習筆記區

### 🤔 我的理解
```
抽象介面的好處：

依賴反轉原則的意義：

為什麼要用Optional[User]：
```

---

## 🚀 下一步預告
**Phase 5.2** 將實作InMemoryUserRepository，具體實現這個介面。

---

## ⏰ 執行記錄
- **開始時間**: ___________
- **完成時間**: ___________