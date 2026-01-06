# Production Deployment: 企業級部署策略與維運實踐

## 🎯 學習目標
- 掌握現代化部署策略與最佳實務
- 學習容器化技術與 Kubernetes 編排
- 建立自動化部署與維運管道
- 實踐高可用性與災難恢復機制

---

## 🐳 容器化部署策略

### Docker 優化與最佳實務

```python
# deployment/docker_optimizer.py
import docker
import json
from typing import Dict, List, Any, Optional
from dataclasses import dataclass
from datetime import datetime
import subprocess
import structlog

logger = structlog.get_logger()

@dataclass
class ImageBuildConfig:
    """映像建構配置"""
    dockerfile_path: str
    context_path: str
    image_name: str
    build_args: Dict[str, str]
    target_stage: Optional[str] = None
    no_cache: bool = False

@dataclass
class SecurityScanResult:
    """安全掃描結果"""
    image_id: str
    vulnerabilities: List[Dict[str, Any]]
    total_vulnerabilities: int
    critical_count: int
    high_count: int
    medium_count: int
    low_count: int

class DockerImageOptimizer:
    """Docker 映像優化器"""

    def __init__(self):
        self.client = docker.from_env()

    def build_optimized_image(self, config: ImageBuildConfig) -> str:
        """建構優化的映像"""
        logger.info("Building optimized Docker image",
                   image_name=config.image_name,
                   target_stage=config.target_stage)

        # 建構映像
        image, build_logs = self.client.images.build(
            path=config.context_path,
            dockerfile=config.dockerfile_path,
            tag=config.image_name,
            buildargs=config.build_args,
            target=config.target_stage,
            nocache=config.no_cache,
            rm=True,
            forcerm=True
        )

        # 分析建構日誌
        self._analyze_build_logs(build_logs)

        # 映像大小分析
        image_info = self._analyze_image_size(image.id)

        logger.info("Image build completed",
                   image_id=image.id[:12],
                   image_size=image_info["size_mb"],
                   layer_count=image_info["layer_count"])

        return image.id

    def _analyze_build_logs(self, build_logs) -> None:
        """分析建構日誌"""
        total_steps = 0
        cached_steps = 0

        for log in build_logs:
            if 'stream' in log:
                line = log['stream'].strip()
                if line.startswith('Step '):
                    total_steps += 1
                elif 'Using cache' in line:
                    cached_steps += 1

        cache_efficiency = (cached_steps / total_steps) * 100 if total_steps > 0 else 0

        logger.info("Build analysis",
                   total_steps=total_steps,
                   cached_steps=cached_steps,
                   cache_efficiency=f"{cache_efficiency:.1f}%")

    def scan_security_vulnerabilities(self, image_id: str) -> SecurityScanResult:
        """安全漏洞掃描"""
        logger.info("Scanning image for security vulnerabilities", image_id=image_id[:12])

        try:
            # 使用 Trivy 進行安全掃描
            result = subprocess.run([
                'trivy', 'image', '--format', 'json', image_id
            ], capture_output=True, text=True, check=True)

            scan_data = json.loads(result.stdout)
            vulnerabilities = []

            for target in scan_data.get('Results', []):
                if 'Vulnerabilities' in target:
                    vulnerabilities.extend(target['Vulnerabilities'])

            # 統計漏洞數量
            severity_counts = {'CRITICAL': 0, 'HIGH': 0, 'MEDIUM': 0, 'LOW': 0}
            for vuln in vulnerabilities:
                severity = vuln.get('Severity', 'UNKNOWN')
                if severity in severity_counts:
                    severity_counts[severity] += 1

            return SecurityScanResult(
                image_id=image_id,
                vulnerabilities=vulnerabilities,
                total_vulnerabilities=len(vulnerabilities),
                critical_count=severity_counts['CRITICAL'],
                high_count=severity_counts['HIGH'],
                medium_count=severity_counts['MEDIUM'],
                low_count=severity_counts['LOW']
            )

        except subprocess.CalledProcessError as e:
            logger.error("Security scan failed", error=str(e))
            return SecurityScanResult(
                image_id=image_id,
                vulnerabilities=[],
                total_vulnerabilities=0,
                critical_count=0,
                high_count=0,
                medium_count=0,
                low_count=0
            )

    def create_multi_stage_dockerfile(self, base_config: Dict[str, Any]) -> str:
        """建立多階段 Dockerfile"""
        dockerfile_template = """
# 建構階段
FROM python:3.11-slim as builder

WORKDIR /app

# 安裝建構依賴
RUN apt-get update && \\
    apt-get install -y --no-install-recommends \\
        build-essential \\
        gcc \\
        && rm -rf /var/lib/apt/lists/*

# 複製需求文件
COPY requirements.txt .

# 建立虛擬環境並安裝依賴
RUN python -m venv /opt/venv && \\
    /opt/venv/bin/pip install --no-cache-dir -r requirements.txt

# 生產階段
FROM python:3.11-slim as production

# 建立非 root 使用者
RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app

# 僅複製必要的執行時依賴
RUN apt-get update && \\
    apt-get install -y --no-install-recommends \\
        ca-certificates \\
        && rm -rf /var/lib/apt/lists/* \\
        && apt-get purge -y --auto-remove

# 從建構階段複製虛擬環境
COPY --from=builder /opt/venv /opt/venv

# 複製應用程式碼
COPY --chown=appuser:appuser . .

# 設定環境變數
ENV PATH="/opt/venv/bin:$PATH"
ENV PYTHONPATH="/app"
ENV PYTHONUNBUFFERED=1

# 健康檢查
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \\
    CMD python -c "import requests; requests.get('http://localhost:8000/health')"

# 切換到非 root 使用者
USER appuser

# 暴露端口
EXPOSE 8000

# 啟動命令
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
"""
        return dockerfile_template.strip()

class ContainerRegistryManager:
    """容器註冊表管理器"""

    def __init__(self, registry_url: str, username: str, password: str):
        self.registry_url = registry_url
        self.username = username
        self.password = password
        self.client = docker.from_env()

    def push_image(self, image_tag: str, repository: str) -> bool:
        """推送映像到註冊表"""
        try:
            # 登入註冊表
            self.client.login(
                username=self.username,
                password=self.password,
                registry=self.registry_url
            )

            # 標記映像
            full_tag = f"{self.registry_url}/{repository}:{image_tag}"
            image = self.client.images.get(image_tag)
            image.tag(full_tag)

            # 推送映像
            logger.info("Pushing image to registry", tag=full_tag)

            push_log = self.client.images.push(full_tag, stream=True, decode=True)

            for line in push_log:
                if 'error' in line:
                    logger.error("Push failed", error=line['error'])
                    return False
                elif 'status' in line and 'digest' in line:
                    logger.info("Push completed", digest=line.get('aux', {}).get('Digest'))

            return True

        except Exception as e:
            logger.error("Image push failed", error=str(e))
            return False
```

