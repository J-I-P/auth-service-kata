# Microservices Architecture: 企業級微服務設計與實踐

## 🎯 學習目標
- 掌握微服務架構的設計原則與模式
- 學習服務間通信與資料一致性策略
- 建立分散式系統的錯誤處理與恢復機制
- 實踐微服務的監控、部署與維運策略

---

## 🏗️ 微服務架構設計原則

### 領域驅動設計 (Domain-Driven Design)

```python
# domain/user/models.py
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import List, Optional
from datetime import datetime

# 領域實體 (Entity)
@dataclass
class User:
    """使用者實體"""
    id: str
    username: str
    email: str
    profile: 'UserProfile'
    created_at: datetime

    def change_email(self, new_email: str, email_service: 'EmailService'):
        """變更電子郵件"""
        if not email_service.is_valid_email(new_email):
            raise ValueError("Invalid email format")

        old_email = self.email
        self.email = new_email

        # 領域事件
        return UserEmailChanged(
            user_id=self.id,
            old_email=old_email,
            new_email=new_email,
            timestamp=datetime.utcnow()
        )

# 值物件 (Value Object)
@dataclass(frozen=True)
class UserProfile:
    """使用者個人資料值物件"""
    first_name: str
    last_name: str
    display_name: str

    def __post_init__(self):
        if not self.first_name or not self.last_name:
            raise ValueError("First name and last name are required")

# 領域事件 (Domain Event)
@dataclass
class UserEmailChanged:
    """使用者郵箱變更事件"""
    user_id: str
    old_email: str
    new_email: str
    timestamp: datetime

# 領域服務 (Domain Service)
class UserService:
    """使用者領域服務"""

    def __init__(self, user_repository: 'UserRepository', email_service: 'EmailService'):
        self.user_repository = user_repository
        self.email_service = email_service

    async def register_user(self, username: str, email: str, profile: UserProfile) -> User:
        """註冊使用者"""
        # 檢查使用者名稱唯一性
        existing_user = await self.user_repository.find_by_username(username)
        if existing_user:
            raise UserAlreadyExistsError(f"Username {username} already exists")

        # 檢查郵箱唯一性
        existing_email = await self.user_repository.find_by_email(email)
        if existing_email:
            raise EmailAlreadyExistsError(f"Email {email} already exists")

        # 建立新使用者
        user = User(
            id=str(uuid.uuid4()),
            username=username,
            email=email,
            profile=profile,
            created_at=datetime.utcnow()
        )

        await self.user_repository.save(user)

        # 發送歡迎郵件
        await self.email_service.send_welcome_email(user)

        return user

# 倉庫介面 (Repository Interface)
class UserRepository(ABC):
    """使用者倉庫介面"""

    @abstractmethod
    async def find_by_id(self, user_id: str) -> Optional[User]:
        pass

    @abstractmethod
    async def find_by_username(self, username: str) -> Optional[User]:
        pass

    @abstractmethod
    async def find_by_email(self, email: str) -> Optional[User]:
        pass

    @abstractmethod
    async def save(self, user: User) -> None:
        pass

    @abstractmethod
    async def delete(self, user_id: str) -> None:
        pass

# 應用服務 (Application Service)
class UserApplicationService:
    """使用者應用服務"""

    def __init__(self,
                 user_service: UserService,
                 event_bus: 'EventBus',
                 notification_service: 'NotificationService'):
        self.user_service = user_service
        self.event_bus = event_bus
        self.notification_service = notification_service

    async def register_user(self, command: RegisterUserCommand) -> RegisterUserResponse:
        """註冊使用者用例"""
        try:
            profile = UserProfile(
                first_name=command.first_name,
                last_name=command.last_name,
                display_name=command.display_name
            )

            user = await self.user_service.register_user(
                username=command.username,
                email=command.email,
                profile=profile
            )

            # 發布領域事件
            event = UserRegistered(
                user_id=user.id,
                username=user.username,
                email=user.email,
                timestamp=datetime.utcnow()
            )
            await self.event_bus.publish(event)

            return RegisterUserResponse(
                user_id=user.id,
                success=True,
                message="User registered successfully"
            )

        except (UserAlreadyExistsError, EmailAlreadyExistsError) as e:
            return RegisterUserResponse(
                success=False,
                error_code="USER_EXISTS",
                message=str(e)
            )
```

### 服務邊界定義

