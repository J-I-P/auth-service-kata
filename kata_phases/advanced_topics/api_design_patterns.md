# API Design Patterns: 企業級 RESTful API 設計與最佳實務

## 🎯 學習目標
- 掌握現代 RESTful API 設計原則與模式
- 學習 API 版本控制、分頁、過濾等進階技術
- 實踐 API 安全、錯誤處理與文檔最佳實務
- 建立可維護、可擴展的 API 架構

---

## 🏗️ RESTful API 設計原則

### 資源導向設計 (Resource-Oriented Design)

```python
# api/design_patterns/resource_design.py
from typing import List, Optional, Dict, Any
from pydantic import BaseModel, Field
from fastapi import APIRouter, Depends, Query, Path, HTTPException
from datetime import datetime
import uuid

# 資源模型設計
class UserResource(BaseModel):
    """使用者資源模型"""
    id: str
    username: str
    email: str
    display_name: str
    created_at: datetime
    updated_at: datetime
    is_active: bool
    profile_url: str = Field(..., description="Profile resource URL")

    class Config:
        schema_extra = {
            "example": {
                "id": "550e8400-e29b-41d4-a716-446655440000",
                "username": "johndoe",
                "email": "john@example.com",
                "display_name": "John Doe",
                "created_at": "2024-01-06T10:00:00Z",
                "updated_at": "2024-01-06T10:00:00Z",
                "is_active": True,
                "profile_url": "/api/v1/users/550e8400-e29b-41d4-a716-446655440000/profile"
            }
        }

class UserProfileResource(BaseModel):
    """使用者個人資料資源"""
    user_id: str
    first_name: Optional[str] = None
    last_name: Optional[str] = None
    bio: Optional[str] = None
    avatar_url: Optional[str] = None
    location: Optional[str] = None
    website: Optional[str] = None
    birth_date: Optional[datetime] = None

# 資源集合設計
class ResourceCollection(BaseModel):
    """資源集合的標準格式"""
    items: List[Dict[str, Any]]
    total_count: int
    page: int
    per_page: int
    has_next: bool
    has_previous: bool
    next_url: Optional[str] = None
    previous_url: Optional[str] = None

    @classmethod
    def create(cls, items: List[Any], page: int, per_page: int, total_count: int,
               base_url: str) -> "ResourceCollection":
        """建立資源集合"""
        has_next = (page * per_page) < total_count
        has_previous = page > 1

        next_url = f"{base_url}?page={page + 1}&per_page={per_page}" if has_next else None
        previous_url = f"{base_url}?page={page - 1}&per_page={per_page}" if has_previous else None

        return cls(
            items=items,
            total_count=total_count,
            page=page,
            per_page=per_page,
            has_next=has_next,
            has_previous=has_previous,
            next_url=next_url,
            previous_url=previous_url
        )

# RESTful 路由設計
class RESTfulUsersAPI:
    """RESTful 使用者 API 設計"""

    def __init__(self):
        self.router = APIRouter(prefix="/api/v1/users", tags=["users"])
        self._setup_routes()

    def _setup_routes(self):
        """設定 RESTful 路由"""

        # Collection endpoints
        @self.router.get("/", response_model=ResourceCollection,
                        summary="List users",
                        description="Retrieve a paginated list of users with optional filtering")
        async def list_users(
            page: int = Query(1, ge=1, description="Page number"),
            per_page: int = Query(20, ge=1, le=100, description="Items per page"),
            search: Optional[str] = Query(None, description="Search term"),
            status: Optional[str] = Query(None, enum=["active", "inactive"], description="User status filter"),
            sort: str = Query("created_at", enum=["created_at", "username", "email"], description="Sort field"),
            order: str = Query("desc", enum=["asc", "desc"], description="Sort order")
        ):
            # 實作分頁查詢邏輯
            return await self._get_users_collection(page, per_page, search, status, sort, order)

        @self.router.post("/", response_model=UserResource, status_code=201,
                         summary="Create user",
                         description="Create a new user resource")
        async def create_user(user_data: CreateUserRequest):
            # 實作使用者建立邏輯
            return await self._create_user(user_data)

        # Resource endpoints
        @self.router.get("/{user_id}", response_model=UserResource,
                        summary="Get user",
                        description="Retrieve a specific user by ID")
        async def get_user(
            user_id: str = Path(..., description="User ID")
        ):
            # 實作使用者查詢邏輯
            return await self._get_user(user_id)

        @self.router.put("/{user_id}", response_model=UserResource,
                        summary="Update user",
                        description="Update an existing user resource")
        async def update_user(
            user_id: str = Path(..., description="User ID"),
            user_data: UpdateUserRequest
        ):
            # 實作使用者更新邏輯
            return await self._update_user(user_id, user_data)

        @self.router.patch("/{user_id}", response_model=UserResource,
                          summary="Partially update user",
                          description="Partially update user fields")
        async def patch_user(
            user_id: str = Path(..., description="User ID"),
            user_data: PatchUserRequest
        ):
            # 實作部分更新邏輯
            return await self._patch_user(user_id, user_data)

        @self.router.delete("/{user_id}", status_code=204,
                           summary="Delete user",
                           description="Delete a user resource")
        async def delete_user(
            user_id: str = Path(..., description="User ID")
        ):
            # 實作使用者刪除邏輯
            return await self._delete_user(user_id)

        # Sub-resource endpoints
        @self.router.get("/{user_id}/profile", response_model=UserProfileResource,
                        summary="Get user profile",
                        description="Retrieve user profile information")
        async def get_user_profile(
            user_id: str = Path(..., description="User ID")
        ):
            return await self._get_user_profile(user_id)

        @self.router.put("/{user_id}/profile", response_model=UserProfileResource,
                        summary="Update user profile",
                        description="Update user profile information")
        async def update_user_profile(
            user_id: str = Path(..., description="User ID"),
            profile_data: UpdateProfileRequest
        ):
            return await self._update_user_profile(user_id, profile_data)

        # Action endpoints (non-standard RESTful operations)
        @self.router.post("/{user_id}/activate", status_code=200,
                         summary="Activate user",
                         description="Activate a user account")
        async def activate_user(
            user_id: str = Path(..., description="User ID")
        ):
            return await self._activate_user(user_id)

        @self.router.post("/{user_id}/deactivate", status_code=200,
                         summary="Deactivate user",
                         description="Deactivate a user account")
        async def deactivate_user(
            user_id: str = Path(..., description="User ID")
        ):
            return await self._deactivate_user(user_id)

# HTTP 方法語義
class HTTPMethodSemantics:
    """HTTP 方法語義指南"""

    @staticmethod
    def get_method_characteristics():
        """HTTP 方法特性"""
        return {
            "GET": {
                "safe": True,      # 不會修改資源
                "idempotent": True, # 多次調用結果相同
                "cacheable": True,  # 可快取
                "has_body": False   # 不應該有請求體
            },
            "POST": {
                "safe": False,
                "idempotent": False,
                "cacheable": False,
                "has_body": True
            },
            "PUT": {
                "safe": False,
                "idempotent": True,  # 重要！PUT 應該是冪等的
                "cacheable": False,
                "has_body": True
            },
            "PATCH": {
                "safe": False,
                "idempotent": False, # PATCH 不一定是冪等的
                "cacheable": False,
                "has_body": True
            },
            "DELETE": {
                "safe": False,
                "idempotent": True,  # 刪除操作是冪等的
                "cacheable": False,
                "has_body": False
            }
        }

# 狀態碼使用指南
class HTTPStatusCodes:
    """HTTP 狀態碼使用指南"""

    @staticmethod
    def get_status_code_guidelines():
        """狀態碼使用指南"""
        return {
            # 2xx Success
            200: {
                "name": "OK",
                "usage": "GET, PUT, PATCH requests when resource is returned",
                "example": "Successfully retrieved user data"
            },
            201: {
                "name": "Created",
                "usage": "POST requests when resource is created",
                "example": "User successfully created",
                "location_header": True
            },
            202: {
                "name": "Accepted",
                "usage": "Request accepted for processing but not completed",
                "example": "Email verification request accepted"
            },
            204: {
                "name": "No Content",
                "usage": "DELETE requests, PUT/PATCH when no content returned",
                "example": "User successfully deleted"
            },

            # 3xx Redirection
            301: {
                "name": "Moved Permanently",
                "usage": "Resource has permanently moved",
                "example": "API endpoint moved to new location"
            },
            302: {
                "name": "Found",
                "usage": "Temporary redirect",
                "example": "OAuth authorization redirect"
            },
            304: {
                "name": "Not Modified",
                "usage": "Resource hasn't changed (ETag/If-Modified-Since)",
                "example": "User data not modified since last request"
            },

            # 4xx Client Error
            400: {
                "name": "Bad Request",
                "usage": "Invalid request syntax or parameters",
                "example": "Invalid JSON payload or missing required fields"
            },
            401: {
                "name": "Unauthorized",
                "usage": "Authentication required or failed",
                "example": "Invalid or missing authentication token"
            },
            403: {
                "name": "Forbidden",
                "usage": "Authenticated but not authorized",
                "example": "User doesn't have permission to access resource"
            },
            404: {
                "name": "Not Found",
                "usage": "Resource doesn't exist",
                "example": "User with specified ID not found"
            },
            405: {
                "name": "Method Not Allowed",
                "usage": "HTTP method not supported for resource",
                "example": "DELETE not allowed on user collection"
            },
            409: {
                "name": "Conflict",
                "usage": "Request conflicts with current state",
                "example": "Username already exists"
            },
            410: {
                "name": "Gone",
                "usage": "Resource existed but no longer available",
                "example": "User account permanently deleted"
            },
            422: {
                "name": "Unprocessable Entity",
                "usage": "Request is syntactically correct but semantically invalid",
                "example": "Validation errors in request data"
            },
            429: {
                "name": "Too Many Requests",
                "usage": "Rate limit exceeded",
                "example": "API rate limit exceeded"
            },

            # 5xx Server Error
            500: {
                "name": "Internal Server Error",
                "usage": "Generic server error",
                "example": "Unexpected server error occurred"
            },
            502: {
                "name": "Bad Gateway",
                "usage": "Invalid response from upstream server",
                "example": "Database connection failed"
            },
            503: {
                "name": "Service Unavailable",
                "usage": "Server temporarily unavailable",
                "example": "Server under maintenance"
            },
            504: {
                "name": "Gateway Timeout",
                "usage": "Upstream server timeout",
                "example": "Database query timeout"
            }
        }
```