---

## ☸️ Kubernetes 編排與管理

### 部署配置與自動擴展

```python
# kubernetes/deployment_manager.py
from kubernetes import client, config
from kubernetes.client.rest import ApiException
from typing import Dict, List, Any, Optional
from dataclasses import dataclass
import yaml
import structlog
from datetime import datetime

logger = structlog.get_logger()

@dataclass
class DeploymentConfig:
    """部署配置"""
    name: str
    namespace: str
    image: str
    replicas: int
    resources: Dict[str, Any]
    environment_vars: Dict[str, str]
    health_check: Dict[str, Any]
    service_config: Dict[str, Any]

class KubernetesDeploymentManager:
    """Kubernetes 部署管理器"""

    def __init__(self, kubeconfig_path: str = None):
        if kubeconfig_path:
            config.load_kube_config(config_file=kubeconfig_path)
        else:
            try:
                config.load_incluster_config()
            except:
                config.load_kube_config()

        self.apps_v1 = client.AppsV1Api()
        self.core_v1 = client.CoreV1Api()
        self.autoscaling_v1 = client.AutoscalingV1Api()

    def create_deployment(self, deployment_config: DeploymentConfig) -> bool:
        """建立部署"""
        try:
            # 建立 Deployment 物件
            deployment = self._build_deployment(deployment_config)

            # 建立部署
            self.apps_v1.create_namespaced_deployment(
                namespace=deployment_config.namespace,
                body=deployment
            )

            # 建立服務
            service = self._build_service(deployment_config)
            self.core_v1.create_namespaced_service(
                namespace=deployment_config.namespace,
                body=service
            )

            # 建立 HPA（如果配置了）
            if deployment_config.resources.get('auto_scaling'):
                hpa = self._build_hpa(deployment_config)
                self.autoscaling_v1.create_namespaced_horizontal_pod_autoscaler(
                    namespace=deployment_config.namespace,
                    body=hpa
                )

            logger.info("Deployment created successfully",
                       name=deployment_config.name,
                       namespace=deployment_config.namespace)
            return True

        except ApiException as e:
            logger.error("Failed to create deployment",
                        name=deployment_config.name,
                        error=str(e))
            return False

    def _build_deployment(self, config: DeploymentConfig) -> client.V1Deployment:
        """建構 Deployment 物件"""
        # 環境變數
        env_vars = [
            client.V1EnvVar(name=k, value=v)
            for k, v in config.environment_vars.items()
        ]

        # 資源限制
        resources = client.V1ResourceRequirements(
            requests=config.resources.get('requests', {}),
            limits=config.resources.get('limits', {})
        )

        # 健康檢查
        liveness_probe = None
        readiness_probe = None

        if config.health_check:
            probe_config = config.health_check

            if probe_config.get('liveness'):
                liveness_probe = client.V1Probe(
                    http_get=client.V1HTTPGetAction(
                        path=probe_config['liveness']['path'],
                        port=probe_config['liveness']['port']
                    ),
                    initial_delay_seconds=probe_config['liveness'].get('initial_delay', 30),
                    period_seconds=probe_config['liveness'].get('period', 10),
                    timeout_seconds=probe_config['liveness'].get('timeout', 3),
                    failure_threshold=probe_config['liveness'].get('failure_threshold', 3)
                )

            if probe_config.get('readiness'):
                readiness_probe = client.V1Probe(
                    http_get=client.V1HTTPGetAction(
                        path=probe_config['readiness']['path'],
                        port=probe_config['readiness']['port']
                    ),
                    initial_delay_seconds=probe_config['readiness'].get('initial_delay', 5),
                    period_seconds=probe_config['readiness'].get('period', 5),
                    timeout_seconds=probe_config['readiness'].get('timeout', 3),
                    failure_threshold=probe_config['readiness'].get('failure_threshold', 3)
                )

        # 容器配置
        container = client.V1Container(
            name=config.name,
            image=config.image,
            env=env_vars,
            resources=resources,
            liveness_probe=liveness_probe,
            readiness_probe=readiness_probe,
            ports=[client.V1ContainerPort(container_port=8000)]
        )

        # Pod 模板
        pod_template = client.V1PodTemplateSpec(
            metadata=client.V1ObjectMeta(
                labels={"app": config.name}
            ),
            spec=client.V1PodSpec(
                containers=[container],
                security_context=client.V1PodSecurityContext(
                    run_as_non_root=True,
                    run_as_user=1000,
                    fs_group=2000
                )
            )
        )

        # Deployment 規格
        deployment_spec = client.V1DeploymentSpec(
            replicas=config.replicas,
            selector=client.V1LabelSelector(
                match_labels={"app": config.name}
            ),
            template=pod_template,
            strategy=client.V1DeploymentStrategy(
                type="RollingUpdate",
                rolling_update=client.V1RollingUpdateDeployment(
                    max_unavailable="25%",
                    max_surge="25%"
                )
            )
        )

        return client.V1Deployment(
            api_version="apps/v1",
            kind="Deployment",
            metadata=client.V1ObjectMeta(name=config.name),
            spec=deployment_spec
        )

    def get_deployment_status(self, name: str, namespace: str) -> Dict[str, Any]:
        """取得部署狀態"""
        try:
            deployment = self.apps_v1.read_namespaced_deployment(name=name, namespace=namespace)

            status = {
                "replicas": deployment.status.replicas or 0,
                "ready_replicas": deployment.status.ready_replicas or 0,
                "available_replicas": deployment.status.available_replicas or 0,
                "updated_replicas": deployment.status.updated_replicas or 0,
                "conditions": []
            }

            if deployment.status.conditions:
                for condition in deployment.status.conditions:
                    status["conditions"].append({
                        "type": condition.type,
                        "status": condition.status,
                        "reason": condition.reason,
                        "message": condition.message,
                        "last_transition_time": condition.last_transition_time.isoformat() if condition.last_transition_time else None
                    })

            return status

        except ApiException as e:
            logger.error("Failed to get deployment status", name=name, error=str(e))
            return {}

class BlueGreenDeployment:
    """藍綠部署管理器"""

    def __init__(self, k8s_manager: KubernetesDeploymentManager):
        self.k8s_manager = k8s_manager

    async def deploy(self, config: DeploymentConfig, current_version: str, new_version: str) -> bool:
        """執行藍綠部署"""
        logger.info("Starting blue-green deployment",
                   current_version=current_version,
                   new_version=new_version)

        try:
            # Step 1: 建立新版本部署（綠環境）
            green_config = self._create_green_config(config, new_version)
            if not self.k8s_manager.create_deployment(green_config):
                logger.error("Failed to create green deployment")
                return False

            # Step 2: 等待綠環境就緒
            if not await self._wait_for_deployment_ready(green_config.name, green_config.namespace):
                logger.error("Green deployment not ready")
                await self._cleanup_green_deployment(green_config)
                return False

            # Step 3: 健康檢查
            if not await self._health_check(green_config):
                logger.error("Green deployment health check failed")
                await self._cleanup_green_deployment(green_config)
                return False

            # Step 4: 切換流量到綠環境
            if not await self._switch_traffic_to_green(config, green_config):
                logger.error("Failed to switch traffic")
                await self._cleanup_green_deployment(green_config)
                return False

            # Step 5: 清理藍環境（舊版本）
            blue_deployment_name = f"{config.name}-{current_version}"
            await self._cleanup_blue_deployment(blue_deployment_name, config.namespace)

            logger.info("Blue-green deployment completed successfully")
            return True

        except Exception as e:
            logger.error("Blue-green deployment failed", error=str(e))
            return False

class CanaryDeployment:
    """金絲雀部署管理器"""

    def __init__(self, k8s_manager: KubernetesDeploymentManager):
        self.k8s_manager = k8s_manager

    async def deploy(self, config: DeploymentConfig, canary_percentage: int = 10) -> bool:
        """執行金絲雀部署"""
        logger.info("Starting canary deployment",
                   canary_percentage=canary_percentage)

        try:
            # Step 1: 建立金絲雀部署
            canary_config = self._create_canary_config(config, canary_percentage)
            if not self.k8s_manager.create_deployment(canary_config):
                return False

            # Step 2: 監控金絲雀指標
            if not await self._monitor_canary_metrics(canary_config):
                await self._rollback_canary(canary_config)
                return False

            # Step 3: 逐步增加流量
            for percentage in [25, 50, 75, 100]:
                logger.info(f"Increasing canary traffic to {percentage}%")
                await self._adjust_traffic_split(config, canary_config, percentage)

                if not await self._monitor_canary_metrics(canary_config):
                    await self._rollback_canary(canary_config)
                    return False

            # Step 4: 完成部署，移除舊版本
            await self._complete_canary_deployment(config, canary_config)

            logger.info("Canary deployment completed successfully")
            return True

        except Exception as e:
            logger.error("Canary deployment failed", error=str(e))
            return False
```

