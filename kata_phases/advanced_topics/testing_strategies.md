# Testing Strategies: 企業級測試架構與策略

## 🎯 學習目標
- 掌握現代軟體測試的完整生態系統
- 學習測試驅動開發 (TDD) 與行為驅動開發 (BDD)
- 建立企業級自動化測試流水線
- 理解測試在 DevOps 中的關鍵作用

---

## 🧪 測試金字塔與策略框架

### 現代測試金字塔重新定義

```python
# 測試策略架構圖
"""
           🔺 E2E Tests (5%)
          /   \
         /     \  Integration Tests (15%)
        /       \
       /         \
      /___________\ Unit Tests (80%)

額外層次：
- Contract Tests (服務間契約)
- Performance Tests (效能測試)
- Security Tests (安全測試)
- Chaos Tests (混沌工程)
"""

class TestingStrategy:
    """企業級測試策略管理"""

    def __init__(self):
        self.test_categories = {
            "unit": {"coverage": 80, "speed": "fast", "isolation": "high"},
            "integration": {"coverage": 15, "speed": "medium", "isolation": "medium"},
            "e2e": {"coverage": 5, "speed": "slow", "isolation": "low"},
            "contract": {"coverage": "cross-service", "speed": "fast", "isolation": "high"},
            "performance": {"coverage": "critical-paths", "speed": "slow", "isolation": "low"},
            "security": {"coverage": "all-inputs", "speed": "medium", "isolation": "medium"}
        }

    def get_optimal_test_mix(self, project_type: str, team_size: int) -> dict:
        """根據專案類型和團隊規模推薦測試組合"""
        if project_type == "microservice":
            return {
                "unit": 70,
                "integration": 20,
                "contract": 8,
                "e2e": 2
            }
        elif project_type == "monolith":
            return {
                "unit": 80,
                "integration": 15,
                "e2e": 5
            }
        elif project_type == "api-service":
            return {
                "unit": 75,
                "integration": 20,
                "contract": 3,
                "e2e": 2
            }
```

---

## 🔬 單元測試：深度與廣度

### 進階單元測試模式

```python
import pytest
from unittest.mock import Mock, patch, AsyncMock
from pytest_mock import MockerFixture

class AdvancedUnitTestPatterns:
    """進階單元測試模式展示"""

class TestAuthServiceAdvanced:
    """AuthService 進階測試案例"""

    @pytest.fixture
    def auth_service(self, mocker: MockerFixture):
        """設定測試用 AuthService"""
        mock_user_repo = mocker.Mock()
        mock_token_manager = mocker.Mock()
        return AuthService(mock_user_repo, mock_token_manager)

    @pytest.mark.asyncio
    async def test_login_success_with_mfa(self, auth_service, mocker):
        """測試包含 MFA 的登入流程"""
        # Arrange
        mock_user = User(id="123", username="testuser", mfa_enabled=True)
        auth_service.user_repo.find_by_username.return_value = mock_user
        auth_service.password_verifier.verify.return_value = True
        auth_service.mfa_manager.verify_totp.return_value = True
        auth_service.token_manager.generate_token.return_value = "mock_token"

        request = LoginRequest(
            username="testuser",
            password="password123",
            mfa_token="123456"
        )

        # Act
        result = await auth_service.login(request)

        # Assert
        assert result is not None
        assert result.access_token == "mock_token"
        auth_service.mfa_manager.verify_totp.assert_called_once_with("123", "123456")

    @pytest.mark.parametrize("username,password,mfa_token,expected_error", [
        ("", "password", "123456", "Username cannot be empty"),
        ("user", "", "123456", "Password cannot be empty"),
        ("user", "password", "", "MFA token required"),
        ("user", "short", "123456", "Password too short"),
    ])
    async def test_login_validation_errors(self, auth_service, username, password, mfa_token, expected_error):
        """參數化測試驗證錯誤"""
        request = LoginRequest(username=username, password=password, mfa_token=mfa_token)

        with pytest.raises(ValidationError) as exc_info:
            await auth_service.login(request)

        assert expected_error in str(exc_info.value)

    @patch('app.services.auth_service.datetime')
    async def test_token_expiration_logic(self, mock_datetime, auth_service):
        """使用 patch 測試時間相關邏輯"""
        # 設定固定時間
        mock_datetime.utcnow.return_value = datetime(2024, 1, 1, 12, 0, 0)

        # 測試邏輯...
        result = await auth_service.generate_token_with_expiry("user123")

        expected_expiry = datetime(2024, 1, 1, 13, 0, 0)  # 1小時後
        assert result.expires_at == expected_expiry

class TestWithDependencyInjection:
    """依賴注入環境下的測試"""

    @pytest.fixture
    def app_with_test_deps(self):
        """建立帶測試依賴的應用"""
        from fastapi.testclient import TestClient

        # 建立測試版本的依賴
        test_user_repo = InMemoryUserRepository()
        test_auth_service = AuthService(test_user_repo)

        # 覆蓋依賴
        app.dependency_overrides[get_user_repository] = lambda: test_user_repo
        app.dependency_overrides[get_auth_service] = lambda: test_auth_service

        yield TestClient(app)

        # 清理
        app.dependency_overrides.clear()

    def test_login_endpoint_integration(self, app_with_test_deps):
        """測試完整的 endpoint 整合"""
        client = app_with_test_deps

        # 先建立測試用戶
        response = client.post("/auth/register", json={
            "username": "testuser",
            "password": "password123"
        })
        assert response.status_code == 201

        # 測試登入
        response = client.post("/auth/login", json={
            "username": "testuser",
            "password": "password123"
        })
        assert response.status_code == 200
        assert "access_token" in response.json()

class PropertyBasedTesting:
    """屬性基礎測試（Property-Based Testing）"""

    @pytest.mark.hypothesis
    def test_password_hashing_properties(self, st=strategies):
        """使用 Hypothesis 進行屬性測試"""
        @given(password=st.text(min_size=1, max_size=100))
        def test_hash_verify_roundtrip(password):
            """測試密碼雜湊的往返屬性"""
            hashed = hash_password(password)
            assert verify_password(password, hashed)

            # 不同密碼不應該產生相同雜湊
            if len(password) > 1:
                different_password = password[:-1] + 'X'
                assert not verify_password(different_password, hashed)

        test_hash_verify_roundtrip()

class MutationTesting:
    """變異測試 - 測試你的測試"""

    def configure_mutation_testing(self):
        """配置變異測試"""
        # 使用 mutmut 進行變異測試
        """
        # 安裝: pip install mutmut
        # 執行: mutmut run --paths-to-mutate=app/
        # 查看結果: mutmut results

        變異測試會故意修改程式碼，檢查測試是否能偵測到錯誤
        高品質的測試套件應該能捕捉到大部分變異
        """
        pass
```