```python
# services/auth_service/service_definition.py
from dataclasses import dataclass
from typing import List, Dict, Any
from enum import Enum

class ServiceType(Enum):
    CORE = "core"          # 核心業務服務
    PLATFORM = "platform"  # 平台服務
    EDGE = "edge"          # 邊緣服務

@dataclass
class ServiceDefinition:
    """服務定義"""
    name: str
    version: str
    service_type: ServiceType
    domain: str
    responsibilities: List[str]
    dependencies: List[str]
    exposed_apis: List[str]
    data_ownership: List[str]

class AuthServiceDefinition:
    """認證服務定義"""

    DEFINITION = ServiceDefinition(
        name="auth-service",
        version="1.0.0",
        service_type=ServiceType.CORE,
        domain="identity-and-access",
        responsibilities=[
            "User authentication",
            "Token management",
            "Session management",
            "Password policies",
            "Multi-factor authentication"
        ],
        dependencies=[
            "user-service",
            "notification-service",
            "audit-service"
        ],
        exposed_apis=[
            "/auth/login",
            "/auth/logout",
            "/auth/refresh",
            "/auth/verify",
            "/auth/mfa/setup",
            "/auth/mfa/verify"
        ],
        data_ownership=[
            "authentication_sessions",
            "refresh_tokens",
            "mfa_configurations",
            "login_attempts"
        ]
    )

class ServiceBoundaryAnalyzer:
    """服務邊界分析器"""

    def __init__(self):
        self.services = {}

    def analyze_coupling(self, service1: str, service2: str) -> Dict[str, Any]:
        """分析服務間耦合度"""
        coupling_analysis = {
            "data_coupling": self._analyze_data_coupling(service1, service2),
            "temporal_coupling": self._analyze_temporal_coupling(service1, service2),
            "spatial_coupling": self._analyze_spatial_coupling(service1, service2),
            "recommendation": ""
        }

        # 計算總體耦合分數 (0-100)
        total_score = (
            coupling_analysis["data_coupling"]["score"] * 0.4 +
            coupling_analysis["temporal_coupling"]["score"] * 0.3 +
            coupling_analysis["spatial_coupling"]["score"] * 0.3
        )

        coupling_analysis["total_score"] = total_score
        coupling_analysis["recommendation"] = self._get_coupling_recommendation(total_score)

        return coupling_analysis

    def _analyze_data_coupling(self, service1: str, service2: str) -> Dict[str, Any]:
        """分析資料耦合"""
        # 分析共享的資料結構、API 依賴等
        return {
            "score": 30,  # 0-100, 越低越好
            "shared_data_structures": ["User", "Session"],
            "api_dependencies": ["/user/profile", "/user/permissions"]
        }

    def _get_coupling_recommendation(self, score: float) -> str:
        """根據耦合分數提供建議"""
        if score < 30:
            return "Good separation - maintain current boundaries"
        elif score < 60:
            return "Moderate coupling - consider refactoring shared dependencies"
        else:
            return "High coupling - strongly consider service consolidation or redesign"

class ServiceDecompositionStrategy:
    """服務分解策略"""

    @staticmethod
    def decompose_by_business_capability():
        """按業務能力分解"""
        return {
            "user-management": [
                "User registration",
                "User profile management",
                "User preferences"
            ],
            "authentication": [
                "Login/logout",
                "Token management",
                "Session management"
            ],
            "authorization": [
                "Permission management",
                "Role-based access control",
                "Resource authorization"
            ],
            "notification": [
                "Email notifications",
                "SMS notifications",
                "Push notifications"
            ]
        }

    @staticmethod
    def decompose_by_data():
        """按資料邊界分解"""
        return {
            "user-service": {
                "owns": ["users", "user_profiles", "user_preferences"],
                "references": ["roles", "permissions"]
            },
            "auth-service": {
                "owns": ["sessions", "tokens", "login_attempts"],
                "references": ["users"]
            },
            "role-service": {
                "owns": ["roles", "permissions", "role_assignments"],
                "references": ["users"]
            }
        }
```

---

## 🔗 服務間通信模式

### 同步通信：REST API 與 gRPC

```python
# communication/rest_client.py
import httpx
import asyncio
from typing import Optional, Dict, Any
from dataclasses import dataclass
from circuitbreaker import circuit_breaker
import structlog

logger = structlog.get_logger()

@dataclass
class ServiceEndpoint:
    """服務端點配置"""
    name: str
    base_url: str
    timeout: float = 30.0
    retry_attempts: int = 3
    circuit_breaker_threshold: int = 5

class ResilientHttpClient:
    """具備恢復能力的 HTTP 客戶端"""

    def __init__(self, endpoint: ServiceEndpoint):
        self.endpoint = endpoint
        self.client = self._create_client()

    def _create_client(self) -> httpx.AsyncClient:
        """建立 HTTP 客戶端"""
        return httpx.AsyncClient(
            base_url=self.endpoint.base_url,
            timeout=httpx.Timeout(self.endpoint.timeout),
            limits=httpx.Limits(max_keepalive_connections=20, max_connections=100)
        )

    @circuit_breaker(failure_threshold=5, recovery_timeout=30, expected_exception=httpx.RequestError)
    async def get(self, path: str, params: Optional[Dict] = None) -> Optional[Dict[str, Any]]:
        """GET 請求"""
        return await self._request_with_retry("GET", path, params=params)

    @circuit_breaker(failure_threshold=5, recovery_timeout=30, expected_exception=httpx.RequestError)
    async def post(self, path: str, json_data: Optional[Dict] = None) -> Optional[Dict[str, Any]]:
        """POST 請求"""
        return await self._request_with_retry("POST", path, json=json_data)

    async def _request_with_retry(self, method: str, path: str, **kwargs) -> Optional[Dict[str, Any]]:
        """帶重試機制的請求"""
        last_exception = None

        for attempt in range(self.endpoint.retry_attempts):
            try:
                response = await self.client.request(method, path, **kwargs)
                response.raise_for_status()

                return response.json()

            except httpx.RequestError as e:
                last_exception = e
                wait_time = 2 ** attempt  # 指數退避
                logger.warning(f"Request failed, retrying in {wait_time}s",
                             service=self.endpoint.name,
                             attempt=attempt + 1,
                             error=str(e))

                if attempt < self.endpoint.retry_attempts - 1:
                    await asyncio.sleep(wait_time)

            except httpx.HTTPStatusError as e:
                if e.response.status_code >= 500:
                    # 伺服器錯誤，重試
                    last_exception = e
                    if attempt < self.endpoint.retry_attempts - 1:
                        await asyncio.sleep(2 ** attempt)
                        continue
                else:
                    # 客戶端錯誤，不重試
                    logger.error("Client error, not retrying",
                               service=self.endpoint.name,
                               status_code=e.response.status_code)
                    return None

        logger.error("All retry attempts failed",
                    service=self.endpoint.name,
                    error=str(last_exception))
        return None

# communication/grpc_client.py
import grpc
from grpc import aio
from typing import Optional, AsyncGenerator
import user_service_pb2_grpc
import user_service_pb2

class UserServiceClient:
    """使用者服務 gRPC 客戶端"""

    def __init__(self, channel: aio.Channel):
        self.stub = user_service_pb2_grpc.UserServiceStub(channel)

    async def get_user(self, user_id: str) -> Optional[user_service_pb2.User]:
        """取得使用者資訊"""
        try:
            request = user_service_pb2.GetUserRequest(user_id=user_id)
            response = await self.stub.GetUser(request)
            return response
        except grpc.RpcError as e:
            logger.error("gRPC call failed",
                        method="GetUser",
                        error_code=e.code(),
                        error_details=e.details())
            return None

    async def stream_user_events(self, user_id: str) -> AsyncGenerator[user_service_pb2.UserEvent, None]:
        """串流使用者事件"""
        try:
            request = user_service_pb2.StreamEventsRequest(user_id=user_id)
            async for event in self.stub.StreamUserEvents(request):
                yield event
        except grpc.RpcError as e:
            logger.error("gRPC stream failed", error=str(e))

class GrpcServiceRegistry:
    """gRPC 服務註冊中心"""

    def __init__(self):
        self.channels = {}
        self.clients = {}

    async def get_user_service_client(self) -> UserServiceClient:
        """取得使用者服務客戶端"""
        if "user-service" not in self.clients:
            channel = await self._create_channel("user-service", "localhost:50051")
            self.clients["user-service"] = UserServiceClient(channel)

        return self.clients["user-service"]

    async def _create_channel(self, service_name: str, address: str) -> aio.Channel:
        """建立 gRPC 頻道"""
        if service_name not in self.channels:
            self.channels[service_name] = aio.insecure_channel(
                address,
                options=[
                    ('grpc.keepalive_time_ms', 30000),
                    ('grpc.keepalive_timeout_ms', 5000),
                    ('grpc.keepalive_permit_without_calls', True),
                    ('grpc.http2.max_pings_without_data', 0),
                    ('grpc.http2.min_time_between_pings_ms', 10000),
                    ('grpc.http2.min_ping_interval_without_data_ms', 300000)
                ]
            )

        return self.channels[service_name]

    async def close_all(self):
        """關閉所有連接"""
        for channel in self.channels.values():
            await channel.close()
```