---

## 🔄 部署管道自動化

### CI/CD 管道整合

```python
# cicd/pipeline_manager.py
import json
import yaml
from typing import Dict, List, Any, Optional
from dataclasses import dataclass
from datetime import datetime
from enum import Enum
import subprocess
import asyncio

class PipelineStage(Enum):
    BUILD = "build"
    TEST = "test"
    SECURITY_SCAN = "security_scan"
    QUALITY_GATE = "quality_gate"
    DEPLOY = "deploy"
    SMOKE_TEST = "smoke_test"
    ROLLBACK = "rollback"

class PipelineStatus(Enum):
    PENDING = "pending"
    RUNNING = "running"
    SUCCESS = "success"
    FAILED = "failed"
    CANCELLED = "cancelled"

@dataclass
class PipelineStageResult:
    """管道階段結果"""
    stage: PipelineStage
    status: PipelineStatus
    start_time: datetime
    end_time: Optional[datetime] = None
    duration_seconds: Optional[float] = None
    artifacts: Dict[str, Any] = None
    logs: List[str] = None
    error_message: Optional[str] = None

@dataclass
class PipelineExecution:
    """管道執行記錄"""
    pipeline_id: str
    commit_sha: str
    branch: str
    triggered_by: str
    start_time: datetime
    end_time: Optional[datetime] = None
    status: PipelineStatus = PipelineStatus.PENDING
    stages: List[PipelineStageResult] = None

class CICDPipelineManager:
    """CI/CD 管道管理器"""

    def __init__(self, config: Dict[str, Any]):
        self.config = config
        self.executions = {}

    async def execute_pipeline(self, commit_sha: str, branch: str,
                             triggered_by: str) -> PipelineExecution:
        """執行 CI/CD 管道"""
        pipeline_id = f"{commit_sha[:8]}_{int(datetime.utcnow().timestamp())}"

        execution = PipelineExecution(
            pipeline_id=pipeline_id,
            commit_sha=commit_sha,
            branch=branch,
            triggered_by=triggered_by,
            start_time=datetime.utcnow(),
            stages=[]
        )

        self.executions[pipeline_id] = execution

        logger.info("Starting CI/CD pipeline",
                   pipeline_id=pipeline_id,
                   commit_sha=commit_sha,
                   branch=branch)

        try:
            execution.status = PipelineStatus.RUNNING

            # 定義管道階段
            stages = [
                PipelineStage.BUILD,
                PipelineStage.TEST,
                PipelineStage.SECURITY_SCAN,
                PipelineStage.QUALITY_GATE
            ]

            # 僅在主分支部署
            if branch in ['main', 'master', 'production']:
                stages.extend([
                    PipelineStage.DEPLOY,
                    PipelineStage.SMOKE_TEST
                ])

            # 依序執行階段
            for stage in stages:
                result = await self._execute_stage(stage, execution)
                execution.stages.append(result)

                if result.status == PipelineStatus.FAILED:
                    execution.status = PipelineStatus.FAILED
                    break
            else:
                execution.status = PipelineStatus.SUCCESS

        except Exception as e:
            logger.error("Pipeline execution failed",
                        pipeline_id=pipeline_id,
                        error=str(e))
            execution.status = PipelineStatus.FAILED

        execution.end_time = datetime.utcnow()

        # 發送通知
        await self._send_pipeline_notification(execution)

        return execution

    async def _execute_build_stage(self, result: PipelineStageResult,
                                 execution: PipelineExecution):
        """執行建構階段"""
        # Docker 建構
        build_args = self.config.get('build', {}).get('args', {})
        build_args['GIT_COMMIT'] = execution.commit_sha

        # 建構命令
        build_command = [
            'docker', 'build',
            '-t', f"auth-service:{execution.commit_sha}",
            '--build-arg', f"GIT_COMMIT={execution.commit_sha}",
            '.'
        ]

        process = await asyncio.create_subprocess_exec(
            *build_command,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE
        )

        stdout, stderr = await process.communicate()

        if process.returncode != 0:
            raise Exception(f"Build failed: {stderr.decode()}")

        result.artifacts = {
            'image_tag': execution.commit_sha,
            'build_logs': stdout.decode()
        }

    async def _execute_deploy_stage(self, result: PipelineStageResult,
                                  execution: PipelineExecution):
        """執行部署階段"""
        environment = self._get_target_environment(execution.branch)

        # 推送映像到註冊表
        registry_url = self.config.get('registry', {}).get('url')
        image_name = f"{registry_url}/auth-service:{execution.commit_sha}"

        push_command = ['docker', 'push', image_name]
        process = await asyncio.create_subprocess_exec(*push_command)
        await process.wait()

        if process.returncode != 0:
            raise Exception("Failed to push image to registry")

        # 部署到 Kubernetes
        kubectl_command = [
            'kubectl', 'set', 'image',
            f'deployment/auth-service',
            f'auth-service={image_name}',
            f'--namespace={environment}'
        ]

        process = await asyncio.create_subprocess_exec(*kubectl_command)
        await process.wait()

        if process.returncode != 0:
            raise Exception("Failed to deploy to Kubernetes")

        result.artifacts = {
            'environment': environment,
            'image_name': image_name,
            'deployment_time': datetime.utcnow().isoformat()
        }

class DeploymentRollbackManager:
    """部署回滾管理器"""

    def __init__(self, k8s_manager: KubernetesDeploymentManager):
        self.k8s_manager = k8s_manager

    async def rollback_deployment(self, deployment_name: str, namespace: str,
                                 target_revision: Optional[int] = None) -> bool:
        """回滾部署"""
        try:
            logger.info("Starting deployment rollback",
                       deployment=deployment_name,
                       namespace=namespace,
                       target_revision=target_revision)

            # 執行回滾
            kubectl_command = ['kubectl', 'rollout', 'undo', f'deployment/{deployment_name}']

            if target_revision:
                kubectl_command.extend(['--to-revision', str(target_revision)])

            kubectl_command.extend(['-n', namespace])

            process = await asyncio.create_subprocess_exec(
                *kubectl_command,
                stdout=asyncio.subprocess.PIPE,
                stderr=asyncio.subprocess.PIPE
            )

            stdout, stderr = await process.communicate()

            if process.returncode != 0:
                logger.error("Rollback failed", error=stderr.decode())
                return False

            logger.info("Deployment rollback completed successfully",
                       deployment=deployment_name)
            return True

        except Exception as e:
            logger.error("Rollback failed", deployment=deployment_name, error=str(e))
            return False
```

