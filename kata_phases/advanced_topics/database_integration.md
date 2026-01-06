# Database Integration: 企業級資料庫架構與最佳實務

## 🎯 學習目標
- 掌握現代資料庫架構設計模式
- 學習 SQLAlchemy 進階特性與效能優化
- 建立資料庫遷移與版本控制策略
- 實踐資料庫監控與效能調優

---

## 🗄️ SQLAlchemy 進階架構設計

### 企業級資料庫連接管理

```python
# app/database/connection.py
from sqlalchemy.ext.asyncio import (
    AsyncSession,
    create_async_engine,
    AsyncConnection,
    async_sessionmaker
)
from sqlalchemy.pool import QueuePool, NullPool
from sqlalchemy.engine.events import event
from typing import AsyncGenerator
import structlog

logger = structlog.get_logger()

class DatabaseManager:
    """企業級資料庫管理器"""

    def __init__(self, config: DatabaseConfig):
        self.config = config
        self.engine = self._create_engine()
        self.session_factory = self._create_session_factory()
        self._setup_event_listeners()

    def _create_engine(self):
        """建立優化的資料庫引擎"""
        engine_kwargs = {
            "echo": self.config.debug,
            "echo_pool": self.config.debug_pool,
            "future": True,  # 使用 SQLAlchemy 2.0 樣式
            "pool_size": self.config.pool_size,
            "max_overflow": self.config.max_overflow,
            "pool_timeout": self.config.pool_timeout,
            "pool_recycle": self.config.pool_recycle,
            "pool_pre_ping": True,  # 連接健康檢查
        }

        # 根據環境選擇連接池
        if self.config.environment == "test":
            engine_kwargs["poolclass"] = NullPool  # 測試環境不使用連接池
        else:
            engine_kwargs["poolclass"] = QueuePool

        return create_async_engine(self.config.url, **engine_kwargs)

    def _create_session_factory(self):
        """建立會話工廠"""
        return async_sessionmaker(
            self.engine,
            class_=AsyncSession,
            autoflush=False,  # 手動控制 flush 時機
            autocommit=False,
            expire_on_commit=False  # 避免懶載入問題
        )

    def _setup_event_listeners(self):
        """設定資料庫事件監聽器"""

        @event.listens_for(self.engine.sync_engine, "connect")
        def set_sqlite_pragma(dbapi_connection, connection_record):
            """SQLite 專用設定"""
            if "sqlite" in str(self.engine.url):
                cursor = dbapi_connection.cursor()
                cursor.execute("PRAGMA foreign_keys=ON")
                cursor.execute("PRAGMA journal_mode=WAL")
                cursor.close()

        @event.listens_for(self.engine.sync_engine, "checkout")
        def receive_checkout(dbapi_connection, connection_record, connection_proxy):
            """連接取出時的處理"""
            logger.debug("Database connection checked out",
                        connection_id=id(dbapi_connection))

        @event.listens_for(self.engine.sync_engine, "checkin")
        def receive_checkin(dbapi_connection, connection_record):
            """連接歸還時的處理"""
            logger.debug("Database connection checked in",
                        connection_id=id(dbapi_connection))

    async def get_session(self) -> AsyncGenerator[AsyncSession, None]:
        """取得資料庫會話"""
        async with self.session_factory() as session:
            try:
                yield session
            except Exception:
                await session.rollback()
                raise
            finally:
                await session.close()

    async def health_check(self) -> bool:
        """資料庫健康檢查"""
        try:
            async with self.engine.begin() as conn:
                await conn.execute(text("SELECT 1"))
            return True
        except Exception as e:
            logger.error("Database health check failed", error=str(e))
            return False

    async def get_connection_stats(self) -> dict:
        """取得連接池統計"""
        pool = self.engine.pool
        return {
            "pool_size": pool.size(),
            "checked_in": pool.checkedin(),
            "checked_out": pool.checkedout(),
            "overflow": pool.overflow(),
            "invalid": pool.invalid()
        }

    async def close(self):
        """關閉資料庫引擎"""
        await self.engine.dispose()

class ReadWriteSplitManager:
    """讀寫分離管理器"""

    def __init__(self, write_config: DatabaseConfig, read_configs: List[DatabaseConfig]):
        self.write_db = DatabaseManager(write_config)
        self.read_dbs = [DatabaseManager(config) for config in read_configs]
        self.read_db_index = 0

    def get_read_db(self) -> DatabaseManager:
        """取得讀取資料庫（負載均衡）"""
        db = self.read_dbs[self.read_db_index]
        self.read_db_index = (self.read_db_index + 1) % len(self.read_dbs)
        return db

    def get_write_db(self) -> DatabaseManager:
        """取得寫入資料庫"""
        return self.write_db

    async def transaction_context(self, read_only: bool = False):
        """交易上下文管理"""
        db = self.get_read_db() if read_only else self.get_write_db()
        async with db.get_session() as session:
            yield session
```

### 進階模型設計模式