### 異步通信：事件驅動架構

```python
# events/event_bus.py
from abc import ABC, abstractmethod
from typing import Any, List, Callable, Dict
import asyncio
import json
from dataclasses import dataclass
from datetime import datetime

@dataclass
class Event:
    """基礎事件類別"""
    event_id: str
    event_type: str
    aggregate_id: str
    payload: Dict[str, Any]
    timestamp: datetime
    version: int = 1

class EventHandler(ABC):
    """事件處理器介面"""

    @abstractmethod
    async def handle(self, event: Event) -> None:
        pass

class EventBus(ABC):
    """事件匯流排介面"""

    @abstractmethod
    async def publish(self, event: Event) -> None:
        pass

    @abstractmethod
    async def subscribe(self, event_type: str, handler: EventHandler) -> None:
        pass

class InMemoryEventBus(EventBus):
    """記憶體事件匯流排實作"""

    def __init__(self):
        self.handlers: Dict[str, List[EventHandler]] = {}
        self.dead_letter_queue: List[Event] = []

    async def publish(self, event: Event) -> None:
        """發布事件"""
        logger.info("Publishing event",
                   event_type=event.event_type,
                   event_id=event.event_id)

        handlers = self.handlers.get(event.event_type, [])

        if not handlers:
            logger.warning("No handlers for event type", event_type=event.event_type)
            return

        # 並行處理所有處理器
        tasks = []
        for handler in handlers:
            task = asyncio.create_task(self._handle_event_safely(handler, event))
            tasks.append(task)

        await asyncio.gather(*tasks, return_exceptions=True)

    async def subscribe(self, event_type: str, handler: EventHandler) -> None:
        """訂閱事件"""
        if event_type not in self.handlers:
            self.handlers[event_type] = []

        self.handlers[event_type].append(handler)
        logger.info("Handler subscribed",
                   event_type=event_type,
                   handler=handler.__class__.__name__)

    async def _handle_event_safely(self, handler: EventHandler, event: Event) -> None:
        """安全地處理事件"""
        try:
            await handler.handle(event)
            logger.debug("Event handled successfully",
                        handler=handler.__class__.__name__,
                        event_id=event.event_id)

        except Exception as e:
            logger.error("Event handling failed",
                        handler=handler.__class__.__name__,
                        event_id=event.event_id,
                        error=str(e))

            # 將失敗的事件加入死信佇列
            self.dead_letter_queue.append(event)

class RabbitMQEventBus(EventBus):
    """RabbitMQ 事件匯流排實作"""

    def __init__(self, connection_url: str):
        self.connection_url = connection_url
        self.connection = None
        self.channel = None
        self.handlers = {}

    async def connect(self):
        """建立連接"""
        import aio_pika

        self.connection = await aio_pika.connect_robust(self.connection_url)
        self.channel = await self.connection.channel()

        # 設定事件交換器
        self.exchange = await self.channel.declare_exchange(
            "events",
            aio_pika.ExchangeType.TOPIC,
            durable=True
        )

    async def publish(self, event: Event) -> None:
        """發布事件到 RabbitMQ"""
        if not self.channel:
            await self.connect()

        message_body = json.dumps({
            "event_id": event.event_id,
            "event_type": event.event_type,
            "aggregate_id": event.aggregate_id,
            "payload": event.payload,
            "timestamp": event.timestamp.isoformat(),
            "version": event.version
        })

        message = aio_pika.Message(
            message_body.encode(),
            content_type="application/json",
            headers={
                "event_type": event.event_type,
                "event_id": event.event_id
            }
        )

        await self.exchange.publish(
            message,
            routing_key=event.event_type
        )

        logger.info("Event published to RabbitMQ",
                   event_type=event.event_type,
                   event_id=event.event_id)

    async def subscribe(self, event_type: str, handler: EventHandler) -> None:
        """訂閱事件"""
        if not self.channel:
            await self.connect()

        # 建立專用佇列
        queue_name = f"{handler.__class__.__name__}_{event_type}"
        queue = await self.channel.declare_queue(
            queue_name,
            durable=True
        )

        # 綁定佇列到交換器
        await queue.bind(self.exchange, routing_key=event_type)

        # 設定消費者
        async def message_handler(message: aio_pika.IncomingMessage):
            async with message.process():
                try:
                    event_data = json.loads(message.body.decode())
                    event = Event(
                        event_id=event_data["event_id"],
                        event_type=event_data["event_type"],
                        aggregate_id=event_data["aggregate_id"],
                        payload=event_data["payload"],
                        timestamp=datetime.fromisoformat(event_data["timestamp"]),
                        version=event_data["version"]
                    )

                    await handler.handle(event)

                except Exception as e:
                    logger.error("Message processing failed",
                               queue=queue_name,
                               error=str(e))
                    raise  # 重新拋出異常以觸發消息重新佇列

        await queue.consume(message_handler)

        logger.info("Subscribed to event",
                   event_type=event_type,
                   queue=queue_name,
                   handler=handler.__class__.__name__)

# 事件處理器範例
class UserRegisteredHandler(EventHandler):
    """使用者註冊事件處理器"""

    def __init__(self, notification_service: 'NotificationService'):
        self.notification_service = notification_service

    async def handle(self, event: Event) -> None:
        """處理使用者註冊事件"""
        if event.event_type != "UserRegistered":
            return

        user_data = event.payload

        # 發送歡迎郵件
        await self.notification_service.send_welcome_email(
            email=user_data["email"],
            username=user_data["username"]
        )

        logger.info("Welcome email sent for new user",
                   user_id=event.aggregate_id,
                   email=user_data["email"])

class UserEmailChangedHandler(EventHandler):
    """使用者郵箱變更事件處理器"""

    def __init__(self, audit_service: 'AuditService'):
        self.audit_service = audit_service

    async def handle(self, event: Event) -> None:
        """處理使用者郵箱變更事件"""
        if event.event_type != "UserEmailChanged":
            return

        # 記錄審計日誌
        await self.audit_service.log_security_event(
            user_id=event.aggregate_id,
            event_type="email_change",
            details={
                "old_email": event.payload["old_email"],
                "new_email": event.payload["new_email"],
                "timestamp": event.timestamp
            }
        )
```

