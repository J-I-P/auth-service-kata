# Security Deep Dive: 企業級安全架構設計

## 🎯 學習目標
- 掌握現代Web應用的安全威脅模型
- 深入理解JWT與OAuth2的安全設計
- 學習企業級身份認證與授權架構
- 實踐Defense in Depth安全策略

---

## 🔐 現代認證架構演進

### 傳統Session vs JWT vs OAuth2.0

#### **Session-Based Authentication**
```python
# 傳統session模式的安全考量
class SessionManager:
    def __init__(self):
        self.sessions = {}  # 應該使用Redis等外部存儲
        self.session_timeout = 3600

    def create_session(self, user_id: str) -> str:
        session_id = secrets.token_urlsafe(32)
        self.sessions[session_id] = {
            "user_id": user_id,
            "created_at": datetime.utcnow(),
            "last_accessed": datetime.utcnow(),
            "ip_address": request.client.host,  # IP綁定
            "user_agent": request.headers.get("user-agent")  # 設備指紋
        }
        return session_id

    def validate_session(self, session_id: str, client_ip: str) -> Optional[dict]:
        if session_id not in self.sessions:
            return None

        session = self.sessions[session_id]

        # 檢查過期
        if datetime.utcnow() - session["last_accessed"] > timedelta(seconds=self.session_timeout):
            del self.sessions[session_id]
            return None

        # 檢查IP變更（可選，取決於需求）
        if session["ip_address"] != client_ip:
            logger.warning(f"Session {session_id} IP changed from {session['ip_address']} to {client_ip}")
            # 可選擇是否終止session

        # 更新最後訪問時間
        session["last_accessed"] = datetime.utcnow()
        return session

# 優點：容易撤銷、集中控制
# 缺點：需要共享存儲、不適合分散式系統
```

#### **JWT (JSON Web Token) 深度解析**
```python
import jwt
from datetime import datetime, timedelta
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.primitives.asymmetric import rsa

class JWTManager:
    def __init__(self):
        # 生產環境應該從安全存儲載入
        self.private_key = self._load_private_key()
        self.public_key = self._load_public_key()
        self.algorithm = "RS256"  # 使用RSA而非HS256

    def generate_token(self, user_id: str, permissions: List[str], expires_hours: int = 1) -> str:
        """生成安全的JWT token"""
        now = datetime.utcnow()
        payload = {
            # 標準聲明
            "iss": "auth-service",  # Issuer
            "sub": user_id,         # Subject
            "aud": "api-service",   # Audience
            "exp": now + timedelta(hours=expires_hours),  # Expiration
            "nbf": now,             # Not Before
            "iat": now,             # Issued At
            "jti": str(uuid.uuid4()),  # JWT ID (用於撤銷)

            # 自定義聲明
            "permissions": permissions,
            "session_id": str(uuid.uuid4()),
            "device_id": self._get_device_fingerprint()
        }

        return jwt.encode(payload, self.private_key, algorithm=self.algorithm)

    def verify_token(self, token: str) -> Optional[dict]:
        """驗證JWT token"""
        try:
            # 首先檢查是否在黑名單中
            if self._is_token_blacklisted(token):
                return None

            payload = jwt.decode(
                token,
                self.public_key,
                algorithms=[self.algorithm],
                audience="api-service",
                issuer="auth-service"
            )

            # 額外的安全檢查
            if not self._validate_device_fingerprint(payload.get("device_id")):
                logger.warning(f"Device fingerprint mismatch for user {payload['sub']}")
                return None

            return payload

        except jwt.ExpiredSignatureError:
            logger.info("Token has expired")
            return None
        except jwt.InvalidTokenError as e:
            logger.warning(f"Invalid token: {e}")
            return None

    def _is_token_blacklisted(self, token: str) -> bool:
        """檢查token是否在黑名單中（用於登出功能）"""
        # 實際實作會查詢Redis或數據庫
        jti = jwt.decode(token, options={"verify_signature": False})["jti"]
        return redis_client.sismember("token_blacklist", jti)

    def revoke_token(self, token: str):
        """撤銷token（加入黑名單）"""
        decoded = jwt.decode(token, options={"verify_signature": False})
        jti = decoded["jti"]
        exp = decoded["exp"]

        # 將jti加入黑名單，設定過期時間
        redis_client.sadd("token_blacklist", jti)
        redis_client.expireat("token_blacklist", exp)

# JWT的優缺點分析：
# 優點：無狀態、適合分散式、包含用戶信息
# 缺點：難以撤銷、token較大、需要妥善管理密鑰
```