```python
# app/models/base.py
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.ext.hybrid import hybrid_property
from sqlalchemy import Column, DateTime, String, Boolean
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.sql import func
from datetime import datetime
import uuid

Base = declarative_base()

class TimestampMixin:
    """時間戳混入類別"""
    created_at = Column(DateTime(timezone=True), server_default=func.now(), nullable=False)
    updated_at = Column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now(), nullable=False)

class SoftDeleteMixin:
    """軟刪除混入類別"""
    deleted_at = Column(DateTime(timezone=True), nullable=True)
    is_deleted = Column(Boolean, default=False, nullable=False)

    @hybrid_property
    def is_active(self):
        return not self.is_deleted

    def soft_delete(self):
        self.is_deleted = True
        self.deleted_at = datetime.utcnow()

class AuditMixin:
    """審計混入類別"""
    created_by = Column(String(255), nullable=True)
    updated_by = Column(String(255), nullable=True)
    version = Column(Integer, default=1, nullable=False)

    def increment_version(self):
        self.version += 1

class BaseModel(Base, TimestampMixin, SoftDeleteMixin, AuditMixin):
    """基礎模型類別"""
    __abstract__ = True

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)

    def to_dict(self) -> dict:
        """轉換為字典"""
        return {c.name: getattr(self, c.name) for c in self.__table__.columns}

    def __repr__(self):
        return f"<{self.__class__.__name__}(id={self.id})>"

# app/models/user.py
from sqlalchemy import Column, String, Boolean, Integer, Index, UniqueConstraint
from sqlalchemy.orm import relationship
from sqlalchemy.ext.hybrid import hybrid_property
from .base import BaseModel

class User(BaseModel):
    """使用者模型"""
    __tablename__ = "users"

    # 基本欄位
    username = Column(String(50), unique=True, nullable=False)
    email = Column(String(255), unique=True, nullable=False)
    hashed_password = Column(String(255), nullable=False)

    # 個人資訊
    first_name = Column(String(100), nullable=True)
    last_name = Column(String(100), nullable=True)
    display_name = Column(String(200), nullable=True)

    # 帳戶狀態
    is_active = Column(Boolean, default=True, nullable=False)
    is_verified = Column(Boolean, default=False, nullable=False)
    email_verified_at = Column(DateTime(timezone=True), nullable=True)

    # 安全欄位
    failed_login_attempts = Column(Integer, default=0, nullable=False)
    locked_until = Column(DateTime(timezone=True), nullable=True)
    password_changed_at = Column(DateTime(timezone=True), nullable=True)
    last_login_at = Column(DateTime(timezone=True), nullable=True)
    last_login_ip = Column(String(45), nullable=True)  # IPv6 支援

    # MFA 設定
    mfa_enabled = Column(Boolean, default=False, nullable=False)
    mfa_secret = Column(String(255), nullable=True)
    backup_codes = Column(JSON, nullable=True)

    # 關聯
    user_sessions = relationship("UserSession", back_populates="user", cascade="all, delete-orphan")
    login_history = relationship("LoginHistory", back_populates="user", cascade="all, delete-orphan")

    # 索引
    __table_args__ = (
        Index('ix_users_username_active', 'username', 'is_active'),
        Index('ix_users_email_active', 'email', 'is_active'),
        Index('ix_users_created_at', 'created_at'),
        UniqueConstraint('email', name='uq_users_email'),
        UniqueConstraint('username', name='uq_users_username'),
    )

    @hybrid_property
    def full_name(self):
        """完整姓名"""
        if self.first_name and self.last_name:
            return f"{self.first_name} {self.last_name}"
        return self.display_name or self.username

    @hybrid_property
    def is_locked(self):
        """檢查帳戶是否被鎖定"""
        if self.locked_until:
            return datetime.utcnow() < self.locked_until
        return False

    def lock_account(self, duration_minutes: int = 30):
        """鎖定帳戶"""
        self.locked_until = datetime.utcnow() + timedelta(minutes=duration_minutes)

    def unlock_account(self):
        """解鎖帳戶"""
        self.locked_until = None
        self.failed_login_attempts = 0

    def record_failed_login(self):
        """記錄登入失敗"""
        self.failed_login_attempts += 1
        if self.failed_login_attempts >= 5:  # 5次失敗後鎖定
            self.lock_account()

    def record_successful_login(self, ip_address: str):
        """記錄成功登入"""
        self.last_login_at = datetime.utcnow()
        self.last_login_ip = ip_address
        self.failed_login_attempts = 0
        self.locked_until = None

class UserSession(BaseModel):
    """使用者會話模型"""
    __tablename__ = "user_sessions"

    user_id = Column(UUID(as_uuid=True), ForeignKey("users.id"), nullable=False)
    session_token = Column(String(255), unique=True, nullable=False)
    refresh_token = Column(String(255), unique=True, nullable=True)

    expires_at = Column(DateTime(timezone=True), nullable=False)
    last_activity = Column(DateTime(timezone=True), server_default=func.now(), nullable=False)

    # 會話資訊
    ip_address = Column(String(45), nullable=True)
    user_agent = Column(String(500), nullable=True)
    device_fingerprint = Column(String(255), nullable=True)

    # 關聯
    user = relationship("User", back_populates="user_sessions")

    # 索引
    __table_args__ = (
        Index('ix_sessions_user_id', 'user_id'),
        Index('ix_sessions_expires_at', 'expires_at'),
        Index('ix_sessions_token', 'session_token'),
    )

    @hybrid_property
    def is_expired(self):
        """檢查會話是否過期"""
        return datetime.utcnow() > self.expires_at

    @hybrid_property
    def is_active(self):
        """檢查會話是否活躍"""
        return not self.is_expired and not self.is_deleted

    def extend_session(self, hours: int = 24):
        """延長會話"""
        self.expires_at = datetime.utcnow() + timedelta(hours=hours)
        self.last_activity = datetime.utcnow()

class LoginHistory(BaseModel):
    """登入歷史記錄"""
    __tablename__ = "login_history"

    user_id = Column(UUID(as_uuid=True), ForeignKey("users.id"), nullable=False)
    ip_address = Column(String(45), nullable=False)
    user_agent = Column(String(500), nullable=True)
    location = Column(String(255), nullable=True)  # 地理位置
    success = Column(Boolean, nullable=False)
    failure_reason = Column(String(255), nullable=True)

    # 風險評分
    risk_score = Column(Integer, default=0, nullable=False)
    unusual_activity = Column(Boolean, default=False, nullable=False)

    # 關聯
    user = relationship("User", back_populates="login_history")

    # 索引
    __table_args__ = (
        Index('ix_login_history_user_id', 'user_id'),
        Index('ix_login_history_created_at', 'created_at'),
        Index('ix_login_history_ip', 'ip_address'),
        Index('ix_login_history_success', 'success'),
    )
```