### 測試資料管理策略

```python
class TestDataManagement:
    """測試資料管理最佳實務"""

    @pytest.fixture
    def sample_users(self):
        """測試用戶資料工廠"""
        return {
            "basic_user": User(
                id="user_001",
                username="basicuser",
                email="basic@example.com",
                role="user"
            ),
            "admin_user": User(
                id="admin_001",
                username="adminuser",
                email="admin@example.com",
                role="admin"
            ),
            "premium_user": User(
                id="premium_001",
                username="premiumuser",
                email="premium@example.com",
                role="premium"
            )
        }

    @pytest.fixture
    def user_factory(self):
        """用戶工廠函數"""
        def _create_user(username=None, email=None, **kwargs):
            return User(
                id=kwargs.get('id', f"user_{uuid.uuid4().hex[:8]}"),
                username=username or f"user_{uuid.uuid4().hex[:8]}",
                email=email or f"test_{uuid.uuid4().hex[:8]}@example.com",
                **kwargs
            )
        return _create_user

class DatabaseTestingPatterns:
    """資料庫測試模式"""

    @pytest.fixture(scope="function")
    async def db_session(self):
        """每個測試獨立的資料庫會話"""
        # 建立測試資料庫連接
        engine = create_test_database_engine()
        async with engine.begin() as conn:
            # 建立所有表
            await conn.run_sync(Base.metadata.create_all)

            # 建立 session
            TestSessionLocal = sessionmaker(
                autocommit=False, autoflush=False, bind=conn, class_=AsyncSession
            )

            async with TestSessionLocal() as session:
                yield session

                # 回滾所有變更
                await session.rollback()

    async def test_with_transaction_rollback(self, db_session):
        """使用交易回滾的測試"""
        # 測試資料操作
        user = User(username="testuser", email="test@example.com")
        db_session.add(user)
        await db_session.commit()

        # 驗證資料存在
        result = await db_session.execute(
            select(User).where(User.username == "testuser")
        )
        assert result.scalar_one() is not None

        # 測試結束後會自動回滾，不影響其他測試

class TestContainerization:
    """容器化測試環境"""

    @pytest.fixture(scope="session")
    def postgres_container(self):
        """PostgreSQL 測試容器"""
        with DockerContainer("postgres:13") \
            .with_exposed_ports(5432) \
            .with_env("POSTGRES_PASSWORD", "testpass") \
            .with_env("POSTGRES_DB", "testdb") as container:

            # 等待容器啟動
            container.wait_for_logs("database system is ready to accept connections")

            yield container

    def test_with_real_database(self, postgres_container):
        """使用真實資料庫的測試"""
        port = postgres_container.get_exposed_port(5432)
        db_url = f"postgresql://postgres:testpass@localhost:{port}/testdb"

        # 使用真實資料庫進行測試
        engine = create_engine(db_url)
        # ... 測試邏輯
```

---

## 🔗 整合測試：系統協作驗證

### API 整合測試策略