#### **OAuth 2.0 + OIDC 企業級實作**
```python
from authlib.integrations.fastapi_oauth2 import AuthorizationServer
from authlib.oauth2.rfc6749 import grants

class OAuth2AuthorizationServer:
    """企業級OAuth2授權服務器"""

    def __init__(self):
        self.authorization_server = AuthorizationServer()
        self._setup_grants()

    def _setup_grants(self):
        """配置OAuth2授權類型"""
        # Authorization Code Grant (最安全)
        self.authorization_server.register_grant(AuthorizationCodeGrant)

        # Client Credentials Grant (服務間通信)
        self.authorization_server.register_grant(ClientCredentialsGrant)

        # Refresh Token Grant
        self.authorization_server.register_grant(RefreshTokenGrant)

        # 不建議使用Implicit Grant和Password Grant

class EnterpriseAuthFlow:
    """企業級認證流程"""

    def __init__(self):
        self.mfa_manager = MFAManager()
        self.risk_engine = RiskAssessmentEngine()

    async def authenticate_user(self, username: str, password: str, context: AuthContext) -> AuthResult:
        """多層次認證流程"""

        # 第一層：基本驗證
        user = await self.verify_credentials(username, password)
        if not user:
            await self._log_failed_attempt(username, context)
            return AuthResult(success=False, reason="invalid_credentials")

        # 第二層：風險評估
        risk_score = await self.risk_engine.assess_risk(user, context)
        if risk_score > RISK_THRESHOLD:
            # 觸發額外驗證
            return AuthResult(success=False, reason="additional_verification_required",
                            challenge="mfa_required")

        # 第三層：設備信任檢查
        if not await self._is_trusted_device(user.id, context.device_fingerprint):
            # 需要設備驗證
            await self._send_device_verification_email(user.email)
            return AuthResult(success=False, reason="device_verification_required")

        # 第四層：MFA（如果啟用）
        if user.mfa_enabled and not context.mfa_token:
            return AuthResult(success=False, reason="mfa_required")

        if user.mfa_enabled:
            if not await self.mfa_manager.verify_token(user.id, context.mfa_token):
                return AuthResult(success=False, reason="invalid_mfa_token")

        # 生成tokens
        access_token = await self._generate_access_token(user)
        refresh_token = await self._generate_refresh_token(user)

        return AuthResult(
            success=True,
            access_token=access_token,
            refresh_token=refresh_token,
            expires_in=3600
        )

class RiskAssessmentEngine:
    """智能風險評估引擎"""

    async def assess_risk(self, user: User, context: AuthContext) -> float:
        """計算登入風險分數 (0-100)"""
        risk_score = 0

        # 地理位置風險
        if await self._is_unusual_location(user.id, context.ip_address):
            risk_score += 30

        # 時間模式風險
        if await self._is_unusual_time(user.id, context.timestamp):
            risk_score += 20

        # 設備風險
        if await self._is_new_device(user.id, context.device_fingerprint):
            risk_score += 25

        # 行為模式風險
        behavior_score = await self._analyze_behavior_patterns(user.id, context)
        risk_score += behavior_score

        # 威脅情報風險
        if await self._check_threat_intelligence(context.ip_address):
            risk_score += 40

        return min(risk_score, 100)
```

---

## 🛡️ 密碼安全與認證機制

### 進階密碼安全策略

