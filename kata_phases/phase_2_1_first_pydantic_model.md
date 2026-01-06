# Phase 2.1: 第一個Pydantic Model

## 🎯 本Phase學習目標
- 理解Pydantic的資料驗證概念
- 建立第一個資料模型
- 學會使用型別提示
- 體驗自動資料驗證

---

## 📋 具體任務描述
建立一個簡單的User model，並實作endpoint使用這個model進行資料驗證。

## 🔧 實作要求

### ✅ 必須完成的項目
1. **建立schemas目錄**: 在app/下建立schemas資料夾
2. **建立User model**: 定義基本的使用者資料結構
3. **實作validation endpoint**: 使用model驗證輸入資料
4. **測試驗證功能**: 驗證正確和錯誤的資料

### 📝 檔案結構
```
app/
├── schemas/
│   └── user.py  # 新建立
└── main.py      # 修改
```

### 🎯 User Model設計
```python
# app/schemas/user.py
from pydantic import BaseModel

class User(BaseModel):
    name: str
    age: int
    email: str
```

---

## 📚 學習重點

### 🔍 核心概念
1. **BaseModel**: Pydantic的基礎類別
2. **型別提示**: Python的type hints
3. **自動驗證**: Pydantic自動檢查資料型別
4. **序列化**: 物件與JSON之間的轉換

### 🌟 FastAPI + Pydantic 的強大整合
這是FastAPI的核心優勢之一：
- **零配置驗證**: 只要定義Pydantic model，FastAPI自動處理request validation
- **自動文檔更新**: model的欄位和型別會自動出現在Swagger UI中
- **智能錯誤訊息**: 驗證失敗時回傳詳細的422錯誤，告訴用戶哪個欄位有問題
- **編輯器支援**: IDE可以提供自動補全和型別檢查

### 💡 為什麼Pydantic這麼重要
- **型別安全**: 編譯時期就能發現型別錯誤
- **資料驗證**: 防止無效資料進入系統
- **自動轉換**: 字串 "123" 自動轉為 int 123
- **效能優化**: 使用Rust實作的驗證引擎，速度極快

---

## 📖 學習筆記區

### 🤔 我的理解
```
Pydantic vs 普通dict的優勢：

型別提示的作用：
```

---

## ⏰ 執行記錄
- **開始時間**: ___________
- **完成時間**: ___________