---

## ⚡ 效能與可靠性優化

### 負載均衡與快取策略

```python
# performance/load_balancer.py
from typing import List, Dict, Any, Optional
from dataclasses import dataclass
from enum import Enum
import hashlib
import time
import random
import structlog

logger = structlog.get_logger()

class LoadBalancingStrategy(Enum):
    ROUND_ROBIN = "round_robin"
    WEIGHTED_ROUND_ROBIN = "weighted_round_robin"
    LEAST_CONNECTIONS = "least_connections"
    IP_HASH = "ip_hash"
    HEALTH_BASED = "health_based"

@dataclass
class BackendServer:
    """後端伺服器定義"""
    id: str
    host: str
    port: int
    weight: int = 100
    max_connections: int = 1000
    current_connections: int = 0
    is_healthy: bool = True
    response_time_ms: float = 0.0
    error_rate: float = 0.0
    last_health_check: Optional[float] = None

class LoadBalancer:
    """負載均衡器"""

    def __init__(self, config: Dict[str, Any]):
        self.config = config
        self.servers: List[BackendServer] = []
        self.round_robin_index = 0

    def get_server(self, client_ip: Optional[str] = None) -> Optional[BackendServer]:
        """根據策略選擇伺服器"""
        healthy_servers = [s for s in self.servers if s.is_healthy]

        if not healthy_servers:
            logger.warning("No healthy servers available")
            return None

        strategy = self.config.get('strategy', LoadBalancingStrategy.ROUND_ROBIN)

        if strategy == LoadBalancingStrategy.ROUND_ROBIN:
            return self._round_robin_selection(healthy_servers)
        elif strategy == LoadBalancingStrategy.LEAST_CONNECTIONS:
            return self._least_connections_selection(healthy_servers)
        elif strategy == LoadBalancingStrategy.IP_HASH:
            return self._ip_hash_selection(healthy_servers, client_ip)

        return healthy_servers[0]

    def _round_robin_selection(self, servers: List[BackendServer]) -> BackendServer:
        """輪詢選擇"""
        selected = servers[self.round_robin_index % len(servers)]
        self.round_robin_index += 1
        return selected

    def _least_connections_selection(self, servers: List[BackendServer]) -> BackendServer:
        """最少連接選擇"""
        return min(servers, key=lambda s: s.current_connections)

    def _ip_hash_selection(self, servers: List[BackendServer], client_ip: str) -> BackendServer:
        """IP 雜湊選擇"""
        if not client_ip:
            return self._round_robin_selection(servers)

        hash_value = int(hashlib.md5(client_ip.encode()).hexdigest(), 16)
        return servers[hash_value % len(servers)]

class CacheManager:
    """快取管理器"""

    def __init__(self, max_size: int = 1000, default_ttl: Optional[float] = None):
        self.max_size = max_size
        self.default_ttl = default_ttl
        self.cache: Dict[str, Any] = {}

    def get(self, key: str) -> Optional[Any]:
        """取得快取值"""
        if key in self.cache:
            entry = self.cache[key]
            if not self._is_expired(entry):
                return entry['value']
            else:
                del self.cache[key]
        return None

    def set(self, key: str, value: Any, ttl: Optional[float] = None) -> None:
        """設定快取值"""
        if len(self.cache) >= self.max_size:
            self._evict()

        self.cache[key] = {
            'value': value,
            'created_at': time.time(),
            'ttl': ttl or self.default_ttl
        }

    def _is_expired(self, entry: Dict[str, Any]) -> bool:
        """檢查條目是否過期"""
        if entry.get('ttl') is None:
            return False
        return time.time() - entry['created_at'] > entry['ttl']

    def _evict(self) -> None:
        """淘汰條目（LRU 策略）"""
        if self.cache:
            oldest_key = min(self.cache.keys(),
                           key=lambda k: self.cache[k]['created_at'])
            del self.cache[oldest_key]
```

