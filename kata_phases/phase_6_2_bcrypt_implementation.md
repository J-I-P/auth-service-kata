# Phase 6.2: bcrypt使用實作

## 🎯 本Phase學習目標
- 實作密碼雜湊utility函數
- 整合bcrypt到專案中
- 建立可重複使用的密碼處理模組
- 遵循安全最佳實務

## 📋 任務描述
建立 `app/utils/password.py`，實作密碼雜湊和驗證函數。

## 🔧 實作要求
1. 建立utils目錄和password模組
2. 實作hash_password函數
3. 實作verify_password函數
4. 設定適當的bcrypt cost
5. 加入錯誤處理

## 📝 實作結構
```python
# app/utils/password.py
import bcrypt

def hash_password(password: str) -> str:
    """將明碼密碼轉為雜湊值"""
    pass

def verify_password(password: str, hashed: str) -> bool:
    """驗證密碼是否正確"""
    pass
```

## 📚 學習重點
- bcrypt實際使用
- 安全的密碼處理
- Utility函數設計

## ⏰ 執行記錄
- **開始時間**: ___________
- **完成時間**: ___________