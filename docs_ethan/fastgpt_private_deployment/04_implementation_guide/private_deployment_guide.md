# FastGPT 私有化部署实施指南

## 🎯 实施目标

本指南提供FastGPT完全私有化部署的详细实施步骤，实现**零外网依赖**的企业级AI知识库平台，满足企业IT部门"禁止随意外网访问"的安全合规要求。

## 📋 部署前准备清单

### 1. 硬件资源需求

```yaml
# 最小配置要求 (测试环境)
minimum_requirements:
  compute_nodes:
    - cpu: "16 cores"
      memory: "64GB RAM"
      storage: "500GB NVMe SSD"
      gpu: "RTX 4090 24GB (1张)"
      quantity: 2
      
  network_equipment:
    - layer3_switch: "千兆交换机"
      ports: 48
      vlan_support: true
    - firewall: "企业级防火墙"
      throughput: "1Gbps+"
      
  storage:
    - shared_storage: "NAS/SAN"
      capacity: "10TB"
      raid_level: "RAID 5"
      backup: "3-2-1备份策略"

# 生产环境推荐配置
production_requirements:
  compute_cluster:
    master_nodes:
      - cpu: "32 cores"
        memory: "128GB RAM" 
        storage: "1TB NVMe SSD"
        quantity: 3
        role: "Kubernetes Master"
        
    worker_nodes:
      - cpu: "64 cores"
        memory: "256GB RAM"
        storage: "2TB NVMe SSD"
        gpu: "A100 80GB (2张)"
        quantity: 4
        role: "AI计算节点"
        
    data_nodes:
      - cpu: "16 cores"
        memory: "64GB RAM"
        storage: "4TB SATA SSD"
        quantity: 3
        role: "数据存储节点"
        
  network_infrastructure:
    core_switches:
      - bandwidth: "10Gbps"
        redundancy: "双链路冗余"
        quantity: 2
    firewall_cluster:
      - throughput: "10Gbps+"
        ha_mode: "主备模式"
        quantity: 2
        
  storage_system:
    distributed_storage:
      - type: "分布式对象存储"
        capacity: "100TB"
        replication: "3副本"
        backup: "增量备份"
```

### 2. 软件环境准备

```bash
#!/bin/bash
# 软件环境安装脚本

# 操作系统要求: Ubuntu 22.04 LTS 或 CentOS 8+

# 1. 更新系统
sudo apt update && sudo apt upgrade -y

# 2. 安装基础软件
sudo apt install -y curl wget vim git unzip jq

# 3. 安装Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER

# 4. 安装Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# 5. 安装Kubernetes (可选)
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

# 6. 安装NVIDIA驱动和容器工具包 (如果使用GPU)
sudo apt install -y nvidia-driver-535
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list
sudo apt-get update && sudo apt-get install -y nvidia-container-toolkit
sudo systemctl restart docker
```

### 3. 网络规划配置

```yaml
# 网络规划配置文件
network_configuration:
  # IP地址分配
  ip_allocation:
    management_network: "192.168.10.0/24"
    dmz_network: "10.1.0.0/24"
    application_network: "10.2.0.0/24"
    data_network: "10.3.0.0/24"
    ai_services_network: "10.4.0.0/24"
    monitoring_network: "10.5.0.0/24"
    
  # DNS配置
  dns_configuration:
    primary_dns: "10.1.0.253"
    secondary_dns: "10.1.0.254"
    search_domains: ["internal.company.com", "fastgpt.internal"]
    
  # 时间同步
  ntp_configuration:
    primary_ntp: "ntp1.internal.company.com"
    secondary_ntp: "ntp2.internal.company.com"
    timezone: "Asia/Shanghai"
```

## 🚀 分阶段实施步骤

### 阶段1: 基础设施部署 (第1-2周)

#### 1.1 网络基础设施配置

```bash
#!/bin/bash
# 网络基础配置脚本

# 1. 配置防火墙规则
sudo ufw --force enable

# 2. 创建网络桥接
sudo docker network create --driver bridge \
  --subnet=10.1.0.0/24 \
  --ip-range=10.1.0.0/24 \
  --gateway=10.1.0.1 \
  fastgpt-dmz

sudo docker network create --driver bridge \
  --subnet=10.2.0.0/24 \
  --ip-range=10.2.0.0/24 \
  --gateway=10.2.0.1 \
  fastgpt-app

sudo docker network create --driver bridge \
  --subnet=10.3.0.0/24 \
  --ip-range=10.3.0.0/24 \
  --gateway=10.3.0.1 \
  fastgpt-data

sudo docker network create --driver bridge \
  --subnet=10.4.0.0/24 \
  --ip-range=10.4.0.0/24 \
  --gateway=10.4.0.1 \
  fastgpt-ai

sudo docker network create --driver bridge \
  --subnet=10.5.0.0/24 \
  --ip-range=10.5.0.0/24 \
  --gateway=10.5.0.1 \
  fastgpt-mgmt

# 3. 配置iptables规则
cat > /tmp/firewall-rules.sh << 'EOF'
#!/bin/bash
# 清空现有规则
iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -X

# 默认策略：拒绝所有
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT DROP

# 允许本地回环
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT

# 允许已建立的连接
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# 允许内网通信
iptables -A INPUT -s 10.0.0.0/8 -j ACCEPT
iptables -A OUTPUT -d 10.0.0.0/8 -j ACCEPT

# 允许SSH管理访问
iptables -A INPUT -p tcp --dport 22 -s 192.168.10.0/24 -j ACCEPT

# 允许HTTP/HTTPS访问DMZ
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# 禁止所有外网出站连接
iptables -A OUTPUT -d 0.0.0.0/0 -j DROP

# 保存规则
iptables-save > /etc/iptables/rules.v4
EOF

chmod +x /tmp/firewall-rules.sh
sudo /tmp/firewall-rules.sh
```

#### 1.2 内网服务基础设施