#### **密碼強度與策略**
```python
import re
from passlib.context import CryptContext
from passlib.hash import argon2

class AdvancedPasswordManager:
    def __init__(self):
        # 使用Argon2而非bcrypt（更現代的選擇）
        self.pwd_context = CryptContext(
            schemes=["argon2", "bcrypt"],
            default="argon2",
            argon2__memory_cost=65536,  # 64MB
            argon2__time_cost=3,        # 3次迭代
            argon2__parallelism=2,      # 2個並行線程
        )

    def validate_password_strength(self, password: str) -> PasswordValidationResult:
        """企業級密碼強度檢查"""
        issues = []
        score = 0

        # 長度檢查
        if len(password) < 12:
            issues.append("密碼長度至少需要12個字符")
        elif len(password) >= 16:
            score += 20
        else:
            score += 10

        # 複雜度檢查
        if re.search(r'[a-z]', password):
            score += 10
        else:
            issues.append("需要包含小寫字母")

        if re.search(r'[A-Z]', password):
            score += 10
        else:
            issues.append("需要包含大寫字母")

        if re.search(r'\d', password):
            score += 10
        else:
            issues.append("需要包含數字")

        if re.search(r'[!@#$%^&*(),.?":{}|<>]', password):
            score += 15
        else:
            issues.append("需要包含特殊字符")

        # 常見密碼檢查
        if self._is_common_password(password):
            issues.append("這是一個常見密碼，請選擇更獨特的密碼")
            score -= 30

        # 重複字符檢查
        if self._has_excessive_repetition(password):
            issues.append("密碼包含過多重複字符")
            score -= 10

        # 順序字符檢查
        if self._has_sequential_characters(password):
            issues.append("避免使用順序字符（如123、abc）")
            score -= 15

        return PasswordValidationResult(
            is_valid=len(issues) == 0,
            score=max(score, 0),
            issues=issues
        )

    def hash_password(self, password: str) -> str:
        """安全的密碼雜湊"""
        return self.pwd_context.hash(password)

    def verify_password(self, plain_password: str, hashed_password: str) -> bool:
        """驗證密碼"""
        return self.pwd_context.verify(plain_password, hashed_password)

    def needs_update(self, hashed_password: str) -> bool:
        """檢查是否需要更新雜湊演算法"""
        return self.pwd_context.needs_update(hashed_password)

class PasswordPolicy:
    """密碼策略管理"""

    def __init__(self):
        self.history_size = 12  # 記住最近12個密碼
        self.min_age_hours = 24  # 最小密碼年齡
        self.max_age_days = 90   # 最大密碼年齡

    async def can_change_password(self, user_id: str, new_password: str) -> bool:
        """檢查是否可以更改密碼"""
        user = await self.user_repo.get_user(user_id)

        # 檢查密碼年齡
        last_change = user.password_changed_at
        if datetime.utcnow() - last_change < timedelta(hours=self.min_age_hours):
            return False

        # 檢查密碼歷史
        password_history = await self.get_password_history(user_id)
        for old_hash in password_history[-self.history_size:]:
            if self.pwd_manager.verify_password(new_password, old_hash):
                return False

        return True
```

