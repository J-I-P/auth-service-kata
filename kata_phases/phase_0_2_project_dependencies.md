# Phase 0.2: 專案依賴設定

## 🎯 本Phase學習目標
- 理解Python專案的依賴管理
- 學會使用pyproject.toml設定專案
- 安裝FastAPI和uvicorn
- 建立可重現的開發環境

---

## 📋 具體任務描述
設定 `pyproject.toml` 檔案，定義專案基本資訊和依賴，安裝FastAPI開發環境。

## 🔧 實作要求

### ✅ 必須完成的項目
1. **檢查pyproject.toml**: 確認檔案存在，如果是空的需要補充內容
2. **定義專案基本資訊**: name, version, description
3. **設定依賴**: 新增fastapi和uvicorn到dependencies
4. **安裝依賴**: 使用pip或poetry安裝dependencies
5. **驗證安裝**: 確認可以import fastapi

### 📝 pyproject.toml基本結構
```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "auth-service-kata"
version = "0.1.0"
description = "Authentication service practice kata"
dependencies = [
    # 你需要新增 FastAPI 和 uvicorn
]

[tool.hatch.envs.default]
dependencies = [
    # 開發依賴
]
```

### 🎯 依賴版本建議
- fastapi: 最新版本即可
- uvicorn[standard]: 包含標準ASGI server功能

---

## 📚 學習重點

### 🔍 核心概念
1. **pyproject.toml**:
   - 現代Python專案的標準設定檔案
   - 替代舊的setup.py + requirements.txt

2. **依賴管理**:
   - 生產環境依賴 vs 開發環境依賴
   - 版本鎖定的重要性

3. **ASGI Server**:
   - uvicorn是ASGI server
   - 負責運行FastAPI應用程式

### 💡 深度思考
- 為什麼要明確定義依賴？
- 生產環境和開發環境的依賴有什麼差異？
- 虛擬環境的作用是什麼？

---

## 🔍 Code Review重點

### ✅ 設定檔檢查
- [ ] pyproject.toml格式正確
- [ ] 包含必要的專案資訊
- [ ] 依賴清單完整
- [ ] 版本約束合理

### 🛠️ 環境檢查
- [ ] 依賴安裝成功
- [ ] 可以import fastapi
- [ ] 可以import uvicorn

---

## 📝 測試驗證

```bash
# 1. 安裝依賴
pip install -e .

# 2. 驗證安裝
python -c "import fastapi; import uvicorn; print('Dependencies installed!')"

# 3. 檢查版本
python -c "import fastapi; print(f'FastAPI: {fastapi.__version__}')"
```

---

## 📖 學習筆記區

### 🤔 我的理解
```
1. pyproject.toml的作用：


2. 為什麼需要依賴管理：


3. uvicorn的角色：

```

---

## 🚀 下一步預告
**Phase 0.3** 將學習如何啟動FastAPI開發服務器，真正運行你的程式碼！

---

## ⏰ 執行記錄
- **開始時間**: ___________
- **完成時間**: ___________