---

## 🔐 生產環境安全配置

### 安全加固與配置管理

```python
# security/production_security.py
from typing import Dict, List, Any, Optional
import secrets
import hashlib
from dataclasses import dataclass
from cryptography.fernet import Fernet
import base64
import structlog

logger = structlog.get_logger()

@dataclass
class SecurityConfig:
    """安全配置"""
    encryption_key: str
    jwt_secret: str
    password_salt: str
    api_rate_limits: Dict[str, int]
    allowed_origins: List[str]
    security_headers: Dict[str, str]

class ProductionSecurityManager:
    """生產環境安全管理器"""

    def __init__(self, config: SecurityConfig):
        self.config = config
        self.cipher = Fernet(config.encryption_key.encode())

    def generate_secure_config(self) -> Dict[str, str]:
        """生成安全配置"""
        return {
            'encryption_key': base64.urlsafe_b64encode(Fernet.generate_key()).decode(),
            'jwt_secret': secrets.token_urlsafe(32),
            'password_salt': secrets.token_urlsafe(16),
            'session_secret': secrets.token_urlsafe(32)
        }

    def encrypt_sensitive_data(self, data: str) -> str:
        """加密敏感資料"""
        return self.cipher.encrypt(data.encode()).decode()

    def decrypt_sensitive_data(self, encrypted_data: str) -> str:
        """解密敏感資料"""
        return self.cipher.decrypt(encrypted_data.encode()).decode()

    def get_security_headers(self) -> Dict[str, str]:
        """取得安全標頭"""
        return {
            'X-Content-Type-Options': 'nosniff',
            'X-Frame-Options': 'DENY',
            'X-XSS-Protection': '1; mode=block',
            'Strict-Transport-Security': 'max-age=31536000; includeSubDomains',
            'Content-Security-Policy': "default-src 'self'",
            'Referrer-Policy': 'strict-origin-when-cross-origin'
        }

    def validate_environment_security(self) -> List[str]:
        """驗證環境安全配置"""
        issues = []

        # 檢查金鑰強度
        if len(self.config.jwt_secret) < 32:
            issues.append("JWT secret too short")

        # 檢查 CORS 配置
        if '*' in self.config.allowed_origins:
            issues.append("CORS allows all origins")

        # 檢查速率限制
        if not self.config.api_rate_limits:
            issues.append("No rate limits configured")

        return issues

class EnvironmentConfigManager:
    """環境配置管理器"""

    def __init__(self, environment: str):
        self.environment = environment

    def get_deployment_config(self) -> Dict[str, Any]:
        """取得部署配置"""
        base_config = {
            'replicas': 1,
            'resources': {
                'requests': {'cpu': '100m', 'memory': '128Mi'},
                'limits': {'cpu': '500m', 'memory': '256Mi'}
            },
            'auto_scaling': False
        }

        if self.environment == 'production':
            return {
                **base_config,
                'replicas': 3,
                'resources': {
                    'requests': {'cpu': '200m', 'memory': '256Mi'},
                    'limits': {'cpu': '1000m', 'memory': '512Mi'}
                },
                'auto_scaling': {
                    'min_replicas': 3,
                    'max_replicas': 10,
                    'target_cpu': 70
                }
            }
        elif self.environment == 'staging':
            return {
                **base_config,
                'replicas': 2,
                'resources': {
                    'requests': {'cpu': '100m', 'memory': '128Mi'},
                    'limits': {'cpu': '500m', 'memory': '256Mi'}
                }
            }

        return base_config  # development

    def get_security_policies(self) -> Dict[str, Any]:
        """取得安全策略"""
        policies = {
            'network_policies': {
                'ingress': ['allowed-namespaces'],
                'egress': ['database', 'cache', 'external-apis']
            },
            'pod_security': {
                'run_as_non_root': True,
                'read_only_root_filesystem': True,
                'allow_privilege_escalation': False
            }
        }

        if self.environment == 'production':
            policies['pod_security']['security_context'] = {
                'runAsUser': 1000,
                'runAsGroup': 1000,
                'fsGroup': 1000
            }

        return policies
```

