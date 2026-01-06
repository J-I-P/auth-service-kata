# 🎯 Complete Learning Phases Outline

## 總覽
本document列出所有學習phases的完整大綱，每個phase都會有對應的詳細實作指南文件。

---

## Phase 0: FastAPI 基礎建立

### Phase 0.1: 第一個FastAPI應用程式
**檔案**: `phase_0_1_basic_fastapi.md`
**任務**: 建立FastAPI app，實作 `GET /ping`
**學習重點**: FastAPI instance, decorator routing, 基本JSON回應

### Phase 0.2: 專案依賴設定
**檔案**: `phase_0_2_project_dependencies.md`
**任務**: 設定pyproject.toml，安裝FastAPI + uvicorn
**學習重點**: Python專案管理, poetry/pip使用, 虛擬環境

### Phase 0.3: 開發環境啟動
**檔案**: `phase_0_3_dev_environment.md`
**任務**: 啟動uvicorn server，測試熱重載
**學習重點**: ASGI server概念, 開發workflow建立

---

## Phase 1: HTTP 與 API 基礎

### Phase 1.1: HTTP方法理解
**檔案**: `phase_1_1_http_methods.md`
**任務**: 新增 `POST /echo` endpoint，回傳接收到的資料
**學習重點**: GET vs POST差異, request body處理

### Phase 1.2: HTTP狀態碼實作
**檔案**: `phase_1_2_http_status_codes.md`
**任務**: 實作不同狀態碼的endpoints (200, 400, 404)
**學習重點**: 狀態碼意義, FastAPI的Response handling

### Phase 1.3: 路徑參數和查詢參數
**檔案**: `phase_1_3_url_parameters.md`
**任務**: 實作 `GET /user/{user_id}` 和查詢參數處理
**學習重點**: URL design, 參數解析, type conversion

---

## Phase 2: 資料驗證與Pydantic

### Phase 2.1: 第一個Pydantic Model
**檔案**: `phase_2_1_first_pydantic_model.md`
**任務**: 建立簡單的User model，驗證資料
**學習重點**: Pydantic基礎, type hints, 資料驗證概念

### Phase 2.2: 請求資料驗證
**檔案**: `phase_2_2_request_validation.md`
**任務**: 使用Pydantic model驗證POST請求
**學習重點**: Request body validation, 錯誤處理

### Phase 2.3: 回應資料序列化
**檔案**: `phase_2_3_response_serialization.md`
**任務**: 定義response model並序列化回傳
**學習重點**: Response model, 資料序列化概念

---

## Phase 3: 認證資料模型設計

### Phase 3.1: LoginRequest Model
**檔案**: `phase_3_1_login_request_model.md`
**任務**: 設計登入請求的schema，包含驗證規則
**學習重點**: 業務規則in model, 字串驗證, 必填欄位

### Phase 3.2: LoginResponse Model
**檔案**: `phase_3_2_login_response_model.md`
**任務**: 設計登入回應的schema (token格式)
**學習重點**: API response設計, OAuth2標準格式

### Phase 3.3: 錯誤回應Model
**檔案**: `phase_3_3_error_response_model.md`
**任務**: 統一錯誤回應格式設計
**學習重點**: 錯誤處理標準化, HTTP error response

---

## Phase 4: 資料儲存基礎

### Phase 4.1: 記憶體資料結構設計
**檔案**: `phase_4_1_memory_data_structure.md`
**任務**: 使用dict建立簡單的使用者資料庫
**學習重點**: 資料結構選擇, in-memory storage概念

### Phase 4.2: User資料模型
**檔案**: `phase_4_2_user_data_model.md`
**任務**: 設計User entity，包含密碼儲存策略
**學習重點**: Entity design, 資料模型vs schema差異

### Phase 4.3: 基本CRUD操作
**檔案**: `phase_4_3_basic_crud.md`
**任務**: 實作基本的Create, Read操作
**學習重點**: 資料存取操作, 簡單查詢邏輯

---

## Phase 5: Repository Pattern

### Phase 5.1: Repository抽象介面
**檔案**: `phase_5_1_repository_interface.md`
**任務**: 定義UserRepository的抽象介面
**學習重點**: 抽象化概念, interface design, dependency inversion

### Phase 5.2: InMemoryUserRepository實作
**檔案**: `phase_5_2_inmemory_repository.md`
**任務**: 實作記憶體版本的repository
**學習重點**: 介面實作, 資料存取邏輯分離

### Phase 5.3: Repository使用和依賴注入
**檔案**: `phase_5_3_repository_dependency.md`
**任務**: 在service中使用repository
**學習重點**: 依賴管理, 物件生命週期

---

## Phase 6: 密碼安全處理

### Phase 6.1: 密碼雜湊概念理解
**檔案**: `phase_6_1_password_hashing_concept.md`
**任務**: 理解為什麼不能明碼儲存，學習bcrypt
**學習重點**: 安全性概念, 雜湊vs加密, salt的作用