---

## 🔄 資料庫遷移與版本控制

### Alembic 進階遷移策略

```python
# alembic/env.py
from logging.config import fileConfig
from sqlalchemy import engine_from_config, pool
from alembic import context
from app.models import Base
import os

# Alembic Config object
config = context.config

# 設定資料庫 URL
database_url = os.environ.get("DATABASE_URL")
if database_url:
    config.set_main_option("sqlalchemy.url", database_url)

# Interpret the config file for Python logging
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

# 目標 metadata
target_metadata = Base.metadata

def include_object(object, name, type_, reflected, compare_to):
    """過濾要包含在遷移中的物件"""
    # 排除臨時表
    if type_ == "table" and name.startswith("temp_"):
        return False

    # 排除視圖
    if type_ == "table" and reflected and object.info.get("is_view"):
        return False

    return True

def run_migrations_offline():
    """離線模式遷移"""
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
        include_object=include_object,
        compare_type=True,
        compare_server_default=True,
    )

    with context.begin_transaction():
        context.run_migrations()

def run_migrations_online():
    """線上模式遷移"""
    connectable = engine_from_config(
        config.get_section(config.config_ini_section),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    with connectable.connect() as connection:
        context.configure(
            connection=connection,
            target_metadata=target_metadata,
            include_object=include_object,
            compare_type=True,
            compare_server_default=True,
        )

        with context.begin_transaction():
            context.run_migrations()

# 判斷執行模式
if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()

# scripts/migration_manager.py
class MigrationManager:
    """遷移管理器"""

    def __init__(self, config: AlembicConfig):
        self.config = config

    async def create_migration(self, message: str, auto_generate: bool = True) -> str:
        """建立新的遷移檔案"""
        cmd_opts = []
        if auto_generate:
            cmd_opts.append("--autogenerate")

        cmd_opts.extend(["-m", message])

        # 執行 alembic revision
        revision_id = alembic.command.revision(self.config, message, autogenerate=auto_generate)

        # 遷移檔案後處理
        await self._post_process_migration(revision_id)

        return revision_id

    async def _post_process_migration(self, revision_id: str):
        """遷移檔案後處理"""
        migration_file = self._get_migration_file(revision_id)

        # 添加安全檢查
        await self._add_safety_checks(migration_file)

        # 驗證遷移語法
        await self._validate_migration_syntax(migration_file)

        # 估算遷移時間
        estimated_time = await self._estimate_migration_time(migration_file)

        logger.info(f"Migration {revision_id} created",
                   estimated_time=estimated_time)

    async def apply_migrations(self, target_revision: str = "head", dry_run: bool = False):
        """應用遷移"""
        if dry_run:
            # 檢查遷移計劃
            current = alembic.command.current(self.config)
            plan = alembic.command.show(self.config, target_revision)

            logger.info("Migration plan", current=current, target=target_revision, plan=plan)
            return

        # 備份資料庫（生產環境）
        if self._is_production():
            backup_id = await self._backup_database()
            logger.info("Database backed up", backup_id=backup_id)

        try:
            # 執行遷移
            alembic.command.upgrade(self.config, target_revision)
            logger.info("Migration applied successfully", target=target_revision)

        except Exception as e:
            if self._is_production():
                # 回復備份
                await self._restore_database(backup_id)
            raise MigrationError(f"Migration failed: {str(e)}")

    async def rollback_migration(self, target_revision: str):
        """回滾遷移"""
        # 檢查回滾安全性
        if not await self._is_safe_to_rollback(target_revision):
            raise MigrationError("Rollback not safe - data loss may occur")

        # 備份當前狀態
        backup_id = await self._backup_database()

        try:
            alembic.command.downgrade(self.config, target_revision)
            logger.info("Migration rolled back", target=target_revision)

        except Exception as e:
            await self._restore_database(backup_id)
            raise MigrationError(f"Rollback failed: {str(e)}")

class ZeroDowntimeMigration:
    """零停機時間遷移"""

    async def add_column_safely(self, table_name: str, column_def: dict):
        """安全地添加欄位"""
        steps = [
            # 1. 添加欄位（允許 NULL）
            f"ALTER TABLE {table_name} ADD COLUMN {column_def['name']} {column_def['type']} NULL",

            # 2. 填充預設值（分批處理）
            self._create_backfill_script(table_name, column_def),

            # 3. 添加非空約束（如果需要）
            f"ALTER TABLE {table_name} ALTER COLUMN {column_def['name']} SET NOT NULL" if column_def.get('not_null') else None,

            # 4. 添加索引（並發建立）
            f"CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_{table_name}_{column_def['name']} ON {table_name} ({column_def['name']})" if column_def.get('indexed') else None
        ]

        for step in filter(None, steps):
            await self._execute_with_retry(step)

    async def remove_column_safely(self, table_name: str, column_name: str):
        """安全地移除欄位"""
        steps = [
            # 1. 移除索引
            f"DROP INDEX IF EXISTS idx_{table_name}_{column_name}",

            # 2. 移除約束
            f"ALTER TABLE {table_name} ALTER COLUMN {column_name} DROP NOT NULL",

            # 3. 等待確認沒有應用程式使用此欄位
            self._wait_for_deployment_confirmation(),

            # 4. 移除欄位
            f"ALTER TABLE {table_name} DROP COLUMN {column_name}"
        ]

        for step in steps:
            await self._execute_with_retry(step)

    def _create_backfill_script(self, table_name: str, column_def: dict) -> str:
        """建立回填腳本"""
        batch_size = 10000
        default_value = column_def.get('default', 'NULL')

        return f"""
        DO $$
        DECLARE
            batch_start INTEGER := 0;
            batch_end INTEGER := {batch_size};
            affected_rows INTEGER;
        BEGIN
            LOOP
                UPDATE {table_name}
                SET {column_def['name']} = {default_value}
                WHERE id IN (
                    SELECT id FROM {table_name}
                    WHERE {column_def['name']} IS NULL
                    ORDER BY id
                    LIMIT {batch_size}
                );

                GET DIAGNOSTICS affected_rows = ROW_COUNT;
                EXIT WHEN affected_rows = 0;

                -- 暫停以避免鎖定過久
                PERFORM pg_sleep(0.1);
            END LOOP;
        END
        $$;
        """
```

