# Phase 3.3: 錯誤回應Model

## 🎯 本Phase學習目標
- 設計統一的錯誤回應格式
- 理解HTTP錯誤回應的標準化
- 學會錯誤訊息的安全設計
- 建立可維護的錯誤處理系統

---

## 📋 具體任務描述
設計統一的錯誤回應model，確保所有API錯誤都有一致的格式。

## 🔧 實作要求

### ✅ 必須完成的項目
1. **ErrorResponse model**: 統一錯誤格式
2. **錯誤類型設計**: 不同類型的錯誤回應
3. **安全考量**: 錯誤訊息不洩漏敏感資訊
4. **使用範例**: 在實際endpoint中使用

### 📝 Model設計
```python
# app/schemas/error.py
from pydantic import BaseModel
from typing import Optional

class ErrorResponse(BaseModel):
    detail: str
    error_code: Optional[str] = None

class ValidationErrorResponse(BaseModel):
    detail: str = "Validation error"
    errors: list[dict]
```

---

## 📚 學習重點

### 🔍 核心概念
1. **錯誤標準化**: 統一的錯誤格式
2. **安全性**: 不洩漏系統內部資訊
3. **可讀性**: 清楚的錯誤訊息
4. **可操作性**: 前端能據此處理

---

## 📖 學習筆記區

### 🤔 我的理解
```
統一錯誤格式的好處：

錯誤訊息的安全考量：
```

---

## ⏰ 執行記錄
- **開始時間**: ___________
- **完成時間**: ___________