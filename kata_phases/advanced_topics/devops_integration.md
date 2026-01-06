# DevOps Integration: 現代開發運維一體化實踐

## 🎯 學習目標
- 掌握 DevOps 文化與技術實踐的深度整合
- 學習現代 CI/CD 流水線設計與最佳實務
- 建立基礎設施即代碼 (IaC) 的完整解決方案
- 實踐 GitOps 與雲原生部署策略

---

## 🏗️ CI/CD 流水線設計

### GitHub Actions 企業級流水線

```yaml
# .github/workflows/auth-service-pipeline.yml
name: Auth Service CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]
  release:
    types: [ published ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}
  PYTHON_VERSION: "3.11"

jobs:
  # ============ 代碼品質檢查 ============
  code-quality:
    name: Code Quality & Security Scan
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # 完整歷史記錄用於 SonarQube

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: ${{ env.PYTHON_VERSION }}

      - name: Cache dependencies
        uses: actions/cache@v3
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements*.txt') }}

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements-dev.txt

      - name: Code formatting check (Black)
        run: black --check --diff app/ tests/

      - name: Import sorting check (isort)
        run: isort --check-only --diff app/ tests/

      - name: Type checking (mypy)
        run: mypy app/

      - name: Linting (flake8)
        run: flake8 app/ tests/

      - name: Security scan (bandit)
        run: bandit -r app/ -f json -o bandit-report.json

      - name: Dependency vulnerability scan (safety)
        run: safety check --json --output safety-report.json

      - name: Upload security reports
        uses: actions/upload-artifact@v3
        with:
          name: security-reports
          path: |
            bandit-report.json
            safety-report.json

  # ============ 單元測試與覆蓋率 ============
  unit-tests:
    name: Unit Tests & Coverage
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.10", "3.11", "3.12"]

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v4
        with:
          python-version: ${{ matrix.python-version }}

      - name: Install dependencies
        run: |
          pip install -r requirements-dev.txt

      - name: Run unit tests with coverage
        run: |
          pytest tests/unit/ \
            --cov=app \
            --cov-report=xml \
            --cov-report=html \
            --cov-fail-under=90 \
            --junitxml=junit.xml

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.xml
          flags: unittests
          name: codecov-umbrella

      - name: Publish test results
        uses: EnricoMi/publish-unit-test-result-action@v2
        if: always()
        with:
          files: junit.xml

  # ============ 整合測試 ============
  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-latest
    needs: unit-tests

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: ${{ env.PYTHON_VERSION }}

      - name: Install dependencies
        run: pip install -r requirements-dev.txt

      - name: Run integration tests
        env:
          DATABASE_URL: postgresql://postgres:testpass@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379
        run: |
          pytest tests/integration/ \
            --junitxml=integration-junit.xml \
            -v

  # ============ 容器構建 ============
  build-image:
    name: Build Container Image
    runs-on: ubuntu-latest
    needs: [code-quality, unit-tests]
    outputs:
      image-digest: ${{ steps.build.outputs.digest }}
      image-tag: ${{ steps.meta.outputs.tags }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=sha,prefix={{branch}}-
            type=semver,pattern={{version}}

      - name: Build and push
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            PYTHON_VERSION=${{ env.PYTHON_VERSION }}

  # ============ 安全掃描 ============
  container-security:
    name: Container Security Scan
    runs-on: ubuntu-latest
    needs: build-image

    steps:
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ needs.build-image.outputs.image-tag }}
          format: 'sarif'
          output: 'trivy-results.sarif'

      - name: Upload Trivy scan results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'

  # ============ 端對端測試 ============
  e2e-tests:
    name: End-to-End Tests
    runs-on: ubuntu-latest
    needs: build-image
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - name: Set up test environment
        run: |
          docker-compose -f docker-compose.test.yml up -d
          sleep 30  # 等待服務啟動

      - name: Run E2E tests
        run: |
          npm install -g @playwright/test
          playwright install
          pytest tests/e2e/ -v

      - name: Cleanup
        if: always()
        run: docker-compose -f docker-compose.test.yml down

  # ============ 部署到 Staging ============
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: [integration-tests, build-image, container-security]
    if: github.ref == 'refs/heads/main'
    environment:
      name: staging
      url: https://auth-staging.example.com

    steps:
      - name: Deploy to staging
        uses: ./.github/actions/deploy
        with:
          environment: staging
          image: ${{ needs.build-image.outputs.image-tag }}
          kubeconfig: ${{ secrets.KUBECONFIG_STAGING }}

      - name: Run smoke tests
        run: |
          curl -f https://auth-staging.example.com/health || exit 1

  # ============ 部署到 Production ============
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: [deploy-staging, e2e-tests]
    if: github.event_name == 'release'
    environment:
      name: production
      url: https://auth.example.com

    steps:
      - name: Deploy to production
        uses: ./.github/actions/deploy
        with:
          environment: production
          image: ${{ needs.build-image.outputs.image-tag }}
          kubeconfig: ${{ secrets.KUBECONFIG_PRODUCTION }}

      - name: Run production smoke tests
        run: |
          curl -f https://auth.example.com/health || exit 1

      - name: Notify deployment success
        uses: 8398a7/action-slack@v3
        with:
          status: success
          text: '🚀 Auth Service deployed to production successfully!'
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### 進階 CI/CD 策略

```python
# scripts/deployment/deployment_manager.py
class DeploymentManager:
    """智能部署管理器"""

    def __init__(self, config):
        self.config = config
        self.k8s_client = KubernetesClient()
        self.metrics_client = PrometheusClient()

    async def deploy_with_canary(self, image_tag: str, canary_percentage: int = 10):
        """金絲雀部署策略"""
        try:
            # 1. 部署金絲雀版本
            await self._deploy_canary_version(image_tag, canary_percentage)

            # 2. 監控關鍵指標
            metrics = await self._monitor_canary_metrics(duration_minutes=10)

            # 3. 自動決策是否繼續部署
            if self._should_proceed_deployment(metrics):
                await self._promote_canary_to_stable(image_tag)
                return {"status": "success", "deployment_type": "canary"}
            else:
                await self._rollback_canary()
                return {"status": "failed", "reason": "metrics_threshold_exceeded"}

        except Exception as e:
            await self._rollback_canary()
            raise DeploymentError(f"Canary deployment failed: {str(e)}")

    async def blue_green_deployment(self, image_tag: str):
        """藍綠部署策略"""
        current_env = await self._get_current_environment()
        target_env = "green" if current_env == "blue" else "blue"

        try:
            # 1. 部署到目標環境
            await self._deploy_to_environment(target_env, image_tag)

            # 2. 運行健康檢查
            health_check_passed = await self._run_health_checks(target_env)

            if health_check_passed:
                # 3. 切換流量
                await self._switch_traffic(target_env)

                # 4. 清理舊環境
                await self._cleanup_old_environment(current_env)

                return {"status": "success", "new_environment": target_env}
            else:
                await self._cleanup_environment(target_env)
                raise DeploymentError("Health checks failed")

        except Exception as e:
            await self._cleanup_environment(target_env)
            raise DeploymentError(f"Blue-green deployment failed: {str(e)}")

    def _should_proceed_deployment(self, metrics: dict) -> bool:
        """基於指標決定是否繼續部署"""
        thresholds = {
            "error_rate": 0.01,        # 1% 錯誤率
            "response_time_p95": 1000,  # 95% 回應時間 < 1s
            "cpu_usage": 80,           # CPU 使用率 < 80%
            "memory_usage": 80         # 記憶體使用率 < 80%
        }

        for metric, threshold in thresholds.items():
            if metrics.get(metric, 0) > threshold:
                logger.warning(f"Metric {metric} exceeded threshold: {metrics[metric]} > {threshold}")
                return False

        return True