```yaml
# docker-compose-infrastructure.yml
version: '3.8'
services:
  # 内网DNS服务器
  dns-server:
    image: pihole/pihole:latest
    container_name: internal-dns
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "8053:80/tcp"
    environment:
      TZ: 'Asia/Shanghai'
      WEBPASSWORD: 'admin123'
    volumes:
      - ./dns-data/etc-pihole/:/etc/pihole/
      - ./dns-data/etc-dnsmasq.d/:/etc/dnsmasq.d/
    networks:
      fastgpt-mgmt:
        ipv4_address: 10.5.0.253
        
  # 内网NTP服务器
  ntp-server:
    image: cturra/ntp:latest
    container_name: internal-ntp
    ports:
      - "123:123/udp"
    environment:
      ENABLE_NTP_CONF: "true"
      NTP_SERVERS: "pool.ntp.org"
    networks:
      fastgpt-mgmt:
        ipv4_address: 10.5.0.252
        
  # 内网容器镜像仓库
  docker-registry:
    image: registry:2
    container_name: internal-registry
    ports:
      - "5000:5000"
    environment:
      REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY: /data
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_REALM: Registry Realm
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
    volumes:
      - ./registry-data:/data
      - ./registry-auth:/auth
    networks:
      fastgpt-mgmt:
        ipv4_address: 10.5.0.40
        
  # Harbor镜像仓库 (企业版推荐)
  harbor:
    image: goharbor/harbor-core:latest
    container_name: harbor-core
    depends_on:
      - harbor-db
      - harbor-redis
    ports:
      - "8080:8080"
    environment:
      CORE_SECRET: harbor-secret
      JOBSERVICE_SECRET: jobservice-secret
    volumes:
      - ./harbor-data:/data
    networks:
      fastgpt-mgmt:
        ipv4_address: 10.5.0.41
        
  harbor-db:
    image: postgres:13
    container_name: harbor-postgres
    environment:
      POSTGRES_DB: registry
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: harbor123
    volumes:
      - ./harbor-db:/var/lib/postgresql/data
    networks:
      fastgpt-mgmt:
        ipv4_address: 10.5.0.42
        
  harbor-redis:
    image: redis:7-alpine
    container_name: harbor-redis
    networks:
      fastgpt-mgmt:
        ipv4_address: 10.5.0.43

networks:
  fastgpt-mgmt:
    external: true
```

### 阶段2: 数据服务部署 (第3周)

#### 2.1 数据库集群部署

```yaml
# docker-compose-databases.yml
version: '3.8'
services:
  # MongoDB主从集群
  mongodb-primary:
    image: mongo:6.0
    container_name: mongodb-primary
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: mongodb123
      MONGO_REPLICA_SET_NAME: fastgpt-rs
    command: mongod --replSet fastgpt-rs --bind_ip_all
    volumes:
      - ./mongodb-primary:/data/db
      - ./mongodb-config:/docker-entrypoint-initdb.d
    networks:
      fastgpt-data:
        ipv4_address: 10.3.0.10
        
  mongodb-secondary1:
    image: mongo:6.0
    container_name: mongodb-secondary1
    ports:
      - "27018:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: mongodb123
      MONGO_REPLICA_SET_NAME: fastgpt-rs
    command: mongod --replSet fastgpt-rs --bind_ip_all
    volumes:
      - ./mongodb-secondary1:/data/db
    networks:
      fastgpt-data:
        ipv4_address: 10.3.0.11
        
  mongodb-secondary2:
    image: mongo:6.0
    container_name: mongodb-secondary2
    ports:
      - "27019:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: mongodb123
      MONGO_REPLICA_SET_NAME: fastgpt-rs
    command: mongod --replSet fastgpt-rs --bind_ip_all
    volumes:
      - ./mongodb-secondary2:/data/db
    networks:
      fastgpt-data:
        ipv4_address: 10.3.0.12
        
  # Redis集群
  redis-master:
    image: redis:7-alpine
    container_name: redis-master
    ports:
      - "6379:6379"
    command: redis-server --requirepass redis123 --masterauth redis123
    volumes:
      - ./redis-master:/data
    networks:
      fastgpt-data:
        ipv4_address: 10.3.0.20
        
  redis-slave:
    image: redis:7-alpine
    container_name: redis-slave
    ports:
      - "6380:6379"
    command: redis-server --slaveof redis-master 6379 --requirepass redis123 --masterauth redis123
    depends_on:
      - redis-master
    volumes:
      - ./redis-slave:/data
    networks:
      fastgpt-data:
        ipv4_address: 10.3.0.21
        
  # PostgreSQL + pgvector (向量数据库)
  postgres-vector:
    image: pgvector/pgvector:pg15
    container_name: postgres-vector
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: fastgpt_vector
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres123
    volumes:
      - ./postgres-data:/var/lib/postgresql/data
      - ./postgres-init:/docker-entrypoint-initdb.d
    networks:
      fastgpt-data:
        ipv4_address: 10.3.0.30
        
  # MinIO对象存储集群
  minio1:
    image: minio/minio:latest
    container_name: minio1
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: minio
      MINIO_ROOT_PASSWORD: minio123456
    command: server http://minio{1...4}/data{1...2} --console-address ":9001"
    volumes:
      - ./minio1-data1:/data1
      - ./minio1-data2:/data2
    networks:
      fastgpt-data:
        ipv4_address: 10.3.0.40
        
  minio2:
    image: minio/minio:latest
    container_name: minio2
    ports:
      - "9002:9000"
      - "9003:9001"
    environment:
      MINIO_ROOT_USER: minio
      MINIO_ROOT_PASSWORD: minio123456
    command: server http://minio{1...4}/data{1...2} --console-address ":9001"
    volumes:
      - ./minio2-data1:/data1
      - ./minio2-data2:/data2
    networks:
      fastgpt-data:
        ipv4_address: 10.3.0.41

networks:
  fastgpt-data:
    external: true
```

#### 2.2 数据库初始化脚本