#### **多因子認證 (MFA) 實作**
```python
import pyotp
import qrcode
from io import BytesIO

class MFAManager:
    """多因子認證管理器"""

    def __init__(self):
        self.app_name = "AuthService"

    def setup_totp(self, user: User) -> TOTPSetupResult:
        """設置TOTP (Time-based One-Time Password)"""
        secret = pyotp.random_base32()
        totp = pyotp.TOTP(secret)

        # 生成QR碼
        provisioning_uri = totp.provisioning_uri(
            name=user.email,
            issuer_name=self.app_name
        )

        qr = qrcode.QRCode(version=1, box_size=10, border=5)
        qr.add_data(provisioning_uri)
        qr.make(fit=True)

        img = qr.make_image(fill_color="black", back_color="white")

        # 轉換為base64以便傳輸
        buffer = BytesIO()
        img.save(buffer, format='PNG')
        qr_code_data = base64.b64encode(buffer.getvalue()).decode()

        return TOTPSetupResult(
            secret=secret,
            qr_code=qr_code_data,
            backup_codes=self._generate_backup_codes()
        )

    def verify_totp(self, secret: str, token: str, window: int = 1) -> bool:
        """驗證TOTP令牌"""
        totp = pyotp.TOTP(secret)
        return totp.verify(token, valid_window=window)

    def _generate_backup_codes(self, count: int = 10) -> List[str]:
        """生成備用恢復碼"""
        return [secrets.token_hex(4).upper() for _ in range(count)]

    async def verify_backup_code(self, user_id: str, code: str) -> bool:
        """驗證備用恢復碼"""
        user_backup_codes = await self.get_user_backup_codes(user_id)

        if code in user_backup_codes:
            # 使用後立即刪除
            await self.remove_backup_code(user_id, code)
            return True
        return False

class WebAuthnManager:
    """WebAuthn (FIDO2) 實作"""

    def __init__(self):
        from webauthn import generate_registration_options, verify_registration_response
        from webauthn import generate_authentication_options, verify_authentication_response

        self.rp_id = "auth-service.com"
        self.rp_name = "Auth Service"

    async def begin_registration(self, user: User) -> dict:
        """開始WebAuthn註冊流程"""
        options = generate_registration_options(
            rp_id=self.rp_id,
            rp_name=self.rp_name,
            user_id=user.id.encode(),
            user_name=user.username,
            user_display_name=user.display_name,
        )

        # 存儲challenge以供後續驗證
        await self.store_challenge(user.id, options.challenge)

        return options

    async def complete_registration(self, user_id: str, credential: dict) -> bool:
        """完成WebAuthn註冊"""
        stored_challenge = await self.get_stored_challenge(user_id)

        verification = verify_registration_response(
            credential=credential,
            expected_challenge=stored_challenge,
            expected_origin=f"https://{self.rp_id}",
            expected_rp_id=self.rp_id,
        )

        if verification.verified:
            # 存儲用戶的憑證
            await self.store_user_credential(user_id, {
                "id": verification.credential_id,
                "public_key": verification.credential_public_key,
                "sign_count": verification.sign_count,
            })
            return True
        return False
```

---

## 🚨 威脅建模與防護

### 常見安全威脅與防護策略

#### **注入攻擊防護**
```python
from sqlalchemy.orm import Session
from sqlalchemy import text

class SecureDataAccess:
    """安全的數據訪問層"""

    def __init__(self, db: Session):
        self.db = db

    async def safe_user_lookup(self, username: str) -> Optional[User]:
        """防止SQL注入的用戶查詢"""
        # 錯誤的做法：
        # query = f"SELECT * FROM users WHERE username = '{username}'"

        # 正確的做法：使用參數化查詢
        result = await self.db.execute(
            text("SELECT * FROM users WHERE username = :username"),
            {"username": username}
        )
        return result.fetchone()

    async def search_users(self, search_term: str, limit: int = 10) -> List[User]:
        """安全的用戶搜索（防止NoSQL注入）"""
        # 輸入驗證
        if not re.match(r'^[a-zA-Z0-9\s\-_.]+$', search_term):
            raise ValueError("Invalid search term")

        # 限制長度
        if len(search_term) > 100:
            raise ValueError("Search term too long")

        # 使用參數化查詢
        result = await self.db.execute(
            text("""
                SELECT * FROM users
                WHERE username ILIKE :search_term
                OR email ILIKE :search_term
                LIMIT :limit
            """),
            {
                "search_term": f"%{search_term}%",
                "limit": limit
            }
        )
        return result.fetchall()

class InputSanitizer:
    """輸入數據清理器"""

    @staticmethod
    def sanitize_html(input_text: str) -> str:
        """防止XSS攻擊的HTML清理"""
        import bleach

        allowed_tags = ['b', 'i', 'u', 'em', 'strong', 'p', 'br']
        allowed_attributes = {}

        return bleach.clean(input_text, tags=allowed_tags, attributes=allowed_attributes)

    @staticmethod
    def validate_email(email: str) -> bool:
        """安全的郵箱驗證"""
        pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
        return re.match(pattern, email) is not None

    @staticmethod
    def sanitize_filename(filename: str) -> str:
        """文件名安全清理"""
        # 移除路徑遍歷字符
        filename = os.path.basename(filename)

        # 只允許特定字符
        filename = re.sub(r'[^a-zA-Z0-9._-]', '', filename)

        # 限制長度
        if len(filename) > 255:
            name, ext = os.path.splitext(filename)
            filename = name[:255-len(ext)] + ext

        return filename
```

