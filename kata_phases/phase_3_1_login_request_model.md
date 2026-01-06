# Phase 3.1: LoginRequest Model

## 🎯 本Phase學習目標
- 設計認證系統的資料模型
- 學會業務規則的模型驗證
- 實作字串長度和格式檢查
- 理解安全的資料接收方式

---

## 📋 具體任務描述
在 `app/schemas/auth.py` 中建立登入請求的Pydantic model，定義username和password的驗證規則。

## 🔧 實作要求

### ✅ 必須完成的項目
1. **建立auth.py**: 新檔案存放認證相關schemas
2. **LoginRequest model**: 定義登入請求格式
3. **欄位驗證**: username和password的基本驗證規則
4. **測試endpoint**: 建立endpoint測試model驗證

### 📝 Model設計需求
```python
# app/schemas/auth.py
from pydantic import BaseModel, Field

class LoginRequest(BaseModel):
    username: str = Field(min_length=3, max_length=50)
    password: str = Field(min_length=6)

    # 可選：model_config或其他設定
```

### 🎯 驗證規則
- **username**: 3-50字元，必填
- **password**: 最少6字元，必填
- **格式**: JSON格式接收

---

## 📚 學習重點

### 🔍 核心概念
1. **Field validation**: 使用Field()定義驗證規則
2. **業務規則**: 將安全需求轉為驗證邏輯
3. **資料邊界**: 在資料進入系統時就驗證
4. **錯誤回應**: Pydantic的自動錯誤訊息

### 💡 安全考量
- 密碼長度限制的平衡
- 使用者名稱的合理範圍
- 不在log中記錄敏感資料

### 🌟 Pydantic進階驗證功能
FastAPI + Pydantic 提供豐富的驗證選項：

#### **進階Field驗證**
```python
from pydantic import BaseModel, Field, validator
import re

class LoginRequest(BaseModel):
    username: str = Field(
        min_length=3,
        max_length=50,
        regex="^[a-zA-Z0-9_]+$",  # 只允許字母數字和底線
        description="用戶名：3-50字元，只能包含字母、數字、底線"
    )
    password: str = Field(
        min_length=6,
        max_length=128,
        description="密碼：最少6字元"
    )

    @validator('username')
    def username_must_not_be_reserved(cls, v):
        """自定義驗證：避免保留字"""
        reserved = ['admin', 'root', 'test', 'user']
        if v.lower() in reserved:
            raise ValueError('Username cannot be a reserved word')
        return v

    @validator('password')
    def password_complexity(cls, v):
        """自定義密碼複雜度驗證"""
        if not re.search(r'[A-Za-z]', v):
            raise ValueError('Password must contain at least one letter')
        if not re.search(r'\d', v):
            raise ValueError('Password must contain at least one number')
        return v

    class Config:
        schema_extra = {
            "example": {
                "username": "john_doe",
                "password": "secure123"
            }
        }
```

#### **FastAPI的自動錯誤處理**
當驗證失敗時，FastAPI自動回傳詳細的422錯誤：
```json
{
  "detail": [
    {
      "loc": ["body", "username"],
      "msg": "ensure this value has at least 3 characters",
      "type": "value_error.any_str.min_length",
      "ctx": {"limit_value": 3}
    }
  ]
}
```

#### **自動文檔更新**
- Field的description自動出現在Swagger UI
- regex patterns顯示在文檔中
- example values提供使用指引

---

## 🔍 Code Review重點

### ✅ 功能檢查
- [ ] LoginRequest model正確定義
- [ ] 驗證規則符合需求
- [ ] 測試endpoint正常運作
- [ ] 錯誤訊息清楚明確

### 🛡️ 安全檢查
- [ ] 密碼欄位不會在回應中洩漏
- [ ] 驗證規則合理不過度嚴格
- [ ] 錯誤訊息不洩漏系統資訊

---

## 📝 測試案例

### ✅ 有效測試
```json
{
  "username": "testuser",
  "password": "securepass123"
}
```

### ❌ 無效測試
```json
// 太短的username
{"username": "ab", "password": "securepass123"}

// 太短的password
{"username": "testuser", "password": "123"}

// 缺少欄位
{"username": "testuser"}
```

---

## 📖 學習筆記區

### 🤔 我的理解
```
Field()驗證的好處：

為什麼要在model層做驗證：

密碼長度限制的考量：
```

---

## 🚀 下一步預告
**Phase 3.2** 將設計LoginResponse model，定義登入成功後的回應格式。

---

## ⏰ 執行記錄
- **開始時間**: ___________
- **完成時間**: ___________