```bash
#!/bin/bash
# 数据库初始化脚本

# 1. 初始化MongoDB副本集
cat > init-mongodb-replica.js << 'EOF'
rs.initiate({
  _id: "fastgpt-rs",
  members: [
    { _id: 0, host: "10.3.0.10:27017", priority: 2 },
    { _id: 1, host: "10.3.0.11:27017", priority: 1 },
    { _id: 2, host: "10.3.0.12:27017", priority: 1 }
  ]
});

// 等待副本集初始化完成
sleep(10000);

// 创建FastGPT数据库和用户
use fastgpt;
db.createUser({
  user: "fastgpt",
  pwd: "fastgpt123",
  roles: [
    { role: "dbOwner", db: "fastgpt" }
  ]
});
EOF

# 执行MongoDB初始化
docker exec mongodb-primary mongosh --eval "$(cat init-mongodb-replica.js)"

# 2. 初始化PostgreSQL向量扩展
cat > init-postgres-vector.sql << 'EOF'
-- 创建向量扩展
CREATE EXTENSION IF NOT EXISTS vector;

-- 创建FastGPT向量表
CREATE TABLE IF NOT EXISTS vector_store (
    id SERIAL PRIMARY KEY,
    collection_name VARCHAR(255) NOT NULL,
    document_id VARCHAR(255) NOT NULL,
    vector vector(1536),
    metadata JSONB,
    content TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 创建向量索引
CREATE INDEX IF NOT EXISTS vector_store_vector_idx ON vector_store USING ivfflat (vector vector_cosine_ops);
CREATE INDEX IF NOT EXISTS vector_store_collection_idx ON vector_store (collection_name);
CREATE INDEX IF NOT EXISTS vector_store_document_idx ON vector_store (document_id);

-- 创建FastGPT用户
CREATE USER fastgpt WITH PASSWORD 'fastgpt123';
GRANT ALL PRIVILEGES ON DATABASE fastgpt_vector TO fastgpt;
GRANT ALL PRIVILEGES ON TABLE vector_store TO fastgpt;
GRANT USAGE, SELECT ON SEQUENCE vector_store_id_seq TO fastgpt;
EOF

# 执行PostgreSQL初始化
docker exec postgres-vector psql -U postgres -d fastgpt_vector -f /docker-entrypoint-initdb.d/init-postgres-vector.sql

# 3. 创建MinIO存储桶
docker exec minio1 mc alias set local http://localhost:9000 minio minio123456
docker exec minio1 mc mb local/fastgpt-data
docker exec minio1 mc policy set public local/fastgpt-data
```

### 阶段3: AI服务部署 (第4-5周)

#### 3.1 本地AI模型服务集群

```yaml
# docker-compose-ai-services.yml
version: '3.8'
services:
  # Ollama LLM服务集群
  ollama-node1:
    image: ollama/ollama:latest
    container_name: ollama-node1
    ports:
      - "11434:11434"
    environment:
      OLLAMA_ORIGINS: "*"
      OLLAMA_HOST: "0.0.0.0"
    volumes:
      - ./ollama-node1:/root/.ollama
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    networks:
      fastgpt-ai:
        ipv4_address: 10.4.0.10
        
  ollama-node2:
    image: ollama/ollama:latest
    container_name: ollama-node2
    ports:
      - "11435:11434"
    environment:
      OLLAMA_ORIGINS: "*"
      OLLAMA_HOST: "0.0.0.0"
    volumes:
      - ./ollama-node2:/root/.ollama
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    networks:
      fastgpt-ai:
        ipv4_address: 10.4.0.11
        
  # Embedding服务
  text-embeddings:
    image: ghcr.io/huggingface/text-embeddings-inference:cpu-1.2
    container_name: text-embeddings
    ports:
      - "8080:80"
    command: --model-id BAAI/bge-small-zh-v1.5 --port 80
    volumes:
      - ./embedding-models:/data
    networks:
      fastgpt-ai:
        ipv4_address: 10.4.0.20
        
  # Rerank服务
  rerank-service:
    image: registry.cn-hangzhou.aliyuncs.com/fastgpt/rerank:latest
    container_name: rerank-service
    ports:
      - "8081:8000"
    environment:
      MODEL_NAME: BAAI/bge-reranker-base
      MODEL_PATH: /models
    volumes:
      - ./rerank-models:/models
    networks:
      fastgpt-ai:
        ipv4_address: 10.4.0.30
        
  # AI服务负载均衡器
  ai-loadbalancer:
    image: nginx:alpine
    container_name: ai-lb
    ports:
      - "8090:80"
    volumes:
      - ./nginx-ai-lb.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - ollama-node1
      - ollama-node2
      - text-embeddings
      - rerank-service
    networks:
      fastgpt-ai:
        ipv4_address: 10.4.0.100

networks:
  fastgpt-ai:
    external: true
```

#### 3.2 AI模型下载和部署脚本

```bash
#!/bin/bash
# AI模型部署脚本

# 1. 下载Ollama模型
echo "下载Ollama模型..."
docker exec ollama-node1 ollama pull qwen2:7b
docker exec ollama-node1 ollama pull qwen2:14b
docker exec ollama-node1 ollama pull llama3.1:8b
docker exec ollama-node1 ollama pull nomic-embed-text

docker exec ollama-node2 ollama pull qwen2:7b
docker exec ollama-node2 ollama pull llama3.1:8b

# 2. 下载Embedding模型
echo "下载Embedding模型..."
mkdir -p ./embedding-models
cd ./embedding-models
git clone https://huggingface.co/BAAI/bge-small-zh-v1.5
cd ..

# 3. 下载Rerank模型
echo "下载Rerank模型..."
mkdir -p ./rerank-models
cd ./rerank-models
git clone https://huggingface.co/BAAI/bge-reranker-base
cd ..

# 4. 配置AI服务负载均衡
cat > nginx-ai-lb.conf << 'EOF'
events {
    worker_connections 1024;
}

http {
    upstream ollama_backend {
        server 10.4.0.10:11434;
        server 10.4.0.11:11434;
    }
    
    upstream embedding_backend {
        server 10.4.0.20:80;
    }
    
    upstream rerank_backend {
        server 10.4.0.30:8000;
    }
    
    server {
        listen 80;
        
        location /v1/chat/completions {
            proxy_pass http://ollama_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_buffering off;
        }
        
        location /v1/embeddings {
            proxy_pass http://embedding_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
        
        location /rerank {
            proxy_pass http://rerank_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
        
        location /health {
            return 200 "AI Services OK";
            add_header Content-Type text/plain;
        }
    }
}
EOF

# 5. 验证AI服务
echo "验证AI服务状态..."
curl -X POST "http://10.4.0.100:80/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen2:7b",
    "messages": [{"role": "user", "content": "你好，请介绍一下你自己"}],
    "max_tokens": 100
  }'
```

### 阶段4: FastGPT应用部署 (第6周)

#### 4.1 FastGPT核心服务部署