#### **CSRF與CORS防護**
```python
from fastapi.middleware.cors import CORSMiddleware
from fastapi.middleware.csrf import CSRFMiddleware

class SecurityMiddleware:
    """安全中間件配置"""

    @staticmethod
    def setup_cors(app: FastAPI, allowed_origins: List[str]):
        """配置CORS防護"""
        app.add_middleware(
            CORSMiddleware,
            allow_origins=allowed_origins,  # 絕不使用 ["*"] 在生產環境
            allow_credentials=True,
            allow_methods=["GET", "POST", "PUT", "DELETE"],
            allow_headers=["*"],
            expose_headers=["X-Total-Count"],  # 暴露特定header
            max_age=3600,  # 預檢請求快取時間
        )

    @staticmethod
    def setup_csrf(app: FastAPI):
        """配置CSRF防護"""
        app.add_middleware(
            CSRFMiddleware,
            secret_key="your-secret-key",
            cookie_name="csrftoken",
            header_name="X-CSRFToken",
            cookie_secure=True,  # 只在HTTPS下傳輸
            cookie_samesite="strict",
        )

class APISecurityHeaders:
    """API安全標頭管理"""

    @staticmethod
    def add_security_headers(response: Response):
        """添加安全響應標頭"""
        # 防止點擊劫持
        response.headers["X-Frame-Options"] = "DENY"

        # 防止MIME類型嗅探
        response.headers["X-Content-Type-Options"] = "nosniff"

        # XSS保護
        response.headers["X-XSS-Protection"] = "1; mode=block"

        # 強制HTTPS
        response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"

        # CSP (Content Security Policy)
        response.headers["Content-Security-Policy"] = "default-src 'self'; script-src 'self'"

        # 隱藏服務器信息
        response.headers["Server"] = "API-Gateway"

        return response
```

#### **Rate Limiting與DDoS防護**
```python
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

class AdvancedRateLimiter:
    """進階限流管理器"""

    def __init__(self):
        self.limiter = Limiter(
            key_func=self._get_identifier,
            storage_uri="redis://localhost:6379"
        )

    def _get_identifier(self, request: Request) -> str:
        """智能識別器組合多個因素"""
        # 優先使用認證用戶ID
        if hasattr(request.state, 'user_id'):
            return f"user:{request.state.user_id}"

        # 其次使用IP地址
        return f"ip:{get_remote_address(request)}"

    def create_adaptive_limiter(self, base_rate: str = "100/hour"):
        """自適應限流器"""
        def adaptive_rate_limit(request: Request) -> str:
            # 根據用戶等級調整限制
            if hasattr(request.state, 'user_tier'):
                tier = request.state.user_tier
                if tier == "premium":
                    return "1000/hour"
                elif tier == "business":
                    return "5000/hour"

            # 根據時間調整（夜間放寬限制）
            current_hour = datetime.utcnow().hour
            if 0 <= current_hour <= 6:  # 夜間時段
                return "200/hour"

            return base_rate

        return adaptive_rate_limit

class DDoSProtection:
    """DDoS攻擊防護"""

    def __init__(self):
        self.suspicious_ips = set()
        self.request_patterns = {}

    async def analyze_request_pattern(self, request: Request) -> bool:
        """分析請求模式檢測異常"""
        client_ip = get_remote_address(request)
        current_time = time.time()

        # 記錄請求模式
        if client_ip not in self.request_patterns:
            self.request_patterns[client_ip] = []

        self.request_patterns[client_ip].append({
            "timestamp": current_time,
            "path": request.url.path,
            "user_agent": request.headers.get("user-agent", ""),
        })

        # 保留最近5分鐘的請求
        five_minutes_ago = current_time - 300
        self.request_patterns[client_ip] = [
            req for req in self.request_patterns[client_ip]
            if req["timestamp"] > five_minutes_ago
        ]

        # 檢測異常模式
        recent_requests = self.request_patterns[client_ip]

        # 檢測過快的請求頻率
        if len(recent_requests) > 100:  # 5分鐘內超過100個請求
            logger.warning(f"High request frequency from {client_ip}")
            self.suspicious_ips.add(client_ip)
            return False

        # 檢測相同User-Agent的過多請求
        user_agents = [req["user_agent"] for req in recent_requests]
        if len(set(user_agents)) == 1 and len(recent_requests) > 50:
            logger.warning(f"Suspicious user agent pattern from {client_ip}")
            return False

        return True

    def is_ip_blocked(self, client_ip: str) -> bool:
        """檢查IP是否被封鎖"""
        return client_ip in self.suspicious_ips
```