---

## 🔄 API 版本控制策略

### 多版本管理與向後相容

```python
# api/versioning/version_manager.py
from enum import Enum
from typing import Dict, Any, Optional, List
from fastapi import APIRouter, Request, Depends, HTTPException
from pydantic import BaseModel
import semver

class VersioningStrategy(Enum):
    URL_PATH = "url_path"           # /api/v1/users
    HEADER = "header"               # Accept-Version: v1
    QUERY_PARAMETER = "query"       # ?version=v1
    CONTENT_NEGOTIATION = "content" # Accept: application/vnd.api.v1+json

class APIVersion:
    """API 版本定義"""

    def __init__(self, version: str, status: str = "stable"):
        self.version = version
        self.semantic_version = semver.VersionInfo.parse(version.lstrip('v'))
        self.status = status  # stable, deprecated, beta
        self.deprecation_date = None
        self.sunset_date = None

    @property
    def major(self) -> int:
        return self.semantic_version.major

    @property
    def minor(self) -> int:
        return self.semantic_version.minor

    @property
    def patch(self) -> int:
        return self.semantic_version.patch

    def is_compatible_with(self, other: "APIVersion") -> bool:
        """檢查版本相容性"""
        return (
            self.major == other.major and
            self.minor >= other.minor
        )

class VersionManager:
    """API 版本管理器"""

    def __init__(self):
        self.versions: Dict[str, APIVersion] = {}
        self.default_version = None
        self.supported_versions: List[str] = []

    def register_version(self, version: str, status: str = "stable",
                        is_default: bool = False):
        """註冊 API 版本"""
        api_version = APIVersion(version, status)
        self.versions[version] = api_version

        if status in ["stable", "beta"]:
            self.supported_versions.append(version)

        if is_default or self.default_version is None:
            self.default_version = version

    def deprecate_version(self, version: str, deprecation_date: datetime,
                         sunset_date: datetime):
        """標記版本為已棄用"""
        if version in self.versions:
            api_version = self.versions[version]
            api_version.status = "deprecated"
            api_version.deprecation_date = deprecation_date
            api_version.sunset_date = sunset_date

    def get_version_from_request(self, request: Request,
                               strategy: VersioningStrategy = VersioningStrategy.URL_PATH) -> str:
        """從請求中提取版本資訊"""
        if strategy == VersioningStrategy.URL_PATH:
            # 從 URL 路徑提取版本 /api/v1/users
            path_parts = request.url.path.split('/')
            for part in path_parts:
                if part.startswith('v') and part[1:].replace('.', '').isdigit():
                    return part

        elif strategy == VersioningStrategy.HEADER:
            # 從 Header 提取版本
            return request.headers.get("Accept-Version", self.default_version)

        elif strategy == VersioningStrategy.QUERY_PARAMETER:
            # 從查詢參數提取版本
            return request.query_params.get("version", self.default_version)

        elif strategy == VersioningStrategy.CONTENT_NEGOTIATION:
            # 從 Accept Header 提取版本
            accept_header = request.headers.get("Accept", "")
            if "application/vnd.api.v" in accept_header:
                version_part = accept_header.split("vnd.api.v")[1].split("+")[0]
                return f"v{version_part}"

        return self.default_version

    def validate_version(self, version: str) -> bool:
        """驗證版本是否支援"""
        return version in self.supported_versions

    def get_version_info(self, version: str) -> Dict[str, Any]:
        """取得版本資訊"""
        if version in self.versions:
            api_version = self.versions[version]
            return {
                "version": version,
                "status": api_version.status,
                "semantic_version": str(api_version.semantic_version),
                "deprecation_date": api_version.deprecation_date.isoformat() if api_version.deprecation_date else None,
                "sunset_date": api_version.sunset_date.isoformat() if api_version.sunset_date else None
            }
        return {}

# 版本相容性處理
class BackwardCompatibility:
    """向後相容性管理"""

    def __init__(self, version_manager: VersionManager):
        self.version_manager = version_manager
        self.field_mappings = {}
        self.transformation_rules = {}

    def register_field_mapping(self, from_version: str, to_version: str,
                              field_mappings: Dict[str, str]):
        """註冊欄位映射規則"""
        key = f"{from_version}->{to_version}"
        self.field_mappings[key] = field_mappings

    def register_transformation_rule(self, from_version: str, to_version: str,
                                   transformer_func):
        """註冊轉換規則"""
        key = f"{from_version}->{to_version}"
        self.transformation_rules[key] = transformer_func

    def transform_response(self, data: Dict[str, Any], from_version: str,
                         to_version: str) -> Dict[str, Any]:
        """轉換響應資料以保持相容性"""
        key = f"{from_version}->{to_version}"

        # 應用欄位映射
        if key in self.field_mappings:
            data = self._apply_field_mapping(data, self.field_mappings[key])

        # 應用轉換規則
        if key in self.transformation_rules:
            data = self.transformation_rules[key](data)

        return data

    def _apply_field_mapping(self, data: Dict[str, Any],
                           mappings: Dict[str, str]) -> Dict[str, Any]:
        """應用欄位映射"""
        transformed_data = data.copy()

        for old_field, new_field in mappings.items():
            if old_field in transformed_data:
                transformed_data[new_field] = transformed_data[old_field]
                # 可選：移除舊欄位或保留以相容性

        return transformed_data

# 版本化 API 路由器
class VersionedAPIRouter:
    """版本化 API 路由器"""

    def __init__(self, version_manager: VersionManager):
        self.version_manager = version_manager
        self.routers = {}

    def create_versioned_router(self, version: str, prefix: str = None) -> APIRouter:
        """建立版本化路由器"""
        if prefix is None:
            prefix = f"/api/{version}"

        router = APIRouter(
            prefix=prefix,
            tags=[f"v{version.lstrip('v')}"]
        )

        # 添加版本驗證中間件
        @router.middleware("http")
        async def validate_version_middleware(request: Request, call_next):
            version = self.version_manager.get_version_from_request(request)

            if not self.version_manager.validate_version(version):
                raise HTTPException(
                    status_code=400,
                    detail=f"Unsupported API version: {version}",
                    headers={"X-Supported-Versions": ",".join(self.version_manager.supported_versions)}
                )

            # 檢查版本狀態
            version_info = self.version_manager.get_version_info(version)
            if version_info.get("status") == "deprecated":
                # 添加棄用警告標頭
                response = await call_next(request)
                response.headers["Warning"] = f"299 - API version {version} is deprecated"
                if version_info.get("sunset_date"):
                    response.headers["Sunset"] = version_info["sunset_date"]
                return response

            return await call_next(request)

        self.routers[version] = router
        return router

    def get_router(self, version: str) -> APIRouter:
        """取得指定版本的路由器"""
        return self.routers.get(version)

# 實際使用範例
class UserAPIVersioning:
    """使用者 API 版本管理範例"""

    def __init__(self):
        self.version_manager = VersionManager()
        self.compatibility = BackwardCompatibility(self.version_manager)
        self._setup_versions()
        self._setup_compatibility_rules()

    def _setup_versions(self):
        """設定版本"""
        # 註冊支援的版本
        self.version_manager.register_version("v1", "stable", is_default=True)
        self.version_manager.register_version("v2", "stable")
        self.version_manager.register_version("v3", "beta")

        # 標記舊版本為棄用
        self.version_manager.deprecate_version(
            "v1",
            datetime(2024, 6, 1),   # 棄用日期
            datetime(2024, 12, 1)   # 停用日期
        )

    def _setup_compatibility_rules(self):
        """設定相容性規則"""
        # v1 -> v2 欄位映射
        self.compatibility.register_field_mapping("v1", "v2", {
            "username": "login_name",  # v2 中 username 改名為 login_name
            "created_at": "created_date"
        })

        # v2 -> v3 轉換規則
        def v2_to_v3_transformer(data):
            # v3 中添加了新的 metadata 欄位
            if "metadata" not in data:
                data["metadata"] = {}
            return data

        self.compatibility.register_transformation_rule("v2", "v3", v2_to_v3_transformer)

# 版本資訊端點
def create_version_info_endpoint(version_manager: VersionManager):
    """建立版本資訊端點"""
    router = APIRouter()

    @router.get("/api/versions",
               summary="Get API version information",
               description="Retrieve information about all available API versions")
    async def get_api_versions():
        """取得 API 版本資訊"""
        versions_info = []

        for version in version_manager.supported_versions:
            version_info = version_manager.get_version_info(version)
            versions_info.append(version_info)

        return {
            "default_version": version_manager.default_version,
            "supported_versions": version_manager.supported_versions,
            "versions": versions_info
        }

    return router
```

