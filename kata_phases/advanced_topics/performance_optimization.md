# Performance Optimization: 企業級效能調優與擴展策略

## 🎯 學習目標
- 掌握現代 Web 應用效能分析與優化方法論
- 學習 FastAPI 與 Python 效能調優最佳實務
- 建立可擴展的快取與負載平衡策略
- 實踐效能監控與自動化優化系統

---

## 🔍 效能分析與瓶頸識別

### 全面效能分析框架

```python
# app/performance/profiler.py
import cProfile
import pstats
import time
import asyncio
from functools import wraps
from typing import Dict, List, Callable, Any
from dataclasses import dataclass
import structlog

logger = structlog.get_logger()

@dataclass
class PerformanceMetrics:
    """效能指標資料結構"""
    function_name: str
    execution_time: float
    memory_usage: int
    cpu_usage: float
    call_count: int
    avg_time_per_call: float

class PerformanceProfiler:
    """企業級效能分析器"""

    def __init__(self):
        self.metrics = {}
        self.active_sessions = {}

    def profile_function(self, track_memory: bool = True, track_cpu: bool = True):
        """函數效能分析裝飾器"""
        def decorator(func: Callable):
            @wraps(func)
            async def async_wrapper(*args, **kwargs):
                start_time = time.perf_counter()
                start_memory = self._get_memory_usage() if track_memory else 0
                start_cpu = self._get_cpu_usage() if track_cpu else 0

                try:
                    # 執行函數
                    if asyncio.iscoroutinefunction(func):
                        result = await func(*args, **kwargs)
                    else:
                        result = func(*args, **kwargs)

                    return result

                finally:
                    # 計算效能指標
                    end_time = time.perf_counter()
                    execution_time = end_time - start_time

                    end_memory = self._get_memory_usage() if track_memory else 0
                    end_cpu = self._get_cpu_usage() if track_cpu else 0

                    # 記錄指標
                    await self._record_metrics(
                        function_name=func.__name__,
                        execution_time=execution_time,
                        memory_delta=end_memory - start_memory,
                        cpu_usage=end_cpu - start_cpu
                    )

            return async_wrapper
        return decorator

    async def _record_metrics(self, function_name: str, execution_time: float,
                            memory_delta: int, cpu_usage: float):
        """記錄效能指標"""
        if function_name not in self.metrics:
            self.metrics[function_name] = {
                "total_time": 0,
                "call_count": 0,
                "min_time": float('inf'),
                "max_time": 0,
                "memory_peak": 0,
                "cpu_peak": 0
            }

        metrics = self.metrics[function_name]
        metrics["total_time"] += execution_time
        metrics["call_count"] += 1
        metrics["min_time"] = min(metrics["min_time"], execution_time)
        metrics["max_time"] = max(metrics["max_time"], execution_time)
        metrics["memory_peak"] = max(metrics["memory_peak"], memory_delta)
        metrics["cpu_peak"] = max(metrics["cpu_peak"], cpu_usage)

        # 記錄慢函數
        if execution_time > 1.0:  # 超過 1 秒
            logger.warning("Slow function execution",
                         function=function_name,
                         duration=execution_time,
                         memory_delta=memory_delta)

    def get_performance_report(self) -> List[PerformanceMetrics]:
        """生成效能報告"""
        report = []
        for func_name, data in self.metrics.items():
            avg_time = data["total_time"] / data["call_count"] if data["call_count"] > 0 else 0

            metrics = PerformanceMetrics(
                function_name=func_name,
                execution_time=data["total_time"],
                memory_usage=data["memory_peak"],
                cpu_usage=data["cpu_peak"],
                call_count=data["call_count"],
                avg_time_per_call=avg_time
            )
            report.append(metrics)

        # 按平均執行時間排序
        return sorted(report, key=lambda x: x.avg_time_per_call, reverse=True)

    def _get_memory_usage(self) -> int:
        """取得記憶體使用量"""
        import psutil
        process = psutil.Process()
        return process.memory_info().rss

    def _get_cpu_usage(self) -> float:
        """取得 CPU 使用率"""
        import psutil
        return psutil.cpu_percent()

class AsyncProfiler:
    """異步程式效能分析器"""

    def __init__(self):
        self.async_metrics = {}
        self.concurrency_stats = {}

    async def profile_async_endpoint(self, endpoint_name: str):
        """分析異步 endpoint 效能"""
        async def decorator(func):
            @wraps(func)
            async def wrapper(*args, **kwargs):
                request_id = id(asyncio.current_task())
                start_time = time.perf_counter()

                # 記錄併發請求數
                self._increment_concurrency(endpoint_name)

                try:
                    result = await func(*args, **kwargs)
                    return result

                finally:
                    end_time = time.perf_counter()
                    execution_time = end_time - start_time

                    await self._record_async_metrics(endpoint_name, execution_time, request_id)
                    self._decrement_concurrency(endpoint_name)

            return wrapper
        return decorator

    def _increment_concurrency(self, endpoint_name: str):
        """增加併發計數"""
        if endpoint_name not in self.concurrency_stats:
            self.concurrency_stats[endpoint_name] = {"current": 0, "peak": 0}

        self.concurrency_stats[endpoint_name]["current"] += 1
        current = self.concurrency_stats[endpoint_name]["current"]
        self.concurrency_stats[endpoint_name]["peak"] = max(
            self.concurrency_stats[endpoint_name]["peak"], current
        )

    def _decrement_concurrency(self, endpoint_name: str):
        """減少併發計數"""
        if endpoint_name in self.concurrency_stats:
            self.concurrency_stats[endpoint_name]["current"] -= 1

    async def analyze_async_performance(self) -> Dict[str, Any]:
        """分析異步效能"""
        analysis = {}

        for endpoint, stats in self.concurrency_stats.items():
            analysis[endpoint] = {
                "peak_concurrency": stats["peak"],
                "current_concurrency": stats["current"],
                "bottleneck_risk": "high" if stats["peak"] > 50 else "low"
            }

        return analysis

class MemoryProfiler:
    """記憶體使用分析器"""

    def __init__(self):
        self.memory_snapshots = []

    async def profile_memory_usage(self, interval: float = 1.0, duration: float = 60.0):
        """監控記憶體使用情況"""
        import psutil
        import gc

        start_time = time.time()
        process = psutil.Process()

        while time.time() - start_time < duration:
            snapshot = {
                "timestamp": time.time(),
                "rss": process.memory_info().rss,
                "vms": process.memory_info().vms,
                "percent": process.memory_percent(),
                "gc_stats": {
                    "generation_0": gc.get_count()[0],
                    "generation_1": gc.get_count()[1],
                    "generation_2": gc.get_count()[2],
                }
            }

            self.memory_snapshots.append(snapshot)
            await asyncio.sleep(interval)

    def detect_memory_leaks(self) -> List[Dict[str, Any]]:
        """檢測記憶體洩漏"""
        if len(self.memory_snapshots) < 10:
            return []

        leaks = []
        window_size = 10

        for i in range(window_size, len(self.memory_snapshots)):
            recent_avg = sum(s["rss"] for s in self.memory_snapshots[i-window_size:i]) / window_size
            current = self.memory_snapshots[i]["rss"]

            # 記憶體增長超過 50%
            if current > recent_avg * 1.5:
                leaks.append({
                    "timestamp": self.memory_snapshots[i]["timestamp"],
                    "memory_usage": current,
                    "growth_rate": (current - recent_avg) / recent_avg,
                    "severity": "high" if current > recent_avg * 2 else "medium"
                })

        return leaks

    def get_memory_recommendations(self) -> List[str]:
        """取得記憶體優化建議"""
        recommendations = []

        if not self.memory_snapshots:
            return recommendations

        latest = self.memory_snapshots[-1]
        peak_memory = max(s["rss"] for s in self.memory_snapshots)

        # 記憶體使用率高
        if latest["percent"] > 80:
            recommendations.append("Consider increasing available memory or optimizing memory usage")

        # 垃圾回收頻率高
        if latest["gc_stats"]["generation_0"] > 1000:
            recommendations.append("High garbage collection activity - consider object pooling")

        # 記憶體峰值過高
        if peak_memory > 2 * 1024**3:  # 2GB
            recommendations.append("Peak memory usage is high - analyze large object allocations")

        return recommendations
```