---

## 🔍 安全審計與監控

### 企業級安全日誌與監控

```python
import structlog
from enum import Enum

class SecurityEventType(Enum):
    LOGIN_SUCCESS = "login_success"
    LOGIN_FAILURE = "login_failure"
    PASSWORD_CHANGE = "password_change"
    MFA_ENABLED = "mfa_enabled"
    SUSPICIOUS_ACTIVITY = "suspicious_activity"
    TOKEN_ISSUED = "token_issued"
    TOKEN_REVOKED = "token_revoked"
    PERMISSION_ESCALATION = "permission_escalation"

class SecurityAuditLogger:
    """安全審計日誌記錄器"""

    def __init__(self):
        self.logger = structlog.get_logger("security_audit")

    async def log_security_event(
        self,
        event_type: SecurityEventType,
        user_id: Optional[str] = None,
        ip_address: Optional[str] = None,
        user_agent: Optional[str] = None,
        details: Optional[dict] = None
    ):
        """記錄安全事件"""
        event_data = {
            "event_type": event_type.value,
            "timestamp": datetime.utcnow().isoformat(),
            "user_id": user_id,
            "ip_address": ip_address,
            "user_agent": user_agent,
            "session_id": self._get_current_session_id(),
            "details": details or {}
        }

        # 結構化日誌
        self.logger.info("Security event", **event_data)

        # 高風險事件立即告警
        if self._is_high_risk_event(event_type):
            await self._send_security_alert(event_data)

    def _is_high_risk_event(self, event_type: SecurityEventType) -> bool:
        """判斷是否為高風險事件"""
        high_risk_events = {
            SecurityEventType.PERMISSION_ESCALATION,
            SecurityEventType.SUSPICIOUS_ACTIVITY,
        }
        return event_type in high_risk_events

    async def _send_security_alert(self, event_data: dict):
        """發送安全告警"""
        # 發送到監控系統
        await self.monitoring_client.send_alert(
            severity="high",
            title="Security Event Detected",
            description=f"High-risk security event: {event_data['event_type']}",
            details=event_data
        )

class SecurityMetrics:
    """安全指標收集器"""

    def __init__(self):
        self.metrics_client = PrometheusMetrics()

    def record_login_attempt(self, success: bool, user_id: str, ip: str):
        """記錄登入嘗試"""
        labels = {
            "success": str(success).lower(),
            "user_id": user_id,
            "ip_class": self._get_ip_class(ip)
        }
        self.metrics_client.increment("login_attempts_total", labels)

        if not success:
            self.metrics_client.increment("login_failures_total", labels)

    def record_token_usage(self, token_type: str, action: str):
        """記錄token使用情況"""
        self.metrics_client.increment("token_operations_total", {
            "token_type": token_type,
            "action": action
        })

    def record_security_incident(self, incident_type: str, severity: str):
        """記錄安全事件"""
        self.metrics_client.increment("security_incidents_total", {
            "type": incident_type,
            "severity": severity
        })

class SecurityDashboard:
    """安全監控儀表板數據提供者"""

    async def get_security_overview(self, timeframe: str = "24h") -> dict:
        """獲取安全概覽"""
        return {
            "login_stats": await self._get_login_statistics(timeframe),
            "threat_indicators": await self._get_threat_indicators(timeframe),
            "system_health": await self._get_system_health(),
            "active_sessions": await self._get_active_sessions_count(),
            "recent_incidents": await self._get_recent_security_incidents(5)
        }

    async def _get_threat_indicators(self, timeframe: str) -> dict:
        """獲取威脅指標"""
        return {
            "failed_login_attempts": await self.count_failed_logins(timeframe),
            "suspicious_ips": await self.count_suspicious_ips(timeframe),
            "blocked_requests": await self.count_blocked_requests(timeframe),
            "anomaly_score": await self.calculate_anomaly_score(timeframe)
        }
```

