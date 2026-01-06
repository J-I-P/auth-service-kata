# Phase 5.2: InMemoryUserRepository實作

## 🎯 本Phase學習目標
- 實作repository介面的具體實現
- 學會介面實作的最佳實務
- 建立可測試的資料存取層
- 理解依賴注入的實際應用

---

## 📋 具體任務描述
建立InMemoryUserRepository類別，實作Phase 5.1定義的抽象介面。

## 🔧 實作要求

### ✅ 必須完成的項目
1. **繼承抽象介面**: 實作UserRepository
2. **記憶體儲存**: 使用dict儲存使用者資料
3. **實作所有方法**: find_by_username, create_user等
4. **預設測試資料**: 建立一些測試用的使用者

### 📝 實作結構
```python
# app/repos/user_repo.py 擴展
class InMemoryUserRepository(UserRepository):
    def __init__(self):
        # 初始化記憶體儲存和測試資料
        self._users = {}
        self._next_id = 1
        self._init_test_data()

    async def find_by_username(self, username: str) -> Optional[User]:
        # 實作查詢邏輯
        pass

    async def create_user(self, user: User) -> User:
        # 實作創建邏輯
        pass

    def _init_test_data(self):
        # 建立測試用的使用者資料
        pass
```

---

## 📚 學習重點

### 🔍 核心概念
1. **介面實作**: 如何正確實現抽象方法
2. **資料存取邏輯**: 查詢、創建、更新的實作
3. **記憶體管理**: 簡單的資料結構操作
4. **測試資料**: 開發階段的資料準備

---

## 📖 學習筆記區

### 🤔 我的理解
```
實作抽象介面的好處：

記憶體儲存的優缺點：

如何設計測試資料：
```

---

## ⏰ 執行記錄
- **開始時間**: ___________
- **完成時間**: ___________