---

## 📊 資料庫效能優化

### 查詢優化與索引策略

```python
# app/database/query_optimizer.py
from sqlalchemy import text, event
from sqlalchemy.engine import Engine
import time
import structlog

logger = structlog.get_logger()

class QueryPerformanceMonitor:
    """查詢效能監控器"""

    def __init__(self, slow_query_threshold: float = 1.0):
        self.slow_query_threshold = slow_query_threshold
        self.query_stats = defaultdict(lambda: {"count": 0, "total_time": 0, "avg_time": 0})

    def setup_monitoring(self, engine: Engine):
        """設定查詢監控"""

        @event.listens_for(engine, "before_cursor_execute")
        def receive_before_cursor_execute(conn, cursor, statement, parameters, context, executemany):
            context._query_start_time = time.time()

        @event.listens_for(engine, "after_cursor_execute")
        def receive_after_cursor_execute(conn, cursor, statement, parameters, context, executemany):
            total = time.time() - context._query_start_time

            # 記錄統計
            query_hash = hash(statement)
            stats = self.query_stats[query_hash]
            stats["count"] += 1
            stats["total_time"] += total
            stats["avg_time"] = stats["total_time"] / stats["count"]

            # 記錄慢查詢
            if total > self.slow_query_threshold:
                logger.warning("Slow query detected",
                             duration=total,
                             statement=statement[:200],
                             parameters=parameters)

    def get_performance_report(self) -> dict:
        """取得效能報告"""
        slow_queries = []
        for query_hash, stats in self.query_stats.items():
            if stats["avg_time"] > self.slow_query_threshold:
                slow_queries.append({
                    "query_hash": query_hash,
                    "avg_time": stats["avg_time"],
                    "count": stats["count"],
                    "total_time": stats["total_time"]
                })

        return {
            "total_queries": sum(stats["count"] for stats in self.query_stats.values()),
            "slow_queries": sorted(slow_queries, key=lambda x: x["avg_time"], reverse=True),
            "avg_query_time": sum(stats["avg_time"] for stats in self.query_stats.values()) / len(self.query_stats)
        }

class IndexAnalyzer:
    """索引分析器"""

    def __init__(self, session: AsyncSession):
        self.session = session

    async def analyze_missing_indexes(self) -> List[dict]:
        """分析缺失的索引"""
        # PostgreSQL 專用查詢
        query = text("""
        SELECT
            schemaname,
            tablename,
            attname,
            n_distinct,
            correlation,
            most_common_vals
        FROM pg_stats
        WHERE schemaname = 'public'
        AND n_distinct > 100
        AND correlation < 0.1
        ORDER BY n_distinct DESC;
        """)

        result = await self.session.execute(query)
        return [dict(row) for row in result.fetchall()]

    async def analyze_unused_indexes(self) -> List[dict]:
        """分析未使用的索引"""
        query = text("""
        SELECT
            schemaname,
            tablename,
            indexname,
            idx_tup_read,
            idx_tup_fetch,
            pg_size_pretty(pg_relation_size(indexrelid)) as size
        FROM pg_stat_user_indexes
        WHERE idx_tup_read = 0
        ORDER BY pg_relation_size(indexrelid) DESC;
        """)

        result = await self.session.execute(query)
        return [dict(row) for row in result.fetchall()]

    async def get_index_usage_stats(self) -> List[dict]:
        """取得索引使用統計"""
        query = text("""
        SELECT
            schemaname,
            tablename,
            indexname,
            idx_tup_read,
            idx_tup_fetch,
            idx_scan,
            pg_size_pretty(pg_relation_size(indexrelid)) as size
        FROM pg_stat_user_indexes
        ORDER BY idx_scan DESC;
        """)

        result = await self.session.execute(query)
        return [dict(row) for row in result.fetchall()]

class QueryOptimizer:
    """查詢優化器"""

    @staticmethod
    def optimize_user_queries():
        """優化使用者相關查詢"""
        # 使用索引的高效查詢
        efficient_queries = {
            "find_active_users": """
                SELECT u.* FROM users u
                WHERE u.is_active = true
                AND u.is_deleted = false
                ORDER BY u.created_at DESC
                LIMIT :limit
            """,

            "find_users_by_email": """
                SELECT u.* FROM users u
                WHERE u.email = :email
                AND u.is_deleted = false
            """,

            "get_user_login_history": """
                SELECT lh.* FROM login_history lh
                WHERE lh.user_id = :user_id
                AND lh.created_at >= :since
                ORDER BY lh.created_at DESC
                LIMIT :limit
            """,

            "count_active_sessions": """
                SELECT COUNT(*) FROM user_sessions us
                WHERE us.user_id = :user_id
                AND us.expires_at > NOW()
                AND us.is_deleted = false
            """
        }

        return efficient_queries

    @staticmethod
    def create_optimized_indexes():
        """建立優化索引的 SQL"""
        return [
            # 複合索引用於常見查詢
            "CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_users_active_email ON users (is_active, email) WHERE is_deleted = false",

            # 部分索引用於活躍使用者
            "CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_users_active_created ON users (created_at) WHERE is_active = true AND is_deleted = false",

            # 會話過期查詢優化
            "CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_sessions_user_expires ON user_sessions (user_id, expires_at) WHERE is_deleted = false",

            # 登入歷史時間範圍查詢
            "CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_login_history_user_time ON login_history (user_id, created_at DESC)",

            # 失敗登入查詢
            "CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_login_history_failed ON login_history (ip_address, created_at) WHERE success = false",
        ]

class ConnectionPoolOptimizer:
    """連接池優化器"""

    def __init__(self):
        self.metrics = {
            "pool_size": [],
            "overflow": [],
            "checked_out": [],
            "response_times": []
        }

    async def analyze_pool_performance(self, db_manager: DatabaseManager) -> dict:
        """分析連接池效能"""
        stats = await db_manager.get_connection_stats()

        # 記錄指標
        self.metrics["pool_size"].append(stats["pool_size"])
        self.metrics["overflow"].append(stats["overflow"])
        self.metrics["checked_out"].append(stats["checked_out"])

        # 計算建議
        recommendations = []

        # 連接池溢出檢查
        if stats["overflow"] > 0:
            recommendations.append("Consider increasing pool_size - overflow detected")

        # 連接池利用率檢查
        utilization = stats["checked_out"] / stats["pool_size"] if stats["pool_size"] > 0 else 0
        if utilization > 0.8:
            recommendations.append("High pool utilization - consider increasing pool_size")
        elif utilization < 0.2:
            recommendations.append("Low pool utilization - consider decreasing pool_size")

        return {
            "current_stats": stats,
            "utilization": utilization,
            "recommendations": recommendations
        }

    def get_optimal_pool_config(self, concurrent_users: int, avg_query_time: float) -> dict:
        """計算最佳連接池配置"""
        # 基於經驗公式計算
        base_pool_size = min(max(concurrent_users // 10, 5), 50)
        max_overflow = base_pool_size // 2

        # 根據查詢時間調整
        if avg_query_time > 1.0:  # 慢查詢需要更大的池
            base_pool_size = int(base_pool_size * 1.5)

        return {
            "pool_size": base_pool_size,
            "max_overflow": max_overflow,
            "pool_timeout": 30,
            "pool_recycle": 3600  # 1 小時
        }
```

