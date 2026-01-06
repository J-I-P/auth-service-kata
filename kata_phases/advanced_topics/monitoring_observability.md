# Monitoring & Observability: 企業級可觀測性架構與實踐

## 🎯 學習目標
- 掌握現代可觀測性的三大支柱：日誌、指標、追蹤
- 學習建立全面的監控與告警系統
- 實踐 SRE (Site Reliability Engineering) 最佳實務
- 建立故障檢測、診斷與恢復機制

---

## 📊 可觀測性三大支柱

### 結構化日誌 (Structured Logging)

```python
# logging/structured_logger.py
import structlog
import logging.config
from typing import Any, Dict, Optional
import json
import traceback
from datetime import datetime

class StructuredLogger:
    """結構化日誌管理器"""

    def __init__(self, service_name: str, environment: str, log_level: str = "INFO"):
        self.service_name = service_name
        self.environment = environment

        # 配置 structlog
        structlog.configure(
            processors=[
                structlog.stdlib.filter_by_level,
                structlog.stdlib.add_logger_name,
                structlog.stdlib.add_log_level,
                structlog.stdlib.PositionalArgumentsFormatter(),
                self._add_service_context,
                structlog.processors.TimeStamper(fmt="iso"),
                structlog.dev.ConsoleRenderer() if environment == "development"
                else structlog.processors.JSONRenderer()
            ],
            context_class=dict,
            logger_factory=structlog.stdlib.LoggerFactory(),
            wrapper_class=structlog.stdlib.BoundLogger,
            cache_logger_on_first_use=True,
        )

        # 設定標準日誌等級
        logging.basicConfig(level=getattr(logging, log_level.upper()))

    def _add_service_context(self, logger, method_name, event_dict):
        """添加服務上下文資訊"""
        event_dict["service"] = self.service_name
        event_dict["environment"] = self.environment
        event_dict["version"] = self._get_service_version()
        return event_dict

    def _get_service_version(self) -> str:
        """取得服務版本（從環境變數或文件）"""
        import os
        return os.environ.get("SERVICE_VERSION", "unknown")

    def get_logger(self, name: str = None):
        """取得結構化 logger"""
        return structlog.get_logger(name or self.service_name)

class ApplicationLogger:
    """應用程式日誌記錄器"""

    def __init__(self, structured_logger: StructuredLogger):
        self.logger = structured_logger.get_logger()

    def log_request(self, method: str, path: str, status_code: int,
                   response_time: float, user_id: str = None, **kwargs):
        """記錄 HTTP 請求日誌"""
        self.logger.info(
            "HTTP request completed",
            http_method=method,
            http_path=path,
            http_status=status_code,
            response_time_ms=response_time * 1000,
            user_id=user_id,
            **kwargs
        )

    def log_business_event(self, event_type: str, user_id: str = None,
                          metadata: Dict[str, Any] = None):
        """記錄業務事件日誌"""
        self.logger.info(
            "Business event",
            event_type=event_type,
            user_id=user_id,
            metadata=metadata or {}
        )

    def log_security_event(self, event_type: str, severity: str,
                          details: Dict[str, Any] = None):
        """記錄安全事件日誌"""
        self.logger.warning(
            "Security event",
            security_event_type=event_type,
            severity=severity,
            details=details or {}
        )

    def log_error(self, error: Exception, context: Dict[str, Any] = None):
        """記錄錯誤日誌"""
        self.logger.error(
            "Application error",
            error_type=error.__class__.__name__,
            error_message=str(error),
            error_traceback=traceback.format_exc(),
            context=context or {}
        )

    def log_performance_metric(self, operation: str, duration: float,
                              success: bool, **kwargs):
        """記錄效能指標日誌"""
        self.logger.info(
            "Performance metric",
            operation=operation,
            duration_ms=duration * 1000,
            success=success,
            **kwargs
        )

class AuditLogger:
    """審計日誌記錄器"""

    def __init__(self, structured_logger: StructuredLogger):
        self.logger = structured_logger.get_logger("audit")

    def log_data_access(self, user_id: str, resource_type: str, resource_id: str,
                       action: str, success: bool, ip_address: str = None):
        """記錄資料存取審計"""
        self.logger.info(
            "Data access",
            audit_type="data_access",
            user_id=user_id,
            resource_type=resource_type,
            resource_id=resource_id,
            action=action,
            success=success,
            client_ip=ip_address
        )

    def log_privilege_escalation(self, user_id: str, old_role: str, new_role: str,
                                granted_by: str):
        """記錄權限提升審計"""
        self.logger.warning(
            "Privilege escalation",
            audit_type="privilege_change",
            user_id=user_id,
            old_role=old_role,
            new_role=new_role,
            granted_by=granted_by
        )

    def log_configuration_change(self, user_id: str, config_key: str,
                                old_value: str, new_value: str):
        """記錄配置變更審計"""
        self.logger.info(
            "Configuration change",
            audit_type="config_change",
            user_id=user_id,
            config_key=config_key,
            old_value=old_value,
            new_value=new_value
        )

class LogAggregator:
    """日誌聚合器（ELK Stack 整合）"""

    def __init__(self, elasticsearch_url: str, index_pattern: str = "app-logs"):
        self.elasticsearch_url = elasticsearch_url
        self.index_pattern = index_pattern

    async def query_logs(self, query: Dict[str, Any], start_time: datetime,
                        end_time: datetime, size: int = 100) -> List[Dict[str, Any]]:
        """查詢日誌"""
        from elasticsearch import AsyncElasticsearch

        es = AsyncElasticsearch([self.elasticsearch_url])

        search_body = {
            "query": {
                "bool": {
                    "must": [query],
                    "filter": [
                        {
                            "range": {
                                "@timestamp": {
                                    "gte": start_time.isoformat(),
                                    "lte": end_time.isoformat()
                                }
                            }
                        }
                    ]
                }
            },
            "sort": [{"@timestamp": {"order": "desc"}}],
            "size": size
        }

        result = await es.search(
            index=f"{self.index_pattern}-*",
            body=search_body
        )

        await es.close()

        return [hit["_source"] for hit in result["hits"]["hits"]]

    async def aggregate_logs(self, aggregation: Dict[str, Any],
                           start_time: datetime, end_time: datetime) -> Dict[str, Any]:
        """聚合日誌統計"""
        from elasticsearch import AsyncElasticsearch

        es = AsyncElasticsearch([self.elasticsearch_url])

        search_body = {
            "query": {
                "range": {
                    "@timestamp": {
                        "gte": start_time.isoformat(),
                        "lte": end_time.isoformat()
                    }
                }
            },
            "aggs": aggregation,
            "size": 0
        }

        result = await es.search(
            index=f"{self.index_pattern}-*",
            body=search_body
        )

        await es.close()

        return result["aggregations"]

    async def get_error_trends(self, hours: int = 24) -> Dict[str, Any]:
        """取得錯誤趨勢"""
        end_time = datetime.utcnow()
        start_time = end_time - timedelta(hours=hours)

        aggregation = {
            "error_over_time": {
                "date_histogram": {
                    "field": "@timestamp",
                    "interval": "1h"
                },
                "aggs": {
                    "error_count": {
                        "filter": {
                            "term": {"level": "error"}
                        }
                    }
                }
            },
            "top_errors": {
                "filter": {
                    "term": {"level": "error"}
                },
                "aggs": {
                    "error_types": {
                        "terms": {
                            "field": "error_type.keyword",
                            "size": 10
                        }
                    }
                }
            }
        }

        return await self.aggregate_logs(aggregation, start_time, end_time)
```