```yaml
# docker-compose-fastgpt.yml
version: '3.8'
services:
  # FastGPT后端服务
  fastgpt-backend:
    image: registry.cn-hangzhou.aliyuncs.com/fastgpt/fastgpt:v4.11.0
    container_name: fastgpt-backend
    ports:
      - "3000:3000"
    environment:
      # 数据库配置
      MONGODB_URI: mongodb://fastgpt:fastgpt123@10.3.0.10:27017,10.3.0.11:27017,10.3.0.12:27017/fastgpt?replicaSet=fastgpt-rs
      REDIS_URI: redis://:redis123@10.3.0.20:6379
      PG_URL: postgresql://fastgpt:fastgpt123@10.3.0.30:5432/fastgpt_vector
      
      # 对象存储配置
      MINIO_ENDPOINT: http://10.3.0.40:9000
      MINIO_ACCESS_KEY: minio
      MINIO_SECRET_KEY: minio123456
      MINIO_BUCKET: fastgpt-data
      
      # AI服务配置 - 使用内网AI负载均衡器
      OPENAI_BASE_URL: http://10.4.0.100:80/v1
      CHAT_API_KEY: sk-internal-placeholder
      
      # 禁用外部服务
      DISABLE_EXTERNAL_SERVICES: "true"
      DISABLE_WECHAT_OFFICIAL: "true"
      DISABLE_PAYMENT_SERVICES: "true"
      
      # 内网服务配置
      INTERNAL_MODE: "true"
      SANDBOX_URL: http://10.2.0.40:3030
      
    volumes:
      - ./fastgpt-logs:/app/logs
    networks:
      fastgpt-app:
        ipv4_address: 10.2.0.10
    depends_on:
      - fastgpt-sandbox
      
  # FastGPT Web前端
  fastgpt-web:
    image: registry.cn-hangzhou.aliyuncs.com/fastgpt/fastgpt-web:v4.11.0
    container_name: fastgpt-web
    ports:
      - "3001:3000"
    environment:
      NEXT_PUBLIC_API_BASE_URL: http://10.2.0.10:3000
      NEXT_PUBLIC_INTERNAL_MODE: "true"
    networks:
      fastgpt-app:
        ipv4_address: 10.2.0.20
    depends_on:
      - fastgpt-backend
      
  # 代码执行沙箱
  fastgpt-sandbox:
    image: registry.cn-hangzhou.aliyuncs.com/fastgpt/fastgpt-sandbox:v4.10.1
    container_name: fastgpt-sandbox
    ports:
      - "3030:3000"
    environment:
      SANDBOX_MODE: "secure"
      MAX_EXECUTION_TIME: "30"
    networks:
      fastgpt-app:
        ipv4_address: 10.2.0.40
        
  # API网关
  api-gateway:
    image: kong:latest
    container_name: api-gateway
    ports:
      - "8000:8000"
      - "8443:8443"
      - "8001:8001"
      - "8444:8444"
    environment:
      KONG_DATABASE: "off"
      KONG_DECLARATIVE_CONFIG: /kong/declarative/kong.yml
      KONG_PROXY_ACCESS_LOG: /dev/stdout
      KONG_ADMIN_ACCESS_LOG: /dev/stdout
      KONG_PROXY_ERROR_LOG: /dev/stderr
      KONG_ADMIN_ERROR_LOG: /dev/stderr
      KONG_ADMIN_LISTEN: "0.0.0.0:8001, 0.0.0.0:8444 ssl"
    volumes:
      - ./kong-config:/kong/declarative
    networks:
      fastgpt-app:
        ipv4_address: 10.2.0.30
    depends_on:
      - fastgpt-backend
      - fastgpt-web

networks:
  fastgpt-app:
    external: true
  fastgpt-data:
    external: true
  fastgpt-ai:
    external: true
```

#### 4.2 反向代理和负载均衡

```yaml
# docker-compose-proxy.yml
version: '3.8'
services:
  # 主负载均衡器
  main-loadbalancer:
    image: nginx:alpine
    container_name: main-lb
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx-main.conf:/etc/nginx/nginx.conf:ro
      - ./ssl-certs:/etc/nginx/ssl:ro
    networks:
      fastgpt-dmz:
        ipv4_address: 10.1.0.10
      fastgpt-app:
        ipv4_address: 10.2.0.200
    depends_on:
      - waf
      
  # Web应用防火墙
  waf:
    image: owasp/modsecurity:nginx-alpine
    container_name: web-waf
    ports:
      - "8080:80"
    volumes:
      - ./modsecurity-config:/etc/modsecurity
      - ./nginx-waf.conf:/etc/nginx/nginx.conf:ro
    networks:
      fastgpt-dmz:
        ipv4_address: 10.1.0.20
      
  # 反向代理
  reverse-proxy:
    image: nginx:alpine
    container_name: reverse-proxy
    ports:
      - "8090:80"
    volumes:
      - ./nginx-reverse-proxy.conf:/etc/nginx/nginx.conf:ro
    networks:
      fastgpt-dmz:
        ipv4_address: 10.1.0.30
      fastgpt-app:
        ipv4_address: 10.2.0.201

networks:
  fastgpt-dmz:
    external: true
  fastgpt-app:
    external: true
```

#### 4.3 主负载均衡器配置

```nginx
# nginx-main.conf
events {
    worker_connections 2048;
}

http {
    # 日志格式
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                   '$status $body_bytes_sent "$http_referer" '
                   '"$http_user_agent" "$http_x_forwarded_for"';
    
    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log warn;
    
    # 上游服务器定义
    upstream fastgpt_backend {
        server 10.2.0.10:3000;
        keepalive 32;
    }
    
    upstream fastgpt_web {
        server 10.2.0.20:3000;
        keepalive 32;
    }
    
    # 限流配置
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=web:10m rate=30r/s;
    
    # HTTP服务器 - 重定向到HTTPS
    server {
        listen 80;
        server_name fastgpt.internal.com;
        return 301 https://$server_name$request_uri;
    }
    
    # HTTPS服务器
    server {
        listen 443 ssl http2;
        server_name fastgpt.internal.com;
        
        # SSL配置
        ssl_certificate /etc/nginx/ssl/fastgpt.internal.com.crt;
        ssl_certificate_key /etc/nginx/ssl/fastgpt.internal.com.key;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512;
        ssl_prefer_server_ciphers off;
        ssl_session_cache shared:SSL:10m;
        ssl_session_timeout 10m;
        
        # 安全头
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
        add_header X-Frame-Options DENY always;
        add_header X-Content-Type-Options nosniff always;
        add_header X-XSS-Protection "1; mode=block" always;
        add_header Referrer-Policy "strict-origin-when-cross-origin" always;
        
        # API代理
        location /api/ {
            limit_req zone=api burst=20 nodelay;
            proxy_pass http://fastgpt_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            # 流式响应支持
            proxy_buffering off;
            proxy_cache_bypass $http_upgrade;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }
        
        # Web前端代理
        location / {
            limit_req zone=web burst=50 nodelay;
            proxy_pass http://fastgpt_web;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
        
        # 健康检查
        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
        
        # 静态文件缓存
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
            access_log off;
        }
    }
}
```