---

## 📊 監控與告警整合

### 生產環境監控配置

```python
# monitoring/production_monitoring.py
from typing import Dict, List, Any
from dataclasses import dataclass
import structlog

logger = structlog.get_logger()

@dataclass
class MonitoringConfig:
    """監控配置"""
    prometheus_endpoint: str
    grafana_endpoint: str
    alert_manager_endpoint: str
    log_aggregation_endpoint: str

class ProductionMonitoringSetup:
    """生產環境監控設定"""

    def __init__(self, config: MonitoringConfig):
        self.config = config

    def get_prometheus_config(self) -> Dict[str, Any]:
        """取得 Prometheus 配置"""
        return {
            'global': {
                'scrape_interval': '15s',
                'evaluation_interval': '15s'
            },
            'rule_files': [
                'auth_service_alerts.yml'
            ],
            'scrape_configs': [
                {
                    'job_name': 'auth-service',
                    'kubernetes_sd_configs': [
                        {
                            'role': 'pod',
                            'namespaces': {
                                'names': ['production', 'staging']
                            }
                        }
                    ],
                    'relabel_configs': [
                        {
                            'source_labels': ['__meta_kubernetes_pod_label_app'],
                            'action': 'keep',
                            'regex': 'auth-service'
                        }
                    ]
                }
            ],
            'alerting': {
                'alertmanagers': [
                    {
                        'static_configs': [
                            {
                                'targets': [self.config.alert_manager_endpoint]
                            }
                        ]
                    }
                ]
            }
        }

    def get_alert_rules(self) -> Dict[str, Any]:
        """取得告警規則"""
        return {
            'groups': [
                {
                    'name': 'auth-service-alerts',
                    'rules': [
                        {
                            'alert': 'AuthServiceHighErrorRate',
                            'expr': 'rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.1',
                            'for': '2m',
                            'labels': {
                                'severity': 'critical',
                                'service': 'auth-service'
                            },
                            'annotations': {
                                'summary': 'Auth service high error rate',
                                'description': 'Error rate is {{ $value | humanizePercentage }}'
                            }
                        },
                        {
                            'alert': 'AuthServiceHighLatency',
                            'expr': 'histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2',
                            'for': '5m',
                            'labels': {
                                'severity': 'warning',
                                'service': 'auth-service'
                            },
                            'annotations': {
                                'summary': 'Auth service high latency',
                                'description': '95th percentile latency is {{ $value }}s'
                            }
                        }
                    ]
                }
            ]
        }

    def get_grafana_dashboard(self) -> Dict[str, Any]:
        """取得 Grafana 儀表板配置"""
        return {
            'dashboard': {
                'title': 'Auth Service Production Dashboard',
                'panels': [
                    {
                        'title': 'Request Rate',
                        'type': 'graph',
                        'targets': [
                            {
                                'expr': 'rate(http_requests_total[5m])',
                                'legendFormat': 'RPS'
                            }
                        ]
                    },
                    {
                        'title': 'Error Rate',
                        'type': 'singlestat',
                        'targets': [
                            {
                                'expr': 'rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])',
                                'legendFormat': 'Error Rate'
                            }
                        ]
                    },
                    {
                        'title': 'Response Time',
                        'type': 'graph',
                        'targets': [
                            {
                                'expr': 'histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[5m]))',
                                'legendFormat': 'p50'
                            },
                            {
                                'expr': 'histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))',
                                'legendFormat': 'p95'
                            }
                        ]
                    }
                ]
            }
        }

# 部署配置範例
def create_production_deployment_config() -> Dict[str, Any]:
    """建立生產環境部署配置"""
    return {
        'apiVersion': 'apps/v1',
        'kind': 'Deployment',
        'metadata': {
            'name': 'auth-service',
            'namespace': 'production',
            'labels': {
                'app': 'auth-service',
                'environment': 'production'
            }
        },
        'spec': {
            'replicas': 3,
            'selector': {
                'matchLabels': {
                    'app': 'auth-service'
                }
            },
            'template': {
                'metadata': {
                    'labels': {
                        'app': 'auth-service',
                        'environment': 'production'
                    }
                },
                'spec': {
                    'serviceAccountName': 'auth-service',
                    'securityContext': {
                        'runAsNonRoot': True,
                        'runAsUser': 1000,
                        'fsGroup': 1000
                    },
                    'containers': [
                        {
                            'name': 'auth-service',
                            'image': 'auth-service:latest',
                            'ports': [
                                {
                                    'containerPort': 8000,
                                    'name': 'http'
                                }
                            ],
                            'env': [
                                {
                                    'name': 'ENVIRONMENT',
                                    'value': 'production'
                                },
                                {
                                    'name': 'LOG_LEVEL',
                                    'value': 'INFO'
                                }
                            ],
                            'resources': {
                                'requests': {
                                    'cpu': '200m',
                                    'memory': '256Mi'
                                },
                                'limits': {
                                    'cpu': '1000m',
                                    'memory': '512Mi'
                                }
                            },
                            'livenessProbe': {
                                'httpGet': {
                                    'path': '/health',
                                    'port': 8000
                                },
                                'initialDelaySeconds': 30,
                                'periodSeconds': 10
                            },
                            'readinessProbe': {
                                'httpGet': {
                                    'path': '/ready',
                                    'port': 8000
                                },
                                'initialDelaySeconds': 5,
                                'periodSeconds': 5
                            }
                        }
                    ]
                }
            }
        }
    }
```