### 指標收集與監控 (Metrics)

```python
# metrics/prometheus_metrics.py
from prometheus_client import Counter, Histogram, Gauge, Summary, CollectorRegistry, push_to_gateway
from typing import Dict, Any, List
import time
import psutil
import asyncio

class PrometheusMetrics:
    """Prometheus 指標收集器"""

    def __init__(self, namespace: str = "auth_service"):
        self.namespace = namespace
        self.registry = CollectorRegistry()

        # HTTP 請求指標
        self.http_requests_total = Counter(
            'http_requests_total',
            'Total HTTP requests',
            ['method', 'endpoint', 'status_code'],
            namespace=namespace,
            registry=self.registry
        )

        self.http_request_duration = Histogram(
            'http_request_duration_seconds',
            'HTTP request duration',
            ['method', 'endpoint'],
            namespace=namespace,
            registry=self.registry
        )

        # 業務指標
        self.user_registrations_total = Counter(
            'user_registrations_total',
            'Total user registrations',
            namespace=namespace,
            registry=self.registry
        )

        self.login_attempts_total = Counter(
            'login_attempts_total',
            'Total login attempts',
            ['result'],  # success, failure
            namespace=namespace,
            registry=self.registry
        )

        self.active_sessions = Gauge(
            'active_sessions',
            'Number of active user sessions',
            namespace=namespace,
            registry=self.registry
        )

        # 系統指標
        self.system_cpu_usage = Gauge(
            'system_cpu_usage_percent',
            'System CPU usage percentage',
            namespace=namespace,
            registry=self.registry
        )

        self.system_memory_usage = Gauge(
            'system_memory_usage_bytes',
            'System memory usage in bytes',
            namespace=namespace,
            registry=self.registry
        )

        # 資料庫指標
        self.db_connection_pool_size = Gauge(
            'db_connection_pool_size',
            'Database connection pool size',
            namespace=namespace,
            registry=self.registry
        )

        self.db_query_duration = Histogram(
            'db_query_duration_seconds',
            'Database query duration',
            ['query_type'],
            namespace=namespace,
            registry=self.registry
        )

    def record_http_request(self, method: str, endpoint: str, status_code: int, duration: float):
        """記錄 HTTP 請求指標"""
        self.http_requests_total.labels(
            method=method,
            endpoint=endpoint,
            status_code=status_code
        ).inc()

        self.http_request_duration.labels(
            method=method,
            endpoint=endpoint
        ).observe(duration)

    def record_user_registration(self):
        """記錄使用者註冊指標"""
        self.user_registrations_total.inc()

    def record_login_attempt(self, success: bool):
        """記錄登入嘗試指標"""
        result = "success" if success else "failure"
        self.login_attempts_total.labels(result=result).inc()

    def update_active_sessions(self, count: int):
        """更新活躍會話數量"""
        self.active_sessions.set(count)

    def update_system_metrics(self):
        """更新系統指標"""
        # CPU 使用率
        cpu_percent = psutil.cpu_percent()
        self.system_cpu_usage.set(cpu_percent)

        # 記憶體使用率
        memory = psutil.virtual_memory()
        self.system_memory_usage.set(memory.used)

    def record_db_query(self, query_type: str, duration: float):
        """記錄資料庫查詢指標"""
        self.db_query_duration.labels(query_type=query_type).observe(duration)

    def update_db_connection_pool(self, size: int):
        """更新資料庫連接池大小"""
        self.db_connection_pool_size.set(size)

class CustomMetrics:
    """自定義業務指標"""

    def __init__(self, prometheus_metrics: PrometheusMetrics):
        self.prometheus_metrics = prometheus_metrics

        # 自定義指標
        self.failed_login_rate = Gauge(
            'failed_login_rate',
            'Failed login rate in the last 5 minutes',
            namespace=prometheus_metrics.namespace,
            registry=prometheus_metrics.registry
        )

        self.token_generation_rate = Counter(
            'token_generation_total',
            'Total tokens generated',
            ['token_type'],
            namespace=prometheus_metrics.namespace,
            registry=prometheus_metrics.registry
        )

        self.cache_hit_rate = Gauge(
            'cache_hit_rate',
            'Cache hit rate percentage',
            ['cache_type'],
            namespace=prometheus_metrics.namespace,
            registry=prometheus_metrics.registry
        )

    async def calculate_failed_login_rate(self) -> float:
        """計算失敗登入率"""
        # 這裡應該查詢最近 5 分鐘的登入記錄
        # 簡化實作
        total_attempts = 100  # 從日誌或資料庫查詢
        failed_attempts = 15  # 從日誌或資料庫查詢

        rate = failed_attempts / total_attempts if total_attempts > 0 else 0
        self.failed_login_rate.set(rate)
        return rate

    def record_token_generation(self, token_type: str):
        """記錄 token 生成"""
        self.token_generation_rate.labels(token_type=token_type).inc()

    def update_cache_hit_rate(self, cache_type: str, hit_rate: float):
        """更新快取命中率"""
        self.cache_hit_rate.labels(cache_type=cache_type).set(hit_rate)

class MetricsCollector:
    """指標收集器（定期執行）"""

    def __init__(self, prometheus_metrics: PrometheusMetrics, custom_metrics: CustomMetrics):
        self.prometheus_metrics = prometheus_metrics
        self.custom_metrics = custom_metrics
        self.collection_interval = 30  # 30 秒收集一次
        self.running = False

    async def start_collection(self):
        """開始指標收集"""
        self.running = True
        while self.running:
            try:
                await self._collect_metrics()
                await asyncio.sleep(self.collection_interval)
            except Exception as e:
                logger.error("Metrics collection failed", error=str(e))
                await asyncio.sleep(60)  # 失敗時延長間隔

    async def stop_collection(self):
        """停止指標收集"""
        self.running = False

    async def _collect_metrics(self):
        """收集所有指標"""
        # 系統指標
        self.prometheus_metrics.update_system_metrics()

        # 資料庫指標
        await self._collect_database_metrics()

        # 業務指標
        await self._collect_business_metrics()

    async def _collect_database_metrics(self):
        """收集資料庫指標"""
        try:
            # 這裡應該從實際的資料庫連接池取得數據
            pool_size = 20  # 從資料庫管理器取得
            self.prometheus_metrics.update_db_connection_pool(pool_size)

            # 活躍會話數
            active_sessions_count = 150  # 從資料庫查詢
            self.prometheus_metrics.update_active_sessions(active_sessions_count)

        except Exception as e:
            logger.error("Database metrics collection failed", error=str(e))

    async def _collect_business_metrics(self):
        """收集業務指標"""
        try:
            # 計算失敗登入率
            await self.custom_metrics.calculate_failed_login_rate()

            # 其他業務指標...

        except Exception as e:
            logger.error("Business metrics collection failed", error=str(e))

class AlertManager:
    """告警管理器"""

    def __init__(self, prometheus_metrics: PrometheusMetrics, webhook_url: str = None):
        self.prometheus_metrics = prometheus_metrics
        self.webhook_url = webhook_url
        self.alert_rules = self._load_alert_rules()

    def _load_alert_rules(self) -> List[Dict[str, Any]]:
        """載入告警規則"""
        return [
            {
                "name": "high_error_rate",
                "expression": "rate(http_requests_total{status_code=~'5..'}[5m]) > 0.1",
                "for_duration": "2m",
                "severity": "critical",
                "description": "HTTP error rate > 10%"
            },
            {
                "name": "high_response_time",
                "expression": "http_request_duration_seconds{quantile='0.95'} > 2",
                "for_duration": "5m",
                "severity": "warning",
                "description": "95th percentile response time > 2s"
            },
            {
                "name": "high_failed_login_rate",
                "expression": "failed_login_rate > 0.3",
                "for_duration": "1m",
                "severity": "warning",
                "description": "Failed login rate > 30%"
            },
            {
                "name": "low_cache_hit_rate",
                "expression": "cache_hit_rate < 0.7",
                "for_duration": "5m",
                "severity": "warning",
                "description": "Cache hit rate < 70%"
            }
        ]

    async def evaluate_alerts(self):
        """評估告警條件"""
        # 在實際環境中，這會由 Prometheus Alertmanager 處理
        # 這裡提供簡化的實作範例

        for rule in self.alert_rules:
            try:
                # 簡化的條件檢查
                should_alert = await self._evaluate_rule(rule)

                if should_alert:
                    await self._send_alert(rule)

            except Exception as e:
                logger.error("Alert evaluation failed", rule=rule["name"], error=str(e))

    async def _evaluate_rule(self, rule: Dict[str, Any]) -> bool:
        """評估單個告警規則"""
        # 實際實作會查詢 Prometheus 指標
        # 這裡使用簡化邏輯
        return False

    async def _send_alert(self, rule: Dict[str, Any]):
        """發送告警"""
        alert_payload = {
            "alert_name": rule["name"],
            "severity": rule["severity"],
            "description": rule["description"],
            "timestamp": datetime.utcnow().isoformat(),
            "service": "auth-service"
        }

        if self.webhook_url:
            # 發送到 Slack/Teams 等
            import httpx
            async with httpx.AsyncClient() as client:
                await client.post(self.webhook_url, json=alert_payload)

        logger.warning("Alert triggered", alert=alert_payload)
```