### 阶段5: 监控和安全加固 (第7周)

#### 5.1 监控系统部署

```yaml
# docker-compose-monitoring.yml
version: '3.8'
services:
  # Prometheus监控
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-lifecycle'
    volumes:
      - ./prometheus-config:/etc/prometheus
      - ./prometheus-data:/prometheus
    networks:
      fastgpt-mgmt:
        ipv4_address: 10.5.0.10
        
  # Grafana监控面板
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin123
      GF_USERS_ALLOW_SIGN_UP: "false"
      GF_INSTALL_PLUGINS: "grafana-piechart-panel,grafana-worldmap-panel"
    volumes:
      - ./grafana-data:/var/lib/grafana
      - ./grafana-dashboards:/etc/grafana/provisioning/dashboards
      - ./grafana-datasources:/etc/grafana/provisioning/datasources
    networks:
      fastgpt-mgmt:
        ipv4_address: 10.5.0.20
    depends_on:
      - prometheus
      
  # AlertManager告警管理
  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager-config:/etc/alertmanager
    networks:
      fastgpt-mgmt:
        ipv4_address: 10.5.0.30
        
  # 日志收集系统
  loki:
    image: grafana/loki:latest
    container_name: loki
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml
    volumes:
      - ./loki-config:/etc/loki
      - ./loki-data:/tmp/loki
    networks:
      fastgpt-mgmt:
        ipv4_address: 10.5.0.40
        
  # 日志采集代理
  promtail:
    image: grafana/promtail:latest
    container_name: promtail
    volumes:
      - ./promtail-config:/etc/promtail
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
    command: -config.file=/etc/promtail/config.yml
    networks:
      fastgpt-mgmt:
        ipv4_address: 10.5.0.50
    depends_on:
      - loki

networks:
  fastgpt-mgmt:
    external: true
```

#### 5.2 安全加固配置

```bash
#!/bin/bash
# 安全加固脚本

# 1. 系统安全加固
echo "开始系统安全加固..."

# 禁用不必要的服务
systemctl disable bluetooth
systemctl disable cups
systemctl disable avahi-daemon

# 配置系统审计
cat > /etc/audit/rules.d/fastgpt.rules << 'EOF'
# 监控文件系统变化
-w /etc/passwd -p wa -k passwd_changes
-w /etc/shadow -p wa -k shadow_changes
-w /etc/group -p wa -k group_changes
-w /etc/sudoers -p wa -k sudoers_changes

# 监控网络变化
-w /etc/hosts -p wa -k network_changes
-w /etc/network/ -p wa -k network_changes

# 监控系统调用
-a always,exit -F arch=b64 -S adjtimex -S settimeofday -k time_change
-a always,exit -F arch=b64 -S clock_settime -k time_change

# 监控特权命令
-w /usr/bin/sudo -p x -k privilege_escalation
-w /usr/bin/su -p x -k privilege_escalation

# 监控Docker相关
-w /usr/bin/docker -p x -k docker_commands
-w /var/lib/docker -p wa -k docker_changes
EOF

systemctl restart auditd

# 2. Docker安全配置
echo "配置Docker安全设置..."

# 创建Docker daemon配置
mkdir -p /etc/docker
cat > /etc/docker/daemon.json << 'EOF'
{
    "log-level": "warn",
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "10m",
        "max-file": "3"
    },
    "live-restore": true,
    "userland-proxy": false,
    "no-new-privileges": true,
    "seccomp-profile": "/etc/docker/seccomp-profile.json",
    "icc": false,
    "disable-legacy-registry": true
}
EOF

# 3. 防火墙规则优化
echo "优化防火墙规则..."

# 创建详细防火墙规则
cat > /etc/iptables/fastgpt-rules.sh << 'EOF'
#!/bin/bash

# 清空现有规则
iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -X

# 默认策略
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT DROP

# 允许本地回环
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT

# 允许已建立的连接
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A OUTPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# 允许SSH (限制来源)
iptables -A INPUT -p tcp --dport 22 -s 192.168.10.0/24 -m conntrack --ctstate NEW -j ACCEPT

# 允许Web访问
iptables -A INPUT -p tcp --dport 80 -m conntrack --ctstate NEW -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -m conntrack --ctstate NEW -j ACCEPT

# 允许内网通信
iptables -A INPUT -s 10.0.0.0/8 -j ACCEPT
iptables -A OUTPUT -d 10.0.0.0/8 -j ACCEPT

# 允许DNS查询 (限制到内网DNS)
iptables -A OUTPUT -p udp --dport 53 -d 10.5.0.253 -j ACCEPT

# 允许NTP同步 (限制到内网NTP)
iptables -A OUTPUT -p udp --dport 123 -d 10.5.0.252 -j ACCEPT

# DDoS防护
iptables -A INPUT -p tcp --dport 80 -m limit --limit 25/minute --limit-burst 100 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -m limit --limit 25/minute --limit-burst 100 -j ACCEPT

# 记录被拒绝的连接
iptables -A INPUT -m limit --limit 5/min -j LOG --log-prefix "iptables INPUT denied: " --log-level 7
iptables -A FORWARD -m limit --limit 5/min -j LOG --log-prefix "iptables FORWARD denied: " --log-level 7
iptables -A OUTPUT -m limit --limit 5/min -j LOG --log-prefix "iptables OUTPUT denied: " --log-level 7

# 保存规则
iptables-save > /etc/iptables/rules.v4
EOF

chmod +x /etc/iptables/fastgpt-rules.sh
/etc/iptables/fastgpt-rules.sh

# 4. 入侵检测系统
echo "配置入侵检测系统..."

# 安装Fail2ban
apt install -y fail2ban

# 配置Fail2ban
cat > /etc/fail2ban/jail.local << 'EOF'
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 5
backend = systemd

[sshd]
enabled = true
port = ssh
logpath = %(sshd_log)s
maxretry = 3

[nginx-http-auth]
enabled = true
filter = nginx-http-auth
logpath = /var/log/nginx/error.log
maxretry = 3

[nginx-noscript]
enabled = true
filter = nginx-noscript
logpath = /var/log/nginx/access.log
maxretry = 6

[nginx-badbots]
enabled = true
filter = nginx-badbots
logpath = /var/log/nginx/access.log
maxretry = 2

[nginx-noproxy]
enabled = true
filter = nginx-noproxy
logpath = /var/log/nginx/access.log
maxretry = 2
EOF

systemctl enable fail2ban
systemctl start fail2ban

echo "安全加固完成!"
```