---

## 🏗️ 零信任架構設計

### 現代零信任安全模型

```python
class ZeroTrustGateway:
    """零信任網關實作"""

    def __init__(self):
        self.policy_engine = PolicyEngine()
        self.device_trust = DeviceTrustManager()
        self.context_analyzer = ContextAnalyzer()

    async def evaluate_request(self, request: Request, user: User) -> AccessDecision:
        """評估請求的零信任策略"""

        # 1. 身份驗證 (Authentication)
        if not await self._verify_identity(request, user):
            return AccessDecision.DENY

        # 2. 設備信任評估
        device_trust_score = await self.device_trust.evaluate_device(
            request.headers.get("device-id"),
            request.headers.get("user-agent")
        )

        # 3. 環境上下文分析
        context = await self.context_analyzer.analyze(request, user)

        # 4. 策略評估
        policy_result = await self.policy_engine.evaluate({
            "user": user,
            "resource": request.url.path,
            "method": request.method,
            "device_trust": device_trust_score,
            "context": context
        })

        # 5. 即時威脅檢測
        threat_score = await self._assess_threats(request, user, context)

        # 6. 綜合決策
        final_decision = self._make_access_decision(
            policy_result, device_trust_score, threat_score
        )

        # 7. 記錄決策
        await self._log_access_decision(request, user, final_decision)

        return final_decision

class PolicyEngine:
    """策略引擎"""

    def __init__(self):
        self.policies = self._load_policies()

    async def evaluate(self, context: dict) -> PolicyResult:
        """評估策略"""
        for policy in self.policies:
            if await policy.matches(context):
                result = await policy.evaluate(context)
                if result.decision == PolicyDecision.DENY:
                    return result

        # 預設拒絕
        return PolicyResult(
            decision=PolicyDecision.DENY,
            reason="No matching allow policy found"
        )

class MicroSegmentation:
    """微分段網路安全"""

    def __init__(self):
        self.network_policies = NetworkPolicyManager()

    async def apply_network_segmentation(self, service: str, user_role: str) -> NetworkPolicy:
        """應用網路微分段策略"""

        # 基於角色的網路訪問控制
        allowed_services = self._get_allowed_services_for_role(user_role)

        # 動態防火牆規則
        firewall_rules = []
        for allowed_service in allowed_services:
            firewall_rules.extend(
                await self._generate_firewall_rules(service, allowed_service)
            )

        return NetworkPolicy(
            source_service=service,
            allowed_destinations=allowed_services,
            firewall_rules=firewall_rules,
            expiry=datetime.utcnow() + timedelta(hours=1)  # 動態策略過期
        )
```

---

## 🧪 安全測試與驗證

### 自動化安全測試框架