### 分散式追蹤 (Distributed Tracing)

```python
# tracing/opentelemetry_setup.py
from opentelemetry import trace, metrics, logs
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.exporter.prometheus import PrometheusMetricReader
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor
from opentelemetry.instrumentation.redis import RedisInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.logs import LoggerProvider
from opentelemetry.semantic_conventions.trace import SpanAttributes
import structlog

logger = structlog.get_logger()

class OpenTelemetrySetup:
    """OpenTelemetry 可觀測性設定"""

    def __init__(self, service_name: str, service_version: str, environment: str):
        self.service_name = service_name
        self.service_version = service_version
        self.environment = environment

        # 設定資源屬性
        from opentelemetry.sdk.resources import Resource
        self.resource = Resource.create({
            "service.name": service_name,
            "service.version": service_version,
            "deployment.environment": environment
        })

    def setup_tracing(self, jaeger_endpoint: str = "http://localhost:14268/api/traces"):
        """設定分散式追蹤"""
        # 建立 Tracer Provider
        tracer_provider = TracerProvider(resource=self.resource)

        # 設定 Jaeger 導出器
        jaeger_exporter = JaegerExporter(endpoint=jaeger_endpoint)
        span_processor = BatchSpanProcessor(jaeger_exporter)
        tracer_provider.add_span_processor(span_processor)

        # 設定全域 tracer
        trace.set_tracer_provider(tracer_provider)

        logger.info("Distributed tracing configured",
                   service=self.service_name,
                   jaeger_endpoint=jaeger_endpoint)

    def setup_metrics(self, prometheus_port: int = 9090):
        """設定指標收集"""
        # 建立 Meter Provider
        metric_reader = PrometheusMetricReader(port=prometheus_port)
        meter_provider = MeterProvider(
            resource=self.resource,
            metric_readers=[metric_reader]
        )

        # 設定全域 meter
        metrics.set_meter_provider(meter_provider)

        logger.info("Metrics collection configured",
                   service=self.service_name,
                   prometheus_port=prometheus_port)

    def setup_logging(self, log_endpoint: str = None):
        """設定日誌收集"""
        # 建立 Logger Provider
        logger_provider = LoggerProvider(resource=self.resource)

        # 設定日誌導出器（如果需要）
        if log_endpoint:
            # 這裡可以添加 OTLP 或其他日誌導出器
            pass

        # 設定全域 logger
        logs.set_logger_provider(logger_provider)

        logger.info("Logging configured", service=self.service_name)

    def setup_auto_instrumentation(self, app):
        """設定自動儀錶化"""
        # FastAPI 自動儀錶化
        FastAPIInstrumentor.instrument_app(app)

        # SQLAlchemy 自動儀錶化
        SQLAlchemyInstrumentor().instrument()

        # Redis 自動儀錶化
        RedisInstrumentor().instrument()

        # HTTP 請求自動儀錶化
        RequestsInstrumentor().instrument()

        logger.info("Auto instrumentation configured",
                   service=self.service_name)

class CustomTracing:
    """自定義追蹤"""

    def __init__(self, service_name: str):
        self.tracer = trace.get_tracer(service_name)

    def trace_business_operation(self, operation_name: str):
        """業務操作追蹤裝飾器"""
        def decorator(func):
            @wraps(func)
            async def wrapper(*args, **kwargs):
                with self.tracer.start_as_current_span(operation_name) as span:
                    # 設定 span 屬性
                    span.set_attribute("operation.name", operation_name)
                    span.set_attribute("operation.type", "business")

                    try:
                        if asyncio.iscoroutinefunction(func):
                            result = await func(*args, **kwargs)
                        else:
                            result = func(*args, **kwargs)

                        span.set_attribute("operation.success", True)
                        return result

                    except Exception as e:
                        span.set_attribute("operation.success", False)
                        span.set_attribute("error.type", e.__class__.__name__)
                        span.set_attribute("error.message", str(e))
                        span.record_exception(e)
                        raise

            return wrapper
        return decorator

    def trace_database_operation(self, table_name: str, operation: str):
        """資料庫操作追蹤裝飾器"""
        def decorator(func):
            @wraps(func)
            async def wrapper(*args, **kwargs):
                with self.tracer.start_as_current_span(f"db.{operation}.{table_name}") as span:
                    # 設定資料庫相關屬性
                    span.set_attribute(SpanAttributes.DB_OPERATION, operation)
                    span.set_attribute(SpanAttributes.DB_SQL_TABLE, table_name)
                    span.set_attribute("db.system", "postgresql")

                    try:
                        result = await func(*args, **kwargs) if asyncio.iscoroutinefunction(func) else func(*args, **kwargs)

                        # 記錄結果資訊
                        if hasattr(result, '__len__'):
                            span.set_attribute("db.rows_affected", len(result))

                        return result

                    except Exception as e:
                        span.set_attribute("error", True)
                        span.record_exception(e)
                        raise

            return wrapper
        return decorator

    def add_user_context(self, user_id: str, span: trace.Span = None):
        """添加使用者上下文"""
        current_span = span or trace.get_current_span()
        if current_span:
            current_span.set_attribute("user.id", user_id)

    def add_business_context(self, **context):
        """添加業務上下文"""
        current_span = trace.get_current_span()
        if current_span:
            for key, value in context.items():
                current_span.set_attribute(f"business.{key}", str(value))

class TraceAnalyzer:
    """追蹤分析器"""

    def __init__(self, jaeger_query_endpoint: str):
        self.jaeger_query_endpoint = jaeger_query_endpoint

    async def get_service_dependencies(self, service_name: str,
                                     lookback_hours: int = 24) -> Dict[str, Any]:
        """取得服務依賴關係"""
        # 查詢 Jaeger 取得服務依賴
        end_time = datetime.utcnow()
        start_time = end_time - timedelta(hours=lookback_hours)

        dependencies = {
            "service": service_name,
            "dependencies": [
                {"service": "user-service", "call_count": 1250},
                {"service": "notification-service", "call_count": 890},
                {"service": "redis", "call_count": 3420}
            ],
            "dependents": [
                {"service": "web-app", "call_count": 2100},
                {"service": "mobile-app", "call_count": 1850}
            ]
        }

        return dependencies

    async def analyze_trace_performance(self, trace_id: str) -> Dict[str, Any]:
        """分析追蹤效能"""
        # 查詢特定追蹤的詳細資訊
        performance_analysis = {
            "trace_id": trace_id,
            "total_duration": 1250,  # ms
            "span_count": 8,
            "service_breakdown": {
                "auth-service": 150,
                "user-service": 300,
                "database": 800
            },
            "bottlenecks": [
                {
                    "span_name": "db.query.users",
                    "duration": 800,
                    "percentage": 64
                }
            ]
        }

        return performance_analysis

    async def get_error_traces(self, service_name: str, hours: int = 1) -> List[Dict[str, Any]]:
        """取得錯誤追蹤"""
        error_traces = [
            {
                "trace_id": "abc123",
                "timestamp": "2024-01-06T10:30:00Z",
                "error_type": "DatabaseConnectionError",
                "service": service_name,
                "operation": "user_login",
                "duration": 5000
            }
        ]

        return error_traces
```