## 🧪 测试和验证

### 1. 功能测试脚本

```bash
#!/bin/bash
# FastGPT功能测试脚本

echo "开始FastGPT系统功能测试..."

# 1. 基础连通性测试
echo "1. 测试基础网络连通性..."
ping -c 3 10.1.0.10 || echo "负载均衡器连接失败"
ping -c 3 10.2.0.10 || echo "FastGPT后端连接失败"
ping -c 3 10.3.0.10 || echo "MongoDB连接失败"
ping -c 3 10.4.0.10 || echo "AI服务连接失败"

# 2. 服务健康检查
echo "2. 检查服务健康状态..."
curl -f http://10.1.0.10/health || echo "负载均衡器健康检查失败"
curl -f http://10.2.0.10:3000/api/health || echo "FastGPT后端健康检查失败"
curl -f http://10.4.0.100:80/health || echo "AI服务健康检查失败"

# 3. 数据库连接测试
echo "3. 测试数据库连接..."
docker exec mongodb-primary mongosh --eval "db.runCommand('ping')" || echo "MongoDB连接测试失败"
docker exec redis-master redis-cli -a redis123 ping || echo "Redis连接测试失败"
docker exec postgres-vector psql -U postgres -c "SELECT 1;" || echo "PostgreSQL连接测试失败"

# 4. AI服务功能测试
echo "4. 测试AI服务功能..."
curl -X POST "http://10.4.0.100:80/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen2:7b",
    "messages": [{"role": "user", "content": "你好"}],
    "max_tokens": 50
  }' || echo "LLM服务测试失败"

# 5. 文件存储测试
echo "5. 测试文件存储功能..."
echo "test content" > /tmp/test.txt
docker exec minio1 mc cp /tmp/test.txt local/fastgpt-data/test.txt || echo "MinIO上传测试失败"
docker exec minio1 mc cat local/fastgpt-data/test.txt || echo "MinIO下载测试失败"

# 6. Web界面访问测试
echo "6. 测试Web界面访问..."
curl -f http://10.1.0.10/ || echo "Web界面访问失败"

# 7. API接口测试
echo "7. 测试API接口..."
curl -X GET "http://10.1.0.10/api/v1/system/info" \
  -H "Content-Type: application/json" || echo "API接口测试失败"

echo "功能测试完成!"
```

### 2. 性能测试

```bash
#!/bin/bash
# 性能压力测试脚本

echo "开始性能压力测试..."

# 1. 安装测试工具
apt install -y apache2-utils wrk

# 2. Web服务器性能测试
echo "测试Web服务器性能..."
ab -n 1000 -c 10 http://10.1.0.10/ > /tmp/web-performance.log
echo "Web服务器测试结果："
grep -E "Requests per second|Time per request" /tmp/web-performance.log

# 3. API接口性能测试
echo "测试API接口性能..."
wrk -t12 -c400 -d30s --script=api-test.lua http://10.1.0.10/api/v1/system/info

# 4. AI服务性能测试
echo "测试AI服务性能..."
for i in {1..10}; do
  time curl -X POST "http://10.4.0.100:80/v1/chat/completions" \
    -H "Content-Type: application/json" \
    -d '{
      "model": "qwen2:7b",
      "messages": [{"role": "user", "content": "请写一首关于春天的诗"}],
      "max_tokens": 200
    }' > /tmp/ai-test-$i.log &
done
wait

# 5. 数据库性能测试
echo "测试数据库性能..."
docker exec mongodb-primary mongoperf --jsonConfig '{"nThreads":16,"fileSizeMB":1000,"r":true}' > /tmp/mongodb-perf.log

# 6. 生成性能报告
echo "生成性能测试报告..."
cat > /tmp/performance-report.md << 'EOF'
# FastGPT性能测试报告

## 测试环境
- CPU: 16核心
- 内存: 64GB
- 存储: NVMe SSD
- 网络: 千兆网络

## 测试结果

### Web服务器性能
EOF

grep -A 10 "Requests per second" /tmp/web-performance.log >> /tmp/performance-report.md

echo "### AI服务响应时间" >> /tmp/performance-report.md
for i in {1..10}; do
  echo "- 请求 $i: $(grep -o 'real.*' /tmp/ai-test-$i.log | head -1)" >> /tmp/performance-report.md
done

echo "性能测试完成! 报告保存在 /tmp/performance-report.md"
```

### 3. 安全性测试