---

## 💡 學習筆記區

### 🤔 我的理解
```
容器化部署的最佳實務：

Kubernetes 編排的核心概念：

CI/CD 管道的設計原則：

效能優化的關鍵策略：
```

### 🚀 實踐心得
```
生產部署的挑戰與解決方案：

容器安全性的重要考量：

自動化部署的價值：

監控與可觀測性的整合：
```

### 🔮 進階思考
```
雲原生部署的演進趨勢：

GitOps 的實踐策略：

服務網格的部署考量：

零停機部署的實現方式：
```

---

## 📚 延伸學習資源

### 相關技術棧
- **容器技術**: Docker, Podman, Containerd
- **編排平台**: Kubernetes, Docker Swarm, Nomad
- **CI/CD 工具**: Jenkins, GitLab CI, GitHub Actions, ArgoCD
- **監控工具**: Prometheus, Grafana, Jaeger, ELK Stack
- **雲端平台**: AWS EKS, Azure AKS, Google GKE
- **安全掃描**: Trivy, Clair, Anchore

### 最佳實務指南
- **十二要素應用**: 12factor.net
- **雲原生 CNCF**: 雲原生計算基金會資源
- **Kubernetes 官方文檔**: 部署與操作指南
- **Docker 最佳實務**: 映像建構與安全性
- **GitOps 實踐**: Flux, ArgoCD 工作流程
- **SRE 實踐**: Google SRE 書籍與方法論