---

## 🚨 告警與事件回應

### 智能告警系統

```python
# alerting/alert_manager.py
from dataclasses import dataclass
from enum import Enum
from typing import List, Dict, Any, Optional
import asyncio
from datetime import datetime, timedelta

class AlertSeverity(Enum):
    INFO = "info"
    WARNING = "warning"
    CRITICAL = "critical"

class AlertStatus(Enum):
    OPEN = "open"
    ACKNOWLEDGED = "acknowledged"
    RESOLVED = "resolved"

@dataclass
class Alert:
    """告警資料結構"""
    alert_id: str
    alert_name: str
    severity: AlertSeverity
    status: AlertStatus
    service: str
    description: str
    metric_name: str
    current_value: float
    threshold_value: float
    timestamp: datetime
    resolved_at: Optional[datetime] = None
    acknowledged_by: Optional[str] = None
    metadata: Dict[str, Any] = None

class AlertRule:
    """告警規則"""

    def __init__(self, name: str, expression: str, threshold: float,
                 severity: AlertSeverity, for_duration: int = 300):
        self.name = name
        self.expression = expression
        self.threshold = threshold
        self.severity = severity
        self.for_duration = for_duration  # 持續時間（秒）
        self.last_triggered = None
        self.triggered_count = 0

    async def evaluate(self, current_value: float) -> bool:
        """評估告警條件"""
        if current_value > self.threshold:
            if self.last_triggered is None:
                self.last_triggered = datetime.utcnow()
                return False
            elif (datetime.utcnow() - self.last_triggered).total_seconds() >= self.for_duration:
                self.triggered_count += 1
                return True
        else:
            self.last_triggered = None

        return False

class IntelligentAlertManager:
    """智能告警管理器"""

    def __init__(self):
        self.rules = {}
        self.active_alerts = {}
        self.alert_history = []
        self.notification_channels = {}

    def add_rule(self, rule: AlertRule):
        """添加告警規則"""
        self.rules[rule.name] = rule

    def add_notification_channel(self, name: str, channel):
        """添加通知通道"""
        self.notification_channels[name] = channel

    async def process_metric(self, metric_name: str, value: float, service: str = "unknown"):
        """處理指標並檢查告警"""
        for rule_name, rule in self.rules.items():
            if metric_name in rule.expression:
                should_alert = await rule.evaluate(value)

                if should_alert:
                    await self._trigger_alert(rule, metric_name, value, service)

    async def _trigger_alert(self, rule: AlertRule, metric_name: str,
                           current_value: float, service: str):
        """觸發告警"""
        alert_id = f"{rule.name}_{service}_{int(datetime.utcnow().timestamp())}"

        # 檢查是否為重複告警
        existing_alert_key = f"{rule.name}_{service}"
        if existing_alert_key in self.active_alerts:
            # 更新現有告警
            existing_alert = self.active_alerts[existing_alert_key]
            existing_alert.current_value = current_value
            existing_alert.timestamp = datetime.utcnow()
            return

        # 建立新告警
        alert = Alert(
            alert_id=alert_id,
            alert_name=rule.name,
            severity=rule.severity,
            status=AlertStatus.OPEN,
            service=service,
            description=f"Metric {metric_name} exceeded threshold",
            metric_name=metric_name,
            current_value=current_value,
            threshold_value=rule.threshold,
            timestamp=datetime.utcnow(),
            metadata={
                "rule_expression": rule.expression,
                "triggered_count": rule.triggered_count
            }
        )

        self.active_alerts[existing_alert_key] = alert
        self.alert_history.append(alert)

        # 發送通知
        await self._send_notifications(alert)

    async def _send_notifications(self, alert: Alert):
        """發送告警通知"""
        # 根據嚴重程度選擇通知通道
        channels = []

        if alert.severity == AlertSeverity.CRITICAL:
            channels.extend(["slack", "pagerduty", "email"])
        elif alert.severity == AlertSeverity.WARNING:
            channels.extend(["slack", "email"])
        else:
            channels.append("slack")

        for channel_name in channels:
            if channel_name in self.notification_channels:
                try:
                    channel = self.notification_channels[channel_name]
                    await channel.send_alert(alert)
                except Exception as e:
                    logger.error(f"Failed to send alert via {channel_name}", error=str(e))

    async def acknowledge_alert(self, alert_id: str, user_id: str):
        """確認告警"""
        for key, alert in self.active_alerts.items():
            if alert.alert_id == alert_id:
                alert.status = AlertStatus.ACKNOWLEDGED
                alert.acknowledged_by = user_id
                logger.info("Alert acknowledged", alert_id=alert_id, user_id=user_id)
                break

    async def resolve_alert(self, alert_id: str, user_id: str = None):
        """解決告警"""
        for key, alert in list(self.active_alerts.items()):
            if alert.alert_id == alert_id:
                alert.status = AlertStatus.RESOLVED
                alert.resolved_at = datetime.utcnow()
                del self.active_alerts[key]
                logger.info("Alert resolved", alert_id=alert_id, user_id=user_id)
                break

    async def get_active_alerts(self) -> List[Alert]:
        """取得活躍告警"""
        return list(self.active_alerts.values())

    async def get_alert_summary(self) -> Dict[str, Any]:
        """取得告警摘要"""
        active_alerts = await self.get_active_alerts()

        severity_counts = {
            AlertSeverity.CRITICAL: 0,
            AlertSeverity.WARNING: 0,
            AlertSeverity.INFO: 0
        }

        for alert in active_alerts:
            severity_counts[alert.severity] += 1

        return {
            "total_active": len(active_alerts),
            "by_severity": {
                "critical": severity_counts[AlertSeverity.CRITICAL],
                "warning": severity_counts[AlertSeverity.WARNING],
                "info": severity_counts[AlertSeverity.INFO]
            },
            "latest_alerts": active_alerts[:5]  # 最新 5 個告警
        }

class NotificationChannel:
    """通知通道抽象基類"""

    async def send_alert(self, alert: Alert):
        raise NotImplementedError

class SlackChannel(NotificationChannel):
    """Slack 通知通道"""

    def __init__(self, webhook_url: str):
        self.webhook_url = webhook_url

    async def send_alert(self, alert: Alert):
        """發送 Slack 通知"""
        color_map = {
            AlertSeverity.CRITICAL: "danger",
            AlertSeverity.WARNING: "warning",
            AlertSeverity.INFO: "good"
        }

        payload = {
            "attachments": [
                {
                    "color": color_map[alert.severity],
                    "title": f"🚨 {alert.alert_name}",
                    "text": alert.description,
                    "fields": [
                        {
                            "title": "Service",
                            "value": alert.service,
                            "short": True
                        },
                        {
                            "title": "Severity",
                            "value": alert.severity.value.upper(),
                            "short": True
                        },
                        {
                            "title": "Current Value",
                            "value": str(alert.current_value),
                            "short": True
                        },
                        {
                            "title": "Threshold",
                            "value": str(alert.threshold_value),
                            "short": True
                        }
                    ],
                    "footer": "Monitoring System",
                    "ts": int(alert.timestamp.timestamp())
                }
            ]
        }

        import httpx
        async with httpx.AsyncClient() as client:
            response = await client.post(self.webhook_url, json=payload)
            response.raise_for_status()

class EmailChannel(NotificationChannel):
    """郵件通知通道"""

    def __init__(self, smtp_config: Dict[str, Any]):
        self.smtp_config = smtp_config

    async def send_alert(self, alert: Alert):
        """發送郵件通知"""
        subject = f"[{alert.severity.value.upper()}] {alert.alert_name} - {alert.service}"

        body = f"""
        Alert: {alert.alert_name}
        Service: {alert.service}
        Severity: {alert.severity.value.upper()}
        Description: {alert.description}

        Metric: {alert.metric_name}
        Current Value: {alert.current_value}
        Threshold: {alert.threshold_value}

        Timestamp: {alert.timestamp.isoformat()}
        Alert ID: {alert.alert_id}
        """

        # 實際發送郵件邏輯
        await self._send_email(subject, body)

    async def _send_email(self, subject: str, body: str):
        """發送郵件"""
        # 使用 aiosmtplib 或其他異步郵件庫
        pass

class PagerDutyChannel(NotificationChannel):
    """PagerDuty 通知通道"""

    def __init__(self, integration_key: str):
        self.integration_key = integration_key

    async def send_alert(self, alert: Alert):
        """發送 PagerDuty 通知"""
        payload = {
            "routing_key": self.integration_key,
            "event_action": "trigger",
            "dedup_key": f"{alert.service}_{alert.alert_name}",
            "payload": {
                "summary": alert.description,
                "source": alert.service,
                "severity": "critical" if alert.severity == AlertSeverity.CRITICAL else "warning",
                "component": alert.service,
                "group": alert.metric_name,
                "custom_details": {
                    "current_value": alert.current_value,
                    "threshold": alert.threshold_value,
                    "alert_id": alert.alert_id
                }
            }
        }

        import httpx
        async with httpx.AsyncClient() as client:
            response = await client.post(
                "https://events.pagerduty.com/v2/enqueue",
                json=payload
            )
            response.raise_for_status()
```