---

## ⚡ FastAPI 效能優化

### 高效能 FastAPI 配置

```python
# app/performance/fastapi_optimization.py
from fastapi import FastAPI, Request, Response
from fastapi.middleware.gzip import GZipMiddleware
from fastapi.middleware.cors import CORSMiddleware
import uvicorn
from uvloop import EventLoopPolicy
import asyncio

class OptimizedFastAPI:
    """優化的 FastAPI 應用"""

    @staticmethod
    def create_optimized_app() -> FastAPI:
        """建立優化的 FastAPI 應用"""
        app = FastAPI(
            title="Auth Service",
            description="High-performance authentication service",
            version="1.0.0",
            # 效能優化設定
            docs_url="/docs" if settings.ENVIRONMENT != "production" else None,
            redoc_url="/redoc" if settings.ENVIRONMENT != "production" else None,
            openapi_url="/openapi.json" if settings.ENVIRONMENT != "production" else None,
            # 減少 OpenAPI 生成開銷
            generate_unique_id_function=lambda route: f"{route.tags[0]}-{route.name}" if route.tags else route.name,
        )

        # 效能中間件
        OptimizedFastAPI._setup_performance_middleware(app)

        # 異步事件循環優化
        OptimizedFastAPI._setup_event_loop_optimization()

        return app

    @staticmethod
    def _setup_performance_middleware(app: FastAPI):
        """設定效能中間件"""

        # 1. 響應壓縮
        app.add_middleware(GZipMiddleware, minimum_size=1000)

        # 2. 響應快取中間件
        @app.middleware("http")
        async def cache_middleware(request: Request, call_next):
            # 只快取 GET 請求
            if request.method == "GET":
                cache_key = f"response:{request.url.path}:{hash(str(request.query_params))}"
                cached_response = await cache.get(cache_key)

                if cached_response:
                    return Response(
                        content=cached_response["content"],
                        status_code=cached_response["status_code"],
                        headers=cached_response["headers"],
                        media_type=cached_response["media_type"]
                    )

            response = await call_next(request)

            # 快取成功響應
            if request.method == "GET" and response.status_code == 200:
                # 讀取響應內容
                response_body = b""
                async for chunk in response.body_iterator:
                    response_body += chunk

                # 存入快取
                await cache.set(cache_key, {
                    "content": response_body,
                    "status_code": response.status_code,
                    "headers": dict(response.headers),
                    "media_type": response.media_type
                }, expire=300)  # 5 分鐘快取

                # 重建響應
                return Response(
                    content=response_body,
                    status_code=response.status_code,
                    headers=response.headers,
                    media_type=response.media_type
                )

            return response

        # 3. 效能監控中間件
        @app.middleware("http")
        async def performance_middleware(request: Request, call_next):
            start_time = time.perf_counter()

            response = await call_next(request)

            process_time = time.perf_counter() - start_time
            response.headers["X-Process-Time"] = str(process_time)

            # 記錄慢請求
            if process_time > 1.0:
                logger.warning("Slow request detected",
                             path=request.url.path,
                             method=request.method,
                             duration=process_time)

            return response

    @staticmethod
    def _setup_event_loop_optimization():
        """設定事件循環優化"""
        # 使用 uvloop 提升效能 (Unix only)
        import platform
        if platform.system() != 'Windows':
            asyncio.set_event_loop_policy(EventLoopPolicy())

class ConnectionPoolOptimizer:
    """連接池優化器"""

    @staticmethod
    def create_optimized_http_client():
        """建立優化的 HTTP 客戶端"""
        import httpx

        # 連接池配置
        limits = httpx.Limits(
            max_keepalive_connections=100,
            max_connections=200,
            keepalive_expiry=30
        )

        # 超時配置
        timeout = httpx.Timeout(
            connect=5.0,
            read=30.0,
            write=10.0,
            pool=5.0
        )

        return httpx.AsyncClient(
            limits=limits,
            timeout=timeout,
            http2=True,  # 啟用 HTTP/2
            verify=True
        )

    @staticmethod
    def optimize_database_connections():
        """優化資料庫連接"""
        return {
            "pool_size": 20,
            "max_overflow": 30,
            "pool_timeout": 30,
            "pool_recycle": 3600,
            "pool_pre_ping": True,
            "echo": False,
            "echo_pool": False
        }

class RequestOptimizer:
    """請求處理優化器"""

    @staticmethod
    async def optimize_pydantic_parsing():
        """優化 Pydantic 解析效能"""
        from pydantic import BaseModel
        from pydantic.json import pydantic_encoder

        # 使用 orjson 加速 JSON 序列化
        import orjson

        def custom_json_encoder(obj):
            return orjson.dumps(obj, default=pydantic_encoder).decode()

        # 為模型設定自定義編碼器
        BaseModel.__json_encoder__ = custom_json_encoder

    @staticmethod
    def create_response_model_optimization():
        """響應模型優化"""
        from pydantic import BaseModel, Field
        from typing import Optional, List

        class OptimizedBaseModel(BaseModel):
            """優化的基礎模型"""

            class Config:
                # 效能優化設定
                allow_reuse=True
                validate_assignment=False  # 減少驗證開銷
                use_enum_values=True
                json_encoders={
                    datetime: lambda dt: dt.isoformat(),
                    UUID: str
                }

                # 序列化優化
                anystr_strip_whitespace=True
                min_anystr_length=0

        return OptimizedBaseModel

class BackgroundTaskOptimizer:
    """背景任務優化器"""

    def __init__(self):
        self.task_queue = asyncio.Queue(maxsize=1000)
        self.workers = []
        self.metrics = {
            "tasks_completed": 0,
            "tasks_failed": 0,
            "avg_processing_time": 0
        }

    async def start_workers(self, num_workers: int = 4):
        """啟動背景工作者"""
        for i in range(num_workers):
            worker = asyncio.create_task(self._worker(f"worker-{i}"))
            self.workers.append(worker)

    async def _worker(self, name: str):
        """背景工作者"""
        while True:
            try:
                # 從佇列取得任務
                task_func, args, kwargs = await self.task_queue.get()

                start_time = time.perf_counter()

                # 執行任務
                try:
                    if asyncio.iscoroutinefunction(task_func):
                        await task_func(*args, **kwargs)
                    else:
                        task_func(*args, **kwargs)

                    self.metrics["tasks_completed"] += 1

                except Exception as e:
                    logger.error(f"Background task failed in {name}", error=str(e))
                    self.metrics["tasks_failed"] += 1

                finally:
                    processing_time = time.perf_counter() - start_time
                    self._update_avg_processing_time(processing_time)

                    # 標記任務完成
                    self.task_queue.task_done()

            except Exception as e:
                logger.error(f"Worker {name} error", error=str(e))
                await asyncio.sleep(1)

    async def submit_task(self, func, *args, **kwargs):
        """提交背景任務"""
        try:
            await self.task_queue.put((func, args, kwargs))
        except asyncio.QueueFull:
            logger.warning("Background task queue is full, dropping task")

    def _update_avg_processing_time(self, new_time: float):
        """更新平均處理時間"""
        total_tasks = self.metrics["tasks_completed"] + self.metrics["tasks_failed"]
        if total_tasks > 0:
            current_avg = self.metrics["avg_processing_time"]
            self.metrics["avg_processing_time"] = (current_avg * (total_tasks - 1) + new_time) / total_tasks

    async def get_queue_stats(self) -> dict:
        """取得佇列統計"""
        return {
            "queue_size": self.task_queue.qsize(),
            "workers_active": len(self.workers),
            "metrics": self.metrics.copy()
        }
```