---

## 🔍 進階查詢與過濾

### 靈活的查詢介面設計

```python
# api/querying/advanced_filtering.py
from typing import List, Dict, Any, Optional, Union
from pydantic import BaseModel, Field, validator
from fastapi import Query, Depends, HTTPException
from enum import Enum
import operator
from datetime import datetime

class FilterOperator(str, Enum):
    """過濾操作符"""
    EQ = "eq"           # 等於
    NE = "ne"           # 不等於
    GT = "gt"           # 大於
    GTE = "gte"         # 大於等於
    LT = "lt"           # 小於
    LTE = "lte"         # 小於等於
    IN = "in"           # 包含在列表中
    NOT_IN = "not_in"   # 不包含在列表中
    CONTAINS = "contains"    # 包含字串
    STARTS_WITH = "starts_with"  # 開頭為
    ENDS_WITH = "ends_with"      # 結尾為
    IS_NULL = "is_null"          # 為空值
    IS_NOT_NULL = "is_not_null"  # 非空值

class SortDirection(str, Enum):
    ASC = "asc"
    DESC = "desc"

class FilterCriteria(BaseModel):
    """過濾條件"""
    field: str
    operator: FilterOperator
    value: Union[str, int, float, List[Any], None] = None

    @validator('value')
    def validate_value_for_operator(cls, v, values):
        """根據操作符驗證值"""
        operator = values.get('operator')

        if operator in [FilterOperator.IS_NULL, FilterOperator.IS_NOT_NULL]:
            return None
        elif operator in [FilterOperator.IN, FilterOperator.NOT_IN]:
            if not isinstance(v, list):
                raise ValueError(f"Operator {operator} requires a list value")
        elif v is None:
            raise ValueError(f"Operator {operator} requires a value")

        return v

class SortCriteria(BaseModel):
    """排序條件"""
    field: str
    direction: SortDirection = SortDirection.ASC

class QueryParams(BaseModel):
    """查詢參數"""
    filters: List[FilterCriteria] = Field(default_factory=list)
    sorts: List[SortCriteria] = Field(default_factory=list)
    page: int = Field(1, ge=1)
    per_page: int = Field(20, ge=1, le=100)
    include_total: bool = True

class AdvancedQueryBuilder:
    """進階查詢建構器"""

    def __init__(self):
        self.allowed_fields = set()
        self.field_types = {}
        self.field_aliases = {}

    def register_field(self, field: str, field_type: type, alias: str = None):
        """註冊可查詢欄位"""
        self.allowed_fields.add(field)
        self.field_types[field] = field_type

        if alias:
            self.field_aliases[alias] = field

    def validate_query(self, query: QueryParams) -> QueryParams:
        """驗證查詢參數"""
        # 驗證過濾欄位
        for filter_criteria in query.filters:
            field = self._resolve_field_alias(filter_criteria.field)

            if field not in self.allowed_fields:
                raise HTTPException(
                    status_code=400,
                    detail=f"Invalid filter field: {filter_criteria.field}"
                )

            # 驗證欄位類型與操作符相容性
            self._validate_field_operator_compatibility(field, filter_criteria)

        # 驗證排序欄位
        for sort_criteria in query.sorts:
            field = self._resolve_field_alias(sort_criteria.field)

            if field not in self.allowed_fields:
                raise HTTPException(
                    status_code=400,
                    detail=f"Invalid sort field: {sort_criteria.field}"
                )

        return query

    def _resolve_field_alias(self, field: str) -> str:
        """解析欄位別名"""
        return self.field_aliases.get(field, field)

    def _validate_field_operator_compatibility(self, field: str, criteria: FilterCriteria):
        """驗證欄位類型與操作符的相容性"""
        field_type = self.field_types[field]
        operator = criteria.operator

        # 字串欄位操作符
        string_operators = {
            FilterOperator.CONTAINS, FilterOperator.STARTS_WITH, FilterOperator.ENDS_WITH
        }

        # 數值欄位操作符
        numeric_operators = {
            FilterOperator.GT, FilterOperator.GTE, FilterOperator.LT, FilterOperator.LTE
        }

        if operator in string_operators and field_type != str:
            raise HTTPException(
                status_code=400,
                detail=f"String operator {operator} not compatible with field {field} of type {field_type.__name__}"
            )

        if operator in numeric_operators and field_type not in [int, float, datetime]:
            raise HTTPException(
                status_code=400,
                detail=f"Numeric operator {operator} not compatible with field {field} of type {field_type.__name__}"
            )

    def build_sqlalchemy_query(self, base_query, query_params: QueryParams):
        """建構 SQLAlchemy 查詢"""
        # 應用過濾條件
        for filter_criteria in query_params.filters:
            base_query = self._apply_filter(base_query, filter_criteria)

        # 應用排序
        for sort_criteria in query_params.sorts:
            base_query = self._apply_sort(base_query, sort_criteria)

        return base_query

    def _apply_filter(self, query, criteria: FilterCriteria):
        """應用過濾條件到 SQLAlchemy 查詢"""
        # 這裡需要根據實際的 SQLAlchemy 模型實作
        # 簡化範例
        pass

    def _apply_sort(self, query, criteria: SortCriteria):
        """應用排序到 SQLAlchemy 查詢"""
        # 簡化範例
        pass

# URL 查詢參數解析器
class URLQueryParser:
    """URL 查詢參數解析器"""

    @staticmethod
    def parse_filters(filter_params: Optional[str] = Query(None, description="Filter parameters")):
        """解析過濾參數

        支援格式：
        ?filter=username:eq:john,created_at:gte:2024-01-01,status:in:active|inactive
        """
        filters = []

        if filter_params:
            filter_strings = filter_params.split(',')

            for filter_str in filter_strings:
                parts = filter_str.split(':')

                if len(parts) < 2:
                    raise HTTPException(status_code=400, detail=f"Invalid filter format: {filter_str}")

                field = parts[0]
                operator_str = parts[1]

                # 解析值
                if len(parts) >= 3:
                    value_str = ':'.join(parts[2:])  # 重新組合值（防止值中包含冒號）

                    # 處理列表值
                    if operator_str in ['in', 'not_in']:
                        value = value_str.split('|')
                    else:
                        value = value_str
                else:
                    value = None

                try:
                    operator = FilterOperator(operator_str)
                except ValueError:
                    raise HTTPException(
                        status_code=400,
                        detail=f"Invalid filter operator: {operator_str}"
                    )

                filters.append(FilterCriteria(
                    field=field,
                    operator=operator,
                    value=value
                ))

        return filters

    @staticmethod
    def parse_sorts(sort_params: Optional[str] = Query(None, description="Sort parameters")):
        """解析排序參數

        支援格式：
        ?sort=username:asc,created_at:desc
        """
        sorts = []

        if sort_params:
            sort_strings = sort_params.split(',')

            for sort_str in sort_strings:
                parts = sort_str.split(':')
                field = parts[0]
                direction = SortDirection.ASC

                if len(parts) > 1:
                    try:
                        direction = SortDirection(parts[1])
                    except ValueError:
                        raise HTTPException(
                            status_code=400,
                            detail=f"Invalid sort direction: {parts[1]}"
                        )

                sorts.append(SortCriteria(field=field, direction=direction))

        return sorts

# GraphQL 風格查詢
class GraphQLStyleQuery(BaseModel):
    """GraphQL 風格查詢參數"""
    fields: Optional[List[str]] = Field(
        None,
        description="Specific fields to include in response",
        example=["id", "username", "email"]
    )
    expand: Optional[List[str]] = Field(
        None,
        description="Related resources to expand",
        example=["profile", "permissions"]
    )

class FieldSelector:
    """欄位選擇器"""

    def __init__(self):
        self.allowed_fields = set()
        self.expandable_relations = set()

    def register_field(self, field: str):
        """註冊可選擇的欄位"""
        self.allowed_fields.add(field)

    def register_expandable_relation(self, relation: str):
        """註冊可展開的關聯"""
        self.expandable_relations.add(relation)

    def validate_fields(self, fields: List[str]) -> List[str]:
        """驗證欄位選擇"""
        invalid_fields = [f for f in fields if f not in self.allowed_fields]

        if invalid_fields:
            raise HTTPException(
                status_code=400,
                detail=f"Invalid fields: {invalid_fields}"
            )

        return fields

    def validate_expansions(self, expansions: List[str]) -> List[str]:
        """驗證關聯展開"""
        invalid_expansions = [e for e in expansions if e not in self.expandable_relations]

        if invalid_expansions:
            raise HTTPException(
                status_code=400,
                detail=f"Invalid expansions: {invalid_expansions}"
            )

        return expansions

    def apply_field_selection(self, data: Dict[str, Any], selected_fields: List[str]) -> Dict[str, Any]:
        """應用欄位選擇"""
        if not selected_fields:
            return data

        return {field: data[field] for field in selected_fields if field in data}

# 全文搜索
class FullTextSearch:
    """全文搜索功能"""

    def __init__(self):
        self.searchable_fields = set()
        self.search_weights = {}

    def register_searchable_field(self, field: str, weight: float = 1.0):
        """註冊可搜索欄位"""
        self.searchable_fields.add(field)
        self.search_weights[field] = weight

    def build_search_query(self, base_query, search_term: str):
        """建構搜索查詢"""
        if not search_term:
            return base_query

        # 實作全文搜索邏輯
        # 這裡需要根據使用的資料庫和搜索引擎進行實作
        # 例如：PostgreSQL 的 tsvector，Elasticsearch 等
        pass

# 複合查詢範例
class UserAdvancedQueryAPI:
    """使用者進階查詢 API"""

    def __init__(self):
        self.query_builder = AdvancedQueryBuilder()
        self.field_selector = FieldSelector()
        self.full_text_search = FullTextSearch()
        self._setup_query_capabilities()

    def _setup_query_capabilities(self):
        """設定查詢能力"""
        # 註冊可查詢欄位
        self.query_builder.register_field("username", str)
        self.query_builder.register_field("email", str)
        self.query_builder.register_field("created_at", datetime, alias="created")
        self.query_builder.register_field("is_active", bool, alias="active")

        # 註冊可選擇欄位
        for field in ["id", "username", "email", "created_at", "is_active"]:
            self.field_selector.register_field(field)

        # 註冊可展開關聯
        self.field_selector.register_expandable_relation("profile")
        self.field_selector.register_expandable_relation("permissions")

        # 註冊可搜索欄位
        self.full_text_search.register_searchable_field("username", 2.0)
        self.full_text_search.register_searchable_field("email", 1.0)

    def create_query_endpoint(self):
        """建立查詢端點"""
        from fastapi import APIRouter

        router = APIRouter()

        @router.get("/users/search", summary="Advanced user search")
        async def advanced_user_search(
            # 基本查詢參數
            page: int = Query(1, ge=1),
            per_page: int = Query(20, ge=1, le=100),

            # 過濾和排序
            filters: List[FilterCriteria] = Depends(URLQueryParser.parse_filters),
            sorts: List[SortCriteria] = Depends(URLQueryParser.parse_sorts),

            # 欄位選擇
            fields: Optional[List[str]] = Query(None, description="Fields to include"),
            expand: Optional[List[str]] = Query(None, description="Relations to expand"),

            # 全文搜索
            q: Optional[str] = Query(None, description="Search query"),
        ):
            """進階使用者搜索"""

            # 建構查詢參數
            query_params = QueryParams(
                filters=filters or [],
                sorts=sorts or [],
                page=page,
                per_page=per_page
            )

            # 驗證查詢參數
            query_params = self.query_builder.validate_query(query_params)

            # 驗證欄位選擇
            if fields:
                fields = self.field_selector.validate_fields(fields)

            if expand:
                expand = self.field_selector.validate_expansions(expand)

            # 執行查詢
            results = await self._execute_query(query_params, fields, expand, q)

            return results

        return router

    async def _execute_query(self, query_params: QueryParams, fields: Optional[List[str]],
                           expand: Optional[List[str]], search_term: Optional[str]):
        """執行查詢"""
        # 實際查詢實作
        # 這裡需要整合資料庫查詢邏輯
        pass
```