### Phase 6.2: bcrypt使用實作
**檔案**: `phase_6_2_bcrypt_implementation.md`
**任務**: 實作密碼雜湊和驗證函數
**學習重點**: bcrypt library使用, 安全實作最佳實務

### Phase 6.3: 密碼驗證邏輯
**檔案**: `phase_6_3_password_verification.md`
**任務**: 整合密碼驗證到使用者檢查邏輯
**學習重點**: 驗證流程設計, timing attack防護

---

## Phase 7: 業務邏輯層 (Service Layer)

### Phase 7.1: AuthService類別設計
**檔案**: `phase_7_1_auth_service_design.md`
**任務**: 設計AuthService的介面和基本結構
**學習重點**: Service layer概念, 業務邏輯封裝

### Phase 7.2: 登入驗證邏輯實作
**檔案**: `phase_7_2_login_logic.md`
**任務**: 實作完整的登入驗證流程
**學習重點**: 業務流程設計, exception handling

### Phase 7.3: 錯誤處理和例外設計
**檔案**: `phase_7_3_error_handling.md`
**任務**: 定義業務例外和錯誤處理策略
**學習重點**: Exception design, 錯誤傳播策略

---

## Phase 8: Token 生成與處理

### Phase 8.1: 簡單Token生成策略
**檔案**: `phase_8_1_token_generation.md`
**任務**: 實作基本的token生成邏輯 (uuid或簡單字串)
**學習重點**: Token概念, 生成策略選擇

### Phase 8.2: Token格式和回應
**檔案**: `phase_8_2_token_response.md`
**任務**: 實作符合OAuth2格式的token回應
**學習重點**: 標準格式遵循, API design consistency

### Phase 8.3: Token驗證基礎
**檔案**: `phase_8_3_token_validation.md`
**任務**: 實作基本的token驗證邏輯
**學習重點**: Token lifecycle, 驗證策略

---

## Phase 9: API Router 實作

### Phase 9.1: 路由檔案分離
**檔案**: `phase_9_1_router_separation.md`
**任務**: 將路由邏輯分離到獨立的auth.py檔案
**學習重點**: 專案結構組織, 關注點分離

### Phase 9.2: POST /login Endpoint實作
**檔案**: `phase_9_2_login_endpoint.md`
**任務**: 實作完整的登入endpoint，整合所有layer
**學習重點**: Layer整合, HTTP endpoint design

### Phase 9.3: 錯誤處理和狀態碼
**檔案**: `phase_9_3_error_handling_http.md`
**任務**: 實作正確的HTTP錯誤回應和狀態碼
**學習重點**: HTTP error handling, 狀態碼選擇

---

## Phase 10: 測試撰寫

### Phase 10.1: 單元測試基礎
**檔案**: `phase_10_1_unit_testing_basics.md`
**任務**: 為AuthService撰寫單元測試
**學習重點**: 單元測試概念, pytest使用

### Phase 10.2: Repository測試
**檔案**: `phase_10_2_repository_testing.md`
**任務**: 測試repository的基本功能
**學習重點**: 資料存取測試, test isolation

### Phase 10.3: API整合測試
**檔案**: `phase_10_3_api_integration_testing.md`
**任務**: 測試完整的/login API功能
**學習重點**: 整合測試, FastAPI測試工具

---

## Phase 11: 程式碼品質與重構

### Phase 11.1: Code Review和靜態分析
**檔案**: `phase_11_1_code_review.md`
**任務**: 使用工具檢查程式碼品質，修正問題
**學習重點**: 程式碼品質標準, linting tools

### Phase 11.2: 重構和優化
**檔案**: `phase_11_2_refactoring.md`
**任務**: 基於學到的概念重構和優化程式碼
**學習重點**: 重構技巧, 程式碼優化策略

### Phase 11.3: 文檔撰寫
**檔案**: `phase_11_3_documentation.md`
**任務**: 撰寫API文檔和程式碼註解
**學習重點**: 技術文檔撰寫, 自動文檔生成

---

## 📊 學習進度追蹤

- [ ] Phase 0: FastAPI基礎 (3 sub-phases)
- [ ] Phase 1: HTTP與API基礎 (3 sub-phases)
- [ ] Phase 2: 資料驗證與Pydantic (3 sub-phases)
- [ ] Phase 3: 認證資料模型 (3 sub-phases)
- [ ] Phase 4: 資料儲存基礎 (3 sub-phases)
- [ ] Phase 5: Repository Pattern (3 sub-phases)
- [ ] Phase 6: 密碼安全 (3 sub-phases)
- [ ] Phase 7: 業務邏輯層 (3 sub-phases)
- [ ] Phase 8: Token處理 (3 sub-phases)
- [ ] Phase 9: API Router (3 sub-phases)
- [ ] Phase 10: 測試撰寫 (3 sub-phases)
- [ ] Phase 11: 程式碼品質 (3 sub-phases)

**總計**: 12個主要Phase，36個詳細sub-phases

---

## 🎯 下一步
現在要開始建立每個phase的詳細實作指南文件。

**目前狀態**: 完成大綱
**下一個任務**: 建立Phase 0.1的詳細實作指南