```bash
#!/bin/bash
# 安全性测试脚本

echo "开始安全性测试..."

# 1. 端口扫描测试
echo "1. 执行端口扫描测试..."
nmap -sS -O 10.1.0.10 > /tmp/port-scan.log
echo "开放端口列表："
grep -E "open|closed|filtered" /tmp/port-scan.log

# 2. SSL/TLS配置测试
echo "2. 测试SSL/TLS配置..."
testssl.sh --quiet https://fastgpt.internal.com > /tmp/ssl-test.log
echo "SSL测试结果："
grep -E "Grade|Protocol|Cipher" /tmp/ssl-test.log

# 3. Web应用安全测试
echo "3. 执行Web应用安全扫描..."
nikto -h http://10.1.0.10 -output /tmp/nikto-scan.log

# 4. SQL注入测试
echo "4. 测试SQL注入防护..."
sqlmap -u "http://10.1.0.10/api/v1/search?q=test" --batch --level=1 --risk=1 > /tmp/sqlmap.log

# 5. XSS攻击测试
echo "5. 测试XSS防护..."
curl -X POST "http://10.1.0.10/api/v1/chat" \
  -H "Content-Type: application/json" \
  -d '{"message": "<script>alert(\"XSS\")</script>"}' > /tmp/xss-test.log

# 6. 访问控制测试
echo "6. 测试访问控制..."
curl -I http://10.3.0.10:27017 > /tmp/access-control.log
curl -I http://10.4.0.10:11434 > /tmp/access-control.log

# 7. 生成安全测试报告
echo "生成安全测试报告..."
cat > /tmp/security-report.md << 'EOF'
# FastGPT安全测试报告

## 测试项目
1. 端口扫描测试
2. SSL/TLS配置测试
3. Web应用安全扫描
4. SQL注入测试
5. XSS攻击测试
6. 访问控制测试

## 测试结果

### 端口扫描结果
EOF

grep -E "open|closed" /tmp/port-scan.log >> /tmp/security-report.md

echo "### 安全漏洞统计" >> /tmp/security-report.md
echo "- 高危漏洞: $(grep -c "High" /tmp/nikto-scan.log)" >> /tmp/security-report.md
echo "- 中危漏洞: $(grep -c "Medium" /tmp/nikto-scan.log)" >> /tmp/security-report.md
echo "- 低危漏洞: $(grep -c "Low" /tmp/nikto-scan.log)" >> /tmp/security-report.md

echo "安全性测试完成! 报告保存在 /tmp/security-report.md"
```

## 🔧 故障排除指南

### 常见问题和解决方案

```yaml
troubleshooting_guide:
  # 网络连接问题
  network_issues:
    - problem: "服务间无法通信"
      symptoms: ["连接超时", "DNS解析失败", "路由不可达"]
      solutions:
        - "检查防火墙规则配置"
        - "验证Docker网络配置"
        - "确认IP地址分配正确"
        - "测试网络连通性"
      commands:
        - "docker network ls"
        - "iptables -L -n"
        - "ping <target_ip>"
        - "telnet <target_ip> <port>"
        
  # AI服务问题
  ai_service_issues:
    - problem: "AI模型响应慢或失败"
      symptoms: ["请求超时", "模型加载失败", "GPU内存不足"]
      solutions:
        - "检查GPU资源使用情况"
        - "重启Ollama服务"
        - "检查模型文件完整性"
        - "调整模型并发参数"
      commands:
        - "nvidia-smi"
        - "docker logs ollama-node1"
        - "docker exec ollama-node1 ollama list"
        - "docker restart ollama-node1"
        
  # 数据库问题  
  database_issues:
    - problem: "数据库连接失败"
      symptoms: ["连接被拒绝", "认证失败", "副本集状态异常"]
      solutions:
        - "检查数据库服务状态"
        - "验证认证信息"
        - "检查副本集配置"
        - "查看数据库日志"
      commands:
        - "docker logs mongodb-primary"
        - "docker exec mongodb-primary mongosh --eval 'rs.status()'"
        - "docker exec redis-master redis-cli ping"
        
  # 存储问题
  storage_issues:
    - problem: "文件上传下载失败"
      symptoms: ["MinIO连接失败", "存储空间不足", "权限被拒绝"]
      solutions:
        - "检查MinIO服务状态"
        - "验证存储空间"
        - "检查访问密钥配置"
        - "查看存储权限设置"
      commands:
        - "docker logs minio1"
        - "docker exec minio1 mc admin info local"
        - "df -h"
        
  # 应用问题
  application_issues:
    - problem: "FastGPT服务异常"
      symptoms: ["应用启动失败", "API错误", "页面无法访问"]
      solutions:
        - "检查应用日志"
        - "验证环境变量配置"
        - "重启相关服务"
        - "检查依赖服务状态"
      commands:
        - "docker logs fastgpt-backend"
        - "docker exec fastgpt-backend env | grep -E 'MONGODB|REDIS|OPENAI'"
        - "curl http://10.2.0.10:3000/api/health"
```

## 📋 部署后运维

### 1. 日常监控检查清单

```bash
#!/bin/bash
# 日常监控检查脚本

echo "=== FastGPT日常监控检查 $(date) ==="

# 1. 系统资源检查
echo "1. 系统资源使用情况："
echo "CPU使用率: $(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1)%"
echo "内存使用率: $(free | grep Mem | awk '{printf("%.2f%%", $3/$2 * 100.0)}')"
echo "磁盘使用率:"
df -h | grep -E "/$|/data"

# 2. 服务状态检查
echo -e "\n2. 核心服务状态："
services=("mongodb-primary" "redis-master" "postgres-vector" "minio1" "ollama-node1" "fastgpt-backend" "fastgpt-web")
for service in "${services[@]}"; do
    status=$(docker ps --filter "name=$service" --format "table {{.Names}}\t{{.Status}}" | tail -n +2)
    if [[ -n "$status" ]]; then
        echo "✓ $service: Running"
    else
        echo "✗ $service: Stopped"
    fi
done

# 3. 网络连通性检查
echo -e "\n3. 网络连通性检查："
endpoints=("10.1.0.10:80" "10.2.0.10:3000" "10.3.0.10:27017" "10.4.0.10:11434")
for endpoint in "${endpoints[@]}"; do
    if timeout 5 bash -c "</dev/tcp/${endpoint/:/ }" 2>/dev/null; then
        echo "✓ $endpoint: 可达"
    else
        echo "✗ $endpoint: 不可达"
    fi
done

# 4. 应用功能检查
echo -e "\n4. 应用功能检查："
web_status=$(curl -s -o /dev/null -w "%{http_code}" http://10.1.0.10/)
if [[ "$web_status" == "200" ]]; then
    echo "✓ Web界面: 正常"
else
    echo "✗ Web界面: 异常 (HTTP $web_status)"
fi

api_status=$(curl -s -o /dev/null -w "%{http_code}" http://10.1.0.10/api/health)
if [[ "$api_status" == "200" ]]; then
    echo "✓ API接口: 正常"
else
    echo "✗ API接口: 异常 (HTTP $api_status)"
fi

# 5. 安全日志检查
echo -e "\n5. 安全事件检查："
failed_logins=$(grep "Failed password" /var/log/auth.log | wc -l)
echo "最近24小时失败登录次数: $failed_logins"

blocked_ips=$(fail2ban-client status sshd | grep "Currently banned" | awk '{print $4}')
echo "当前被封禁IP数量: $blocked_ips"

# 6. 备份状态检查
echo -e "\n6. 备份状态检查："
if [[ -f "/backup/$(date +%Y%m%d)/mongodb_backup.tar.gz" ]]; then
    echo "✓ 今日MongoDB备份: 已完成"
else
    echo "✗ 今日MongoDB备份: 未完成"
fi

# 7. 磁盘空间预警
echo -e "\n7. 磁盘空间预警："
disk_usage=$(df / | tail -1 | awk '{print $5}' | sed 's/%//')
if [[ $disk_usage -gt 80 ]]; then
    echo "⚠ 根分区使用率超过80%: ${disk_usage}%"
else
    echo "✓ 磁盘空间充足: ${disk_usage}%"
fi

echo -e "\n=== 监控检查完成 ==="
```