---

## 🔄 資料一致性策略

### Saga 模式實作

```python
# saga/saga_manager.py
from abc import ABC, abstractmethod
from typing import List, Dict, Any, Optional
from dataclasses import dataclass, field
from datetime import datetime
from enum import Enum
import uuid

class SagaStatus(Enum):
    PENDING = "pending"
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"
    COMPENSATING = "compensating"

class StepStatus(Enum):
    PENDING = "pending"
    COMPLETED = "completed"
    FAILED = "failed"
    COMPENSATED = "compensated"

@dataclass
class SagaStep:
    """Saga 步驟"""
    step_id: str
    service: str
    action: str
    payload: Dict[str, Any]
    compensation_action: Optional[str] = None
    status: StepStatus = StepStatus.PENDING
    result: Optional[Dict[str, Any]] = None
    error: Optional[str] = None
    executed_at: Optional[datetime] = None

@dataclass
class SagaTransaction:
    """Saga 交易"""
    saga_id: str
    saga_type: str
    status: SagaStatus
    steps: List[SagaStep]
    created_at: datetime
    updated_at: datetime
    context: Dict[str, Any] = field(default_factory=dict)

class SagaStep(ABC):
    """Saga 步驟抽象基類"""

    @abstractmethod
    async def execute(self, context: Dict[str, Any]) -> Dict[str, Any]:
        """執行步驟"""
        pass

    @abstractmethod
    async def compensate(self, context: Dict[str, Any]) -> None:
        """補償步驟"""
        pass

class SagaOrchestrator:
    """Saga 協調器"""

    def __init__(self, saga_repository: 'SagaRepository', service_clients: Dict[str, Any]):
        self.saga_repository = saga_repository
        self.service_clients = service_clients

    async def execute_saga(self, saga: SagaTransaction) -> SagaTransaction:
        """執行 Saga 交易"""
        saga.status = SagaStatus.RUNNING
        saga.updated_at = datetime.utcnow()
        await self.saga_repository.save(saga)

        try:
            # 順序執行所有步驟
            for step in saga.steps:
                await self._execute_step(saga, step)

                if step.status == StepStatus.FAILED:
                    # 步驟失敗，開始補償流程
                    await self._compensate_saga(saga, step)
                    saga.status = SagaStatus.FAILED
                    break
            else:
                # 所有步驟成功完成
                saga.status = SagaStatus.COMPLETED

        except Exception as e:
            logger.error("Saga execution failed", saga_id=saga.saga_id, error=str(e))
            await self._compensate_saga(saga)
            saga.status = SagaStatus.FAILED

        saga.updated_at = datetime.utcnow()
        await self.saga_repository.save(saga)
        return saga

    async def _execute_step(self, saga: SagaTransaction, step: SagaStep) -> None:
        """執行單個步驟"""
        try:
            logger.info("Executing saga step",
                       saga_id=saga.saga_id,
                       step_id=step.step_id,
                       service=step.service,
                       action=step.action)

            # 調用服務執行步驟
            service_client = self.service_clients[step.service]
            result = await service_client.execute_action(step.action, step.payload)

            step.result = result
            step.status = StepStatus.COMPLETED
            step.executed_at = datetime.utcnow()

            # 更新 Saga 上下文
            saga.context.update(result)

        except Exception as e:
            logger.error("Saga step failed",
                        saga_id=saga.saga_id,
                        step_id=step.step_id,
                        error=str(e))

            step.status = StepStatus.FAILED
            step.error = str(e)

    async def _compensate_saga(self, saga: SagaTransaction, failed_step: Optional[SagaStep] = None) -> None:
        """執行 Saga 補償"""
        saga.status = SagaStatus.COMPENSATING
        await self.saga_repository.save(saga)

        # 反向執行補償操作
        completed_steps = [s for s in saga.steps if s.status == StepStatus.COMPLETED]
        for step in reversed(completed_steps):
            if step.compensation_action:
                await self._compensate_step(saga, step)

    async def _compensate_step(self, saga: SagaTransaction, step: SagaStep) -> None:
        """執行步驟補償"""
        try:
            logger.info("Compensating saga step",
                       saga_id=saga.saga_id,
                       step_id=step.step_id,
                       compensation_action=step.compensation_action)

            service_client = self.service_clients[step.service]
            await service_client.execute_action(step.compensation_action, step.result)

            step.status = StepStatus.COMPENSATED

        except Exception as e:
            logger.error("Saga step compensation failed",
                        saga_id=saga.saga_id,
                        step_id=step.step_id,
                        error=str(e))

# 具體 Saga 實作範例
class UserRegistrationSaga:
    """使用者註冊 Saga"""

    def __init__(self, orchestrator: SagaOrchestrator):
        self.orchestrator = orchestrator

    async def execute(self, username: str, email: str, profile_data: Dict[str, Any]) -> SagaTransaction:
        """執行使用者註冊 Saga"""
        saga_id = str(uuid.uuid4())

        steps = [
            SagaStep(
                step_id="create_user",
                service="user-service",
                action="create_user",
                payload={"username": username, "email": email},
                compensation_action="delete_user"
            ),
            SagaStep(
                step_id="create_profile",
                service="profile-service",
                action="create_profile",
                payload={"user_id": "${user_id}", "profile_data": profile_data},
                compensation_action="delete_profile"
            ),
            SagaStep(
                step_id="send_welcome_email",
                service="notification-service",
                action="send_welcome_email",
                payload={"user_id": "${user_id}", "email": email},
                compensation_action=None  # 郵件發送不需要補償
            ),
            SagaStep(
                step_id="create_auth_record",
                service="auth-service",
                action="create_auth_record",
                payload={"user_id": "${user_id}"},
                compensation_action="delete_auth_record"
            )
        ]

        saga = SagaTransaction(
            saga_id=saga_id,
            saga_type="user_registration",
            status=SagaStatus.PENDING,
            steps=steps,
            created_at=datetime.utcnow(),
            updated_at=datetime.utcnow()
        )

        return await self.orchestrator.execute_saga(saga)

class OrderProcessingSaga:
    """訂單處理 Saga 範例"""

    async def execute(self, order_data: Dict[str, Any]) -> SagaTransaction:
        """執行訂單處理 Saga"""
        steps = [
            SagaStep(
                step_id="validate_inventory",
                service="inventory-service",
                action="reserve_items",
                payload={"items": order_data["items"]},
                compensation_action="release_items"
            ),
            SagaStep(
                step_id="process_payment",
                service="payment-service",
                action="charge_payment",
                payload={"amount": order_data["total"], "payment_method": order_data["payment"]},
                compensation_action="refund_payment"
            ),
            SagaStep(
                step_id="create_order",
                service="order-service",
                action="create_order",
                payload=order_data,
                compensation_action="cancel_order"
            ),
            SagaStep(
                step_id="schedule_shipment",
                service="shipping-service",
                action="schedule_shipment",
                payload={"order_id": "${order_id}"},
                compensation_action="cancel_shipment"
            )
        ]

        saga = SagaTransaction(
            saga_id=str(uuid.uuid4()),
            saga_type="order_processing",
            status=SagaStatus.PENDING,
            steps=steps,
            created_at=datetime.utcnow(),
            updated_at=datetime.utcnow()
        )

        return await self.orchestrator.execute_saga(saga)
```