```python
class APIIntegrationTestSuite:
    """API 整合測試套件"""

    @pytest.fixture
    def api_client(self):
        """API 客戶端設定"""
        from httpx import AsyncClient

        return AsyncClient(
            base_url="http://testserver",
            headers={"Content-Type": "application/json"}
        )

    @pytest.mark.asyncio
    async def test_complete_auth_flow(self, api_client):
        """完整認證流程測試"""
        # 1. 註冊用戶
        register_response = await api_client.post("/auth/register", json={
            "username": "integration_user",
            "email": "integration@example.com",
            "password": "SecurePass123!"
        })
        assert register_response.status_code == 201
        user_id = register_response.json()["user_id"]

        # 2. 登入獲取 token
        login_response = await api_client.post("/auth/login", json={
            "username": "integration_user",
            "password": "SecurePass123!"
        })
        assert login_response.status_code == 200
        token = login_response.json()["access_token"]

        # 3. 使用 token 訪問保護的資源
        headers = {"Authorization": f"Bearer {token}"}
        profile_response = await api_client.get("/user/profile", headers=headers)
        assert profile_response.status_code == 200
        assert profile_response.json()["username"] == "integration_user"

        # 4. 更新個人資料
        update_response = await api_client.put("/user/profile",
            headers=headers,
            json={"display_name": "Integration User"}
        )
        assert update_response.status_code == 200

        # 5. 登出
        logout_response = await api_client.post("/auth/logout", headers=headers)
        assert logout_response.status_code == 200

        # 6. 驗證 token 已失效
        profile_response = await api_client.get("/user/profile", headers=headers)
        assert profile_response.status_code == 401

    @pytest.mark.asyncio
    async def test_concurrent_requests(self, api_client):
        """並發請求測試"""
        import asyncio

        # 建立多個並發登入請求
        login_tasks = []
        for i in range(10):
            task = api_client.post("/auth/login", json={
                "username": f"user_{i}",
                "password": "password123"
            })
            login_tasks.append(task)

        # 並發執行
        responses = await asyncio.gather(*login_tasks, return_exceptions=True)

        # 驗證所有請求都正確處理
        for response in responses:
            if isinstance(response, Exception):
                pytest.fail(f"Request failed with exception: {response}")
            assert response.status_code in [200, 401]  # 成功或認證失敗

class ServiceIntegrationTests:
    """服務間整合測試"""

    @pytest.fixture
    def mock_external_services(self, httpx_mock):
        """模擬外部服務"""
        # 模擬郵件服務
        httpx_mock.add_response(
            method="POST",
            url="https://api.emailservice.com/send",
            json={"status": "sent", "message_id": "12345"}
        )

        # 模擬 SMS 服務
        httpx_mock.add_response(
            method="POST",
            url="https://api.smsservice.com/send",
            json={"status": "delivered", "sms_id": "67890"}
        )

    async def test_notification_integration(self, api_client, mock_external_services):
        """通知服務整合測試"""
        # 觸發需要發送通知的操作
        response = await api_client.post("/user/request-password-reset", json={
            "email": "test@example.com"
        })

        assert response.status_code == 200
        # 驗證外部服務被正確調用（透過 httpx_mock）

class DatabaseIntegrationTests:
    """資料庫整合測試"""

    @pytest.mark.integration
    async def test_complex_query_performance(self, db_session):
        """複雜查詢效能測試"""
        # 建立大量測試資料
        users = []
        for i in range(1000):
            user = User(
                username=f"user_{i}",
                email=f"user_{i}@example.com"
            )
            users.append(user)

        db_session.add_all(users)
        await db_session.commit()

        # 測試複雜查詢的效能
        start_time = time.time()

        result = await db_session.execute(
            select(User)
            .where(User.email.like("%@example.com"))
            .order_by(User.created_at.desc())
            .limit(50)
        )

        end_time = time.time()
        query_time = end_time - start_time

        # 驗證查詢效能在可接受範圍內
        assert query_time < 1.0  # 應該在 1 秒內完成
        assert len(result.fetchall()) == 50

    async def test_transaction_isolation(self, db_session):
        """交易隔離測試"""
        # 測試 ACID 特性
        async with db_session.begin():
            user = User(username="tx_user", email="tx@example.com")
            db_session.add(user)

            # 在同一交易中查詢
            result = await db_session.execute(
                select(User).where(User.username == "tx_user")
            )
            assert result.scalar_one() is not None

            # 模擬交易回滾
            raise Exception("Rollback transaction")

        # 驗證回滾後資料不存在
        result = await db_session.execute(
            select(User).where(User.username == "tx_user")
        )
        assert result.scalar_one_or_none() is None
```

---

## 🚀 端對端測試：使用者旅程驗證

### 使用 Playwright 的現代 E2E 測試

