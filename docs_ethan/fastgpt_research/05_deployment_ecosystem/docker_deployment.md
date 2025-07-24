# FastGPT Docker 部署架构深度分析

## 🐳 Docker 部署架构概述

FastGPT 提供了**完整的容器化部署方案**，支持多种数据库后端和部署环境。通过精心设计的 Docker 架构，实现了**一键部署**、**高可用性**和**横向扩展**能力。

### 核心架构特点

- **多数据库支持** - PostgreSQL+pgvector、Milvus、OceanBase、Zilliz Cloud
- **微服务架构** - 应用、沙箱、MCP 服务、插件系统独立部署
- **云原生设计** - 支持 Docker Compose 和 Kubernetes 部署
- **安全优化** - 非 root 用户、网络隔离、密钥管理
- **CI/CD 集成** - 自动化构建、多架构支持、镜像分发

## 📦 Docker 配置文件架构

### 1. 主应用 Dockerfile

**文件路径**: `projects/app/Dockerfile`

```dockerfile
# 多阶段构建优化
FROM node:20.14.0-alpine AS deps
WORKDIR /app

# 依赖安装阶段
COPY package.json pnpm-lock.yaml ./
COPY packages/global packages/global
COPY packages/service packages/service
COPY packages/web packages/web

# 中国镜像加速
RUN if [ "${CHINA_REGISTRY}" = "true" ]; then \
    npm config set registry https://registry.npmmirror.com/; \
    fi

# 安装依赖（包含原生模块）
RUN apk add --no-cache \
    python3 \
    py3-pip \
    build-base \
    cairo-dev \
    pango-dev \
    giflib-dev \
    && npm install -g pnpm@8.15.5 \
    && pnpm install --frozen-lockfile

# 构建阶段
FROM node:20.14.0-alpine AS builder
WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules
COPY . .

# 内存优化构建
ENV NODE_OPTIONS="--max-old-space-size=4096"
RUN npm install -g pnpm@8.15.5 \
    && pnpm build:app

# 运行时阶段
FROM node:20.14.0-alpine AS runner
WORKDIR /app

# 安全用户配置
RUN addgroup --system --gid 1001 nodejs \
    && adduser --system --uid 1001 nextjs

# 运行时依赖
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

# 应用配置
USER nextjs
EXPOSE 3000
ENV PORT 3000
ENV HOSTNAME "0.0.0.0"

# 启动命令
CMD ["node", "server.js"]
```

### 2. 沙箱服务 Dockerfile

**文件路径**: `projects/sandbox/Dockerfile`

```dockerfile
# Python + Node.js 混合环境
FROM python:3.11 AS base

# 基础系统配置
RUN apt-get update && apt-get install -y \
    libseccomp-dev \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Node.js 安装
ENV NODE_VERSION=20.14.0
RUN curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash \
    && . ~/.bashrc \
    && nvm install $NODE_VERSION \
    && nvm use $NODE_VERSION

# Python 依赖
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Node.js 依赖
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN npm install -g pnpm@8.15.5 \
    && pnpm install --frozen-lockfile

# 应用代码
COPY . .

# 安全配置
USER nobody
EXPOSE 3001

# 启动命令
CMD ["python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "3001"]
```

### 3. MCP 服务 Dockerfile

**文件路径**: `projects/mcp_server/Dockerfile`

```dockerfile
# 轻量级 Node.js 镜像
FROM node:20.14.0-alpine AS base

WORKDIR /app

# 系统依赖
RUN apk add --no-cache \
    curl \
    unzip

# Bun 运行时安装
RUN curl -fsSL https://bun.sh/install | bash \
    && ln -s ~/.bun/bin/bun /usr/local/bin/bun

# 依赖安装
COPY package.json pnpm-lock.yaml ./
RUN npm install -g pnpm@8.15.5 \
    && pnpm install --frozen-lockfile

# 应用代码
COPY . .

# 构建应用
RUN pnpm build

# 非 root 用户
USER node
EXPOSE 3000

# 启动命令
CMD ["node", "dist/index.js"]
```

## 🔧 容器编排配置

### 1. PostgreSQL + pgvector 配置