### 2. 备份和恢复策略

```bash
#!/bin/bash
# 自动化备份脚本

BACKUP_DIR="/backup/$(date +%Y%m%d)"
mkdir -p $BACKUP_DIR

echo "开始自动备份 $(date)"

# 1. MongoDB备份
echo "备份MongoDB数据..."
docker exec mongodb-primary mongodump --uri="mongodb://admin:mongodb123@localhost:27017/fastgpt?authSource=admin" --out /tmp/mongodb_backup
docker cp mongodb-primary:/tmp/mongodb_backup $BACKUP_DIR/
tar -czf $BACKUP_DIR/mongodb_backup.tar.gz -C $BACKUP_DIR mongodb_backup
rm -rf $BACKUP_DIR/mongodb_backup

# 2. PostgreSQL备份
echo "备份PostgreSQL数据..."
docker exec postgres-vector pg_dump -U postgres -d fastgpt_vector > $BACKUP_DIR/postgres_backup.sql

# 3. Redis备份
echo "备份Redis数据..."
docker exec redis-master redis-cli -a redis123 --rdb /tmp/redis_backup.rdb
docker cp redis-master:/tmp/redis_backup.rdb $BACKUP_DIR/

# 4. MinIO备份
echo "备份MinIO数据..."
docker exec minio1 mc mirror local/fastgpt-data /tmp/minio_backup
docker cp minio1:/tmp/minio_backup $BACKUP_DIR/
tar -czf $BACKUP_DIR/minio_backup.tar.gz -C $BACKUP_DIR minio_backup
rm -rf $BACKUP_DIR/minio_backup

# 5. 配置文件备份
echo "备份配置文件..."
tar -czf $BACKUP_DIR/config_backup.tar.gz \
  ./docker-compose*.yml \
  ./nginx*.conf \
  ./*.env \
  ./prometheus-config \
  ./grafana-dashboards

# 6. AI模型备份 (可选，模型文件较大)
if [[ "$BACKUP_AI_MODELS" == "true" ]]; then
    echo "备份AI模型..."
    tar -czf $BACKUP_DIR/ai_models_backup.tar.gz ./ollama-node*/
fi

# 7. 清理过期备份
echo "清理过期备份..."
find /backup -type d -mtime +30 -exec rm -rf {} \;

# 8. 备份到远程存储 (可选)
if [[ "$REMOTE_BACKUP" == "true" ]]; then
    echo "同步到远程存储..."
    rsync -av $BACKUP_DIR/ backup-server:/backup/fastgpt/
fi

echo "备份完成 $(date)"
```

## 📈 性能优化建议

### 1. 系统级优化

```bash
#!/bin/bash
# 系统性能优化脚本

echo "开始系统性能优化..."

# 1. 内核参数优化
cat >> /etc/sysctl.conf << 'EOF'
# 网络优化
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 87380 134217728
net.ipv4.tcp_wmem = 4096 65536 134217728
net.ipv4.tcp_congestion_control = bbr
net.core.netdev_max_backlog = 5000
net.ipv4.tcp_max_syn_backlog = 8192

# 文件系统优化
fs.file-max = 65536
fs.inotify.max_user_watches = 524288

# 虚拟内存优化
vm.swappiness = 10
vm.dirty_ratio = 10
vm.dirty_background_ratio = 5
vm.vfs_cache_pressure = 50
EOF

sysctl -p

# 2. 系统资源限制优化
cat >> /etc/security/limits.conf << 'EOF'
* soft nofile 65536
* hard nofile 65536
* soft nproc 32768
* hard nproc 32768
EOF

# 3. Docker性能优化
cat > /etc/docker/daemon.json << 'EOF'
{
    "storage-driver": "overlay2",
    "log-level": "warn",
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "10m",
        "max-file": "3"
    },
    "live-restore": true,
    "max-concurrent-downloads": 10,
    "max-concurrent-uploads": 5,
    "default-ulimits": {
        "nofile": {
            "Name": "nofile",
            "Hard": 65536,
            "Soft": 65536
        }
    }
}
EOF

systemctl restart docker

echo "系统性能优化完成!"
```

### 2. 应用级优化配置

```yaml
# 性能优化的docker-compose配置
version: '3.8'
services:
  fastgpt-backend:
    deploy:
      resources:
        limits:
          cpus: '4.0'
          memory: 8G
        reservations:
          cpus: '2.0'
          memory: 4G
    environment:
      # Node.js性能优化
      NODE_OPTIONS: "--max-old-space-size=6144 --max-semi-space-size=128"
      # 并发处理优化
      WORKER_PROCESSES: "4"
      MAX_CONNECTIONS: "1000"
      # 缓存优化
      REDIS_MAX_CONNECTIONS: "100"
      MONGODB_MAX_POOL_SIZE: "50"
    ulimits:
      nofile:
        soft: 65536
        hard: 65536
        
  mongodb-primary:
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 4G
        reservations:
          cpus: '1.0'
          memory: 2G
    command: |
      mongod --replSet fastgpt-rs 
             --bind_ip_all 
             --wiredTigerCacheSizeGB 2
             --wiredTigerCollectionBlockCompressor zstd
             --wiredTigerIndexPrefixCompression true
             
  redis-master:
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 2G
        reservations:
          cpus: '0.5'
          memory: 1G
    command: |
      redis-server --requirepass redis123
                   --maxmemory 1536mb
                   --maxmemory-policy allkeys-lru
                   --save 900 1
                   --tcp-keepalive 60
```

---

*本实施指南为FastGPT私有化部署提供了完整的实操流程，从基础设施准备到系统优化，确保部署过程的标准化和可重复性。*