```python
import pytest
from playwright.async_api import async_playwright, Page, BrowserContext

class E2ETestSuite:
    """端對端測試套件"""

    @pytest.fixture
    async def browser_context(self):
        """瀏覽器上下文設定"""
        async with async_playwright() as p:
            browser = await p.chromium.launch(headless=True)
            context = await browser.new_context(
                viewport={"width": 1280, "height": 720},
                locale="zh-TW",
                timezone_id="Asia/Taipei"
            )
            yield context
            await browser.close()

    @pytest.fixture
    async def authenticated_page(self, browser_context):
        """已認證的頁面"""
        page = await browser_context.new_page()

        # 執行登入流程
        await page.goto("/login")
        await page.fill('[data-testid="username"]', "testuser")
        await page.fill('[data-testid="password"]', "password123")
        await page.click('[data-testid="login-button"]')

        # 等待登入完成
        await page.wait_for_selector('[data-testid="user-menu"]')

        yield page

class UserJourneyTests:
    """使用者旅程測試"""

    @pytest.mark.e2e
    async def test_new_user_registration_journey(self, browser_context):
        """新用戶註冊流程"""
        page = await browser_context.new_page()

        # 1. 訪問首頁
        await page.goto("/")

        # 2. 點擊註冊按鈕
        await page.click('[data-testid="register-link"]')

        # 3. 填寫註冊表單
        await page.fill('[data-testid="username"]', "newuser123")
        await page.fill('[data-testid="email"]', "newuser@example.com")
        await page.fill('[data-testid="password"]', "SecurePassword123!")
        await page.fill('[data-testid="confirm-password"]', "SecurePassword123!")

        # 4. 提交註冊
        await page.click('[data-testid="register-button"]')

        # 5. 驗證註冊成功
        await page.wait_for_selector('[data-testid="registration-success"]')
        success_message = await page.text_content('[data-testid="success-message"]')
        assert "註冊成功" in success_message

        # 6. 檢查是否收到歡迎郵件（模擬）
        # 這裡可以整合測試郵件服務

    @pytest.mark.e2e
    async def test_password_reset_flow(self, browser_context):
        """密碼重設流程"""
        page = await browser_context.new_page()

        # 1. 進入登入頁面
        await page.goto("/login")

        # 2. 點擊忘記密碼
        await page.click('[data-testid="forgot-password-link"]')

        # 3. 輸入 email
        await page.fill('[data-testid="reset-email"]', "user@example.com")
        await page.click('[data-testid="send-reset-button"]')

        # 4. 驗證重設郵件發送確認
        await page.wait_for_selector('[data-testid="reset-email-sent"]')

        # 5. 模擬點擊郵件中的重設連結
        reset_token = "mock_reset_token_12345"
        await page.goto(f"/reset-password?token={reset_token}")

        # 6. 輸入新密碼
        await page.fill('[data-testid="new-password"]', "NewSecurePassword456!")
        await page.fill('[data-testid="confirm-new-password"]', "NewSecurePassword456!")
        await page.click('[data-testid="update-password-button"]')

        # 7. 驗證密碼重設成功
        await page.wait_for_selector('[data-testid="password-reset-success"]')

class ResponsiveAndAccessibilityTests:
    """響應式設計與無障礙測試"""

    @pytest.mark.e2e
    @pytest.mark.parametrize("viewport", [
        {"width": 375, "height": 667},   # Mobile
        {"width": 768, "height": 1024},  # Tablet
        {"width": 1920, "height": 1080}  # Desktop
    ])
    async def test_responsive_design(self, browser_context, viewport):
        """響應式設計測試"""
        await browser_context.set_viewport_size(viewport["width"], viewport["height"])
        page = await browser_context.new_page()

        await page.goto("/login")

        # 檢查關鍵元素在不同尺寸下的可見性
        login_form = await page.locator('[data-testid="login-form"]')
        await expect(login_form).to_be_visible()

        # 檢查導航在小螢幕上的行為
        if viewport["width"] < 768:
            # 移動端應該有漢堡選單
            menu_toggle = await page.locator('[data-testid="mobile-menu-toggle"]')
            await expect(menu_toggle).to_be_visible()

    @pytest.mark.e2e
    async def test_keyboard_navigation(self, browser_context):
        """鍵盤導航測試"""
        page = await browser_context.new_page()
        await page.goto("/login")

        # 使用 Tab 鍵導航
        await page.keyboard.press("Tab")  # 移動到使用者名稱欄位
        await page.keyboard.type("testuser")

        await page.keyboard.press("Tab")  # 移動到密碼欄位
        await page.keyboard.type("password123")

        await page.keyboard.press("Tab")  # 移動到登入按鈕
        await page.keyboard.press("Enter")  # 按下登入

        # 驗證鍵盤操作能正常登入
        await page.wait_for_selector('[data-testid="dashboard"]')

    @pytest.mark.e2e
    async def test_accessibility_standards(self, browser_context):
        """無障礙標準測試"""
        page = await browser_context.new_page()
        await page.goto("/login")

        # 使用 axe-core 進行無障礙檢查
        await page.add_script_tag(
            url="https://unpkg.com/axe-core@4/axe.min.js"
        )

        accessibility_results = await page.evaluate("""
            async () => {
                const results = await axe.run();
                return results;
            }
        """)

        # 驗證沒有嚴重的無障礙問題
        violations = accessibility_results["violations"]
        critical_violations = [v for v in violations if v["impact"] in ["critical", "serious"]]
        assert len(critical_violations) == 0, f"Found critical accessibility violations: {critical_violations}"

class PerformanceE2ETests:
    """效能端對端測試"""

    @pytest.mark.e2e
    async def test_page_load_performance(self, browser_context):
        """頁面載入效能測試"""
        page = await browser_context.new_page()

        # 開始效能監控
        await page.goto("/", wait_until="networkidle")

        # 取得效能指標
        performance_metrics = await page.evaluate("""
            () => {
                const nav = performance.getEntriesByType('navigation')[0];
                return {
                    domContentLoaded: nav.domContentLoadedEventEnd - nav.domContentLoadedEventStart,
                    loadComplete: nav.loadEventEnd - nav.loadEventStart,
                    firstPaint: performance.getEntriesByType('paint')[0]?.startTime || 0,
                    firstContentfulPaint: performance.getEntriesByType('paint')[1]?.startTime || 0
                };
            }
        """)

        # 驗證效能指標在可接受範圍內
        assert performance_metrics["domContentLoaded"] < 2000  # 2秒內完成 DOM 載入
        assert performance_metrics["firstContentfulPaint"] < 1500  # 1.5秒內首次內容繪製

class VisualRegressionTests:
    """視覺回歸測試"""

    @pytest.mark.e2e
    async def test_visual_consistency(self, browser_context):
        """視覺一致性測試"""
        page = await browser_context.new_page()

        # 設定固定視口以確保截圖一致性
        await page.set_viewport_size(1280, 720)
        await page.goto("/login")

        # 等待頁面完全載入
        await page.wait_for_load_state("networkidle")

        # 截取頁面截圖
        screenshot = await page.screenshot(full_page=True)

        # 與基準截圖比較（需要設定基準截圖）
        # 這裡可以整合 Percy、Applitools 或其他視覺測試工具
        await self.compare_with_baseline(screenshot, "login_page")

    async def compare_with_baseline(self, screenshot: bytes, test_name: str):
        """與基準截圖比較"""
        # 實作視覺比較邏輯
        # 可以使用 Pillow 進行像素級比較或整合專業工具
        pass
```

