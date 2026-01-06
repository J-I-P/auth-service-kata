# Phase 11.3: 文檔撰寫

## 🎯 本Phase學習目標
- 理解FastAPI自動文檔生成的機制
- 學會使用OpenAPI/Swagger文檔
- 撰寫清楚的API descriptions和docstrings
- 掌握FastAPI的文檔自定義功能

---

## 📋 具體任務描述
完善專案的API文檔，包括endpoint descriptions、schema說明和使用範例，充分利用FastAPI的自動文檔功能。

## 🔧 實作要求

### ✅ 必須完成的項目
1. **完善endpoint文檔**: 為所有API endpoint加上詳細說明
2. **Schema文檔**: 為Pydantic models加上description和example
3. **自定義Swagger UI**: 設定API title、description、version
4. **API tags分組**: 使用tags組織API endpoints
5. **回應範例**: 提供成功和錯誤回應的範例

### 📝 文檔增強範例
```python
# app/main.py 中的app設定
app = FastAPI(
    title="Authentication Service API",
    description="A practice kata for building secure authentication services",
    version="1.0.0",
    docs_url="/docs",  # Swagger UI
    redoc_url="/redoc"  # ReDoc
)

# endpoint文檔範例
@router.post(
    "/login",
    response_model=LoginResponse,
    status_code=200,
    summary="User Login",
    description="Authenticate user with username and password",
    responses={
        200: {
            "description": "Login successful",
            "model": LoginResponse
        },
        401: {
            "description": "Invalid credentials",
            "content": {
                "application/json": {
                    "example": {"detail": "Invalid username or password"}
                }
            }
        },
        422: {
            "description": "Validation Error"
        }
    }
)
async def login(request: LoginRequest):
    """
    User authentication endpoint.

    This endpoint accepts username and password credentials and returns
    an access token if the credentials are valid.

    - **username**: User's login name (3-50 characters)
    - **password**: User's password (minimum 6 characters)

    Returns access_token and token_type for successful authentication.
    """
    pass
```

### 🎯 Schema文檔範例
```python
# app/schemas/auth.py 中的model文檔
class LoginRequest(BaseModel):
    """Login request model for user authentication."""

    username: str = Field(
        min_length=3,
        max_length=50,
        description="User's login username",
        example="testuser"
    )
    password: str = Field(
        min_length=6,
        description="User's password",
        example="securepass123"
    )

    class Config:
        schema_extra = {
            "example": {
                "username": "testuser",
                "password": "securepass123"
            }
        }

class LoginResponse(BaseModel):
    """Login response model containing access token."""

    access_token: str = Field(
        description="JWT access token for authenticated requests",
        example="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
    )
    token_type: str = Field(
        default="bearer",
        description="Token type, always 'bearer'",
        example="bearer"
    )
```

---

## 📚 學習重點

### 🔍 核心概念
1. **OpenAPI Specification**:
   - FastAPI自動生成OpenAPI 3.0規格
   - Swagger UI和ReDoc的差異

2. **自動文檔生成**:
   - 基於型別提示和Pydantic models
   - 零額外配置即可使用

3. **文檔自定義**:
   - endpoint descriptions和summaries
   - response examples和error cases
   - schema field descriptions

4. **API組織**:
   - 使用tags分組相關endpoints
   - 一致的命名和描述慣例

### 💡 文檔最佳實務
- 提供清楚的API使用說明
- 包含實際的使用範例
- 說明錯誤情況和處理方式
- 保持文檔與程式碼同步

---

## 🔍 實作指南

### 📖 文檔訪問方式
```bash
# 啟動服務後訪問文檔
http://localhost:8000/docs      # Swagger UI (互動式)
http://localhost:8000/redoc     # ReDoc (美觀的文檔)
http://localhost:8000/openapi.json  # OpenAPI JSON規格
```

### 🏷️ Tags使用範例
```python
# app/api/auth.py
router = APIRouter(
    prefix="/auth",
    tags=["Authentication"],
    responses={404: {"description": "Not found"}}
)
```

### 📝 文檔檢查清單
- [ ] FastAPI app有title和description
- [ ] 所有endpoints有summary和description
- [ ] Pydantic models有field descriptions
- [ ] 重要endpoints有response examples
- [ ] API使用tags適當分組
- [ ] 錯誤回應有清楚說明

---

## 📖 學習筆記區

### 🤔 我的理解
```
OpenAPI/Swagger的好處：

FastAPI自動文檔生成的機制：

為什麼文檔很重要：

Swagger UI vs ReDoc的差異：
```

### 🔍 文檔改善發現
```
改善前後的對比：

使用者體驗的提升：

意外學到的功能：
```

---

## 🚀 下一步預告
恭喜！這是最後一個Phase。完成後你將擁有一個具備完整文檔的authentication service。

## 🎉 專案完成檢查
- [ ] 所有API endpoints都有清楚文檔
- [ ] Swagger UI可以正常使用和測試API
- [ ] 文檔包含使用範例和錯誤說明
- [ ] 專案結構和程式碼品質良好

---

## ⏰ 執行記錄
- **開始時間**: ___________
- **完成時間**: ___________
- **文檔完善程度**: ___________