### 事件溯源 (Event Sourcing)

```python
# event_sourcing/event_store.py
from abc import ABC, abstractmethod
from typing import List, Optional, Any, Dict
from dataclasses import dataclass
from datetime import datetime
import json

@dataclass
class StoredEvent:
    """儲存的事件"""
    event_id: str
    aggregate_id: str
    event_type: str
    event_data: Dict[str, Any]
    event_version: int
    timestamp: datetime
    metadata: Optional[Dict[str, Any]] = None

class EventStore(ABC):
    """事件儲存介面"""

    @abstractmethod
    async def append_events(self, aggregate_id: str, events: List[StoredEvent],
                          expected_version: int) -> None:
        pass

    @abstractmethod
    async def get_events(self, aggregate_id: str, from_version: int = 0) -> List[StoredEvent]:
        pass

    @abstractmethod
    async def get_all_events(self, from_timestamp: Optional[datetime] = None) -> List[StoredEvent]:
        pass

class PostgreSQLEventStore(EventStore):
    """PostgreSQL 事件儲存實作"""

    def __init__(self, db_session):
        self.db_session = db_session

    async def append_events(self, aggregate_id: str, events: List[StoredEvent],
                          expected_version: int) -> None:
        """追加事件到事件流"""
        async with self.db_session.begin():
            # 檢查併發衝突
            current_version = await self._get_current_version(aggregate_id)

            if current_version != expected_version:
                raise OptimisticConcurrencyError(
                    f"Expected version {expected_version}, but current is {current_version}"
                )

            # 儲存事件
            for event in events:
                await self._store_event(event)

    async def get_events(self, aggregate_id: str, from_version: int = 0) -> List[StoredEvent]:
        """取得聚合的事件流"""
        query = """
        SELECT event_id, aggregate_id, event_type, event_data,
               event_version, timestamp, metadata
        FROM events
        WHERE aggregate_id = :aggregate_id AND event_version > :from_version
        ORDER BY event_version
        """

        result = await self.db_session.execute(
            text(query),
            {"aggregate_id": aggregate_id, "from_version": from_version}
        )

        return [self._to_stored_event(row) for row in result.fetchall()]

    async def _get_current_version(self, aggregate_id: str) -> int:
        """取得聚合的當前版本"""
        query = """
        SELECT COALESCE(MAX(event_version), 0) as version
        FROM events
        WHERE aggregate_id = :aggregate_id
        """

        result = await self.db_session.execute(
            text(query),
            {"aggregate_id": aggregate_id}
        )

        return result.scalar()

    def _to_stored_event(self, row) -> StoredEvent:
        """轉換資料庫行為 StoredEvent"""
        return StoredEvent(
            event_id=row.event_id,
            aggregate_id=row.aggregate_id,
            event_type=row.event_type,
            event_data=json.loads(row.event_data) if isinstance(row.event_data, str) else row.event_data,
            event_version=row.event_version,
            timestamp=row.timestamp,
            metadata=json.loads(row.metadata) if row.metadata else None
        )

class AggregateRoot:
    """聚合根基類"""

    def __init__(self, aggregate_id: str):
        self.aggregate_id = aggregate_id
        self.version = 0
        self.uncommitted_events: List[StoredEvent] = []

    def apply_event(self, event: StoredEvent) -> None:
        """應用事件到聚合"""
        self.version = event.event_version
        self._when(event)

    def raise_event(self, event_type: str, event_data: Dict[str, Any]) -> None:
        """產生新事件"""
        self.version += 1

        event = StoredEvent(
            event_id=str(uuid.uuid4()),
            aggregate_id=self.aggregate_id,
            event_type=event_type,
            event_data=event_data,
            event_version=self.version,
            timestamp=datetime.utcnow()
        )

        self.uncommitted_events.append(event)
        self.apply_event(event)

    def mark_events_as_committed(self) -> None:
        """標記事件為已提交"""
        self.uncommitted_events.clear()

    def get_uncommitted_events(self) -> List[StoredEvent]:
        """取得未提交的事件"""
        return self.uncommitted_events.copy()

    @abstractmethod
    def _when(self, event: StoredEvent) -> None:
        """處理事件（子類別實作）"""
        pass

class User(AggregateRoot):
    """使用者聚合"""

    def __init__(self, user_id: str):
        super().__init__(user_id)
        self.username = ""
        self.email = ""
        self.is_active = True
        self.created_at = None

    @classmethod
    def create(cls, user_id: str, username: str, email: str) -> 'User':
        """建立新使用者"""
        user = cls(user_id)
        user.raise_event("UserCreated", {
            "username": username,
            "email": email,
            "created_at": datetime.utcnow().isoformat()
        })
        return user

    def change_email(self, new_email: str) -> None:
        """變更郵箱"""
        if self.email == new_email:
            return

        old_email = self.email
        self.raise_event("EmailChanged", {
            "old_email": old_email,
            "new_email": new_email,
            "changed_at": datetime.utcnow().isoformat()
        })

    def deactivate(self) -> None:
        """停用使用者"""
        if not self.is_active:
            return

        self.raise_event("UserDeactivated", {
            "deactivated_at": datetime.utcnow().isoformat()
        })

    def _when(self, event: StoredEvent) -> None:
        """處理事件"""
        if event.event_type == "UserCreated":
            self._when_user_created(event.event_data)
        elif event.event_type == "EmailChanged":
            self._when_email_changed(event.event_data)
        elif event.event_type == "UserDeactivated":
            self._when_user_deactivated(event.event_data)

    def _when_user_created(self, data: Dict[str, Any]) -> None:
        self.username = data["username"]
        self.email = data["email"]
        self.created_at = datetime.fromisoformat(data["created_at"])
        self.is_active = True

    def _when_email_changed(self, data: Dict[str, Any]) -> None:
        self.email = data["new_email"]

    def _when_user_deactivated(self, data: Dict[str, Any]) -> None:
        self.is_active = False

class Repository:
    """事件溯源倉庫"""

    def __init__(self, event_store: EventStore):
        self.event_store = event_store

    async def get_by_id(self, aggregate_class: type, aggregate_id: str) -> Optional[AggregateRoot]:
        """根據 ID 取得聚合"""
        events = await self.event_store.get_events(aggregate_id)

        if not events:
            return None

        aggregate = aggregate_class(aggregate_id)

        for event in events:
            aggregate.apply_event(event)

        return aggregate

    async def save(self, aggregate: AggregateRoot) -> None:
        """儲存聚合"""
        uncommitted_events = aggregate.get_uncommitted_events()

        if not uncommitted_events:
            return

        expected_version = aggregate.version - len(uncommitted_events)

        await self.event_store.append_events(
            aggregate.aggregate_id,
            uncommitted_events,
            expected_version
        )

        aggregate.mark_events_as_committed()

# 讀取模型投影
class UserProjection:
    """使用者讀取模型投影"""

    def __init__(self, event_store: EventStore, read_model_store):
        self.event_store = event_store
        self.read_model_store = read_model_store

    async def project_user_events(self) -> None:
        """投影使用者事件到讀取模型"""
        last_processed_timestamp = await self.read_model_store.get_last_processed_timestamp()
        events = await self.event_store.get_all_events(from_timestamp=last_processed_timestamp)

        for event in events:
            if event.event_type in ["UserCreated", "EmailChanged", "UserDeactivated"]:
                await self._update_user_read_model(event)

        if events:
            await self.read_model_store.update_last_processed_timestamp(events[-1].timestamp)

    async def _update_user_read_model(self, event: StoredEvent) -> None:
        """更新使用者讀取模型"""
        if event.event_type == "UserCreated":
            await self.read_model_store.create_user({
                "user_id": event.aggregate_id,
                "username": event.event_data["username"],
                "email": event.event_data["email"],
                "is_active": True,
                "created_at": event.event_data["created_at"]
            })
        elif event.event_type == "EmailChanged":
            await self.read_model_store.update_user_email(
                event.aggregate_id,
                event.event_data["new_email"]
            )
        elif event.event_type == "UserDeactivated":
            await self.read_model_store.deactivate_user(event.aggregate_id)
```