```python
import pytest
from httpx import AsyncClient

class SecurityTestSuite:
    """安全測試套件"""

    async def test_authentication_vulnerabilities(self, client: AsyncClient):
        """認證漏洞測試"""
        await self._test_brute_force_protection(client)
        await self._test_session_fixation(client)
        await self._test_credential_stuffing(client)

    async def _test_brute_force_protection(self, client: AsyncClient):
        """暴力破解防護測試"""
        # 嘗試多次失敗登入
        for i in range(10):
            response = await client.post("/auth/login", json={
                "username": "testuser",
                "password": f"wrongpassword{i}"
            })

        # 檢查是否觸發限制
        response = await client.post("/auth/login", json={
            "username": "testuser",
            "password": "wrongpassword"
        })
        assert response.status_code == 429  # Too Many Requests

    async def test_injection_vulnerabilities(self, client: AsyncClient):
        """注入攻擊測試"""
        payloads = [
            "'; DROP TABLE users; --",
            "<script>alert('xss')</script>",
            "../../etc/passwd",
            "{{7*7}}",  # Template injection
        ]

        for payload in payloads:
            response = await client.post("/auth/login", json={
                "username": payload,
                "password": "test"
            })
            # 檢查是否正確處理惡意輸入
            assert response.status_code in [400, 422]

    async def test_authorization_bypass(self, client: AsyncClient):
        """授權繞過測試"""
        # 測試未授權訪問
        response = await client.get("/admin/users")
        assert response.status_code == 401

        # 測試權限提升
        user_token = await self._get_user_token(client)
        headers = {"Authorization": f"Bearer {user_token}"}
        response = await client.get("/admin/users", headers=headers)
        assert response.status_code == 403

class PenetrationTestFramework:
    """滲透測試框架"""

    def __init__(self):
        self.test_cases = self._load_test_cases()

    async def run_security_scan(self, target_url: str) -> SecurityScanReport:
        """執行自動化安全掃描"""
        results = []

        for test_case in self.test_cases:
            result = await test_case.execute(target_url)
            results.append(result)

        return SecurityScanReport(
            target=target_url,
            scan_date=datetime.utcnow(),
            results=results,
            risk_score=self._calculate_risk_score(results)
        )
```

---

## 📚 安全最佳實務總結

### 企業級安全檢查清單

#### ✅ 認證與授權
- [ ] 實作強密碼策略和多因子認證
- [ ] 使用安全的密碼雜湊算法 (Argon2, bcrypt)
- [ ] 實作適當的會話管理和token策略
- [ ] 建立完整的權限控制系統 (RBAC/ABAC)
- [ ] 實作零信任網路架構

#### ✅ 數據保護
- [ ] 所有敏感數據加密傳輸 (TLS 1.3)
- [ ] 靜態數據加密存儲
- [ ] 實作數據分類和標記
- [ ] 建立數據備份和恢復機制
- [ ] 遵循數據隱私法規 (GDPR, CCPA)

#### ✅ 威脅防護
- [ ] 實作全面的輸入驗證和清理
- [ ] 部署WAF和DDoS防護
- [ ] 建立入侵檢測和回應系統
- [ ] 實作漏洞掃描和修補管理
- [ ] 建立事件回應計劃

#### ✅ 監控與審計
- [ ] 實作完整的安全事件日誌
- [ ] 建立即時威脅監控
- [ ] 定期安全評估和滲透測試
- [ ] 實作合規性監控
- [ ] 建立安全指標和報告

### 🚀 進階學習方向

1. **雲端安全架構**: AWS IAM, Azure AD, GCP Security
2. **容器安全**: Docker安全, Kubernetes RBAC
3. **DevSecOps**: 安全左移, CI/CD安全整合
4. **威脅建模**: STRIDE, PASTA方法論
5. **密碼學應用**: PKI, HSM, 量子抗性加密

---

## 💡 學習筆記區

### 🤔 我的理解
```
現代認證架構的演進趨勢：

零信任安全模型的核心原則：

企業級安全策略的關鍵要素：

安全與使用者體驗的平衡點：
```

### 🔒 安全實踐心得
```
實作過程中遇到的安全挑戰：

最有價值的安全防護機制：

企業環境中的安全考量：

未來安全技術的發展方向：
```