**文件路径**: `docker-compose-pgvector.yml`

```yaml
version: '3.8'

networks:
  fastgpt:
    driver: bridge

services:
  # PostgreSQL 向量数据库
  pg:
    image: registry.cn-hangzhou.aliyuncs.com/fastgpt/pgvector:v0.7.0
    container_name: pg
    restart: always
    networks:
      - fastgpt
    environment:
      - POSTGRES_USER=username
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=postgres
    volumes:
      - ./pg/data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U username -d postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s

  # MongoDB 主数据库
  mongo:
    image: mongo:5.0.18
    container_name: mongo
    restart: always
    networks:
      - fastgpt
    environment:
      - MONGO_INITDB_ROOT_USERNAME=myusername
      - MONGO_INITDB_ROOT_PASSWORD=mypassword
    volumes:
      - ./mongo/data:/data/db
      - ./mongo/logs:/var/log/mongodb
    command: --replSet rs0 --oplogSize 128
    healthcheck:
      test: ["CMD", "mongo", "--quiet", "--eval", "db.runCommand('ping').ok"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s

  # Redis 缓存
  redis:
    image: redis:7.0.15-alpine
    container_name: redis
    restart: always
    networks:
      - fastgpt
    environment:
      - REDIS_PASSWORD=mypassword
    volumes:
      - ./redis/data:/data
    command: redis-server --requirepass mypassword --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "mypassword", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  # MinIO 对象存储
  minio:
    image: minio/minio:RELEASE.2023-12-20T01-00-02Z
    container_name: minio
    restart: always
    networks:
      - fastgpt
    ports:
      - 9000:9000
      - 9001:9001
    environment:
      - MINIO_ROOT_USER=minioadmin
      - MINIO_ROOT_PASSWORD=minioadmin
    volumes:
      - ./fastgpt-minio:/data
    command: server /data --console-address ":9001"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

  # FastGPT 主应用
  fastgpt:
    container_name: fastgpt
    image: ghcr.io/labring/fastgpt:latest
    restart: always
    networks:
      - fastgpt
    ports:
      - 3000:3000
    environment:
      # 数据库连接
      - MONGODB_URI=mongodb://myusername:mypassword@mongo:27017/fastgpt?authSource=admin
      - REDIS_URL=redis://default:mypassword@redis:6379
      - PG_URL=postgresql://username:password@pg:5432/postgres
      
      # 安全配置
      - DEFAULT_ROOT_PSW=1234
      - TOKEN_KEY=any
      - ROOT_KEY=root_key
      - FILE_TOKEN_KEY=filetoken
      - AES256_SECRET_KEY=fastgptkey
      
      # 性能配置
      - DB_MAX_LINK=30
      - WORKFLOW_MAX_RUN_TIMES=1000
      - LOG_LEVEL=info
    volumes:
      - ./config.json:/app/data/config.json
    depends_on:
      mongo:
        condition: service_healthy
      pg:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # 代码执行沙箱
  sandbox:
    container_name: sandbox
    image: ghcr.io/labring/fastgpt-sandbox:latest
    restart: always
    networks:
      - fastgpt
    environment:
      - APIKEY=fastgpt-sandbox-xxxx
    security_opt:
      - seccomp:unconfined
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3001/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

### 2. Milvus 向量数据库配置

**文件路径**: `docker-compose-milvus.yml`

```yaml
version: '3.8'

services:
  # Milvus 向量数据库
  milvus:
    image: milvusdb/milvus:v2.3.4
    container_name: milvus
    restart: always
    networks:
      - fastgpt
    environment:
      - ETCD_ENDPOINTS=etcd:2379
      - MINIO_ADDRESS=minio:9000
      - MINIO_ACCESS_KEY=minioadmin
      - MINIO_SECRET_KEY=minioadmin
    volumes:
      - ./milvus/data:/var/lib/milvus
      - ./milvus.yaml:/milvus/configs/milvus.yaml
    command: ["milvus", "run", "standalone"]
    depends_on:
      etcd:
        condition: service_healthy
      minio:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9091/healthz"]
      interval: 30s
      timeout: 20s
      retries: 5

  # etcd 协调服务
  etcd:
    image: quay.io/coreos/etcd:v3.5.5
    container_name: etcd
    restart: always
    networks:
      - fastgpt
    environment:
      - ETCD_AUTO_COMPACTION_MODE=revision
      - ETCD_AUTO_COMPACTION_RETENTION=1000
      - ETCD_QUOTA_BACKEND_BYTES=4294967296
      - ETCD_SNAPSHOT_COUNT=50000
    volumes:
      - ./etcd/data:/etcd
    command: etcd -advertise-client-urls=http://127.0.0.1:2379 -listen-client-urls http://0.0.0.0:2379 --data-dir /etcd
    healthcheck:
      test: ["CMD", "etcdctl", "endpoint", "health"]
      interval: 30s
      timeout: 20s
      retries: 3