---

## 🔧 服務治理與配置

### 服務發現與註冊

```python
# service_discovery/consul_client.py
import consul.aio
from typing import List, Dict, Optional, Any
from dataclasses import dataclass
import asyncio

@dataclass
class ServiceInstance:
    """服務實例"""
    service_name: str
    instance_id: str
    host: str
    port: int
    tags: List[str]
    health_check_url: str
    metadata: Dict[str, Any]

class ConsulServiceDiscovery:
    """Consul 服務發現客戶端"""

    def __init__(self, consul_host: str = "localhost", consul_port: int = 8500):
        self.consul = consul.aio.Consul(host=consul_host, port=consul_port)
        self.registered_services = set()

    async def register_service(self, instance: ServiceInstance) -> None:
        """註冊服務實例"""
        check = consul.Check.http(instance.health_check_url, timeout="10s", interval="30s")

        await self.consul.agent.service.register(
            name=instance.service_name,
            service_id=instance.instance_id,
            address=instance.host,
            port=instance.port,
            tags=instance.tags,
            check=check,
            meta=instance.metadata
        )

        self.registered_services.add(instance.instance_id)

        logger.info("Service registered",
                   service=instance.service_name,
                   instance_id=instance.instance_id,
                   address=f"{instance.host}:{instance.port}")

    async def deregister_service(self, instance_id: str) -> None:
        """取消註冊服務實例"""
        await self.consul.agent.service.deregister(instance_id)
        self.registered_services.discard(instance_id)

        logger.info("Service deregistered", instance_id=instance_id)

    async def discover_services(self, service_name: str) -> List[ServiceInstance]:
        """發現服務實例"""
        _, services = await self.consul.health.service(service_name, passing=True)

        instances = []
        for service in services:
            service_info = service["Service"]
            instances.append(ServiceInstance(
                service_name=service_info["Service"],
                instance_id=service_info["ID"],
                host=service_info["Address"],
                port=service_info["Port"],
                tags=service_info["Tags"],
                health_check_url=f"http://{service_info['Address']}:{service_info['Port']}/health",
                metadata=service_info.get("Meta", {})
            ))

        return instances

    async def watch_service(self, service_name: str, callback):
        """監控服務變更"""
        index = None

        while True:
            try:
                index, services = await self.consul.health.service(
                    service_name,
                    index=index,
                    wait="30s"
                )

                instances = [
                    ServiceInstance(
                        service_name=s["Service"]["Service"],
                        instance_id=s["Service"]["ID"],
                        host=s["Service"]["Address"],
                        port=s["Service"]["Port"],
                        tags=s["Service"]["Tags"],
                        health_check_url=f"http://{s['Service']['Address']}:{s['Service']['Port']}/health",
                        metadata=s["Service"].get("Meta", {})
                    )
                    for s in services
                ]

                await callback(instances)

            except Exception as e:
                logger.error("Service watch error", service=service_name, error=str(e))
                await asyncio.sleep(10)

    async def cleanup(self) -> None:
        """清理註冊的服務"""
        for instance_id in list(self.registered_services):
            await self.deregister_service(instance_id)

# configuration/config_manager.py
class ConfigurationManager:
    """配置管理器"""

    def __init__(self, consul_client: ConsulServiceDiscovery):
        self.consul = consul_client
        self.config_cache = {}
        self.watchers = {}

    async def get_config(self, key: str, default: Any = None) -> Any:
        """取得配置值"""
        if key in self.config_cache:
            return self.config_cache[key]

        _, data = await self.consul.consul.kv.get(key)

        if data is None:
            return default

        value = data["Value"].decode("utf-8")

        # 嘗試解析 JSON
        try:
            import json
            value = json.loads(value)
        except json.JSONDecodeError:
            pass  # 保持字串格式

        self.config_cache[key] = value
        return value

    async def set_config(self, key: str, value: Any) -> None:
        """設定配置值"""
        if not isinstance(value, str):
            import json
            value = json.dumps(value)

        await self.consul.consul.kv.put(key, value)
        self.config_cache[key] = value

    async def watch_config(self, key: str, callback) -> None:
        """監控配置變更"""
        async def config_watcher():
            index = None

            while True:
                try:
                    index, data = await self.consul.consul.kv.get(
                        key,
                        index=index,
                        wait="30s"
                    )

                    if data:
                        value = data["Value"].decode("utf-8")
                        try:
                            import json
                            value = json.loads(value)
                        except json.JSONDecodeError:
                            pass

                        self.config_cache[key] = value
                        await callback(key, value)

                except Exception as e:
                    logger.error("Config watch error", key=key, error=str(e))
                    await asyncio.sleep(10)

        if key not in self.watchers:
            self.watchers[key] = asyncio.create_task(config_watcher())

class CircuitBreakerConfig:
    """熔斷器配置"""

    def __init__(self, config_manager: ConfigurationManager):
        self.config_manager = config_manager
        self.circuit_breakers = {}

    async def get_circuit_breaker_config(self, service_name: str) -> Dict[str, Any]:
        """取得熔斷器配置"""
        config_key = f"circuit_breakers/{service_name}"

        default_config = {
            "failure_threshold": 5,
            "recovery_timeout": 30,
            "timeout": 10,
            "expected_exception": "RequestError"
        }

        return await self.config_manager.get_config(config_key, default_config)

    async def update_circuit_breaker_config(self, service_name: str, config: Dict[str, Any]) -> None:
        """更新熔斷器配置"""
        config_key = f"circuit_breakers/{service_name}"
        await self.config_manager.set_config(config_key, config)
```