---

## 📋 契約測試：服務間協作保證

### Pact 契約測試實作

```python
import pact
from pact import Consumer, Provider

class ContractTestingFramework:
    """契約測試框架"""

    def __init__(self):
        self.pact = Consumer('auth-service').has_pact_with(Provider('user-service'))

class UserServiceContractTests:
    """與 User Service 的契約測試"""

    @pytest.fixture
    def user_service_pact(self):
        """設定 Pact mock 服務"""
        pact = Consumer('auth-service').has_pact_with(
            Provider('user-service'),
            host_name='localhost',
            port=1234
        )
        pact.start()
        yield pact
        pact.stop()

    def test_get_user_by_id_contract(self, user_service_pact):
        """測試獲取用戶資訊的契約"""
        # 定義期望的互動
        user_service_pact.given(
            'user with id 123 exists'
        ).upon_receiving(
            'a request for user 123'
        ).with_request(
            method='GET',
            path='/users/123',
            headers={'Authorization': 'Bearer token123'}
        ).will_respond_with(
            status=200,
            headers={'Content-Type': 'application/json'},
            body={
                'id': '123',
                'username': 'testuser',
                'email': 'test@example.com',
                'active': True
            }
        )

        # 執行實際請求
        with user_service_pact:
            response = requests.get(
                'http://localhost:1234/users/123',
                headers={'Authorization': 'Bearer token123'}
            )

            assert response.status_code == 200
            assert response.json()['username'] == 'testuser'

class APISchemaValidationTests:
    """API Schema 驗證測試"""

    def test_login_response_schema(self, api_client):
        """驗證登入回應符合 OpenAPI schema"""
        response = api_client.post("/auth/login", json={
            "username": "testuser",
            "password": "password123"
        })

        # 載入 OpenAPI 規範
        with open("openapi.json") as f:
            openapi_spec = json.load(f)

        # 驗證回應格式
        validator = jsonschema.validators.validator_for(openapi_spec)
        login_response_schema = openapi_spec["paths"]["/auth/login"]["post"]["responses"]["200"]["content"]["application/json"]["schema"]

        try:
            jsonschema.validate(response.json(), login_response_schema)
        except jsonschema.ValidationError as e:
            pytest.fail(f"Response doesn't match OpenAPI schema: {e}")

class BackwardCompatibilityTests:
    """向後相容性測試"""

    @pytest.mark.parametrize("api_version", ["v1", "v2"])
    def test_api_version_compatibility(self, api_client, api_version):
        """測試 API 版本相容性"""
        headers = {"Accept": f"application/vnd.api+json;version={api_version}"}

        response = api_client.post("/auth/login",
            headers=headers,
            json={"username": "testuser", "password": "password123"}
        )

        # v1 和 v2 都應該成功，但回應格式可能不同
        assert response.status_code == 200

        if api_version == "v1":
            # v1 回應格式
            assert "access_token" in response.json()
        elif api_version == "v2":
            # v2 可能有額外欄位
            assert "access_token" in response.json()
            assert "refresh_token" in response.json()
```

---

## 🎭 行為驅動開發 (BDD)

### Gherkin 場景與 Step 實作