```

### 3. OceanBase 配置

**文件路径**: `docker-compose-oceanbase/docker-compose.yml`

```yaml
version: '3.8'

services:
  # OceanBase 数据库
  oceanbase:
    image: oceanbase/oceanbase-ce:4.2.1.6-106000012024013018
    container_name: oceanbase
    restart: always
    networks:
      - fastgpt
    environment:
      - MODE=slim
      - OB_CLUSTER_NAME=obcluster
      - OB_TENANT_NAME=test
      - OB_USER_NAME=root
      - OB_USER_PASSWORD=oceanbase
    ports:
      - "2881:2881"
    volumes:
      - ./oceanbase/data:/root/ob
      - ./oceanbase/init.sql:/root/boot/init.sql
    healthcheck:
      test: ["CMD", "/bin/bash", "-c", "mysql -h127.0.0.1 -P2881 -uroot -poceanbase -e 'select 1'"]
      interval: 10s
      timeout: 5s
      retries: 120
      start_period: 60s

  # 自定义初始化脚本
  ob-init:
    image: oceanbase/oceanbase-ce:4.2.1.6-106000012024013018
    container_name: ob-init
    networks:
      - fastgpt
    volumes:
      - ./oceanbase/init.sql:/init.sql
    depends_on:
      oceanbase:
        condition: service_healthy
    command: >
      bash -c "
        mysql -h oceanbase -P 2881 -u root -p oceanbase < /init.sql
      "
```

## 🌐 网络与服务发现

### 1. 内部网络架构

```yaml
# 单一网络设计
networks:
  fastgpt:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 172.20.0.0/16
```

### 2. 服务通信模式

```yaml
# 服务发现规则
服务间通信:
├── fastgpt → mongo:27017 (数据库连接)
├── fastgpt → redis:6379 (缓存连接)
├── fastgpt → pg:5432 (向量数据库)
├── fastgpt → minio:9000 (对象存储)
├── fastgpt → sandbox:3001 (代码执行)
└── fastgpt → mcp-server:3000 (模型通信)
```

### 3. 端口映射策略

```yaml
外部端口暴露:
├── 3000:3000 (FastGPT 主应用)
├── 9000:9000 (MinIO API - 可选)
├── 9001:9001 (MinIO 控制台 - 可选)
└── 3005:3000 (MCP 服务器 - 可选)

# 生产环境建议仅暴露 3000 端口
```

## 🔒 安全配置与最佳实践

### 1. 容器安全

```yaml
# 安全配置示例
services:
  fastgpt:
    security_opt:
      - no-new-privileges:true
    read_only: true
    tmpfs:
      - /tmp
      - /var/run
    user: "1001:1001"  # 非 root 用户
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
```

### 2. 密钥管理

```yaml
# 生产环境密钥配置
environment:
  # 强密码要求
  - DEFAULT_ROOT_PSW=${FASTGPT_ROOT_PASSWORD}
  - TOKEN_KEY=${JWT_SECRET_KEY}
  - ROOT_KEY=${ROOT_API_KEY}
  - FILE_TOKEN_KEY=${FILE_TOKEN_SECRET}
  - AES256_SECRET_KEY=${AES_ENCRYPTION_KEY}
  
  # 数据库密钥
  - MONGODB_URI=mongodb://${MONGO_USER}:${MONGO_PASSWORD}@mongo:27017/fastgpt?authSource=admin
  - REDIS_URL=redis://default:${REDIS_PASSWORD}@redis:6379
  - PG_URL=postgresql://${PG_USER}:${PG_PASSWORD}@pg:5432/postgres