---

## 💾 資料備份與災難恢復

### 自動化備份策略

```python
# scripts/backup/database_backup.py
class DatabaseBackupManager:
    """資料庫備份管理器"""

    def __init__(self, config: BackupConfig):
        self.config = config
        self.s3_client = boto3.client('s3')
        self.encryption_key = self._load_encryption_key()

    async def create_full_backup(self) -> BackupResult:
        """建立完整備份"""
        backup_id = f"full-{datetime.utcnow().strftime('%Y%m%d-%H%M%S')}"

        try:
            # 1. 建立一致性快照
            snapshot_info = await self._create_consistent_snapshot()

            # 2. 匯出資料庫
            dump_file = await self._export_database(backup_id)

            # 3. 加密備份檔案
            encrypted_file = await self._encrypt_backup(dump_file)

            # 4. 上傳到雲端儲存
            s3_key = await self._upload_to_s3(encrypted_file, backup_id)

            # 5. 驗證備份完整性
            await self._verify_backup_integrity(s3_key)

            # 6. 更新備份清單
            backup_metadata = {
                "backup_id": backup_id,
                "type": "full",
                "timestamp": datetime.utcnow().isoformat(),
                "size_bytes": os.path.getsize(encrypted_file),
                "s3_key": s3_key,
                "checksum": self._calculate_checksum(encrypted_file),
                "schema_version": await self._get_schema_version()
            }

            await self._update_backup_registry(backup_metadata)

            # 7. 清理本地檔案
            os.remove(dump_file)
            os.remove(encrypted_file)

            return BackupResult(success=True, backup_id=backup_id, metadata=backup_metadata)

        except Exception as e:
            await self._cleanup_failed_backup(backup_id)
            raise BackupError(f"Full backup failed: {str(e)}")

    async def create_incremental_backup(self, base_backup_id: str) -> BackupResult:
        """建立增量備份"""
        backup_id = f"inc-{datetime.utcnow().strftime('%Y%m%d-%H%M%S')}"

        try:
            # 1. 取得上次備份的時間點
            base_backup = await self._get_backup_metadata(base_backup_id)
            last_backup_time = base_backup["timestamp"]

            # 2. 匯出增量變更
            changes_file = await self._export_incremental_changes(backup_id, last_backup_time)

            # 3. WAL 檔案備份（PostgreSQL）
            wal_files = await self._backup_wal_files(last_backup_time)

            # 4. 建立增量包
            incremental_package = await self._create_incremental_package(changes_file, wal_files)

            # 5. 加密並上傳
            encrypted_package = await self._encrypt_backup(incremental_package)
            s3_key = await self._upload_to_s3(encrypted_package, backup_id)

            # 6. 記錄增量備份資訊
            backup_metadata = {
                "backup_id": backup_id,
                "type": "incremental",
                "base_backup_id": base_backup_id,
                "timestamp": datetime.utcnow().isoformat(),
                "size_bytes": os.path.getsize(encrypted_package),
                "s3_key": s3_key,
                "changes_count": await self._count_changes(changes_file)
            }

            await self._update_backup_registry(backup_metadata)

            return BackupResult(success=True, backup_id=backup_id, metadata=backup_metadata)

        except Exception as e:
            await self._cleanup_failed_backup(backup_id)
            raise BackupError(f"Incremental backup failed: {str(e)}")

    async def restore_from_backup(self, backup_id: str, target_db: str, point_in_time: datetime = None):
        """從備份恢復"""
        try:
            backup_metadata = await self._get_backup_metadata(backup_id)

            if backup_metadata["type"] == "full":
                await self._restore_full_backup(backup_metadata, target_db)
            else:
                # 增量恢復需要找到基礎備份
                base_backup = await self._find_base_backup(backup_id)
                await self._restore_incremental_backup(base_backup, backup_metadata, target_db)

            # 點對時恢復
            if point_in_time:
                await self._restore_to_point_in_time(target_db, point_in_time)

            # 驗證恢復結果
            await self._verify_restoration(target_db)

            logger.info(f"Database restored successfully from backup {backup_id}")

        except Exception as e:
            raise RestoreError(f"Restoration failed: {str(e)}")

    async def _export_database(self, backup_id: str) -> str:
        """匯出資料庫"""
        dump_file = f"/tmp/backup_{backup_id}.sql"

        # PostgreSQL pg_dump
        cmd = [
            "pg_dump",
            "--host", self.config.db_host,
            "--port", str(self.config.db_port),
            "--username", self.config.db_user,
            "--dbname", self.config.db_name,
            "--format=custom",
            "--compress=9",
            "--verbose",
            "--file", dump_file
        ]

        # 設定環境變數
        env = os.environ.copy()
        env["PGPASSWORD"] = self.config.db_password

        process = await asyncio.create_subprocess_exec(
            *cmd, env=env, stdout=asyncio.subprocess.PIPE, stderr=asyncio.subprocess.PIPE
        )

        stdout, stderr = await process.communicate()

        if process.returncode != 0:
            raise BackupError(f"pg_dump failed: {stderr.decode()}")

        return dump_file

    async def schedule_automatic_backups(self):
        """排程自動備份"""
        scheduler = AsyncIOScheduler()

        # 每日完整備份 (凌晨 2 點)
        scheduler.add_job(
            self.create_full_backup,
            trigger="cron",
            hour=2,
            minute=0,
            id="daily_full_backup"
        )

        # 每 4 小時增量備份
        scheduler.add_job(
            self._create_scheduled_incremental_backup,
            trigger="interval",
            hours=4,
            id="incremental_backup"
        )

        # 每週完整備份驗證
        scheduler.add_job(
            self._verify_all_backups,
            trigger="cron",
            day_of_week=0,  # 星期日
            hour=1,
            id="weekly_backup_verification"
        )

        scheduler.start()

class BackupRetentionManager:
    """備份保留管理器"""

    def __init__(self, config: RetentionConfig):
        self.config = config

    async def apply_retention_policy(self):
        """應用保留策略"""
        all_backups = await self._get_all_backups()

        # 依據策略分類備份
        retention_groups = {
            "daily": timedelta(days=self.config.daily_retention_days),
            "weekly": timedelta(weeks=self.config.weekly_retention_weeks),
            "monthly": timedelta(days=self.config.monthly_retention_months * 30),
            "yearly": timedelta(days=self.config.yearly_retention_years * 365)
        }

        for group, retention_period in retention_groups.items():
            expired_backups = await self._find_expired_backups(all_backups, retention_period, group)

            for backup in expired_backups:
                await self._delete_backup(backup["backup_id"])
                logger.info(f"Deleted expired backup {backup['backup_id']} from {group} group")

    async def _find_expired_backups(self, all_backups: List[dict], retention_period: timedelta, group: str) -> List[dict]:
        """尋找過期的備份"""
        cutoff_date = datetime.utcnow() - retention_period

        expired = []
        for backup in all_backups:
            backup_date = datetime.fromisoformat(backup["timestamp"])
            if backup_date < cutoff_date and backup.get("retention_group") == group:
                expired.append(backup)

        return expired
```