```python
# features/auth.feature
"""
Feature: User Authentication
  As a user
  I want to authenticate with the system
  So that I can access protected resources

  Background:
    Given the authentication service is running
    And user "testuser" with password "password123" exists

  Scenario: Successful login
    When I submit valid credentials
    Then I should receive an access token
    And the token should be valid for 1 hour

  Scenario: Failed login with invalid password
    When I submit username "testuser" and password "wrongpassword"
    Then I should receive an authentication error
    And no token should be issued

  Scenario Outline: Input validation
    When I submit username "<username>" and password "<password>"
    Then I should receive a validation error "<error>"

    Examples:
      | username | password | error |
      |          | password | Username is required |
      | testuser |          | Password is required |
      | ab       | password | Username too short |

  Scenario: Rate limiting protection
    Given I have made 5 failed login attempts
    When I try to login again
    Then I should be temporarily blocked
    And receive a rate limit error
"""

# steps/auth_steps.py
from behave import given, when, then
from hamcrest import assert_that, is_, equal_to, not_none

@given('the authentication service is running')
def step_impl(context):
    context.client = TestClient(app)

@given('user "{username}" with password "{password}" exists')
def step_impl(context, username, password):
    # 建立測試用戶
    context.test_user = create_test_user(username, password)

@when('I submit valid credentials')
def step_impl(context):
    context.response = context.client.post("/auth/login", json={
        "username": "testuser",
        "password": "password123"
    })

@when('I submit username "{username}" and password "{password}"')
def step_impl(context, username, password):
    context.response = context.client.post("/auth/login", json={
        "username": username,
        "password": password
    })

@then('I should receive an access token')
def step_impl(context):
    assert_that(context.response.status_code, is_(200))
    response_data = context.response.json()
    assert_that(response_data.get('access_token'), is_(not_none()))
    context.access_token = response_data['access_token']

@then('the token should be valid for {duration:d} hour')
def step_impl(context, duration):
    # 解析 JWT token 檢查過期時間
    decoded_token = jwt.decode(
        context.access_token,
        options={"verify_signature": False}
    )

    exp_time = datetime.fromtimestamp(decoded_token['exp'])
    expected_duration = timedelta(hours=duration)

    # 允許小量時間差異
    time_diff = exp_time - datetime.utcnow()
    assert abs(time_diff.total_seconds() - expected_duration.total_seconds()) < 60

class BDDTestRunner:
    """BDD 測試執行器"""

    def run_scenario_tests(self):
        """執行場景測試"""
        # 使用 behave 執行 .feature 檔案
        """
        pip install behave
        behave features/ --tags=@auth
        """

    def generate_living_documentation(self):
        """生成活文檔"""
        # 從 Gherkin 場景生成可讀的文檔
        """
        behave features/ --format=html --outfile=reports/scenarios.html
        """
```

---

## 🚄 效能測試：負載與壓力測試

### 使用 Locust 進行負載測試

```python
from locust import HttpUser, task, between
import random

class AuthServiceUser(HttpUser):
    """認證服務使用者行為模擬"""

    wait_time = between(1, 3)  # 請求間隔 1-3 秒

    def on_start(self):
        """使用者開始時的初始化"""
        self.username = f"user_{random.randint(1000, 9999)}"
        self.password = "password123"
        self.register_user()

    def register_user(self):
        """註冊使用者"""
        response = self.client.post("/auth/register", json={
            "username": self.username,
            "email": f"{self.username}@example.com",
            "password": self.password
        })

        if response.status_code != 201:
            print(f"Registration failed: {response.text}")

    @task(10)
    def login(self):
        """登入任務（權重：10）"""
        with self.client.post("/auth/login", json={
            "username": self.username,
            "password": self.password
        }, catch_response=True) as response:

            if response.status_code == 200:
                self.access_token = response.json().get("access_token")
                response.success()
            else:
                response.failure(f"Login failed: {response.text}")

    @task(5)
    def get_profile(self):
        """獲取個人資料（權重：5）"""
        if hasattr(self, 'access_token'):
            headers = {"Authorization": f"Bearer {self.access_token}"}
            with self.client.get("/user/profile", headers=headers, catch_response=True) as response:
                if response.status_code == 200:
                    response.success()
                else:
                    response.failure(f"Profile fetch failed: {response.text}")

    @task(2)
    def update_profile(self):
        """更新個人資料（權重：2）"""
        if hasattr(self, 'access_token'):
            headers = {"Authorization": f"Bearer {self.access_token}"}
            with self.client.put("/user/profile",
                headers=headers,
                json={"display_name": f"User {random.randint(1, 1000)}"},
                catch_response=True
            ) as response:
                if response.status_code == 200:
                    response.success()
                else:
                    response.failure(f"Profile update failed: {response.text}")

    @task(1)
    def logout(self):
        """登出（權重：1）"""
        if hasattr(self, 'access_token'):
            headers = {"Authorization": f"Bearer {self.access_token}"}
            self.client.post("/auth/logout", headers=headers)
            delattr(self, 'access_token')

class StressTestScenarios:
    """壓力測試場景"""

    def spike_test(self):
        """峰值測試 - 模擬突然的流量激增"""
        """
        執行命令：
        locust -f stress_tests.py --host=http://localhost:8000 \
               --users=1000 --spawn-rate=100 --run-time=300s
        """

    def endurance_test(self):
        """耐久性測試 - 長時間穩定負載"""
        """
        執行命令：
        locust -f stress_tests.py --host=http://localhost:8000 \
               --users=200 --spawn-rate=10 --run-time=3600s
        """

    def volume_test(self):
        """容量測試 - 大量數據處理"""
        """
        測試系統處理大量用戶註冊、大量 token 生成等情況
        """

class PerformanceMetricsCollector:
    """效能指標收集器"""

    def __init__(self):
        self.response_times = []
        self.error_rates = []
        self.throughput_data = []

    def collect_custom_metrics(self):
        """收集自定義效能指標"""
        # 整合 Prometheus metrics
        # 監控 CPU、記憶體、資料庫連接池等
        pass

    def generate_performance_report(self):
        """生成效能測試報告"""
        report = {
            "summary": {
                "avg_response_time": statistics.mean(self.response_times),
                "95th_percentile": statistics.quantiles(self.response_times, n=20)[18],
                "error_rate": sum(self.error_rates) / len(self.error_rates),
                "peak_throughput": max(self.throughput_data)
            },
            "recommendations": self._generate_recommendations()
        }
        return report

    def _generate_recommendations(self):
        """基於測試結果生成優化建議"""
        recommendations = []

        avg_response_time = statistics.mean(self.response_times)
        if avg_response_time > 1000:  # 超過 1 秒
            recommendations.append("回應時間過長，建議優化資料庫查詢或增加快取")

        error_rate = sum(self.error_rates) / len(self.error_rates)
        if error_rate > 0.01:  # 超過 1% 錯誤率
            recommendations.append("錯誤率過高，檢查系統穩定性和錯誤處理")

        return recommendations
```