---

## 🚀 快取策略與優化

### 多層次快取架構

```python
# app/cache/cache_manager.py
from abc import ABC, abstractmethod
from typing import Any, Optional, Union, List, Dict
import asyncio
import json
import time
from dataclasses import dataclass

@dataclass
class CacheStats:
    """快取統計"""
    hits: int = 0
    misses: int = 0
    sets: int = 0
    deletes: int = 0
    evictions: int = 0

    @property
    def hit_rate(self) -> float:
        total = self.hits + self.misses
        return self.hits / total if total > 0 else 0

class CacheBackend(ABC):
    """快取後端抽象基類"""

    @abstractmethod
    async def get(self, key: str) -> Optional[Any]:
        pass

    @abstractmethod
    async def set(self, key: str, value: Any, expire: int = None) -> bool:
        pass

    @abstractmethod
    async def delete(self, key: str) -> bool:
        pass

    @abstractmethod
    async def exists(self, key: str) -> bool:
        pass

class InMemoryCache(CacheBackend):
    """記憶體快取實作"""

    def __init__(self, max_size: int = 10000, default_ttl: int = 3600):
        self.max_size = max_size
        self.default_ttl = default_ttl
        self.data = {}
        self.expiry = {}
        self.access_times = {}
        self.stats = CacheStats()

    async def get(self, key: str) -> Optional[Any]:
        # 檢查過期
        if key in self.expiry and time.time() > self.expiry[key]:
            await self.delete(key)
            return None

        if key in self.data:
            self.access_times[key] = time.time()
            self.stats.hits += 1
            return self.data[key]

        self.stats.misses += 1
        return None

    async def set(self, key: str, value: Any, expire: int = None) -> bool:
        # 檢查容量限制
        if len(self.data) >= self.max_size and key not in self.data:
            await self._evict_lru()

        self.data[key] = value
        self.access_times[key] = time.time()

        # 設定過期時間
        if expire:
            self.expiry[key] = time.time() + expire
        else:
            self.expiry[key] = time.time() + self.default_ttl

        self.stats.sets += 1
        return True

    async def delete(self, key: str) -> bool:
        if key in self.data:
            del self.data[key]
            self.expiry.pop(key, None)
            self.access_times.pop(key, None)
            self.stats.deletes += 1
            return True
        return False

    async def exists(self, key: str) -> bool:
        return key in self.data and (key not in self.expiry or time.time() <= self.expiry[key])

    async def _evict_lru(self):
        """移除最久未使用的項目"""
        if not self.access_times:
            return

        lru_key = min(self.access_times, key=self.access_times.get)
        await self.delete(lru_key)
        self.stats.evictions += 1

class RedisCache(CacheBackend):
    """Redis 快取實作"""

    def __init__(self, redis_client, default_ttl: int = 3600):
        self.redis = redis_client
        self.default_ttl = default_ttl
        self.stats = CacheStats()

    async def get(self, key: str) -> Optional[Any]:
        try:
            result = await self.redis.get(key)
            if result is not None:
                self.stats.hits += 1
                return json.loads(result)
            else:
                self.stats.misses += 1
                return None
        except Exception as e:
            logger.error("Redis get error", key=key, error=str(e))
            return None

    async def set(self, key: str, value: Any, expire: int = None) -> bool:
        try:
            ttl = expire or self.default_ttl
            serialized = json.dumps(value, default=str)
            await self.redis.setex(key, ttl, serialized)
            self.stats.sets += 1
            return True
        except Exception as e:
            logger.error("Redis set error", key=key, error=str(e))
            return False

    async def delete(self, key: str) -> bool:
        try:
            result = await self.redis.delete(key)
            if result:
                self.stats.deletes += 1
            return bool(result)
        except Exception as e:
            logger.error("Redis delete error", key=key, error=str(e))
            return False

    async def exists(self, key: str) -> bool:
        try:
            return bool(await self.redis.exists(key))
        except Exception as e:
            logger.error("Redis exists error", key=key, error=str(e))
            return False

class MultiLayerCache:
    """多層次快取管理器"""

    def __init__(self, layers: List[CacheBackend]):
        self.layers = layers  # 按優先級排序，第一層是最快的

    async def get(self, key: str) -> Optional[Any]:
        """從快取層中取得值"""
        for i, layer in enumerate(self.layers):
            value = await layer.get(key)
            if value is not None:
                # 將值寫入較高優先級的層
                await self._write_to_upper_layers(key, value, i)
                return value
        return None

    async def set(self, key: str, value: Any, expire: int = None) -> bool:
        """設定值到所有快取層"""
        results = []
        for layer in self.layers:
            result = await layer.set(key, value, expire)
            results.append(result)
        return any(results)  # 至少一層成功即可

    async def delete(self, key: str) -> bool:
        """從所有快取層刪除"""
        results = []
        for layer in self.layers:
            result = await layer.delete(key)
            results.append(result)
        return any(results)

    async def _write_to_upper_layers(self, key: str, value: Any, found_at_layer: int):
        """將值寫入更高優先級的層"""
        for i in range(found_at_layer):
            await self.layers[i].set(key, value)

    async def get_stats(self) -> Dict[str, CacheStats]:
        """取得所有層的統計"""
        stats = {}
        for i, layer in enumerate(self.layers):
            stats[f"layer_{i}"] = layer.stats
        return stats

class SmartCacheManager:
    """智能快取管理器"""

    def __init__(self, cache: MultiLayerCache):
        self.cache = cache
        self.access_patterns = {}
        self.preload_tasks = set()

    async def get_with_pattern_learning(self, key: str) -> Optional[Any]:
        """帶模式學習的取值"""
        # 記錄訪問模式
        self._record_access_pattern(key)

        value = await self.cache.get(key)

        # 預測相關鍵值並預載入
        if value is not None:
            await self._preload_related_keys(key)

        return value

    def _record_access_pattern(self, key: str):
        """記錄訪問模式"""
        current_time = time.time()

        if key not in self.access_patterns:
            self.access_patterns[key] = {
                "count": 0,
                "last_access": current_time,
                "access_intervals": []
            }

        pattern = self.access_patterns[key]
        if pattern["last_access"]:
            interval = current_time - pattern["last_access"]
            pattern["access_intervals"].append(interval)

            # 只保留最近 10 次間隔
            if len(pattern["access_intervals"]) > 10:
                pattern["access_intervals"].pop(0)

        pattern["count"] += 1
        pattern["last_access"] = current_time

    async def _preload_related_keys(self, key: str):
        """預載入相關鍵值"""
        # 基於訪問模式預測相關鍵值
        related_keys = self._predict_related_keys(key)

        for related_key in related_keys:
            if related_key not in self.preload_tasks:
                task = asyncio.create_task(self._preload_key(related_key))
                self.preload_tasks.add(task)
                task.add_done_callback(self.preload_tasks.discard)

    def _predict_related_keys(self, key: str) -> List[str]:
        """預測相關鍵值"""
        # 簡單的關聯預測邏輯
        related_keys = []

        # 用戶相關數據的關聯
        if "user:" in key:
            user_id = key.split(":")[-1]
            related_keys.extend([
                f"user_profile:{user_id}",
                f"user_permissions:{user_id}",
                f"user_sessions:{user_id}"
            ])

        return related_keys

    async def _preload_key(self, key: str):
        """預載入單個鍵值"""
        try:
            # 如果快取中沒有，從資料庫載入
            if not await self.cache.cache.layers[0].exists(key):
                # 這裡應該調用實際的資料載入邏輯
                value = await self._load_from_source(key)
                if value:
                    await self.cache.set(key, value)
        except Exception as e:
            logger.error("Preload failed", key=key, error=str(e))

    async def _load_from_source(self, key: str) -> Optional[Any]:
        """從資料來源載入（需要實作具體邏輯）"""
        # 這裡應該根據 key 的類型從對應的資料來源載入
        pass

    async def optimize_cache_expiry(self):
        """優化快取過期時間"""
        for key, pattern in self.access_patterns.items():
            if len(pattern["access_intervals"]) >= 5:
                # 計算平均訪問間隔
                avg_interval = sum(pattern["access_intervals"]) / len(pattern["access_intervals"])

                # 設定最優過期時間（訪問間隔的 2-3 倍）
                optimal_ttl = int(avg_interval * 2.5)
                optimal_ttl = max(300, min(optimal_ttl, 86400))  # 限制在 5分鐘到 1天之間

                # 如果當前存在該鍵，更新其過期時間
                if await self.cache.cache.layers[0].exists(key):
                    value = await self.cache.get(key)
                    if value:
                        await self.cache.set(key, value, optimal_ttl)

class CacheDecorator:
    """快取裝飾器"""

    def __init__(self, cache_manager: SmartCacheManager, default_ttl: int = 3600):
        self.cache_manager = cache_manager
        self.default_ttl = default_ttl

    def cached(self, key_func=None, ttl=None, cache_none=False):
        """快取裝飾器"""
        def decorator(func):
            @wraps(func)
            async def wrapper(*args, **kwargs):
                # 生成快取鍵
                if key_func:
                    cache_key = key_func(*args, **kwargs)
                else:
                    cache_key = f"{func.__name__}:{hash(str(args) + str(sorted(kwargs.items())))}"

                # 嘗試從快取取得
                cached_value = await self.cache_manager.get_with_pattern_learning(cache_key)
                if cached_value is not None:
                    return cached_value

                # 執行函數
                result = await func(*args, **kwargs) if asyncio.iscoroutinefunction(func) else func(*args, **kwargs)

                # 快取結果
                if result is not None or cache_none:
                    expire_time = ttl or self.default_ttl
                    await self.cache_manager.cache.set(cache_key, result, expire_time)

                return result

            return wrapper
        return decorator

    def invalidate_pattern(self, pattern: str):
        """使符合模式的快取失效"""
        async def invalidate():
            # 這需要支援模式匹配的快取後端
            # Redis: SCAN + pattern matching
            # 或維護一個鍵值索引
            pass

        return invalidate()
```