---

## 🔍 資料庫監控與告警

### 即時監控系統

```python
# app/monitoring/database_monitor.py
class DatabaseMonitor:
    """資料庫監控系統"""

    def __init__(self, db_manager: DatabaseManager):
        self.db_manager = db_manager
        self.metrics_collector = PrometheusMetrics()
        self.alert_manager = AlertManager()

    async def start_monitoring(self):
        """開始監控"""
        # 啟動各種監控任務
        await asyncio.gather(
            self._monitor_connection_pool(),
            self._monitor_query_performance(),
            self._monitor_database_size(),
            self._monitor_replication_lag(),
            self._monitor_lock_waits()
        )

    async def _monitor_connection_pool(self):
        """監控連接池"""
        while True:
            try:
                stats = await self.db_manager.get_connection_stats()

                # 記錄指標
                self.metrics_collector.gauge("db_pool_size", stats["pool_size"])
                self.metrics_collector.gauge("db_pool_checked_out", stats["checked_out"])
                self.metrics_collector.gauge("db_pool_overflow", stats["overflow"])

                # 檢查告警條件
                if stats["overflow"] > 0:
                    await self.alert_manager.send_alert(
                        severity="warning",
                        title="Database Pool Overflow",
                        description=f"Pool overflow detected: {stats['overflow']} connections"
                    )

                utilization = stats["checked_out"] / stats["pool_size"] if stats["pool_size"] > 0 else 0
                if utilization > 0.9:
                    await self.alert_manager.send_alert(
                        severity="critical",
                        title="High Database Pool Utilization",
                        description=f"Pool utilization: {utilization:.2%}"
                    )

                await asyncio.sleep(30)

            except Exception as e:
                logger.error("Connection pool monitoring failed", error=str(e))
                await asyncio.sleep(60)

    async def _monitor_query_performance(self):
        """監控查詢效能"""
        while True:
            try:
                async with self.db_manager.get_session() as session:
                    # 取得慢查詢統計
                    slow_queries = await self._get_slow_queries(session)

                    for query in slow_queries:
                        self.metrics_collector.histogram(
                            "db_query_duration",
                            query["mean_time"],
                            labels={"query_type": query["query_type"]}
                        )

                        # 慢查詢告警
                        if query["mean_time"] > 5000:  # 5 秒
                            await self.alert_manager.send_alert(
                                severity="warning",
                                title="Slow Query Detected",
                                description=f"Query taking {query['mean_time']}ms on average"
                            )

                    # 監控鎖等待
                    lock_waits = await self._get_lock_waits(session)
                    self.metrics_collector.gauge("db_lock_waits", len(lock_waits))

                    if len(lock_waits) > 10:
                        await self.alert_manager.send_alert(
                            severity="critical",
                            title="High Lock Contention",
                            description=f"{len(lock_waits)} queries waiting for locks"
                        )

                await asyncio.sleep(60)

            except Exception as e:
                logger.error("Query performance monitoring failed", error=str(e))
                await asyncio.sleep(120)

    async def _get_slow_queries(self, session: AsyncSession) -> List[dict]:
        """取得慢查詢統計"""
        # PostgreSQL pg_stat_statements
        query = text("""
        SELECT
            query,
            calls,
            total_time,
            mean_time,
            min_time,
            max_time,
            stddev_time,
            rows
        FROM pg_stat_statements
        WHERE mean_time > 100  -- 超過 100ms
        ORDER BY mean_time DESC
        LIMIT 20;
        """)

        result = await session.execute(query)
        return [dict(row) for row in result.fetchall()]

    async def _monitor_database_size(self):
        """監控資料庫大小"""
        while True:
            try:
                async with self.db_manager.get_session() as session:
                    # 資料庫大小
                    db_size = await self._get_database_size(session)
                    self.metrics_collector.gauge("db_size_bytes", db_size["size_bytes"])

                    # 表格大小
                    table_sizes = await self._get_table_sizes(session)
                    for table in table_sizes:
                        self.metrics_collector.gauge(
                            "db_table_size_bytes",
                            table["size_bytes"],
                            labels={"table": table["table_name"]}
                        )

                    # 檢查磁碟空間告警
                    if db_size["size_bytes"] > 50 * 1024**3:  # 50GB
                        await self.alert_manager.send_alert(
                            severity="warning",
                            title="Database Size Alert",
                            description=f"Database size: {db_size['size_gb']:.1f}GB"
                        )

                await asyncio.sleep(3600)  # 每小時檢查

            except Exception as e:
                logger.error("Database size monitoring failed", error=str(e))
                await asyncio.sleep(3600)

    async def generate_health_report(self) -> dict:
        """生成健康報告"""
        async with self.db_manager.get_session() as session:
            report = {
                "timestamp": datetime.utcnow().isoformat(),
                "connection_pool": await self.db_manager.get_connection_stats(),
                "database_size": await self._get_database_size(session),
                "slow_queries": await self._get_slow_queries(session),
                "lock_waits": await self._get_lock_waits(session),
                "index_usage": await self._get_index_usage_stats(session),
                "replication_status": await self._get_replication_status(session)
            }

        return report

class DatabaseAlerting:
    """資料庫告警系統"""

    def __init__(self):
        self.alert_rules = self._load_alert_rules()
        self.notification_channels = self._setup_notification_channels()

    def _load_alert_rules(self) -> List[AlertRule]:
        """載入告警規則"""
        return [
            AlertRule(
                name="high_connection_usage",
                condition="db_pool_utilization > 0.8",
                severity="warning",
                duration="5m"
            ),
            AlertRule(
                name="query_timeout",
                condition="db_query_duration > 30s",
                severity="critical",
                duration="1m"
            ),
            AlertRule(
                name="replication_lag",
                condition="db_replication_lag > 60s",
                severity="warning",
                duration="2m"
            ),
            AlertRule(
                name="deadlock_detected",
                condition="rate(db_deadlocks_total[5m]) > 0",
                severity="critical",
                duration="0s"
            )
        ]

    async def evaluate_rules(self, metrics: dict):
        """評估告警規則"""
        for rule in self.alert_rules:
            if await self._evaluate_condition(rule.condition, metrics):
                await self._fire_alert(rule, metrics)

    async def _fire_alert(self, rule: AlertRule, metrics: dict):
        """觸發告警"""
        alert = Alert(
            rule_name=rule.name,
            severity=rule.severity,
            message=self._format_alert_message(rule, metrics),
            timestamp=datetime.utcnow(),
            metrics=metrics
        )

        # 發送到各個通知通道
        for channel in self.notification_channels:
            await channel.send_alert(alert)
```

---

## 💡 學習筆記區

### 🤔 我的理解
```
SQLAlchemy 的進階特性：

資料庫遷移的最佳實務：

查詢優化的關鍵策略：

資料庫監控的重要指標：
```

### 💾 實踐心得
```
設計資料庫架構的考量：

效能優化的實戰經驗：

備份策略的重要性：

生產環境的運維挑戰：
```

### 🚀 進階思考
```
分散式資料庫的挑戰：

雲原生資料庫的優勢：

資料庫安全的考量：

未來資料庫技術趨勢：
```