---

## 🔒 安全測試：漏洞掃描與滲透測試

### 自動化安全測試

```python
class SecurityTestFramework:
    """安全測試框架"""

    @pytest.mark.security
    def test_sql_injection_protection(self, api_client):
        """SQL 注入測試"""
        malicious_payloads = [
            "'; DROP TABLE users; --",
            "' OR '1'='1",
            "' UNION SELECT * FROM users --",
            "'; EXEC xp_cmdshell('dir'); --"
        ]

        for payload in malicious_payloads:
            response = api_client.post("/auth/login", json={
                "username": payload,
                "password": "test"
            })

            # 確保不會洩露敏感錯誤信息
            assert response.status_code in [400, 401, 422]
            assert "SQL" not in response.text.upper()
            assert "ERROR" not in response.text.upper()

    @pytest.mark.security
    def test_xss_protection(self, api_client):
        """跨站腳本攻擊測試"""
        xss_payloads = [
            "<script>alert('xss')</script>",
            "javascript:alert('xss')",
            "<img src=x onerror=alert('xss')>",
            "';alert('xss');//"
        ]

        for payload in xss_payloads:
            response = api_client.post("/user/profile", json={
                "display_name": payload
            })

            # 檢查回應中是否已正確轉義
            if response.status_code == 200:
                assert "<script>" not in response.text
                assert "javascript:" not in response.text

    @pytest.mark.security
    def test_authentication_bypass(self, api_client):
        """認證繞過測試"""
        # 測試�� token 訪問
        response = api_client.get("/user/profile")
        assert response.status_code == 401

        # 測試無效 token
        headers = {"Authorization": "Bearer invalid_token"}
        response = api_client.get("/user/profile", headers=headers)
        assert response.status_code == 401

        # 測試過期 token
        expired_token = self._generate_expired_token()
        headers = {"Authorization": f"Bearer {expired_token}"}
        response = api_client.get("/user/profile", headers=headers)
        assert response.status_code == 401

    @pytest.mark.security
    def test_privilege_escalation(self, api_client):
        """權限提升測試"""
        # 建立普通用戶 token
        user_token = self._get_user_token("regular_user")
        headers = {"Authorization": f"Bearer {user_token}"}

        # 嘗試訪問管理員功能
        response = api_client.get("/admin/users", headers=headers)
        assert response.status_code == 403

        response = api_client.delete("/admin/users/123", headers=headers)
        assert response.status_code == 403

class FuzzTesting:
    """模糊測試"""

    def test_input_fuzzing(self, api_client):
        """輸入模糊測試"""
        import string
        import random

        # 生成隨機輸入
        for _ in range(100):
            random_input = ''.join(
                random.choices(
                    string.ascii_letters + string.digits + string.punctuation,
                    k=random.randint(1, 1000)
                )
            )

            response = api_client.post("/auth/login", json={
                "username": random_input,
                "password": random_input
            })

            # 系統應該優雅地處理任何輸入
            assert response.status_code < 500  # 不應該有伺服器錯誤

class DependencySecurityScanning:
    """依賴包安全掃描"""

    def test_vulnerable_dependencies(self):
        """檢查已知漏洞的依賴包"""
        """
        使用 safety 或 bandit 進行掃描：

        pip install safety bandit
        safety check  # 檢查已知漏洞
        bandit -r app/  # 靜態代碼安全分析
        """

    def generate_security_report(self):
        """生成安全掃描報告"""
        # 整合各種安全工具的結果
        # 生成統一的安全報告
        pass
```