---

## 🏗️ 負載平衡與擴展

### 智能負載平衡策略

```python
# app/load_balancing/load_balancer.py
from abc import ABC, abstractmethod
from typing import List, Optional, Dict, Any
from dataclasses import dataclass
import asyncio
import time
import random
from enum import Enum

class BalancingStrategy(Enum):
    ROUND_ROBIN = "round_robin"
    WEIGHTED_ROUND_ROBIN = "weighted_round_robin"
    LEAST_CONNECTIONS = "least_connections"
    LEAST_RESPONSE_TIME = "least_response_time"
    CONSISTENT_HASH = "consistent_hash"

@dataclass
class ServerNode:
    """服務節點"""
    id: str
    host: str
    port: int
    weight: int = 1
    max_connections: int = 1000
    current_connections: int = 0
    total_requests: int = 0
    failed_requests: int = 0
    avg_response_time: float = 0.0
    last_health_check: float = 0
    is_healthy: bool = True

    @property
    def connection_utilization(self) -> float:
        return self.current_connections / self.max_connections

    @property
    def error_rate(self) -> float:
        if self.total_requests == 0:
            return 0.0
        return self.failed_requests / self.total_requests

    @property
    def address(self) -> str:
        return f"{self.host}:{self.port}"

class LoadBalancer(ABC):
    """負載平衡器抽象基類"""

    def __init__(self, nodes: List[ServerNode]):
        self.nodes = nodes
        self.current_index = 0

    @abstractmethod
    async def select_node(self, request_context: Dict[str, Any] = None) -> Optional[ServerNode]:
        """選擇服務節點"""
        pass

    async def get_healthy_nodes(self) -> List[ServerNode]:
        """取得健康的節點"""
        return [node for node in self.nodes if node.is_healthy]

class RoundRobinBalancer(LoadBalancer):
    """輪詢負載平衡器"""

    async def select_node(self, request_context: Dict[str, Any] = None) -> Optional[ServerNode]:
        healthy_nodes = await self.get_healthy_nodes()
        if not healthy_nodes:
            return None

        # 輪詢選擇
        node = healthy_nodes[self.current_index % len(healthy_nodes)]
        self.current_index += 1
        return node

class WeightedRoundRobinBalancer(LoadBalancer):
    """加權輪詢負載平衡器"""

    def __init__(self, nodes: List[ServerNode]):
        super().__init__(nodes)
        self.weighted_list = self._build_weighted_list()

    def _build_weighted_list(self) -> List[ServerNode]:
        """建立加權列表"""
        weighted_list = []
        for node in self.nodes:
            weighted_list.extend([node] * node.weight)
        return weighted_list

    async def select_node(self, request_context: Dict[str, Any] = None) -> Optional[ServerNode]:
        healthy_weighted = [node for node in self.weighted_list if node.is_healthy]
        if not healthy_weighted:
            return None

        node = healthy_weighted[self.current_index % len(healthy_weighted)]
        self.current_index += 1
        return node

class LeastConnectionsBalancer(LoadBalancer):
    """最少連接數負載平衡器"""

    async def select_node(self, request_context: Dict[str, Any] = None) -> Optional[ServerNode]:
        healthy_nodes = await self.get_healthy_nodes()
        if not healthy_nodes:
            return None

        # 選擇連接數最少的節點
        return min(healthy_nodes, key=lambda node: node.current_connections)

class LeastResponseTimeBalancer(LoadBalancer):
    """最短響應時間負載平衡器"""

    async def select_node(self, request_context: Dict[str, Any] = None) -> Optional[ServerNode]:
        healthy_nodes = await self.get_healthy_nodes()
        if not healthy_nodes:
            return None

        # 選擇平均響應時間最短的節點
        return min(healthy_nodes, key=lambda node: node.avg_response_time)

class ConsistentHashBalancer(LoadBalancer):
    """一致性雜湊負載平衡器"""

    def __init__(self, nodes: List[ServerNode], virtual_nodes: int = 150):
        super().__init__(nodes)
        self.virtual_nodes = virtual_nodes
        self.ring = self._build_hash_ring()

    def _build_hash_ring(self) -> Dict[int, ServerNode]:
        """建立雜湊環"""
        ring = {}
        for node in self.nodes:
            for i in range(self.virtual_nodes):
                virtual_key = f"{node.id}:{i}"
                hash_value = hash(virtual_key) % (2**32)
                ring[hash_value] = node
        return ring

    async def select_node(self, request_context: Dict[str, Any] = None) -> Optional[ServerNode]:
        if not self.ring:
            return None

        # 使用請求的某個標識符作為雜湊鍵
        hash_key = request_context.get("user_id", "") if request_context else ""
        hash_value = hash(hash_key) % (2**32)

        # 在環上找到第一個大於等於 hash_value 的節點
        for ring_key in sorted(self.ring.keys()):
            if ring_key >= hash_value:
                node = self.ring[ring_key]
                if node.is_healthy:
                    return node

        # 如果沒找到，返回環上第一個健康節點
        for ring_key in sorted(self.ring.keys()):
            node = self.ring[ring_key]
            if node.is_healthy:
                return node

        return None

class SmartLoadBalancer:
    """智能負載平衡器"""

    def __init__(self, nodes: List[ServerNode]):
        self.nodes = {node.id: node for node in nodes}
        self.balancers = {
            BalancingStrategy.ROUND_ROBIN: RoundRobinBalancer(nodes),
            BalancingStrategy.WEIGHTED_ROUND_ROBIN: WeightedRoundRobinBalancer(nodes),
            BalancingStrategy.LEAST_CONNECTIONS: LeastConnectionsBalancer(nodes),
            BalancingStrategy.LEAST_RESPONSE_TIME: LeastResponseTimeBalancer(nodes),
            BalancingStrategy.CONSISTENT_HASH: ConsistentHashBalancer(nodes),
        }
        self.current_strategy = BalancingStrategy.ROUND_ROBIN
        self.strategy_performance = {strategy: {"requests": 0, "total_response_time": 0}
                                   for strategy in BalancingStrategy}

    async def select_node(self, request_context: Dict[str, Any] = None) -> Optional[ServerNode]:
        """智能選擇節點"""
        # 根據當前負載情況選擇最佳策略
        optimal_strategy = await self._choose_optimal_strategy()

        if optimal_strategy != self.current_strategy:
            logger.info(f"Switching load balancing strategy from {self.current_strategy} to {optimal_strategy}")
            self.current_strategy = optimal_strategy

        balancer = self.balancers[self.current_strategy]
        return await balancer.select_node(request_context)

    async def _choose_optimal_strategy(self) -> BalancingStrategy:
        """選擇最佳策略"""
        # 分析當前系統狀態
        system_stats = await self._analyze_system_state()

        # 高負載情況下使用最少連接數
        if system_stats["avg_connection_utilization"] > 0.8:
            return BalancingStrategy.LEAST_CONNECTIONS

        # 響應時間差異大時使用最短響應時間
        if system_stats["response_time_variance"] > 1000:  # 1秒差異
            return BalancingStrategy.LEAST_RESPONSE_TIME

        # 需要會話親和性時使用一致性雜湊
        if system_stats["requires_session_affinity"]:
            return BalancingStrategy.CONSISTENT_HASH

        # 默認使用加權輪詢
        return BalancingStrategy.WEIGHTED_ROUND_ROBIN

    async def _analyze_system_state(self) -> Dict[str, Any]:
        """分析系統狀態"""
        healthy_nodes = [node for node in self.nodes.values() if node.is_healthy]

        if not healthy_nodes:
            return {"avg_connection_utilization": 0, "response_time_variance": 0, "requires_session_affinity": False}

        # 平均連接利用率
        avg_utilization = sum(node.connection_utilization for node in healthy_nodes) / len(healthy_nodes)

        # 響應時間變異數
        response_times = [node.avg_response_time for node in healthy_nodes]
        avg_response_time = sum(response_times) / len(response_times)
        variance = sum((rt - avg_response_time) ** 2 for rt in response_times) / len(response_times)

        return {
            "avg_connection_utilization": avg_utilization,
            "response_time_variance": variance,
            "requires_session_affinity": False  # 可根據請求類型判斷
        }

    async def update_node_stats(self, node_id: str, response_time: float, success: bool):
        """更新節點統計"""
        if node_id not in self.nodes:
            return

        node = self.nodes[node_id]
        node.total_requests += 1

        if success:
            # 更新平均響應時間
            node.avg_response_time = (
                (node.avg_response_time * (node.total_requests - 1) + response_time) /
                node.total_requests
            )
        else:
            node.failed_requests += 1

        # 更新策略效能統計
        strategy_stats = self.strategy_performance[self.current_strategy]
        strategy_stats["requests"] += 1
        strategy_stats["total_response_time"] += response_time

class HealthChecker:
    """健康檢查器"""

    def __init__(self, load_balancer: SmartLoadBalancer, check_interval: int = 30):
        self.load_balancer = load_balancer
        self.check_interval = check_interval
        self.running = False

    async def start(self):
        """開始健康檢查"""
        self.running = True
        while self.running:
            await self._check_all_nodes()
            await asyncio.sleep(self.check_interval)

    async def stop(self):
        """停止健康檢查"""
        self.running = False

    async def _check_all_nodes(self):
        """檢查所有節點健康狀態"""
        tasks = []
        for node in self.load_balancer.nodes.values():
            task = asyncio.create_task(self._check_node_health(node))
            tasks.append(task)

        await asyncio.gather(*tasks, return_exceptions=True)

    async def _check_node_health(self, node: ServerNode):
        """檢查單個節點健康狀態"""
        try:
            # 發送健康檢查請求
            start_time = time.time()

            import httpx
            async with httpx.AsyncClient(timeout=5.0) as client:
                response = await client.get(f"http://{node.address}/health")

            response_time = time.time() - start_time

            # 更新健康狀態
            if response.status_code == 200:
                node.is_healthy = True
                node.last_health_check = time.time()

                # 更新響應時間
                if node.total_requests == 0:
                    node.avg_response_time = response_time
                else:
                    node.avg_response_time = (
                        (node.avg_response_time * 0.9) + (response_time * 0.1)
                    )
            else:
                node.is_healthy = False

        except Exception as e:
            logger.error(f"Health check failed for node {node.id}", error=str(e))
            node.is_healthy = False

class AutoScaler:
    """自動擴展器"""

    def __init__(self, load_balancer: SmartLoadBalancer):
        self.load_balancer = load_balancer
        self.scaling_rules = self._load_scaling_rules()

    def _load_scaling_rules(self) -> Dict[str, Any]:
        """載入擴展規則"""
        return {
            "scale_out_threshold": 0.8,  # CPU/連接利用率閾值
            "scale_in_threshold": 0.3,
            "min_instances": 2,
            "max_instances": 10,
            "scale_out_cooldown": 300,  # 5分鐘
            "scale_in_cooldown": 600,   # 10分鐘
        }

    async def evaluate_scaling(self):
        """評估是否需要擴展"""
        current_load = await self._calculate_system_load()

        if await self._should_scale_out(current_load):
            await self._scale_out()
        elif await self._should_scale_in(current_load):
            await self._scale_in()

    async def _calculate_system_load(self) -> float:
        """計算系統負載"""
        healthy_nodes = [node for node in self.load_balancer.nodes.values() if node.is_healthy]

        if not healthy_nodes:
            return 1.0

        total_utilization = sum(node.connection_utilization for node in healthy_nodes)
        return total_utilization / len(healthy_nodes)

    async def _should_scale_out(self, current_load: float) -> bool:
        """判斷是否應該橫向擴展"""
        return (
            current_load > self.scaling_rules["scale_out_threshold"] and
            len(self.load_balancer.nodes) < self.scaling_rules["max_instances"]
        )

    async def _should_scale_in(self, current_load: float) -> bool:
        """判斷是否應該縮減"""
        return (
            current_load < self.scaling_rules["scale_in_threshold"] and
            len(self.load_balancer.nodes) > self.scaling_rules["min_instances"]
        )

    async def _scale_out(self):
        """橫向擴展"""
        logger.info("Scaling out - adding new instance")
        # 實際實作會調用容器編排系統（如 Kubernetes）
        # 這裡簡化為添加新節點的邏輯
        new_node = await self._create_new_instance()
        if new_node:
            self.load_balancer.nodes[new_node.id] = new_node
            logger.info(f"New instance added: {new_node.id}")

    async def _scale_in(self):
        """縮減"""
        logger.info("Scaling in - removing instance")
        # 選擇負載最低的節點移除
        node_to_remove = min(
            self.load_balancer.nodes.values(),
            key=lambda n: n.current_connections
        )

        # 優雅關閉
        await self._graceful_shutdown(node_to_remove)
        del self.load_balancer.nodes[node_to_remove.id]
        logger.info(f"Instance removed: {node_to_remove.id}")

    async def _create_new_instance(self) -> Optional[ServerNode]:
        """建立新實例（需要與容器編排系統整合）"""
        # 這裡應該調用 Kubernetes API 或其他編排工具
        pass

    async def _graceful_shutdown(self, node: ServerNode):
        """優雅關閉節點"""
        # 停止接收新請求
        node.is_healthy = False

        # 等待現有連接完成
        timeout = 30  # 30秒超時
        start_time = time.time()

        while node.current_connections > 0 and (time.time() - start_time) < timeout:
            await asyncio.sleep(1)

        logger.info(f"Node {node.id} gracefully shut down")
```