class FeatureFlagManager:
    """功能開關管理"""

    def __init__(self):
        self.redis_client = redis.Redis()

    async def deploy_with_feature_flag(self, feature_name: str, rollout_percentage: int):
        """基於功能開關的漸進式部署"""
        flag_config = {
            "enabled": True,
            "rollout_percentage": rollout_percentage,
            "created_at": datetime.utcnow().isoformat(),
            "rules": {
                "user_attributes": {},
                "request_attributes": {}
            }
        }

        await self.redis_client.hset(
            "feature_flags",
            feature_name,
            json.dumps(flag_config)
        )

    async def gradual_rollout(self, feature_name: str):
        """漸進式功能推出"""
        rollout_stages = [10, 25, 50, 75, 100]

        for percentage in rollout_stages:
            await self.deploy_with_feature_flag(feature_name, percentage)

            # 監控階段
            await asyncio.sleep(300)  # 等待 5 分鐘

            metrics = await self._collect_feature_metrics(feature_name)
            if not self._validate_metrics(metrics):
                # 回滾功能
                await self.disable_feature(feature_name)
                raise FeatureRolloutError(f"Rollout failed at {percentage}%")

            logger.info(f"Feature {feature_name} rolled out to {percentage}%")
```

---

## 🐳 容器化與編排

### 多階段 Dockerfile 最佳實務

```dockerfile
# Dockerfile
# ============ 基礎階段 ============
FROM python:3.11-slim as base