---

## 🚨 錯誤處理與狀態管理

### 統一錯誤回應格式

```python
# api/error_handling/error_manager.py
from typing import Dict, Any, List, Optional, Union
from pydantic import BaseModel, Field
from fastapi import HTTPException, Request, status
from fastapi.responses import JSONResponse
from enum import Enum
import traceback
import uuid
from datetime import datetime

class ErrorCategory(str, Enum):
    """錯誤類別"""
    VALIDATION = "validation"
    AUTHENTICATION = "authentication"
    AUTHORIZATION = "authorization"
    BUSINESS_LOGIC = "business_logic"
    RESOURCE_NOT_FOUND = "resource_not_found"
    CONFLICT = "conflict"
    RATE_LIMIT = "rate_limit"
    INTERNAL_ERROR = "internal_error"
    EXTERNAL_SERVICE = "external_service"

class ErrorSeverity(str, Enum):
    """錯誤嚴重程度"""
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"
    CRITICAL = "critical"

class FieldError(BaseModel):
    """欄位錯誤詳情"""
    field: str
    message: str
    code: str
    rejected_value: Any = None

class ErrorDetail(BaseModel):
    """錯誤詳情"""
    code: str
    message: str
    field: Optional[str] = None
    meta: Dict[str, Any] = Field(default_factory=dict)

class APIError(BaseModel):
    """統一 API 錯誤回應格式"""
    error_id: str = Field(..., description="Unique error identifier for tracking")
    timestamp: datetime = Field(default_factory=datetime.utcnow)
    status_code: int = Field(..., description="HTTP status code")
    error_code: str = Field(..., description="Application-specific error code")
    message: str = Field(..., description="Human-readable error message")
    category: ErrorCategory = Field(..., description="Error category")
    severity: ErrorSeverity = Field(default=ErrorSeverity.MEDIUM)
    details: List[ErrorDetail] = Field(default_factory=list, description="Detailed error information")
    path: str = Field(..., description="API path where error occurred")
    method: str = Field(..., description="HTTP method")
    user_id: Optional[str] = None
    request_id: Optional[str] = None
    suggestions: List[str] = Field(default_factory=list, description="Suggestions to fix the error")
    documentation_url: Optional[str] = None

    class Config:
        schema_extra = {
            "example": {
                "error_id": "550e8400-e29b-41d4-a716-446655440000",
                "timestamp": "2024-01-06T10:00:00Z",
                "status_code": 400,
                "error_code": "VALIDATION_ERROR",
                "message": "Request validation failed",
                "category": "validation",
                "severity": "medium",
                "details": [
                    {
                        "code": "REQUIRED_FIELD_MISSING",
                        "message": "Email is required",
                        "field": "email"
                    }
                ],
                "path": "/api/v1/users",
                "method": "POST",
                "suggestions": [
                    "Ensure all required fields are included in the request",
                    "Check the API documentation for required field formats"
                ],
                "documentation_url": "https://docs.example.com/api/errors#VALIDATION_ERROR"
            }
        }

class BusinessError(Exception):
    """業務邏輯錯誤基類"""

    def __init__(self, message: str, code: str, category: ErrorCategory = ErrorCategory.BUSINESS_LOGIC,
                 severity: ErrorSeverity = ErrorSeverity.MEDIUM, details: List[ErrorDetail] = None):
        self.message = message
        self.code = code
        self.category = category
        self.severity = severity
        self.details = details or []
        super().__init__(self.message)

class ValidationError(BusinessError):
    """驗證錯誤"""

    def __init__(self, message: str = "Validation failed", field_errors: List[FieldError] = None):
        details = []
        if field_errors:
            for field_error in field_errors:
                details.append(ErrorDetail(
                    code=field_error.code,
                    message=field_error.message,
                    field=field_error.field,
                    meta={"rejected_value": field_error.rejected_value} if field_error.rejected_value else {}
                ))

        super().__init__(
            message=message,
            code="VALIDATION_ERROR",
            category=ErrorCategory.VALIDATION,
            severity=ErrorSeverity.MEDIUM,
            details=details
        )

class AuthenticationError(BusinessError):
    """認證錯誤"""

    def __init__(self, message: str = "Authentication failed"):
        super().__init__(
            message=message,
            code="AUTHENTICATION_ERROR",
            category=ErrorCategory.AUTHENTICATION,
            severity=ErrorSeverity.HIGH
        )

class AuthorizationError(BusinessError):
    """授權錯誤"""

    def __init__(self, message: str = "Insufficient permissions"):
        super().__init__(
            message=message,
            code="AUTHORIZATION_ERROR",
            category=ErrorCategory.AUTHORIZATION,
            severity=ErrorSeverity.HIGH
        )

class ResourceNotFoundError(BusinessError):
    """資源未找到錯誤"""

    def __init__(self, resource_type: str, resource_id: str):
        message = f"{resource_type} with id '{resource_id}' not found"
        super().__init__(
            message=message,
            code="RESOURCE_NOT_FOUND",
            category=ErrorCategory.RESOURCE_NOT_FOUND,
            severity=ErrorSeverity.LOW
        )

class ConflictError(BusinessError):
    """衝突錯誤"""

    def __init__(self, message: str, conflicting_field: str = None):
        details = []
        if conflicting_field:
            details.append(ErrorDetail(
                code="CONFLICT",
                message=f"Conflict in field: {conflicting_field}",
                field=conflicting_field
            ))

        super().__init__(
            message=message,
            code="CONFLICT_ERROR",
            category=ErrorCategory.CONFLICT,
            severity=ErrorSeverity.MEDIUM,
            details=details
        )

class RateLimitError(BusinessError):
    """速率限制錯誤"""

    def __init__(self, retry_after: int):
        super().__init__(
            message=f"Rate limit exceeded. Retry after {retry_after} seconds",
            code="RATE_LIMIT_EXCEEDED",
            category=ErrorCategory.RATE_LIMIT,
            severity=ErrorSeverity.MEDIUM,
            details=[ErrorDetail(
                code="RATE_LIMIT",
                message="Too many requests",
                meta={"retry_after": retry_after}
            )]
        )

class ExternalServiceError(BusinessError):
    """外部服務錯誤"""

    def __init__(self, service_name: str, operation: str):
        super().__init__(
            message=f"External service {service_name} is currently unavailable",
            code="EXTERNAL_SERVICE_ERROR",
            category=ErrorCategory.EXTERNAL_SERVICE,
            severity=ErrorSeverity.HIGH,
            details=[ErrorDetail(
                code="SERVICE_UNAVAILABLE",
                message=f"Operation {operation} failed",
                meta={"service": service_name, "operation": operation}
            )]
        )

class ErrorHandler:
    """統一錯誤處理器"""

    def __init__(self):
        self.error_mappings = self._setup_error_mappings()

    def _setup_error_mappings(self) -> Dict[type, Dict[str, Any]]:
        """設定錯誤映射"""
        return {
            ValidationError: {
                "status_code": status.HTTP_422_UNPROCESSABLE_ENTITY,
                "suggestions": [
                    "Check request payload format and required fields",
                    "Refer to API documentation for field specifications"
                ]
            },
            AuthenticationError: {
                "status_code": status.HTTP_401_UNAUTHORIZED,
                "suggestions": [
                    "Ensure valid authentication token is provided",
                    "Check if token has expired and refresh if needed"
                ]
            },
            AuthorizationError: {
                "status_code": status.HTTP_403_FORBIDDEN,
                "suggestions": [
                    "Contact administrator to request necessary permissions",
                    "Verify you are accessing the correct resource"
                ]
            },
            ResourceNotFoundError: {
                "status_code": status.HTTP_404_NOT_FOUND,
                "suggestions": [
                    "Verify the resource ID is correct",
                    "Check if the resource has been deleted"
                ]
            },
            ConflictError: {
                "status_code": status.HTTP_409_CONFLICT,
                "suggestions": [
                    "Use a different value for the conflicting field",
                    "Check if the resource already exists"
                ]
            },
            RateLimitError: {
                "status_code": status.HTTP_429_TOO_MANY_REQUESTS,
                "suggestions": [
                    "Wait before making another request",
                    "Implement exponential backoff in your client"
                ]
            },
            ExternalServiceError: {
                "status_code": status.HTTP_502_BAD_GATEWAY,
                "suggestions": [
                    "Try again later",
                    "Contact support if the issue persists"
                ]
            }
        }

    def handle_business_error(self, request: Request, error: BusinessError) -> JSONResponse:
        """處理業務錯誤"""
        error_mapping = self.error_mappings.get(type(error), {
            "status_code": status.HTTP_500_INTERNAL_SERVER_ERROR,
            "suggestions": ["Contact support for assistance"]
        })

        api_error = APIError(
            error_id=str(uuid.uuid4()),
            status_code=error_mapping["status_code"],
            error_code=error.code,
            message=error.message,
            category=error.category,
            severity=error.severity,
            details=error.details,
            path=request.url.path,
            method=request.method,
            user_id=getattr(request.state, 'user_id', None),
            request_id=getattr(request.state, 'request_id', None),
            suggestions=error_mapping.get("suggestions", [])
        )

        # 記錄錯誤
        self._log_error(api_error, error)

        return JSONResponse(
            status_code=error_mapping["status_code"],
            content=api_error.dict()
        )

    def handle_http_exception(self, request: Request, exc: HTTPException) -> JSONResponse:
        """處理 HTTP 異常"""
        api_error = APIError(
            error_id=str(uuid.uuid4()),
            status_code=exc.status_code,
            error_code=f"HTTP_{exc.status_code}",
            message=exc.detail,
            category=self._categorize_http_error(exc.status_code),
            path=request.url.path,
            method=request.method
        )

        return JSONResponse(
            status_code=exc.status_code,
            content=api_error.dict()
        )

    def handle_internal_error(self, request: Request, exc: Exception) -> JSONResponse:
        """處理內部伺服器錯誤"""
        error_id = str(uuid.uuid4())

        api_error = APIError(
            error_id=error_id,
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            error_code="INTERNAL_SERVER_ERROR",
            message="An internal server error occurred",
            category=ErrorCategory.INTERNAL_ERROR,
            severity=ErrorSeverity.CRITICAL,
            path=request.url.path,
            method=request.method,
            suggestions=["Contact support with the error ID"]
        )

        # 詳細記錄內部錯誤
        self._log_internal_error(api_error, exc)

        return JSONResponse(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            content=api_error.dict()
        )

    def _categorize_http_error(self, status_code: int) -> ErrorCategory:
        """根據 HTTP 狀態碼分類錯誤"""
        if status_code == 401:
            return ErrorCategory.AUTHENTICATION
        elif status_code == 403:
            return ErrorCategory.AUTHORIZATION
        elif status_code == 404:
            return ErrorCategory.RESOURCE_NOT_FOUND
        elif status_code == 409:
            return ErrorCategory.CONFLICT
        elif status_code == 422:
            return ErrorCategory.VALIDATION
        elif status_code == 429:
            return ErrorCategory.RATE_LIMIT
        else:
            return ErrorCategory.INTERNAL_ERROR

    def _log_error(self, api_error: APIError, original_error: Exception):
        """記錄錯誤"""
        logger.warning(
            "Business error occurred",
            error_id=api_error.error_id,
            error_code=api_error.error_code,
            message=api_error.message,
            path=api_error.path,
            method=api_error.method,
            user_id=api_error.user_id,
            category=api_error.category,
            severity=api_error.severity
        )

    def _log_internal_error(self, api_error: APIError, original_error: Exception):
        """記錄內部錯誤"""
        logger.error(
            "Internal server error occurred",
            error_id=api_error.error_id,
            error_message=str(original_error),
            error_type=type(original_error).__name__,
            traceback=traceback.format_exc(),
            path=api_error.path,
            method=api_error.method,
            user_id=api_error.user_id
        )

# 錯誤處理中間件
def setup_error_handlers(app):
    """設定錯誤處理器"""
    error_handler = ErrorHandler()

    @app.exception_handler(BusinessError)
    async def business_error_handler(request: Request, exc: BusinessError):
        return error_handler.handle_business_error(request, exc)

    @app.exception_handler(HTTPException)
    async def http_exception_handler(request: Request, exc: HTTPException):
        return error_handler.handle_http_exception(request, exc)

    @app.exception_handler(Exception)
    async def general_exception_handler(request: Request, exc: Exception):
        return error_handler.handle_internal_error(request, exc)

# 錯誤回報系統
class ErrorReporter:
    """錯誤回報系統"""

    def __init__(self):
        self.error_stats = {}

    async def report_error(self, api_error: APIError):
        """回報錯誤"""
        # 統計錯誤
        error_key = f"{api_error.error_code}_{api_error.path}"
        if error_key not in self.error_stats:
            self.error_stats[error_key] = {
                "count": 0,
                "first_seen": api_error.timestamp,
                "last_seen": api_error.timestamp
            }

        self.error_stats[error_key]["count"] += 1
        self.error_stats[error_key]["last_seen"] = api_error.timestamp

        # 如果是高頻錯誤或嚴重錯誤，發送告警
        if (self.error_stats[error_key]["count"] > 10 or
            api_error.severity == ErrorSeverity.CRITICAL):
            await self._send_error_alert(api_error, self.error_stats[error_key])

    async def _send_error_alert(self, api_error: APIError, stats: Dict[str, Any]):
        """發送錯誤告警"""
        alert_payload = {
            "error_code": api_error.error_code,
            "message": api_error.message,
            "path": api_error.path,
            "severity": api_error.severity,
            "occurrence_count": stats["count"],
            "first_seen": stats["first_seen"].isoformat(),
            "last_seen": stats["last_seen"].isoformat()
        }

        # 發送到監控系統或 Slack
        logger.critical("High-frequency or critical error detected", **alert_payload)

    def get_error_summary(self) -> Dict[str, Any]:
        """取得錯誤摘要"""
        total_errors = sum(stat["count"] for stat in self.error_stats.values())
        most_frequent = max(self.error_stats.items(), key=lambda x: x[1]["count"]) if self.error_stats else None

        return {
            "total_error_types": len(self.error_stats),
            "total_error_count": total_errors,
            "most_frequent_error": {
                "error_key": most_frequent[0],
                "count": most_frequent[1]["count"]
            } if most_frequent else None
        }
```

---

## 💡 學習筆記區

### 🤔 我的理解
```
RESTful API 設計的核心原則：

API 版本控制的策略選擇：

進階查詢與過濾的設計考量：

統一錯誤處理的重要性：
```

### 🔧 實踐心得
```
API 設計過程中的挑戰：

版本相容性維護的經驗：

查詢效能優化的技巧：

錯誤處理的最佳實務：
```

### 🚀 進階思考
```
API 設計的未來趨勢：

GraphQL vs RESTful 的選擇：

API 治理與標準化：

API 安全性的設計考量：
```