---

## 🔍 微服務監控與追蹤

### 分散式追蹤

```python
# tracing/distributed_tracing.py
from opentelemetry import trace, context, baggage
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.instrumentation.asyncio import AsyncioInstrumentor
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from typing import Dict, Any, Optional
import structlog

logger = structlog.get_logger()

class DistributedTracingManager:
    """分散式追蹤管理器"""

    def __init__(self, service_name: str, jaeger_endpoint: str = "http://localhost:14268/api/traces"):
        self.service_name = service_name
        self.tracer_provider = TracerProvider()
        self.tracer = self.tracer_provider.get_tracer(service_name)

        # 設定 Jaeger 導出器
        jaeger_exporter = JaegerExporter(endpoint=jaeger_endpoint)
        span_processor = BatchSpanProcessor(jaeger_exporter)
        self.tracer_provider.add_span_processor(span_processor)

        # 設定全域 tracer
        trace.set_tracer_provider(self.tracer_provider)

    def setup_auto_instrumentation(self, app):
        """設定自動儀錶化"""
        # FastAPI 自動儀錶化
        FastAPIInstrumentor.instrument_app(app)

        # HTTP 請求自動儀錶化
        RequestsInstrumentor().instrument()

        # Asyncio 自動儀錶化
        AsyncioInstrumentor().instrument()

    def start_span(self, operation_name: str, parent_context=None) -> trace.Span:
        """開始新的 span"""
        ctx = parent_context or context.get_current()
        return self.tracer.start_span(operation_name, context=ctx)

    def trace_operation(self, operation_name: str):
        """操作追蹤裝飾器"""
        def decorator(func):
            @wraps(func)
            async def wrapper(*args, **kwargs):
                with self.start_span(operation_name) as span:
                    try:
                        # 設定 span 屬性
                        span.set_attribute("operation.name", operation_name)
                        span.set_attribute("service.name", self.service_name)

                        if asyncio.iscoroutinefunction(func):
                            result = await func(*args, **kwargs)
                        else:
                            result = func(*args, **kwargs)

                        span.set_attribute("operation.status", "success")
                        return result

                    except Exception as e:
                        span.set_attribute("operation.status", "error")
                        span.set_attribute("error.message", str(e))
                        span.record_exception(e)
                        raise

            return wrapper
        return decorator

    def add_baggage(self, key: str, value: str) -> None:
        """添加 baggage"""
        baggage.set_baggage(key, value)

    def get_baggage(self, key: str) -> Optional[str]:
        """取得 baggage"""
        return baggage.get_baggage(key)

class ServiceMeshTracking:
    """服務網格追蹤"""

    def __init__(self, tracing_manager: DistributedTracingManager):
        self.tracing_manager = tracing_manager
        self.service_dependencies = {}

    async def track_service_call(self, target_service: str, operation: str,
                                request_data: Dict[str, Any]) -> None:
        """追蹤服務調用"""
        with self.tracing_manager.start_span(f"call.{target_service}.{operation}") as span:
            # 記錄請求詳情
            span.set_attribute("target.service", target_service)
            span.set_attribute("target.operation", operation)
            span.set_attribute("request.size", len(str(request_data)))

            # 記錄服務依賴關係
            self._record_dependency(target_service)

            # 傳播追蹤上下文
            trace_headers = self._extract_trace_headers()

            return trace_headers

    def _extract_trace_headers(self) -> Dict[str, str]:
        """提取追蹤標頭"""
        current_span = trace.get_current_span()
        span_context = current_span.get_span_context()

        return {
            "X-Trace-ID": f"{span_context.trace_id:032x}",
            "X-Span-ID": f"{span_context.span_id:016x}",
            "X-Service-Name": self.tracing_manager.service_name
        }

    def _record_dependency(self, target_service: str) -> None:
        """記錄服務依賴"""
        if target_service not in self.service_dependencies:
            self.service_dependencies[target_service] = {
                "call_count": 0,
                "first_seen": datetime.utcnow(),
                "last_seen": datetime.utcnow()
            }

        dep = self.service_dependencies[target_service]
        dep["call_count"] += 1
        dep["last_seen"] = datetime.utcnow()

    def get_service_map(self) -> Dict[str, Any]:
        """取得服務地圖"""
        return {
            "service_name": self.tracing_manager.service_name,
            "dependencies": self.service_dependencies,
            "generated_at": datetime.utcnow().isoformat()
        }

class BusinessMetricsCollector:
    """業務指標收集器"""

    def __init__(self, tracing_manager: DistributedTracingManager):
        self.tracing_manager = tracing_manager
        self.business_events = []

    def track_business_event(self, event_type: str, user_id: str = None,
                           metadata: Dict[str, Any] = None) -> None:
        """追蹤業務事件"""
        with self.tracing_manager.start_span(f"business.{event_type}") as span:
            span.set_attribute("business.event_type", event_type)

            if user_id:
                span.set_attribute("user.id", user_id)

            if metadata:
                for key, value in metadata.items():
                    span.set_attribute(f"business.{key}", str(value))

            # 記錄業務事件
            event = {
                "event_type": event_type,
                "user_id": user_id,
                "timestamp": datetime.utcnow().isoformat(),
                "metadata": metadata or {},
                "trace_id": f"{trace.get_current_span().get_span_context().trace_id:032x}"
            }

            self.business_events.append(event)

    def get_business_metrics(self) -> Dict[str, Any]:
        """取得業務指標"""
        event_counts = {}
        for event in self.business_events:
            event_type = event["event_type"]
            event_counts[event_type] = event_counts.get(event_type, 0) + 1

        return {
            "total_events": len(self.business_events),
            "event_counts": event_counts,
            "time_range": {
                "start": self.business_events[0]["timestamp"] if self.business_events else None,
                "end": self.business_events[-1]["timestamp"] if self.business_events else None
            }
        }

# 使用範例
class TracedUserService:
    """帶追蹤的使用者服務"""

    def __init__(self):
        self.tracing_manager = DistributedTracingManager("user-service")
        self.mesh_tracking = ServiceMeshTracking(self.tracing_manager)
        self.business_metrics = BusinessMetricsCollector(self.tracing_manager)

    @trace_operation("create_user")
    async def create_user(self, user_data: Dict[str, Any]) -> str:
        """建立使用者"""
        # 追蹤業務事件
        self.business_metrics.track_business_event(
            "user_registration",
            metadata={"username": user_data.get("username")}
        )

        # 調用其他服務
        trace_headers = await self.mesh_tracking.track_service_call(
            "notification-service",
            "send_welcome_email",
            {"email": user_data["email"]}
        )

        # 實際的使用者建立邏輯...
        user_id = str(uuid.uuid4())

        # 記錄成功事件
        self.business_metrics.track_business_event(
            "user_created",
            user_id=user_id
        )

        return user_id
```

---

## 💡 學習筆記區

### 🤔 我的理解
```
微服務架構的設計原則：

領域驅動設計的核心概念：

服務間通信的選擇考量：

資料一致性的處理策略：
```

### 🏗️ 實踐心得
```
微服務分解的挑戰：

分散式系統的複雜性：

服務治理的重要性：

監控追蹤的必要性：
```

### 🚀 進階思考
```
微服務 vs 單體架構的取捨：

雲原生微服務的演進：

Service Mesh 的價值：

微服務架構的未來趨勢：
```