# 設定工作目錄
WORKDIR /app

# 建立非特權用戶
RUN groupadd -r appgroup && useradd -r -g appgroup appuser

# 安裝系統依賴
RUN apt-get update && apt-get install -y \
    build-essential \
    curl \
    && rm -rf /var/lib/apt/lists/*

# ============ 依賴階段 ============
FROM base as dependencies

# 複製依賴文件
COPY requirements.txt requirements-prod.txt ./

# 安裝 Python 依賴
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir -r requirements-prod.txt

# ============ 開發階段 ============
FROM dependencies as development

# 安裝開發依賴
COPY requirements-dev.txt ./
RUN pip install --no-cache-dir -r requirements-dev.txt

# 複製源碼
COPY . .

# 開發環境設定
ENV PYTHONPATH=/app
ENV FASTAPI_ENV=development

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]

# ============ 測試階段 ============
FROM development as test

# 運行測試
RUN pytest tests/ --cov=app --cov-report=xml --cov-fail-under=90

# ============ 生產構建階段 ============
FROM dependencies as production-build

# 複製應用代碼
COPY app/ ./app/
COPY alembic/ ./alembic/
COPY alembic.ini ./

# 編譯 Python 位元組碼
RUN python -m compileall app/

# ============ 生產階段 ============
FROM python:3.11-slim as production

# 從基礎階段複製用戶設定
COPY --from=base /etc/passwd /etc/passwd
COPY --from=base /etc/group /etc/group

# 工作目錄
WORKDIR /app

# 複製安裝的依賴
COPY --from=dependencies /usr/local/lib/python3.11/site-packages /usr/local/lib/python3.11/site-packages
COPY --from=dependencies /usr/local/bin /usr/local/bin

# 複製應用文件
COPY --from=production-build --chown=appuser:appgroup /app .

# 安全設定
USER appuser

# 健康檢查
HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1

# 環境變數
ENV PYTHONPATH=/app
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# 暴露端口
EXPOSE 8000

# 啟動命令
CMD ["gunicorn", "app.main:app", "-w", "4", "-k", "uvicorn.workers.UvicornWorker", "--bind", "0.0.0.0:8000"]
```

### Kubernetes 部署配置

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: auth-service
  labels:
    name: auth-service
    environment: production

---
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: auth-service-config
  namespace: auth-service
data:
  ENVIRONMENT: "production"
  LOG_LEVEL: "INFO"
  DATABASE_POOL_SIZE: "20"
  REDIS_POOL_SIZE: "10"

---
# k8s/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: auth-service-secrets
  namespace: auth-service
type: Opaque
stringData:
  DATABASE_URL: "postgresql://user:pass@postgres:5432/authdb"
  REDIS_URL: "redis://redis:6379/0"
  JWT_SECRET_KEY: "your-super-secret-jwt-key"

---
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
  namespace: auth-service
  labels:
    app: auth-service
    version: v1
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: auth-service
  template:
    metadata:
      labels:
        app: auth-service
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8000"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: auth-service
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      containers:
      - name: auth-service
        image: ghcr.io/yourorg/auth-service:latest
        ports:
        - name: http
          containerPort: 8000
        envFrom:
        - configMapRef:
            name: auth-service-config
        - secretRef:
            name: auth-service-secrets
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
        lifecycle:
          preStop:
            exec:
              command: ["sleep", "15"]  # 優雅關閉

---
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: auth-service
  namespace: auth-service
  labels:
    app: auth-service
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: http
    protocol: TCP
    name: http
  selector:
    app: auth-service

---
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: auth-service-hpa
  namespace: auth-service
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: auth-service
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80

---
# k8s/pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: auth-service-pdb
  namespace: auth-service
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: auth-service
```

---

## 🏗️ 基礎設施即代碼 (Infrastructure as Code)

### Terraform 雲基礎設施管理

```hcl
# terraform/main.tf
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
  }

  backend "s3" {
    bucket = "yourcompany-terraform-state"
    key    = "auth-service/terraform.tfstate"
    region = "us-west-2"

    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}

# ============ 變數定義 ============
variable "environment" {
  description = "Environment name"
  type        = string
  default     = "production"
}

variable "cluster_name" {
  description = "EKS cluster name"
  type        = string
  default     = "auth-service-cluster"
}

# ============ VPC 配置 ============
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"

  name = "${var.cluster_name}-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-west-2a", "us-west-2b", "us-west-2c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway = true
  enable_vpn_gateway = false
  enable_dns_hostnames = true
  enable_dns_support = true

  tags = {
    "kubernetes.io/cluster/${var.cluster_name}" = "shared"
    Environment = var.environment
  }
}

# ============ EKS 集群 ============
module "eks" {
  source = "terraform-aws-modules/eks/aws"

  cluster_name    = var.cluster_name
  cluster_version = "1.28"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  # 集群端點配置
  cluster_endpoint_private_access = true
  cluster_endpoint_public_access  = true

  # 集群加密
  cluster_encryption_config = [
    {
      provider_key_arn = aws_kms_key.eks.arn
      resources        = ["secrets"]
    }
  ]

  # 節點組配置
  node_groups = {
    auth_service = {
      desired_capacity = 3
      max_capacity     = 6
      min_capacity     = 3

      instance_types = ["t3.medium"]
      capacity_type  = "ON_DEMAND"

      k8s_labels = {
        Environment = var.environment
        Application = "auth-service"
      }

      # 節點組更新配置
      update_config = {
        max_unavailable_percentage = 25
      }
    }
  }

  tags = {
    Environment = var.environment
    Application = "auth-service"
  }
}

# ============ RDS 資料庫 ============
resource "aws_db_subnet_group" "auth_db" {
  name       = "${var.cluster_name}-db-subnet"
  subnet_ids = module.vpc.private_subnets

  tags = {
    Name = "Auth service DB subnet group"
  }
}

resource "aws_db_instance" "auth_db" {
  allocated_storage    = 100
  max_allocated_storage = 1000
  storage_encrypted    = true
  storage_type         = "gp3"

  engine         = "postgres"
  engine_version = "15.4"
  instance_class = "db.t3.medium"

  db_name  = "authdb"
  username = "authuser"
  password = var.db_password

  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name   = aws_db_subnet_group.auth_db.name

  backup_retention_period = 7
  backup_window          = "03:00-04:00"
  maintenance_window     = "sun:04:00-sun:05:00"

  deletion_protection = true
  skip_final_snapshot = false
  final_snapshot_identifier = "${var.cluster_name}-final-snapshot"

  performance_insights_enabled = true
  monitoring_interval         = 60
  monitoring_role_arn         = aws_iam_role.rds_enhanced_monitoring.arn

  tags = {
    Environment = var.environment
    Application = "auth-service"
  }
}

# ============ ElastiCache Redis ============
resource "aws_elasticache_subnet_group" "auth_cache" {
  name       = "${var.cluster_name}-cache-subnet"
  subnet_ids = module.vpc.private_subnets
}

resource "aws_elasticache_replication_group" "auth_cache" {
  description       = "Auth service Redis cache"
  replication_group_id = "${var.cluster_name}-cache"

  port               = 6379
  parameter_group_name = "default.redis7"
  node_type          = "cache.t3.micro"
  num_cache_clusters = 2

  subnet_group_name  = aws_elasticache_subnet_group.auth_cache.name
  security_group_ids = [aws_security_group.elasticache.id]

  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  auth_token                 = var.redis_auth_token

  tags = {
    Environment = var.environment
    Application = "auth-service"
  }
}

# ============ 安全組 ============
resource "aws_security_group" "rds" {
  name_prefix = "${var.cluster_name}-rds"
  vpc_id      = module.vpc.vpc_id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [module.eks.node_security_group_id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.cluster_name}-rds-sg"
  }
}

# ============ 輸出 ============
output "cluster_endpoint" {
  value = module.eks.cluster_endpoint
}

output "database_endpoint" {
  value = aws_db_instance.auth_db.endpoint
}

output "redis_endpoint" {
  value = aws_elasticache_replication_group.auth_cache.primary_endpoint_address
}
```

### Helm Charts 應用部署

```yaml
# helm/auth-service/Chart.yaml
apiVersion: v2
name: auth-service
description: A Helm chart for Auth Service
type: application
version: 0.1.0
appVersion: "1.0.0"

dependencies:
- name: postgresql
  version: 12.1.2
  repository: https://charts.bitnami.com/bitnami
  condition: postgresql.enabled
- name: redis
  version: 17.3.7
  repository: https://charts.bitnami.com/bitnami
  condition: redis.enabled

---
# helm/auth-service/values.yaml
# 預設值配置
replicaCount: 3

image:
  repository: ghcr.io/yourorg/auth-service
  tag: latest
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8000

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/rate-limit: "100"
  hosts:
    - host: auth.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: auth-service-tls
      hosts:
        - auth.example.com

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 256Mi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

# 依賴服務
postgresql:
  enabled: true
  auth:
    postgresPassword: "changeme"
    database: "authdb"
  primary:
    persistence:
      enabled: true
      size: 100Gi

redis:
  enabled: true
  auth:
    enabled: true
    password: "changeme"
  master:
    persistence:
      enabled: true
      size: 8Gi

---
# helm/auth-service/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "auth-service.fullname" . }}
  labels:
    {{- include "auth-service.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "auth-service.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
      labels:
        {{- include "auth-service.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 8000
              protocol: TCP
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: {{ include "auth-service.fullname" . }}-secrets
                  key: database-url
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: {{ include "auth-service.fullname" . }}-secrets
                  key: redis-url
          livenessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

---

## 🔄 GitOps 與持續部署

### ArgoCD GitOps 配置

```yaml
# gitops/applications/auth-service.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: auth-service
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/yourorg/auth-service
    targetRevision: HEAD
    path: k8s/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: auth-service
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
    - PruneLast=true
    retry:
      limit: 2
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

---
# gitops/projects/auth-service-project.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: auth-service-project
  namespace: argocd
spec:
  description: Auth Service Project

  sourceRepos:
  - https://github.com/yourorg/auth-service
  - https://helm.elastic.co
  - https://prometheus-community.github.io/helm-charts

  destinations:
  - namespace: 'auth-service*'
    server: https://kubernetes.default.svc
  - namespace: 'monitoring'
    server: https://kubernetes.default.svc

  clusterResourceWhitelist:
  - group: ''
    kind: Namespace
  - group: rbac.authorization.k8s.io
    kind: ClusterRole
  - group: rbac.authorization.k8s.io
    kind: ClusterRoleBinding

  namespaceResourceWhitelist:
  - group: ''
    kind: '*'
  - group: apps
    kind: '*'
  - group: networking.k8s.io
    kind: '*'

  roles:
  - name: developers
    policies:
    - p, proj:auth-service-project:developers, applications, get, auth-service-project/*, allow
    - p, proj:auth-service-project:developers, applications, sync, auth-service-project/*, allow
    groups:
    - auth-service-developers
```

### Kustomize 環境管理

```yaml
# k8s/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml
- configmap.yaml
- secret.yaml

commonLabels:
  app: auth-service

---
# k8s/overlays/staging/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: auth-service-staging

bases:
- ../../base

patchesStrategicMerge:
- deployment-patch.yaml

configMapGenerator:
- name: auth-service-config
  literals:
  - ENVIRONMENT=staging
  - LOG_LEVEL=DEBUG

---
# k8s/overlays/staging/deployment-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
spec:
  replicas: 2
  template:
    spec:
      containers:
      - name: auth-service
        resources:
          requests:
            memory: "128Mi"
            cpu: "50m"
          limits:
            memory: "256Mi"
            cpu: "200m"

---
# k8s/overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: auth-service

bases:
- ../../base

patchesStrategicMerge:
- deployment-patch.yaml
- hpa.yaml

configMapGenerator:
- name: auth-service-config
  literals:
  - ENVIRONMENT=production
  - LOG_LEVEL=INFO
```

---

## 🔍 監控與可觀測性

### Prometheus 監控配置

```yaml
# monitoring/prometheus/servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: auth-service-metrics
  namespace: auth-service
  labels:
    app: auth-service
    release: prometheus
spec:
  selector:
    matchLabels:
      app: auth-service
  endpoints:
  - port: http
    path: /metrics
    interval: 30s
    scrapeTimeout: 10s

---
# monitoring/prometheus/prometheusrule.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: auth-service-alerts
  namespace: auth-service
  labels:
    prometheus: kube-prometheus
    role: alert-rules
spec:
  groups:
  - name: auth-service.rules
    interval: 30s
    rules:
    # 高錯誤率告警
    - alert: AuthServiceHighErrorRate
      expr: |
        rate(http_requests_total{service="auth-service",status=~"5.."}[5m]) /
        rate(http_requests_total{service="auth-service"}[5m]) > 0.05
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: "Auth service has high error rate"
        description: "Error rate is {{ $value | humanizePercentage }}"

    # 高回應時間告警
    - alert: AuthServiceHighLatency
      expr: |
        histogram_quantile(0.95, rate(http_request_duration_seconds_bucket{service="auth-service"}[5m])) > 1
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Auth service has high latency"
        description: "95th percentile latency is {{ $value }}s"

    # 服務不可用告警
    - alert: AuthServiceDown
      expr: up{service="auth-service"} == 0
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "Auth service is down"
        description: "Auth service has been down for more than 1 minute"
```

### Grafana 儀表板

```json
{
  "dashboard": {
    "title": "Auth Service Dashboard",
    "panels": [
      {
        "title": "Request Rate",
        "type": "stat",
        "targets": [
          {
            "expr": "rate(http_requests_total{service=\"auth-service\"}[5m])",
            "legendFormat": "RPS"
          }
        ]
      },
      {
        "title": "Error Rate",
        "type": "stat",
        "targets": [
          {
            "expr": "rate(http_requests_total{service=\"auth-service\",status=~\"5..\"}[5m]) / rate(http_requests_total{service=\"auth-service\"}[5m])",
            "legendFormat": "Error Rate"
          }
        ]
      },
      {
        "title": "Response Time",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.50, rate(http_request_duration_seconds_bucket{service=\"auth-service\"}[5m]))",
            "legendFormat": "p50"
          },
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket{service=\"auth-service\"}[5m]))",
            "legendFormat": "p95"
          }
        ]
      }
    ]
  }
}
```

---

## 🚨 災難恢復與備份策略

### 自動化備份系統

```python
# scripts/backup/backup_manager.py
class BackupManager:
    """企業級備份管理系統"""

    def __init__(self):
        self.s3_client = boto3.client('s3')
        self.rds_client = boto3.client('rds')
        self.backup_bucket = "auth-service-backups"

    async def create_full_backup(self) -> BackupResult:
        """建立完整備份"""
        backup_id = f"backup-{datetime.utcnow().strftime('%Y%m%d-%H%M%S')}"

        try:
            # 1. 資料庫備份
            db_backup = await self._backup_database(backup_id)

            # 2. 設定檔備份
            config_backup = await self._backup_configurations(backup_id)

            # 3. 密鑰備份（加密）
            secrets_backup = await self._backup_secrets(backup_id)

            # 4. 建立備份清單
            manifest = {
                "backup_id": backup_id,
                "timestamp": datetime.utcnow().isoformat(),
                "components": {
                    "database": db_backup,
                    "configurations": config_backup,
                    "secrets": secrets_backup
                },
                "checksum": self._calculate_backup_checksum([
                    db_backup, config_backup, secrets_backup
                ])
            }

            await self._upload_backup_manifest(backup_id, manifest)

            return BackupResult(
                success=True,
                backup_id=backup_id,
                size_mb=sum([b["size_mb"] for b in manifest["components"].values()])
            )

        except Exception as e:
            await self._cleanup_failed_backup(backup_id)
            raise BackupError(f"Backup failed: {str(e)}")

    async def restore_from_backup(self, backup_id: str, target_env: str):
        """從備份恢復"""
        try:
            # 1. 驗證備份完整性
            manifest = await self._load_backup_manifest(backup_id)
            if not await self._verify_backup_integrity(manifest):
                raise BackupError("Backup integrity check failed")

            # 2. 恢復資料庫
            await self._restore_database(manifest["components"]["database"], target_env)

            # 3. 恢復設定
            await self._restore_configurations(manifest["components"]["configurations"], target_env)

            # 4. 恢復密鑰
            await self._restore_secrets(manifest["components"]["secrets"], target_env)

            # 5. 驗證恢復結果
            await self._verify_restoration(target_env)

            logger.info(f"Successfully restored from backup {backup_id} to {target_env}")

        except Exception as e:
            await self._rollback_restoration(target_env)
            raise RestoreError(f"Restoration failed: {str(e)}")

class DisasterRecoveryOrchestrator:
    """災難恢復協調器"""

    def __init__(self):
        self.backup_manager = BackupManager()
        self.infrastructure_manager = TerraformManager()
        self.deployment_manager = DeploymentManager()

    async def execute_dr_plan(self, disaster_type: str, target_region: str):
        """執行災難恢復計劃"""
        recovery_id = f"dr-{uuid.uuid4().hex[:8]}"

        try:
            # 1. 評估災難範圍
            damage_assessment = await self._assess_disaster_impact(disaster_type)

            # 2. 選擇恢復策略
            recovery_strategy = self._select_recovery_strategy(damage_assessment)

            # 3. 建立 DR 基礎設施
            if recovery_strategy.requires_new_infrastructure:
                await self.infrastructure_manager.deploy_dr_infrastructure(target_region)

            # 4. 恢復資料
            latest_backup = await self.backup_manager.get_latest_backup()
            await self.backup_manager.restore_from_backup(latest_backup.id, target_region)

            # 5. 部署應用
            await self.deployment_manager.deploy_to_dr_site(target_region)

            # 6. 切換流量
            await self._switch_traffic_to_dr(target_region)

            # 7. 驗證 DR 環境
            await self._validate_dr_environment(target_region)

            return RecoveryResult(
                success=True,
                recovery_id=recovery_id,
                rto_achieved=self._calculate_rto(),
                rpo_achieved=self._calculate_rpo()
            )

        except Exception as e:
            await self._abort_dr_procedure(recovery_id)
            raise DisasterRecoveryError(f"DR execution failed: {str(e)}")
```

---

## 📊 DevOps 指標與改進

### DevOps 關鍵指標監控

```python
class DevOpsMetricsCollector:
    """DevOps 關鍵指標收集器"""

    def __init__(self):
        self.metrics_store = MetricsDatabase()

    async def collect_dora_metrics(self) -> DORAMetrics:
        """收集 DORA 四大指標"""

        # 1. Deployment Frequency (部署頻率)
        deployment_frequency = await self._calculate_deployment_frequency()

        # 2. Lead Time for Changes (變更前置時間)
        lead_time = await self._calculate_lead_time()

        # 3. Mean Time to Recovery (平均恢復時間)
        mttr = await self._calculate_mttr()

        # 4. Change Failure Rate (變更失敗率)
        change_failure_rate = await self._calculate_change_failure_rate()

        return DORAMetrics(
            deployment_frequency=deployment_frequency,
            lead_time_hours=lead_time,
            mttr_hours=mttr,
            change_failure_rate=change_failure_rate
        )

    async def _calculate_deployment_frequency(self) -> float:
        """計算部署頻率（每週部署次數）"""
        deployments = await self.metrics_store.get_deployments_last_weeks(4)
        return len(deployments) / 4.0

    async def _calculate_lead_time(self) -> float:
        """計算從 commit 到 production 的時間"""
        recent_deployments = await self.metrics_store.get_recent_deployments(30)

        lead_times = []
        for deployment in recent_deployments:
            commit_time = deployment.first_commit_time
            deploy_time = deployment.production_deploy_time
            lead_times.append((deploy_time - commit_time).total_seconds() / 3600)

        return statistics.mean(lead_times) if lead_times else 0

    async def generate_improvement_recommendations(self, metrics: DORAMetrics) -> List[str]:
        """基於指標生成改進建議"""
        recommendations = []

        if metrics.deployment_frequency < 1:  # 少於每週一次
            recommendations.append("增加部署頻率：考慮實作特性開關和更小的變更集")

        if metrics.lead_time_hours > 24:  # 超過一天
            recommendations.append("縮短前置時間：優化 CI/CD 流水線和自動化測試")

        if metrics.mttr_hours > 4:  # 超過 4 小時
            recommendations.append("改進事故回應：建立更好的監控和自動回滾機制")

        if metrics.change_failure_rate > 0.15:  # 超過 15%
            recommendations.append("提高變更品質：加強代碼審查和測試覆蓋率")

        return recommendations

class ContinuousImprovementEngine:
    """持續改進引擎"""

    def __init__(self):
        self.metrics_collector = DevOpsMetricsCollector()
        self.optimization_rules = self._load_optimization_rules()

    async def analyze_pipeline_performance(self) -> PipelineAnalysis:
        """分析流水線效能"""
        pipeline_runs = await self._get_recent_pipeline_runs(100)

        analysis = {
            "average_duration": statistics.mean([r.duration for r in pipeline_runs]),
            "success_rate": sum(1 for r in pipeline_runs if r.status == "success") / len(pipeline_runs),
            "bottlenecks": self._identify_bottlenecks(pipeline_runs),
            "failure_patterns": self._analyze_failure_patterns(pipeline_runs),
            "optimization_opportunities": self._find_optimization_opportunities(pipeline_runs)
        }

        return PipelineAnalysis(**analysis)

    def _identify_bottlenecks(self, pipeline_runs: List[PipelineRun]) -> List[Bottleneck]:
        """識別流水線瓶頸"""
        stage_durations = defaultdict(list)

        for run in pipeline_runs:
            for stage in run.stages:
                stage_durations[stage.name].append(stage.duration)

        bottlenecks = []
        for stage_name, durations in stage_durations.items():
            avg_duration = statistics.mean(durations)
            if avg_duration > 300:  # 超過 5 分鐘
                bottlenecks.append(Bottleneck(
                    stage=stage_name,
                    average_duration=avg_duration,
                    impact_score=self._calculate_impact_score(durations)
                ))

        return sorted(bottlenecks, key=lambda x: x.impact_score, reverse=True)
```

---

## 💡 學習筆記區

### 🤔 我的理解
```
DevOps 文化與技術實踐的關係：

CI/CD 流水線設計的關鍵原則：

基礎設施即代碼的優勢：

GitOps 與傳統部署的差異：
```

### 🔧 實踐心得
```
實施 DevOps 過程中的挑戰：

最有價值的 DevOps 實踐：

監控與可觀測性的重要性：

團隊協作文化的改變：
```

### 🚀 進階思考
```
雲原生架構對 DevOps 的影響：

AI/ML 在 DevOps 中的應用：

未來 DevOps 工具鏈的演進：

企業 DevOps 轉型策略：
```