```

### 3. 网络安全

```yaml
# 防火墙规则
networks:
  fastgpt:
    driver: bridge
    driver_opts:
      com.docker.network.bridge.enable_icc: "true"
      com.docker.network.bridge.enable_ip_masquerade: "true"
      com.docker.network.bridge.host_binding_ipv4: "127.0.0.1"
```

## 📈 健康检查与监控

### 1. 健康检查配置

```yaml
# 全面的健康检查
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:3000/api/health"]
  interval: 30s      # 检查间隔
  timeout: 10s       # 超时时间
  retries: 3         # 重试次数
  start_period: 60s  # 启动等待期
```

### 2. 数据库健康检查

```yaml
# MongoDB 健康检查
mongo:
  healthcheck:
    test: |
      bash -c 'mongo --quiet --eval "
        try { 
          rs.status().ok 
        } catch (err) { 
          rs.initiate({
            _id: \"rs0\", 
            members: [{_id: 0, host: \"localhost:27017\"}]
          }).ok 
        }"'
    interval: 10s
    timeout: 5s
    retries: 5
    start_period: 30s

# PostgreSQL 健康检查  
pg:
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U username -d postgres"]
    interval: 10s
    timeout: 5s
    retries: 5
    start_period: 10s

# Redis 健康检查
redis:
  healthcheck:
    test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
    interval: 5s
    timeout: 3s
    retries: 5
```

## 🔄 CI/CD 集成

### 1. GitHub Actions 工作流

**文件路径**: `.github/workflows/fastgpt-build-image.yml`

```yaml
name: Build FastGPT Image

on:
  push:
    branches: [ main, develop ]
    paths:
      - 'projects/app/**'
      - 'packages/**'
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        platform: [linux/amd64, linux/arm64]
        
    steps:
    - name: Checkout
      uses: actions/checkout@v4
      
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
      
    - name: Login to GitHub Container Registry
      uses: docker/login-action@v3
      with:
        registry: ghcr.io
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
        
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ghcr.io/labring/fastgpt
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=sha,prefix={{branch}}-
          type=raw,value=latest,enable={{is_default_branch}}
          
    - name: Build and push
      uses: docker/build-push-action@v5
      with:
        context: .
        file: projects/app/Dockerfile
        platforms: ${{ matrix.platform }}
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
        build-args: |
          CHINA_REGISTRY=${{ contains(github.event.head_commit.message, '[china]') }}
```

### 2. 多镜像仓库发布

```yaml
# 镜像分发策略
registries:
├── ghcr.io/labring/fastgpt (GitHub Container Registry)
├── registry.cn-hangzhou.aliyuncs.com/fastgpt/fastgpt (阿里云)
└── docker.io/labring/fastgpt (Docker Hub)

# 自动化发布流程
- 代码推送到 main 分支
- 触发 CI/CD 构建
- 多架构镜像构建 (AMD64, ARM64)
- 同时推送到多个镜像仓库
- 版本标签和 latest 标签管理
```

## ☸️ Kubernetes 部署

### 1. Helm Chart 配置

**文件路径**: `deploy/helm/fastgpt/`

```yaml
# values.yaml 核心配置
global:
  imageRegistry: "ghcr.io"
  imagePullSecrets: []

fastgpt:
  image:
    repository: labring/fastgpt
    tag: "latest"
    pullPolicy: IfNotPresent
  
  replicaCount: 2
  
  resources:
    limits:
      cpu: 2000m
      memory: 4Gi
    requests:
      cpu: 500m
      memory: 1Gi
  
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 10
    targetCPUUtilizationPercentage: 70
    targetMemoryUtilizationPercentage: 80

  service:
    type: ClusterIP
    port: 3000

  ingress:
    enabled: true
    className: "nginx"
    annotations:
      cert-manager.io/cluster-issuer: "letsencrypt-prod"
    hosts:
      - host: fastgpt.example.com
        paths:
          - path: /
            pathType: Prefix
    tls:
      - secretName: fastgpt-tls
        hosts:
          - fastgpt.example.com

# 数据库依赖
mongodb:
  enabled: true
  auth:
    enabled: true
    rootPassword: "secure-password"
  replicaSet:
    enabled: true
    replicas:
      secondary: 2

postgresql:
  enabled: true
  auth:
    postgresPassword: "secure-password"
  primary:
    persistence:
      enabled: true
      size: 100Gi

redis:
  enabled: true
  auth:
    enabled: true
    password: "secure-password"
  master:
    persistence:
      enabled: true
      size: 10Gi
```

### 2. Kubernetes 资源定义

```yaml
# 部署配置
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastgpt
spec:
  replicas: 3
  selector:
    matchLabels:
      app: fastgpt
  template:
    metadata:
      labels:
        app: fastgpt
    spec:
      containers:
      - name: fastgpt
        image: ghcr.io/labring/fastgpt:latest
        ports:
        - containerPort: 3000
        env:
        - name: MONGODB_URI
          valueFrom:
            secretKeyRef:
              name: fastgpt-secrets
              key: mongodb-uri
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
        livenessProbe:
          httpGet:
            path: /api/health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /api/health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
```

## 📊 性能调优与扩展

### 1. 资源配置建议

```yaml
# 生产环境资源配置
resources:
  fastgpt:
    requests:
      memory: "2Gi"
      cpu: "1000m"
    limits:
      memory: "8Gi"  
      cpu: "4000m"
      
  mongodb:
    requests:
      memory: "4Gi"
      cpu: "2000m"
    limits:
      memory: "16Gi"
      cpu: "8000m"
      
  redis:
    requests:
      memory: "1Gi"
      cpu: "500m"
    limits:
      memory: "4Gi"
      cpu: "2000m"
```

### 2. 水平扩展策略

```yaml
# 自动扩展配置
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: fastgpt-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: fastgpt
  minReplicas: 3
  maxReplicas: 20
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
```

### 3. 数据库集群配置

```yaml
# MongoDB 副本集
mongodb:
  replicaSet:
    enabled: true
    name: "rs0"
    replicas:
      secondary: 2
    arbiter:
      enabled: true
  persistence:
    enabled: true
    storageClass: "fast-ssd"
    size: 500Gi

# Redis 集群  
redis:
  cluster:
    enabled: true
    nodes: 6
    replicas: 1
  persistence:
    enabled: true
    storageClass: "fast-ssd"
    size: 100Gi
```

## 🔧 故障排除与维护

### 1. 常见问题诊断

```bash
# 容器状态检查
docker-compose ps
docker-compose logs fastgpt
docker-compose logs mongo
docker-compose logs redis

# 健康检查状态
docker inspect --format='{{.State.Health.Status}}' fastgpt
docker inspect --format='{{.State.Health.Status}}' mongo

# 网络连通性测试
docker-compose exec fastgpt ping mongo
docker-compose exec fastgpt ping redis
docker-compose exec fastgpt ping pg
```

### 2. 备份与恢复

```bash
# MongoDB 备份
docker-compose exec mongo mongodump \
  --host rs0/mongo:27017 \
  --authenticationDatabase admin \
  --username myusername \
  --password mypassword \
  --out /backup

# PostgreSQL 备份
docker-compose exec pg pg_dump \
  -U username \
  -d postgres \
  > fastgpt_pg_backup.sql

# Redis 备份
docker-compose exec redis redis-cli \
  -a mypassword \
  --rdb /data/backup.rdb

# MinIO 备份
docker-compose exec minio mc mirror \
  /data /backup/minio
```

### 3. 升级策略

```yaml
# 滚动升级
services:
  fastgpt:
    deploy:
      update_config:
        parallelism: 1
        delay: 30s
        failure_action: rollback
        order: start-first
      rollback_config:
        parallelism: 1
        delay: 10s
        failure_action: pause
        order: stop-first
```

---

FastGPT 的 Docker 部署架构体现了现代容器化应用的最佳实践，通过精心设计的多服务架构、安全配置和自动化部署，为用户提供了可靠、可扩展的 AI 应用平台部署方案。无论是开发测试还是生产环境，都能快速部署并稳定运行。