---

## 📈 效能監控與告警

### 即時效能監控系統

```python
# app/monitoring/performance_monitor.py
class PerformanceMonitor:
    """即時效能監控系統"""

    def __init__(self):
        self.metrics_collector = PrometheusMetrics()
        self.alert_manager = AlertManager()
        self.performance_thresholds = self._load_thresholds()

    def _load_thresholds(self) -> Dict[str, float]:
        """載入效能閾值"""
        return {
            "response_time_warning": 1.0,    # 1秒
            "response_time_critical": 5.0,   # 5秒
            "memory_usage_warning": 80.0,    # 80%
            "memory_usage_critical": 95.0,   # 95%
            "cpu_usage_warning": 70.0,       # 70%
            "cpu_usage_critical": 90.0,      # 90%
            "error_rate_warning": 0.05,      # 5%
            "error_rate_critical": 0.10,     # 10%
        }

    async def start_monitoring(self):
        """開始效能監控"""
        await asyncio.gather(
            self._monitor_response_times(),
            self._monitor_system_resources(),
            self._monitor_application_metrics(),
            self._monitor_database_performance()
        )

    async def _monitor_response_times(self):
        """監控響應時間"""
        while True:
            try:
                # 收集最近 5 分鐘的響應時間
                recent_response_times = await self._get_recent_response_times()

                if recent_response_times:
                    avg_response_time = sum(recent_response_times) / len(recent_response_times)
                    p95_response_time = self._calculate_percentile(recent_response_times, 0.95)

                    # 記錄指標
                    self.metrics_collector.gauge("app_response_time_avg", avg_response_time)
                    self.metrics_collector.gauge("app_response_time_p95", p95_response_time)

                    # 檢查告警
                    if p95_response_time > self.performance_thresholds["response_time_critical"]:
                        await self.alert_manager.send_alert(
                            severity="critical",
                            title="High Response Time",
                            description=f"95th percentile response time: {p95_response_time:.2f}s"
                        )
                    elif avg_response_time > self.performance_thresholds["response_time_warning"]:
                        await self.alert_manager.send_alert(
                            severity="warning",
                            title="Elevated Response Time",
                            description=f"Average response time: {avg_response_time:.2f}s"
                        )

                await asyncio.sleep(60)  # 每分鐘檢查

            except Exception as e:
                logger.error("Response time monitoring failed", error=str(e))
                await asyncio.sleep(60)

    async def _monitor_system_resources(self):
        """監控系統資源"""
        import psutil

        while True:
            try:
                # CPU 使用率
                cpu_percent = psutil.cpu_percent(interval=1)
                self.metrics_collector.gauge("system_cpu_usage", cpu_percent)

                # 記憶體使用率
                memory = psutil.virtual_memory()
                memory_percent = memory.percent
                self.metrics_collector.gauge("system_memory_usage", memory_percent)

                # 磁碟 I/O
                disk_io = psutil.disk_io_counters()
                if disk_io:
                    self.metrics_collector.gauge("system_disk_read_bytes", disk_io.read_bytes)
                    self.metrics_collector.gauge("system_disk_write_bytes", disk_io.write_bytes)

                # 網路 I/O
                network_io = psutil.net_io_counters()
                if network_io:
                    self.metrics_collector.gauge("system_network_sent_bytes", network_io.bytes_sent)
                    self.metrics_collector.gauge("system_network_recv_bytes", network_io.bytes_recv)

                # 檢查告警
                if memory_percent > self.performance_thresholds["memory_usage_critical"]:
                    await self.alert_manager.send_alert(
                        severity="critical",
                        title="High Memory Usage",
                        description=f"Memory usage: {memory_percent:.1f}%"
                    )

                if cpu_percent > self.performance_thresholds["cpu_usage_critical"]:
                    await self.alert_manager.send_alert(
                        severity="critical",
                        title="High CPU Usage",
                        description=f"CPU usage: {cpu_percent:.1f}%"
                    )

                await asyncio.sleep(30)  # 每 30 秒檢查

            except Exception as e:
                logger.error("System resource monitoring failed", error=str(e))
                await asyncio.sleep(60)

    def _calculate_percentile(self, data: List[float], percentile: float) -> float:
        """計算百分位數"""
        sorted_data = sorted(data)
        k = (len(sorted_data) - 1) * percentile
        f = int(k)
        c = k - f

        if f == len(sorted_data) - 1:
            return sorted_data[f]

        return sorted_data[f] * (1 - c) + sorted_data[f + 1] * c

class PerformanceOptimizer:
    """效能優化器"""

    def __init__(self):
        self.optimization_rules = self._load_optimization_rules()
        self.current_optimizations = set()

    def _load_optimization_rules(self) -> List[Dict[str, Any]]:
        """載入優化規則"""
        return [
            {
                "name": "enable_gzip_compression",
                "condition": lambda metrics: metrics.get("avg_response_size", 0) > 10000,  # 10KB
                "action": self._enable_gzip_compression,
                "priority": 1
            },
            {
                "name": "increase_connection_pool",
                "condition": lambda metrics: metrics.get("db_connection_utilization", 0) > 0.8,
                "action": self._increase_connection_pool,
                "priority": 2
            },
            {
                "name": "enable_query_cache",
                "condition": lambda metrics: metrics.get("db_query_time_avg", 0) > 100,  # 100ms
                "action": self._enable_query_cache,
                "priority": 3
            },
            {
                "name": "optimize_memory_usage",
                "condition": lambda metrics: metrics.get("memory_usage", 0) > 85,
                "action": self._optimize_memory_usage,
                "priority": 4
            }
        ]

    async def evaluate_optimizations(self, metrics: Dict[str, float]):
        """評估和應用優化"""
        applicable_optimizations = []

        for rule in self.optimization_rules:
            if rule["condition"](metrics) and rule["name"] not in self.current_optimizations:
                applicable_optimizations.append(rule)

        # 按優先級排序
        applicable_optimizations.sort(key=lambda x: x["priority"])

        for rule in applicable_optimizations:
            try:
                await rule["action"]()
                self.current_optimizations.add(rule["name"])
                logger.info(f"Applied optimization: {rule['name']}")
            except Exception as e:
                logger.error(f"Optimization failed: {rule['name']}", error=str(e))

    async def _enable_gzip_compression(self):
        """啟用 GZIP 壓縮"""
        # 動態調整 GZIP 設定
        logger.info("Enabling GZIP compression for responses > 1KB")

    async def _increase_connection_pool(self):
        """增加連接池大小"""
        # 動態增加資料庫連接池大小
        logger.info("Increasing database connection pool size")

    async def _enable_query_cache(self):
        """啟用查詢快取"""
        # 啟用更積極的查詢快取
        logger.info("Enabling aggressive query caching")

    async def _optimize_memory_usage(self):
        """優化記憶體使用"""
        # 觸發垃圾回收
        import gc
        gc.collect()
        logger.info("Triggered garbage collection to optimize memory")

class PerformanceAnalyzer:
    """效能分析器"""

    async def generate_performance_report(self) -> Dict[str, Any]:
        """生成效能分析報告"""
        report = {
            "timestamp": datetime.utcnow().isoformat(),
            "response_time_analysis": await self._analyze_response_times(),
            "resource_usage_analysis": await self._analyze_resource_usage(),
            "bottleneck_analysis": await self._identify_bottlenecks(),
            "optimization_recommendations": await self._generate_recommendations()
        }

        return report

    async def _analyze_response_times(self) -> Dict[str, Any]:
        """分析響應時間"""
        # 分析最近 1 小時的響應時間數據
        response_times = await self._get_response_time_history(hours=1)

        if not response_times:
            return {}

        return {
            "average": sum(response_times) / len(response_times),
            "median": self._calculate_percentile(response_times, 0.5),
            "p95": self._calculate_percentile(response_times, 0.95),
            "p99": self._calculate_percentile(response_times, 0.99),
            "max": max(response_times),
            "min": min(response_times),
            "trend": await self._calculate_trend(response_times)
        }

    async def _identify_bottlenecks(self) -> List[Dict[str, Any]]:
        """識別效能瓶頸"""
        bottlenecks = []

        # 分析資料庫效能
        db_bottlenecks = await self._analyze_database_bottlenecks()
        bottlenecks.extend(db_bottlenecks)

        # 分析網路效能
        network_bottlenecks = await self._analyze_network_bottlenecks()
        bottlenecks.extend(network_bottlenecks)

        # 分析應用層效能
        app_bottlenecks = await self._analyze_application_bottlenecks()
        bottlenecks.extend(app_bottlenecks)

        return sorted(bottlenecks, key=lambda x: x["impact_score"], reverse=True)

    async def _generate_recommendations(self) -> List[str]:
        """生成優化建議"""
        recommendations = []

        # 基於分析結果生成建議
        bottlenecks = await self._identify_bottlenecks()

        for bottleneck in bottlenecks[:5]:  # 前 5 個最嚴重的瓶頸
            if bottleneck["type"] == "database":
                recommendations.append(f"優化資料庫查詢: {bottleneck['description']}")
            elif bottleneck["type"] == "memory":
                recommendations.append(f"記憶體優化: {bottleneck['description']}")
            elif bottleneck["type"] == "cpu":
                recommendations.append(f"CPU 優化: {bottleneck['description']}")
            elif bottleneck["type"] == "network":
                recommendations.append(f"網路優化: {bottleneck['description']}")

        return recommendations
```

---

## 💡 學習筆記區

### 🤔 我的理解
```
效能分析的系統性方法：

FastAPI 效能優化的關鍵點：

快取策略的設計原則：

負載平衡的選擇考量：
```

### ⚡ 實踐心得
```
效能調優過程中的挑戰：

最有效的優化技術：

監控系統的重要性：

擴展策略的考量因素：
```

### 🚀 進階思考
```
現代應用的效能挑戰：

雲原生環境的優化策略：

AI 在效能優化中的應用：

未來效能優化技術趨勢：
```