---

## 📊 測試覆蓋率與品質分析

### 進階覆蓋率分析

```python
class CoverageAnalysis:
    """測試覆蓋率分析"""

    def setup_advanced_coverage(self):
        """設定進階覆蓋率分析"""
        """
        # pytest.ini
        [tool:pytest]
        addopts =
            --cov=app
            --cov-report=html:htmlcov
            --cov-report=term-missing
            --cov-report=xml
            --cov-branch  # 分支覆蓋率
            --cov-fail-under=90  # 最低覆蓋率要求
        """

    def analyze_coverage_quality(self):
        """分析覆蓋率品質"""
        # 不只看數字，還要看測試品質
        quality_metrics = {
            "line_coverage": 95,      # 行覆蓋率
            "branch_coverage": 90,    # 分支覆蓋率
            "function_coverage": 100, # 函數覆蓋率
            "assertion_density": 0.8, # 每個測試的平均斷言數
            "test_isolation": 0.95     # 測試隔離度
        }

class MutationTestingIntegration:
    """變異測試整合"""

    def run_mutation_testing(self):
        """執行變異測試"""
        """
        pip install mutmut

        # 執行變異測試
        mutmut run --paths-to-mutate=app/services/

        # 查看結果
        mutmut results
        mutmut show <mutation_id>
        """

    def analyze_mutation_score(self):
        """分析變異測試分數"""
        # 變異測試分數 = 被殺死的變異 / 總變異數
        # 高品質的測試套件應該有 > 80% 的變異分數
        pass

class TestQualityMetrics:
    """測試品質指標"""

    def calculate_test_metrics(self):
        """計算測試指標"""
        return {
            "test_count": self._count_tests(),
            "assertion_count": self._count_assertions(),
            "test_execution_time": self._measure_execution_time(),
            "flaky_test_rate": self._detect_flaky_tests(),
            "test_maintenance_cost": self._calculate_maintenance_cost()
        }

    def _detect_flaky_tests(self):
        """檢測不穩定的測試"""
        # 多次執行測試，檢測結果不一致的測試
        pass

class ContinuousTestingPipeline:
    """持續測試流水線"""

    def setup_test_pipeline(self):
        """設定測試流水線"""
        pipeline_config = {
            "stages": [
                {
                    "name": "unit_tests",
                    "parallel": True,
                    "timeout": "5m",
                    "coverage_threshold": 90
                },
                {
                    "name": "integration_tests",
                    "depends_on": ["unit_tests"],
                    "timeout": "15m"
                },
                {
                    "name": "e2e_tests",
                    "depends_on": ["integration_tests"],
                    "timeout": "30m",
                    "run_on": ["main", "release/*"]
                },
                {
                    "name": "performance_tests",
                    "depends_on": ["integration_tests"],
                    "timeout": "45m",
                    "run_on": ["main"]
                }
            ],
            "notifications": {
                "on_failure": ["team-slack", "email"],
                "on_success": ["metrics-dashboard"]
            }
        }
```

---

## 🎯 測試策略最佳實務總結

### 企業級測試檢查清單

#### ✅ 測試設計原則
- [ ] 遵循測試金字塔原則，單元測試佔主導
- [ ] 每個測試只驗證一個行為
- [ ] 測試名稱清楚描述測試意圖
- [ ] 使用 Arrange-Act-Assert 模式
- [ ] 測試之間完全獨立，可以任意順序執行

#### ✅ 測試覆蓋率
- [ ] 行覆蓋率 > 90%
- [ ] 分支覆蓋率 > 85%
- [ ] 關鍵業務邏輯 100% 覆蓋
- [ ] 變異測試分數 > 80%
- [ ] 定期審查覆蓋率報告

#### ✅ 自動化測試
- [ ] 所有測試可以一鍵執行
- [ ] 測試執行時間控制在合理範圍
- [ ] CI/CD 流水線整合測試
- [ ] 測試失敗時有清楚的錯誤信息
- [ ] 定期清理和維護測試代碼

#### ✅ 測試環境管理
- [ ] 測試環境與生產環境一致
- [ ] 使用容器化測試環境
- [ ] 測試資料管理策略
- [ ] 外部依賴的 mock 策略
- [ ] 測試環境的快速恢復機制

### 🚀 進階測試技術

1. **AI 驅動測試**: 使用機器學習優化測試用例生成
2. **視覺測試**: 使用 AI 進行 UI 回歸測試
3. **混沌工程**: Netflix Chaos Monkey 類型的故障測試
4. **A/B 測試**: 功能發布的實驗性測試
5. **監控驅動測試**: 基於生產監控數據的測試

---

## 💡 學習筆記區

### 🤔 我的理解
```
測試金字塔在現代架構中的演變：

TDD vs BDD 的使用場景：

測試覆蓋率與測試品質的關係：

企業級測試策略的關鍵考量：
```

### 🧪 測試實踐心得
```
實作過程中最大的測試挑戰：

最有價值的測試實踐：

測試維護的經驗分享：

未來測試技術的發展趨勢：
```