---

## 📈 SRE 與可靠性工程

### SLI/SLO/SLA 管理

```python
# sre/sli_slo_manager.py
from dataclasses import dataclass
from typing import Dict, List, Any, Optional
from datetime import datetime, timedelta
from enum import Enum

class SLIType(Enum):
    AVAILABILITY = "availability"
    LATENCY = "latency"
    ERROR_RATE = "error_rate"
    THROUGHPUT = "throughput"

@dataclass
class SLI:
    """服務水平指標 (Service Level Indicator)"""
    name: str
    type: SLIType
    description: str
    query: str
    unit: str
    good_threshold: float = None
    bad_threshold: float = None

@dataclass
class SLO:
    """服務水平目標 (Service Level Objective)"""
    name: str
    description: str
    sli: SLI
    target_percentage: float  # 99.9%
    time_window_days: int     # 30天
    alert_threshold: float    # 98.0% (提前告警)

@dataclass
class SLA:
    """服務水平協議 (Service Level Agreement)"""
    name: str
    description: str
    slos: List[SLO]
    penalty_percentage: float  # 違約金比例
    measurement_period: str    # "monthly", "quarterly"

class SREMetricsManager:
    """SRE 指標管理器"""

    def __init__(self):
        self.slis = {}
        self.slos = {}
        self.slas = {}
        self.measurements = {}

    def register_sli(self, sli: SLI):
        """註冊 SLI"""
        self.slis[sli.name] = sli

    def register_slo(self, slo: SLO):
        """註冊 SLO"""
        self.slos[slo.name] = slo

    def register_sla(self, sla: SLA):
        """註冊 SLA"""
        self.slas[sla.name] = sla

    async def calculate_sli_value(self, sli_name: str, start_time: datetime,
                                 end_time: datetime) -> float:
        """計算 SLI 值"""
        sli = self.slis[sli_name]

        if sli.type == SLIType.AVAILABILITY:
            return await self._calculate_availability(sli, start_time, end_time)
        elif sli.type == SLIType.LATENCY:
            return await self._calculate_latency(sli, start_time, end_time)
        elif sli.type == SLIType.ERROR_RATE:
            return await self._calculate_error_rate(sli, start_time, end_time)
        elif sli.type == SLIType.THROUGHPUT:
            return await self._calculate_throughput(sli, start_time, end_time)

    async def _calculate_availability(self, sli: SLI, start_time: datetime,
                                    end_time: datetime) -> float:
        """計算可用性"""
        # 查詢 Prometheus 指標
        # 這裡使用簡化邏輯
        total_requests = 10000
        successful_requests = 9950
        return (successful_requests / total_requests) * 100

    async def _calculate_latency(self, sli: SLI, start_time: datetime,
                               end_time: datetime) -> float:
        """計算延遲"""
        # P99 延遲時間
        return 150.0  # ms

    async def _calculate_error_rate(self, sli: SLI, start_time: datetime,
                                  end_time: datetime) -> float:
        """計算錯誤率"""
        total_requests = 10000
        error_requests = 50
        return (error_requests / total_requests) * 100

    async def check_slo_compliance(self, slo_name: str) -> Dict[str, Any]:
        """檢查 SLO 合規性"""
        slo = self.slos[slo_name]
        end_time = datetime.utcnow()
        start_time = end_time - timedelta(days=slo.time_window_days)

        current_value = await self.calculate_sli_value(slo.sli.name, start_time, end_time)

        is_compliant = current_value >= slo.target_percentage
        needs_alert = current_value <= slo.alert_threshold

        return {
            "slo_name": slo_name,
            "current_value": current_value,
            "target": slo.target_percentage,
            "is_compliant": is_compliant,
            "needs_alert": needs_alert,
            "time_window_days": slo.time_window_days,
            "measurement_time": end_time.isoformat()
        }

    async def get_error_budget(self, slo_name: str) -> Dict[str, Any]:
        """取得錯誤預算"""
        slo = self.slos[slo_name]
        compliance = await self.check_slo_compliance(slo_name)

        # 計算錯誤預算
        allowed_failure_rate = 100 - slo.target_percentage
        current_failure_rate = 100 - compliance["current_value"]
        remaining_budget = max(0, allowed_failure_rate - current_failure_rate)

        # 預估剩餘時間
        if current_failure_rate > allowed_failure_rate:
            budget_exhausted = True
            estimated_recovery_time = await self._estimate_recovery_time(slo)
        else:
            budget_exhausted = False
            estimated_recovery_time = None

        return {
            "slo_name": slo_name,
            "allowed_failure_rate": allowed_failure_rate,
            "current_failure_rate": current_failure_rate,
            "remaining_budget": remaining_budget,
            "budget_percentage": (remaining_budget / allowed_failure_rate) * 100,
            "budget_exhausted": budget_exhausted,
            "estimated_recovery_time": estimated_recovery_time
        }

    async def _estimate_recovery_time(self, slo: SLO) -> Optional[str]:
        """估算恢復時間"""
        # 基於歷史數據分析
        # 這裡使用簡化邏輯
        return "2 hours"

    async def generate_sla_report(self, sla_name: str, period: str = "monthly") -> Dict[str, Any]:
        """生成 SLA 報告"""
        sla = self.slas[sla_name]

        # 計算時間範圍
        end_time = datetime.utcnow()
        if period == "monthly":
            start_time = end_time.replace(day=1)
        elif period == "quarterly":
            quarter_start = ((end_time.month - 1) // 3) * 3 + 1
            start_time = end_time.replace(month=quarter_start, day=1)

        slo_results = []
        overall_compliant = True

        for slo in sla.slos:
            compliance = await self.check_slo_compliance(slo.name)
            slo_results.append(compliance)

            if not compliance["is_compliant"]:
                overall_compliant = False

        return {
            "sla_name": sla_name,
            "period": period,
            "start_time": start_time.isoformat(),
            "end_time": end_time.isoformat(),
            "overall_compliant": overall_compliant,
            "slo_results": slo_results,
            "penalty_applicable": not overall_compliant,
            "penalty_percentage": sla.penalty_percentage if not overall_compliant else 0
        }

class IncidentManager:
    """事件管理器"""

    def __init__(self):
        self.incidents = {}
        self.incident_counter = 0

    async def create_incident(self, title: str, description: str, severity: str,
                            affected_services: List[str]) -> str:
        """建立事件"""
        self.incident_counter += 1
        incident_id = f"INC-{self.incident_counter:04d}"

        incident = {
            "incident_id": incident_id,
            "title": title,
            "description": description,
            "severity": severity,
            "status": "open",
            "affected_services": affected_services,
            "created_at": datetime.utcnow(),
            "updated_at": datetime.utcnow(),
            "timeline": [
                {
                    "timestamp": datetime.utcnow(),
                    "event": "incident_created",
                    "description": f"Incident {incident_id} created"
                }
            ]
        }

        self.incidents[incident_id] = incident

        # 觸發事件回應流程
        await self._trigger_incident_response(incident)

        return incident_id

    async def update_incident(self, incident_id: str, update: Dict[str, Any]):
        """更新事件"""
        if incident_id in self.incidents:
            incident = self.incidents[incident_id]
            incident.update(update)
            incident["updated_at"] = datetime.utcnow()

            # 添加到時間軸
            incident["timeline"].append({
                "timestamp": datetime.utcnow(),
                "event": "incident_updated",
                "description": f"Incident updated: {update}"
            })

    async def resolve_incident(self, incident_id: str, resolution: str):
        """解決事件"""
        if incident_id in self.incidents:
            incident = self.incidents[incident_id]
            incident["status"] = "resolved"
            incident["resolution"] = resolution
            incident["resolved_at"] = datetime.utcnow()

            # 添加到時間軸
            incident["timeline"].append({
                "timestamp": datetime.utcnow(),
                "event": "incident_resolved",
                "description": resolution
            })

    async def _trigger_incident_response(self, incident: Dict[str, Any]):
        """觸發事件回應"""
        # 根據嚴重程度觸發不同的回應流程
        severity = incident["severity"]

        if severity == "critical":
            # 立即通知所有相關人員
            await self._notify_incident_responders(incident)
            # 啟動戰情室
            await self._setup_war_room(incident)
        elif severity == "high":
            # 通知主要負責人
            await self._notify_primary_responders(incident)

    async def _notify_incident_responders(self, incident: Dict[str, Any]):
        """通知事件回應人員"""
        # 發送緊急通知
        pass

    async def _setup_war_room(self, incident: Dict[str, Any]):
        """設定戰情室"""
        # 建立 Slack 頻道或其他協作空間
        pass

    async def get_incident_metrics(self, days: int = 30) -> Dict[str, Any]:
        """取得事件指標"""
        end_time = datetime.utcnow()
        start_time = end_time - timedelta(days=days)

        recent_incidents = [
            incident for incident in self.incidents.values()
            if incident["created_at"] >= start_time
        ]

        # 計算 MTTR (Mean Time To Resolution)
        resolved_incidents = [
            incident for incident in recent_incidents
            if incident["status"] == "resolved"
        ]

        if resolved_incidents:
            total_resolution_time = sum(
                (incident["resolved_at"] - incident["created_at"]).total_seconds()
                for incident in resolved_incidents
            )
            mttr = total_resolution_time / len(resolved_incidents) / 60  # 分鐘

        else:
            mttr = 0

        severity_counts = {}
        for incident in recent_incidents:
            severity = incident["severity"]
            severity_counts[severity] = severity_counts.get(severity, 0) + 1

        return {
            "total_incidents": len(recent_incidents),
            "resolved_incidents": len(resolved_incidents),
            "mttr_minutes": mttr,
            "by_severity": severity_counts,
            "period_days": days
        }

# SLI/SLO/SLA 配置範例
def setup_auth_service_sre():
    """設定認證服務的 SRE 指標"""
    manager = SREMetricsManager()

    # 註冊 SLIs
    availability_sli = SLI(
        name="auth_availability",
        type=SLIType.AVAILABILITY,
        description="Authentication service availability",
        query="rate(http_requests_total{status_code!~'5..'}[5m]) / rate(http_requests_total[5m])",
        unit="percentage"
    )

    latency_sli = SLI(
        name="auth_latency_p99",
        type=SLIType.LATENCY,
        description="99th percentile latency for authentication requests",
        query="histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))",
        unit="seconds"
    )

    error_rate_sli = SLI(
        name="auth_error_rate",
        type=SLIType.ERROR_RATE,
        description="Error rate for authentication requests",
        query="rate(http_requests_total{status_code=~'5..'}[5m]) / rate(http_requests_total[5m])",
        unit="percentage"
    )

    manager.register_sli(availability_sli)
    manager.register_sli(latency_sli)
    manager.register_sli(error_rate_sli)

    # 註冊 SLOs
    availability_slo = SLO(
        name="auth_availability_slo",
        description="Authentication service should be available 99.9% of the time",
        sli=availability_sli,
        target_percentage=99.9,
        time_window_days=30,
        alert_threshold=99.0
    )

    latency_slo = SLO(
        name="auth_latency_slo",
        description="99% of authentication requests should complete within 200ms",
        sli=latency_sli,
        target_percentage=99.0,
        time_window_days=30,
        alert_threshold=95.0
    )

    manager.register_slo(availability_slo)
    manager.register_slo(latency_slo)

    # 註冊 SLA
    auth_sla = SLA(
        name="auth_service_sla",
        description="Authentication Service Level Agreement",
        slos=[availability_slo, latency_slo],
        penalty_percentage=5.0,
        measurement_period="monthly"
    )

    manager.register_sla(auth_sla)

    return manager
```

---

## 💡 學習筆記區

### 🤔 我的理解
```
可觀測性三大支柱的關係：

SLI/SLO/SLA 的設計原則：

告警系統的智能化：

分散式追蹤的價值：
```

### 📊 實踐心得
```
建立監控系統的挑戰：

告警疲勞的處理方法：

事件回應的最佳實務：

SRE 文化的建立：
```

### 🚀 進階思考
```
可觀測性的未來發展：

AI 在監控中的應用：

雲原生監控的特點：

監控系統的自我監控：
```