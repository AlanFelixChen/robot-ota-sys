# OTA生产环境部署文档

## 1. 概述

### 1.1 文档目的
定义机器人OTA升级系统在**生产环境中的完整部署方案**，包括云端服务部署、设备端预置、运维监控及安全策略，确保OTA系统在生产环境中的高可用、高安全和高效率运行。

1.2 部署架构总览

```
┌─────────────────────────────────────────────────────────────────┐
│                         云端基础设施                              │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │  负载均衡器    │  │  API网关集群  │  │  消息队列集群  │           │
│  │  (Nginx)     │  │  (Kong)      │  │  (Kafka)     │           │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘           │
│         │                 │                 │                   │
│  ┌──────┴─────────────────┴─────────────────┴───────┐           │
│  │                  Kubernetes 集群                  │           │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐          │           │
│  │  │OTA业务服务│ │设备管理服务│ │固件管理服务│          │           │
│  │  │ (3副本)  │ │ (3副本)  │ │ (2副本)  │            │           │
│  │  └──────────┘ └──────────┘ └──────────┘          │           │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐          │           │
│  │  │MQTT Broker│ │证书管理服务│ │升级策略服务│         │           │
│  │  │ (3副本)   │ │ (2副本)  │ │ (2副本)  │           │           │
│  │  └──────────┘ └──────────┘ └──────────┘          │           │
│  └──────────────────────────────────────────────────┘           │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │  Redis集群    │  │ PostgreSQL   │  │  CDN节点      │          │
│  │  (缓存)       │  │  (主从)      │  │  (全球加速)    │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3 环境规划

| 环境 | 用途 | 设备数量 | 部署区域 |
|------|------|---------|---------|
| 开发环境 | 开发调试 | 10台 | 单区域 |
| 测试环境 | 功能测试 | 100台 | 单区域 |
| 预发布环境 | 灰度验证 | 1000台 | 单区域 |
| 生产环境 | 正式服务 | 100000+台 | 全球多区域 |

---

# 2. 云端服务部署

### 2.1 基础设施要求

#### 2.1.1 计算资源

| 服务 | 副本数 | CPU | 内存 | 存储 | 说明 |
|------|--------|-----|------|------|------|
| OTA业务服务 | 3 | 4核 | 8GB | 50GB SSD | 处理升级业务逻辑 |
| 设备管理服务 | 3 | 4核 | 8GB | 50GB SSD | 设备注册、状态管理 |
| 固件管理服务 | 2 | 2核 | 4GB | 100GB SSD | 固件上传、存储管理 |
| MQTT Broker | 3 | 8核 | 16GB | 200GB SSD | EMQX集群 |
| 证书管理服务 | 2 | 2核 | 4GB | 50GB SSD | 证书签发、吊销 |
| 升级策略服务 | 2 | 2核 | 4GB | 50GB SSD | 灰度策略、版本管理 |

#### 2.1.2 网络要求

| 网络组件 | 带宽 | 并发连接 | 说明 |
|---------|------|---------|------|
| MQTT接入 | 1Gbps | 100,000 | 设备长连接 |
| HTTPS下载 | 10Gbps | 50,000 | CDN加速 |
| API网关 | 1Gbps | 10,000 | 管理接口 |
| 数据库 | 10Gbps | - | 内网专线 |

#### 2.1.3 存储要求

| 存储类型 | 容量 | IOPS | 用途 |
|---------|------|------|------|
| PostgreSQL | 500GB | 10000 | 业务数据 |
| Redis | 64GB | 50000 | 缓存、会话 |
| 对象存储 | 10TB | - | 固件文件 |
| 日志存储 | 5TB | - | 操作日志 |

### 2.2 Kubernetes部署配置

#### 2.2.1 OTA业务服务 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ota-business-service
  namespace: ota-production
  labels:
    app: ota-business
    version: v2.0.0
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: ota-business
  template:
    metadata:
      labels:
        app: ota-business
        version: v2.0.0
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - ota-business
            topologyKey: kubernetes.io/hostname
      containers:
      - name: ota-business
        image: registry.example.com/ota-business:v2.0.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
        - containerPort: 9090
          name: metrics
          protocol: TCP
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "production"
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: ota-db-secret
              key: host
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: ota-db-secret
              key: password
        - name: REDIS_HOST
          value: "redis-cluster.ota-production.svc.cluster.local"
        - name: MQTT_BROKER
          value: "emqx-cluster.ota-production.svc.cluster.local"
        resources:
          requests:
            cpu: "2"
            memory: "4Gi"
          limits:
            cpu: "4"
            memory: "8Gi"
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        volumeMounts:
        - name: config
          mountPath: /app/config
          readOnly: true
        - name: logs
          mountPath: /app/logs
      volumes:
      - name: config
        configMap:
          name: ota-business-config
      - name: logs
        emptyDir: {}
```

#### 2.2.2 MQTT Broker StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: emqx-cluster
  namespace: ota-production
spec:
  serviceName: emqx-headless
  replicas: 3
  podManagementPolicy: Parallel
  selector:
    matchLabels:
      app: emqx
  template:
    metadata:
      labels:
        app: emqx
    spec:
      containers:
      - name: emqx
        image: emqx/emqx-enterprise:5.0.0
        ports:
        - containerPort: 1883
          name: mqtt
        - containerPort: 8883
          name: mqtts
        - containerPort: 8083
          name: ws
        - containerPort: 8084
          name: wss
        - containerPort: 18083
          name: dashboard
        - containerPort: 4370
          name: ekka
        env:
        - name: EMQX_NAME
          value: "emqx"
        - name: EMQX_HOST
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: EMQX_CLUSTER__DISCOVERY_STRATEGY
          value: "k8s"
        - name: EMQX_CLUSTER__K8S__SERVICE_NAME
          value: "emqx-headless"
        - name: EMQX_CLUSTER__K8S__NAMESPACE
          value: "ota-production"
        - name: EMQX_CLUSTER__K8S__ADDRESS_TYPE
          value: "hostname"
        - name: EMQX_LISTENERS__SSL__DEFAULT__SSL_OPTIONS__CERTFILE
          value: "/etc/emqx/certs/server.crt"
        - name: EMQX_LISTENERS__SSL__DEFAULT__SSL_OPTIONS__KEYFILE
          value: "/etc/emqx/certs/server.key"
        resources:
          requests:
            cpu: "4"
            memory: "8Gi"
          limits:
            cpu: "8"
            memory: "16Gi"
        volumeMounts:
        - name: emqx-data
          mountPath: /opt/emqx/data
        - name: emqx-certs
          mountPath: /etc/emqx/certs
          readOnly: true
        - name: emqx-config
          mountPath: /opt/emqx/etc/emqx.conf
          subPath: emqx.conf
  volumeClaimTemplates:
  - metadata:
      name: emqx-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "ssd"
      resources:
        requests:
          storage: 200Gi
```

#### 2.2.3 Service配置

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ota-business-service
  namespace: ota-production
spec:
  type: ClusterIP
  selector:
    app: ota-business
  ports:
  - name: http
    port: 8080
    targetPort: 8080
  - name: metrics
    port: 9090
    targetPort: 9090

---
apiVersion: v1
kind: Service
metadata:
  name: emqx-headless
  namespace: ota-production
spec:
  clusterIP: None
  selector:
    app: emqx
  ports:
  - name: mqtt
    port: 1883
  - name: mqtts
    port: 8883
  - name: ekka
    port: 4370

---
apiVersion: v1
kind: Service
metadata:
  name: emqx-public
  namespace: ota-production
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
spec:
  type: LoadBalancer
  selector:
    app: emqx
  ports:
  - name: mqtts
    port: 8883
    targetPort: 8883
  - name: wss
    port: 8084
    targetPort: 8084
```

### 2.3 数据库部署

#### 2.3.1 PostgreSQL主从配置

```yaml
主库配置
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-master-config
  namespace: ota-production
data:
  postgresql.conf: |
    listen_addresses = '*'
    max_connections = 500
    shared_buffers = 4GB
    effective_cache_size = 12GB
    maintenance_work_mem = 1GB
    checkpoint_completion_target = 0.9
    wal_level = replica
    max_wal_senders = 5
    wal_keep_size = 1GB
    hot_standby = on
    max_standby_streaming_delay = 30s
    wal_receiver_status_interval = 10s
    hot_standby_feedback = on
    random_page_cost = 1.1
    effective_io_concurrency = 200
    work_mem = 16MB
    min_wal_size = 1GB
    max_wal_size = 4GB
    
  pg_hba.conf: |
    local   all             all                                     trust
    host    all             all             127.0.0.1/32            trust
    host    all             all             ::1/128                 trust
    host    replication     replicator      10.0.0.0/8              md5
    host    all             ota_app         10.0.0.0/8              scram-sha-256
```

2.3.2 数据库初始化脚本

```sql
-- 创建数据库
CREATE DATABASE ota_production
  WITH ENCODING = 'UTF8'
       LC_COLLATE = 'en_US.UTF-8'
       LC_CTYPE = 'en_US.UTF-8'
       TEMPLATE = template0;

-- 创建用户
CREATE USER ota_app WITH PASSWORD 'secure_password_here';
CREATE USER ota_readonly WITH PASSWORD 'readonly_password_here';

-- 授权
GRANT ALL PRIVILEGES ON DATABASE ota_production TO ota_app;
GRANT CONNECT ON DATABASE ota_production TO ota_readonly;

-- 连接数据库
\c ota_production;

-- 创建Schema
CREATE SCHEMA IF NOT EXISTS ota AUTHORIZATION ota_app;

-- 设备表
CREATE TABLE ota.devices (
    device_id VARCHAR(64) PRIMARY KEY,
    device_type VARCHAR(32) NOT NULL,
    hardware_revision VARCHAR(16),
    current_version VARCHAR(32) NOT NULL,
    status VARCHAR(16) NOT NULL DEFAULT 'ONLINE',
    last_seen TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 固件表
CREATE TABLE ota.firmwares (
    firmware_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version VARCHAR(32) NOT NULL,
    device_type VARCHAR(32) NOT NULL,
    file_url TEXT NOT NULL,
    file_size BIGINT NOT NULL,
    sha256_hash VARCHAR(64) NOT NULL,
    signature TEXT NOT NULL,
    release_notes TEXT,
    status VARCHAR(16) NOT NULL DEFAULT 'DRAFT',
    created_by VARCHAR(64),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE(version, device_type)
);

-- 升级任务表
CREATE TABLE ota.upgrade_tasks (
    task_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    firmware_id UUID REFERENCES ota.firmwares(firmware_id),
    task_name VARCHAR(128) NOT NULL,
    strategy JSONB NOT NULL,
    status VARCHAR(16) NOT NULL DEFAULT 'CREATED',
    total_devices INT DEFAULT 0,
    success_count INT DEFAULT 0,
    failed_count INT DEFAULT 0,
    created_by VARCHAR(64),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    started_at TIMESTAMP WITH TIME ZONE,
    completed_at TIMESTAMP WITH TIME ZONE
);

-- 升级记录表
CREATE TABLE ota.upgrade_records (
    record_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id UUID REFERENCES ota.upgrade_tasks(task_id),
    device_id VARCHAR(64) REFERENCES ota.devices(device_id),
    from_version VARCHAR(32) NOT NULL,
    to_version VARCHAR(32) NOT NULL,
    status VARCHAR(16) NOT NULL,
    progress INT DEFAULT 0,
    error_code INT,
    error_message TEXT,
    started_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    completed_at TIMESTAMP WITH TIME ZONE,
    detail JSONB
);

-- 索引
CREATE INDEX idx_devices_status ON ota.devices(status);
CREATE INDEX idx_devices_type ON ota.devices(device_type);
CREATE INDEX idx_firmwares_version ON ota.firmwares(version, device_type);
CREATE INDEX idx_upgrade_tasks_status ON ota.upgrade_tasks(status);
CREATE INDEX idx_upgrade_records_device ON ota.upgrade_records(device_id);
CREATE INDEX idx_upgrade_records_task ON ota.upgrade_records(task_id);
CREATE INDEX idx_upgrade_records_status ON ota.upgrade_records(status);

-- 分区表（按月分区）
CREATE TABLE ota.upgrade_records_partitioned (
    record_id UUID,
    task_id UUID,
    device_id VARCHAR(64),
    from_version VARCHAR(32),
    to_version VARCHAR(32),
    status VARCHAR(16),
    progress INT,
    error_code INT,
    error_message TEXT,
    started_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    completed_at TIMESTAMP WITH TIME ZONE,
    detail JSONB
) PARTITION BY RANGE (started_at);

-- 创建分区
CREATE TABLE ota.upgrade_records_2026_07 PARTITION OF ota.upgrade_records_partitioned
    FOR VALUES FROM ('2026-07-01') TO ('2026-08-01');
CREATE TABLE ota.upgrade_records_2026_08 PARTITION OF ota.upgrade_records_partitioned
    FOR VALUES FROM ('2026-08-01') TO ('2026-09-01');
```

### 2.4 Redis集群配置

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-cluster-config
  namespace: ota-production
data:
  redis.conf: |
    集群配置
    cluster-enabled yes
    cluster-config-file /data/nodes.conf
    cluster-node-timeout 5000
    cluster-require-full-coverage yes
    
    持久化配置
    appendonly yes
    appendfsync everysec
    auto-aof-rewrite-percentage 100
    auto-aof-rewrite-min-size 64mb
    
    内存管理
    maxmemory 48gb
    maxmemory-policy allkeys-lru
    
    网络配置
    tcp-backlog 511
    timeout 300
    tcp-keepalive 60
    
    慢日志
    slowlog-log-slower-than 10000
    slowlog-max-len 128
    
    安全
    requirepass ${REDIS_PASSWORD}
    masterauth ${REDIS_PASSWORD}
    
    性能优化
    hash-max-ziplist-entries 512
    hash-max-ziplist-value 64
    list-max-ziplist-size -2
    set-max-intset-entries 512
    zset-max-ziplist-entries 128
    zset-max-ziplist-value 64
```

---

## 3. 设备端部署

### 3.1 预置要求

#### 3.1.1 出厂预置清单

| 预置项 | 说明 | 存储位置 |
|--------|------|---------|
| 设备证书 | X.509客户端证书 | ATECC608安全芯片 |
| CA根证书 | 云端CA根证书 | /etc/ota/certs/ca.crt |
| MQTT配置 | Broker地址、端口 | /etc/ota/config/mqtt.conf |
| 初始固件 | 出厂版本固件 | 分区A |
| U-Boot环境 | 启动配置 | Flash环境变量区 |
| RPMB初始值 | 安全存储初始值 | eMMC RPMB分区 |

#### 3.1.2 设备证书烧录

```bash
!/bin/bash
设备证书烧录脚本

DEVICE_ID=$1
CERT_DIR="/path/to/certificates/${DEVICE_ID}"

检查证书文件
if [ ! -f "${CERT_DIR}/device.crt" ] || [ ! -f "${CERT_DIR}/device.key" ]; then
    echo "Error: Certificate files not found for device ${DEVICE_ID}"
    exit 1
fi

烧录到ATECC608
echo "Burning certificate to ATECC608 for device ${DEVICE_ID}..."

写入设备证书
atecc608-tool write-cert \
    --slot 0 \
    --file "${CERT_DIR}/device.crt"

写入私钥
atecc608-tool write-key \
    --slot 0 \
    --file "${CERT_DIR}/device.key"

写入CA证书
atecc608-tool write-cert \
    --slot 1 \
    --file "${CERT_DIR}/ca.crt"

锁定配置区
atecc608-tool lock-config

验证烧录
echo "Verifying certificate..."
atecc608-tool verify-cert --slot 0

echo "Certificate burning completed for device ${DEVICE_ID}"
```

3.1.3 设备初始化配置

```yaml
/etc/ota/config/ota.yaml
ota:
  device_id: "${DEVICE_ID}"
  mqtt:
    broker: "mqtts://ota-broker.example.com:8883"
    client_id: "${DEVICE_ID}"
    keep_alive: 60
    clean_session: false
    qos: 1
    tls:
      cert_file: "/etc/ota/certs/device.crt"
      key_file: "/etc/ota/certs/device.key"
      ca_file: "/etc/ota/certs/ca.crt"
      verify_peer: true
  
  download:
    max_retries: 3
    retry_delay: 30
    timeout: 300
    chunk_size: 1048576  1MB
    verify_sha256: true
  
  storage:
    temp_dir: "/data/ota/temp"
    firmware_dir: "/data/ota/firmware"
    log_dir: "/data/ota/logs"
    max_temp_size: 2147483648  2GB
  
  security:
    secure_element: "atecc608"
    rpmb_device: "/dev/mmcblk0rpmb"
    verify_signature: true
    verify_certificate: true
  
  watchdog:
    enabled: true
    timeout: 60
    feed_interval: 30
  
  health_check:
    enabled: true
    check_interval: 10
    required_services:
      - "ota-daemon"
      - "network-manager"
      - "sensor-service"
      - "motor-control"
```

### 3.2 OTA守护进程部署

#### 3.2.1 Systemd服务配置

```ini
/etc/systemd/system/ota-daemon.service
[Unit]
Description=OTA Upgrade Daemon
Documentation=https://docs.example.com/ota
After=network-online.target time-sync.target
Wants=network-online.target
Requires=network-online.target

[Service]
Type=notify
User=ota
Group=ota
ExecStart=/usr/bin/ota-daemon --config /etc/ota/config/ota.yaml
ExecReload=/bin/kill -HUP $MAINPID
Restart=always
RestartSec=10
StartLimitInterval=60
StartLimitBurst=3

安全加固
NoNewPrivileges=yes
PrivateTmp=yes
ProtectSystem=strict
ProtectHome=yes
ReadWritePaths=/data/ota /var/log/ota
ReadOnlyPaths=/etc/ota
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectControlGroups=yes
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX AF_NETLINK
RestrictRealtime=yes
MemoryDenyWriteExecute=yes
LockPersonality=yes

资源限制
LimitNOFILE=65536
LimitNPROC=4096
MemoryMax=512M
CPUQuota=50%

看门狗
WatchdogSec=60

[Install]
WantedBy=multi-user.target
```

#### 3.2.2 日志轮转配置

```ini
/etc/logrotate.d/ota-daemon
/var/log/ota/ota-daemon.log {
    daily
    rotate 30
    missingok
    notifempty
    compress
    delaycompress
    copytruncate
    maxsize 100M
    dateext
    dateformat -%Y%m%d
    create 0640 ota ota
    postrotate
        /bin/kill -USR1 $(cat /var/run/ota-daemon.pid) 2>/dev/null || true
    endscript
}
```

### 3.3 监控代理部署

```yaml
Prometheus Node Exporter配置
apiVersion: v1
kind: ConfigMap
metadata:
  name: node-exporter-config
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
    
    scrape_configs:
      - job_name: 'ota-device'
        static_configs:
          - targets: ['localhost:9100']
        metric_relabel_configs:
          - source_labels: [__name__]
            regex: 'node_(cpu|memory|disk|network).*'
            action: keep
      
      - job_name: 'ota-daemon'
        static_configs:
          - targets: ['localhost:9090']
        metrics_path: '/metrics'
```

---

## 4. 安全部署

### 4.1 证书管理

#### 4.1.1 证书层级

```
Root CA (离线存储，10年有效期)
├── Intermediate CA 1 (HSM存储，5年有效期)
│   ├── Server Certificate (MQTT Broker，1年有效期)
│   ├── Server Certificate (HTTPS CDN，1年有效期)
│   └── Server Certificate (API Gateway，1年有效期)
└── Intermediate CA 2 (HSM存储，5年有效期)
    └── Device Certificates (设备端，5年有效期)
```

#### 4.1.2 证书自动续期

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: ota-mqtt-broker-cert
  namespace: ota-production
spec:
  secretName: ota-mqtt-tls
  duration: 2160h  90天
  renewBefore: 720h  30天前续期
  subject:
    organizations:
      - "Example Corp"
  commonName: "mqtts.ota.example.com"
  dnsNames:
    - "mqtts.ota.example.com"
    - "*.mqtt.ota.example.com"
  issuerRef:
    name: ota-intermediate-ca
    kind: ClusterIssuer
  privateKey:
    algorithm: ECDSA
    size: 256
    rotationPolicy: Always
```

### 4.2 网络安全策略

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ota-network-policy
  namespace: ota-production
spec:
  podSelector:
    matchLabels:
      app: ota-business
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api-gateway
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - podSelector:
        matchLabels:
          app: redis
    ports:
    - protocol: TCP
      port: 6379
  - to:
    - podSelector:
        matchLabels:
          app: emqx
    ports:
    - protocol: TCP
      port: 1883
```

### 4.3 密钥管理

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: ota-secrets
  namespace: ota-production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: ota-secrets
    creationPolicy: Owner
  data:
  - secretKey: db-password
    remoteRef:
      key: ota/production/database
      property: password
  - secretKey: redis-password
    remoteRef:
      key: ota/production/redis
      property: password
  - secretKey: jwt-secret
    remoteRef:
      key: ota/production/jwt
      property: secret
  - secretKey: mqtt-admin-password
    remoteRef:
      key: ota/production/mqtt
      property: admin-password
```

---

## 5. 监控与告警

### 5.1 Prometheus监控配置

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: ota-business-monitor
  namespace: ota-production
spec:
  selector:
    matchLabels:
      app: ota-business
  endpoints:
  - port: metrics
    interval: 15s
    path: /actuator/prometheus
    relabelings:
    - sourceLabels: [__meta_kubernetes_pod_name]
      targetLabel: pod
    - sourceLabels: [__meta_kubernetes_namespace]
      targetLabel: namespace
```

### 5.2 关键告警规则

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: ota-alert-rules
  namespace: ota-production
spec:
  groups:
  - name: ota-service
    rules:
    - alert: OTAServiceDown
      expr: up{job="ota-business"} == 0
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: "OTA业务服务不可用"
        description: "OTA业务服务 {{ $labels.pod }} 已停止运行超过2分钟"
    
    - alert: OTAMQTTConnectionHigh
      expr: emqx_connections_count > 80000
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "MQTT连接数过高"
        description: "当前MQTT连接数 {{ $value }}，接近上限100000"
    
    - alert: OTAUpgradeFailureRate
      expr: rate(ota_upgrade_failed_total[5m]) / rate(ota_upgrade_total[5m]) > 0.05
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "升级失败率过高"
        description: "最近5分钟升级失败率 {{ $value | humanizePercentage }}"
    
    - alert: OTADownloadSlow
      expr: histogram_quantile(0.95, rate(ota_download_duration_seconds_bucket[5m])) > 600
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "固件下载速度过慢"
        description: "P95下载耗时 {{ $value }}秒，超过600秒阈值"
    
    - alert: OTADatabaseConnectionPool
      expr: hikaricp_connections_active{pool="OTA-Pool"} / hikaricp_connections_max{pool="OTA-Pool"} > 0.8
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "数据库连接池使用率过高"
        description: "连接池使用率 {{ $value | humanizePercentage }}"
```

### 5.3 Grafana仪表板

```json
{
  "dashboard": {
    "title": "OTA系统监控",
    "panels": [
      {
        "title": "升级任务概览",
        "targets": [
          {
            "expr": "sum(ota_upgrade_total) by (status)",
            "legendFormat": "{{status}}"
          }
        ]
      },
      {
        "title": "设备在线数",
        "targets": [
          {
            "expr": "emqx_connections_count",
            "legendFormat": "在线设备"
          }
        ]
      },
      {
        "title": "升级成功率",
        "targets": [
          {
            "expr": "sum(rate(ota_upgrade_success_total[1h])) / sum(rate(ota_upgrade_total[1h])) * 100",
            "legendFormat": "成功率"
          }
        ]
      },
      {
        "title": "固件下载速度",
        "targets": [
          {
            "expr": "histogram_quantile(0.50, rate(ota_download_speed_bytes_bucket[5m]))",
            "legendFormat": "P50"
          },
          {
            "expr": "histogram_quantile(0.95, rate(ota_download_speed_bytes_bucket[5m]))",
            "legendFormat": "P95"
          }
        ]
      }
    ]
  }
}
```

---

## 6. 灰度发布策略

### 6.1 灰度阶段定义

| 阶段 | 设备比例 | 持续时间 | 监控指标 | 通过条件 |
|------|---------|---------|---------|---------|
| 金丝雀 | 1% | 2小时 | 成功率、崩溃率 | 成功率>99% |
| 小规模 | 5% | 6小时 | 成功率、性能指标 | 成功率>99.5% |
| 中规模 | 20% | 12小时 | 成功率、用户反馈 | 成功率>99.9% |
| 大规模 | 50% | 24小时 | 全量指标 | 无严重问题 |
| 全量 | 100% | - | 持续监控 | - |

### 6.2 灰度策略配置

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: canary-strategy
  namespace: ota-production
data:
  strategy.yaml: |
    canary:
      enabled: true
      stages:
        - name: "canary"
          percentage: 1
          duration: 2h
          auto_promote: false
          metrics:
            - name: "upgrade_success_rate"
              threshold: 99.0
            - name: "crash_rate"
              threshold: 0.1
            - name: "rollback_rate"
              threshold: 0.5
        
        - name: "small_batch"
          percentage: 5
          duration: 6h
          auto_promote: true
          metrics:
            - name: "upgrade_success_rate"
              threshold: 99.5
            - name: "device_online_rate"
              threshold: 99.0
        
        - name: "medium_batch"
          percentage: 20
          duration: 12h
          auto_promote: false
          metrics:
            - name: "upgrade_success_rate"
              threshold: 99.9
            - name: "battery_drain_rate"
              threshold: 5.0
        
        - name: "large_batch"
          percentage: 50
          duration: 24h
          auto_promote: false
          
        - name: "full_rollout"
          percentage: 100
          duration: 0
```

### 6.3 自动回滚策略

```yaml
auto_rollback:
  enabled: true
  triggers:
    - metric: "upgrade_success_rate"
      condition: "< 95%"
      action: "immediate_rollback"
    
    - metric: "device_crash_rate"
      condition: "> 1%"
      action: "pause_and_review"
    
    - metric: "critical_events"
      condition: "> 10 in 5min"
      action: "immediate_rollback"
    
    - metric: "battery_anomaly_rate"
      condition: "> 2%"
      action: "pause_and_review"
```

---

## 7. 运维操作手册

### 7.1 日常运维

#### 7.1.1 健康检查脚本

```bash
!/bin/bash
OTA系统健康检查脚本

set -e

echo "=== OTA System Health Check ==="
echo "Time: $(date)"
echo ""

检查Kubernetes Pod状态
echo "1. Checking Kubernetes Pods..."
kubectl get pods -n ota-production | grep -v "Running\|Completed" && echo "WARNING: Some pods not healthy" || echo "OK"

检查MQTT连接数
echo "2. Checking MQTT Connections..."
MQTT_CONNECTIONS=$(curl -s http://emqx-public:18083/api/v4/stats | jq '.data.connections.count')
echo "MQTT Connections: $MQTT_CONNECTIONS"
if [ "$MQTT_CONNECTIONS" -gt 80000 ]; then
    echo "WARNING: High connection count"
fi

检查数据库连接
echo "3. Checking Database..."
DB_CONNECTIONS=$(kubectl exec -n ota-production postgres-0 -- psql -U ota_app -d ota_production -c "SELECT count(*) FROM pg_stat_activity;" -t)
echo "DB Connections: $DB_CONNECTIONS"

检查Redis集群
echo "4. Checking Redis Cluster..."
kubectl exec -n ota-production redis-cluster-0 -- redis-cli cluster info | grep "cluster_state:ok" || echo "ERROR: Redis cluster not healthy"

检查CDN状态
echo "5. Checking CDN..."
curl -sI https://ota-cdn.example.com/health | head -1

检查磁盘使用
echo "6. Checking Disk Usage..."
kubectl exec -n ota-production postgres-0 -- df -h /var/lib/postgresql/data | tail -1

echo ""
echo "=== Health Check Complete ==="
```

#### 7.1.2 日志收集

```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: Flow
metadata:
  name: ota-logs
  namespace: ota-production
spec:
  filters:
  - tag_normaliser:
      format: ${namespace}.${pod}.${container}
  - parser:
      remove_key_name_field: true
      reserve_data: true
      parsers:
      - type: json
        time:
          format: "%Y-%m-%dT%H:%M:%S.%L%z"
  match:
  - select:
      labels:
        app.kubernetes.io/part-of: ota
  localOutputRefs:
  - ota-elasticsearch
```

### 7.2 故障处理

#### 7.2.1 常见故障处理流程

| 故障现象 | 排查步骤 | 处理方案 |
|---------|---------|---------|
| MQTT连接失败 | 1.检查Broker状态<br>2.检查证书有效期<br>3.检查网络连通性 | 重启Broker/更新证书/检查防火墙 |
| 下载速度慢 | 1.检查CDN状态<br>2.检查带宽使用<br>3.检查源站负载 | 切换CDN节点/扩容带宽/增加源站 |
| 升级失败率高 | 1.检查固件完整性<br>2.检查设备兼容性<br>3.检查存储空间 | 暂停升级/回滚/修复固件 |
| 数据库性能下降 | 1.检查慢查询<br>2.检查连接数<br>3.检查磁盘IO | 优化查询/扩容连接池/升级磁盘 |

#### 7.2.2 紧急回滚操作

```bash
!/bin/bash
紧急回滚脚本

UPGRADE_TASK_ID=$1
REASON=$2

if [ -z "$UPGRADE_TASK_ID" ]; then
    echo "Usage: $0 <upgrade_task_id> <reason>"
    exit 1
fi

echo "=== Emergency Rollback ==="
echo "Task ID: $UPGRADE_TASK_ID"
echo "Reason: $REASON"
echo ""

1. 暂停升级任务
echo "1. Pausing upgrade task..."
curl -X POST "https://api.ota.example.com/api/v1/tasks/${UPGRADE_TASK_ID}/pause" \
    -H "Authorization: Bearer ${API_TOKEN}"

2. 获取已升级设备列表
echo "2. Getting upgraded devices..."
UPGRADED_DEVICES=$(curl -s "https://api.ota.example.com/api/v1/tasks/${UPGRADE_TASK_ID}/devices?status=SUCCESS" \
    -H "Authorization: Bearer ${API_TOKEN}" | jq -r '.devices[].device_id')

3. 下发回滚指令
echo "3. Sending rollback commands..."
for DEVICE_ID in $UPGRADED_DEVICES; do
    echo "Rolling back device: $DEVICE_ID"
    mosquitto_pub -h ota-broker.example.com -p 8883 \
        --cafile /etc/ota/certs/ca.crt \
        --cert /etc/ota/certs/admin.crt \
        --key /etc/ota/certs/admin.key \
        -t "ota/${DEVICE_ID}/down/rollback" \
        -m "{\"msg_type\":\"force_rollback\",\"upgrade_id\":\"${UPGRADE_TASK_ID}\",\"reason\":\"${REASON}\"}"
done

4. 更新任务状态
echo "4. Updating task status..."
curl -X PUT "https://api.ota.example.com/api/v1/tasks/${UPGRADE_TASK_ID}" \
    -H "Authorization: Bearer ${API_TOKEN}" \
    -H "Content-Type: application/json" \
    -d "{\"status\":\"ROLLED_BACK\",\"reason\":\"${REASON}\"}"

echo ""
echo "=== Rollback Complete ==="
```

### 7.3 备份与恢复

#### 7.3.1 数据库备份

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: ota-production
spec:
  schedule: "0 2 * * *"  每天凌晨2点
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: postgres:15
            command:
            - /bin/bash
            - -c
            - |
              pg_dump -h postgres-master -U ota_app -d ota_production \
                -F c -f /backup/ota_$(date +%Y%m%d_%H%M%S).dump
              上传到对象存储
              aws s3 cp /backup/ s3://ota-backups/postgres/ --recursive
              清理7天前的备份
              find /backup -name "*.dump" -mtime +7 -delete
            env:
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: ota-db-secret
                  key: password
            volumeMounts:
            - name: backup
              mountPath: /backup
          volumes:
          - name: backup
            emptyDir: {}
          restartPolicy: OnFailure
```

#### 7.3.2 灾难恢复流程

```
灾难恢复步骤：

1. 基础设施恢复
   ├── 创建新的Kubernetes集群
   ├── 部署基础服务（数据库、Redis、MQTT）
   └── 配置网络和安全策略

2. 数据恢复
   ├── 从备份恢复PostgreSQL数据
   ├── 恢复Redis缓存（可从数据库重建）
   └── 恢复固件文件到对象存储

3. 服务恢复
   ├── 部署OTA业务服务
   ├── 部署设备管理服务
   ├── 部署固件管理服务
   └── 配置负载均衡和DNS

4. 验证
   ├── 运行健康检查脚本
   ├── 测试MQTT连接
   ├── 测试固件下载
   └── 执行端到端升级测试

5. 切换流量
   ├── 更新DNS记录
   ├── 监控设备重连
   └── 确认服务正常

预计恢复时间：2小时
数据丢失窗口：最多24小时
```

---

## 8. 性能优化

### 8.1 数据库优化

```sql
-- 查询优化索引
CREATE INDEX CONCURRENTLY idx_upgrade_records_composite 
ON ota.upgrade_records(device_id, status, started_at DESC);

-- 物化视图（设备升级统计）
CREATE MATERIALIZED VIEW ota.device_upgrade_stats AS
SELECT 
    device_id,
    COUNT(*) as total_upgrades,
    COUNT(*) FILTER (WHERE status = 'SUCCESS') as success_count,
    MAX(started_at) as last_upgrade_time,
    AVG(EXTRACT(EPOCH FROM (completed_at - started_at))) as avg_duration_seconds
FROM ota.upgrade_records
GROUP BY device_id;

CREATE UNIQUE INDEX idx_device_upgrade_stats_device 
ON ota.device_upgrade_stats(device_id);

-- 定期刷新物化视图
REFRESH MATERIALIZED VIEW CONCURRENTLY ota.device_upgrade_stats;
```

### 8.2 缓存策略

```yaml
Redis缓存配置
cache_config:
  device_info:
    ttl: 300  5分钟
    strategy: "write-through"
  
  firmware_metadata:
    ttl: 3600  1小时
    strategy: "cache-aside"
  
  upgrade_task:
    ttl: 600  10分钟
    strategy: "write-behind"
  
  download_token:
    ttl: 86400  24小时
    strategy: "cache-only"
  
  rate_limit:
    ttl: 60  1分钟
    strategy: "cache-only"
```

### 8.3 CDN优化

```nginx
CDN配置
location /firmware/ {
    启用缓存
    proxy_cache STATIC;
    proxy_cache_valid 200 7d;
    proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
    
    大文件优化
    proxy_buffering on;
    proxy_buffer_size 128k;
    proxy_buffers 4 256k;
    proxy_busy_buffers_size 256k;
    
    支持断点续传
    proxy_set_header Range $http_range;
    proxy_set_header If-Range $http_if_range;
    
    安全头
    add_header X-Content-Type-Options nosniff;
    add_header Content-Security-Policy "default-src 'none'";
    
    限速
    limit_rate_after 10m;
    limit_rate 50m;
}
```

---

## 9. 合规与审计

### 9.1 审计日志

```yaml
审计日志配置
audit:
  enabled: true
  log_level: INFO
  events:
    - UPGRADE_CREATED
    - UPGRADE_STARTED
    - UPGRADE_COMPLETED
    - UPGRADE_FAILED
    - UPGRADE_ROLLBACK
    - DEVICE_REGISTERED
    - DEVICE_DELETED
    - FIRMWARE_UPLOADED
    - FIRMWARE_DELETED
    - CERTIFICATE_ISSUED
    - CERTIFICATE_REVOKED
  storage:
    type: elasticsearch
    retention_days: 365
    index_pattern: "ota-audit-*"
```

### 9.2 数据保护

```yaml
数据保护策略
data_protection:
  encryption:
    at_rest:
      database: AES-256
      files: AES-256
    in_transit:
      mqtt: TLS 1.3
      https: TLS 1.3
  
  data_retention:
    upgrade_records: 2 years
    device_logs: 90 days
    audit_logs: 1 year
    firmware_files: 5 years
  
  data_anonymization:
    enabled: true
    fields:
      - device_id: hash
      - ip_address: mask_last_octet
```

---

## 10. 容量规划

### 10.1 扩展性规划

| 设备规模 | MQTT Broker | API服务 | 数据库 | CDN带宽 |
|---------|------------|---------|--------|---------|
| 10,000 | 2核4GB x1 | 2核4GB x2 | 4核16GB | 100Mbps |
| 100,000 | 4核8GB x3 | 4核8GB x3 | 8核32GB | 1Gbps |
| 500,000 | 8核16GB x5 | 8核16GB x5 | 16核64GB | 5Gbps |
| 1,000,000 | 16核32GB x10 | 16核32GB x10 | 32核128GB | 10Gbps |

### 10.2 成本估算

| 资源 | 月成本（美元） | 说明 |
|------|-------------|------|
| Kubernetes集群 | $2,000 | 3节点，每节点4核16GB |
| 数据库 | $1,500 | RDS PostgreSQL，4核16GB |
| Redis | $800 | ElastiCache，3节点集群 |
| CDN | $3,000 | 10TB流量/月 |
| 负载均衡 | $500 | NLB + ALB |
| 对象存储 | $1,000 | 50TB存储 |
| 监控告警 | $500 | Prometheus + Grafana |
| 日志系统 | $800 | Elasticsearch集群 |
| 证书管理 | $200 | Private CA |
| 总计 | $10,300 | 月成本 |

---

## 附录

A. 部署检查清单

- [ ] Kubernetes集群就绪
- [ ] 数据库主从配置完成
- [ ] Redis集群初始化完成
- [ ] MQTT Broker集群部署完成
- [ ] CDN配置完成
- [ ] SSL证书部署完成
- [ ] 监控系统部署完成
- [ ] 日志系统部署完成
- [ ] 备份策略配置完成
- [ ] 告警规则配置完成
- [ ] 灰度策略配置完成
- [ ] 安全策略应用完成
- [ ] 负载测试通过
- [ ] 故障恢复演练完成
- [ ] 文档更新完成

B. 联系方式

| 角色 | 联系方式 | 说明 |
|------|---------|------|
| OTA运维 | ota-ops@example.com | 日常运维 |
| 安全团队 | security@example.com | 安全事件 |
| DBA | dba@example.com | 数据库问题 |
| 网络团队 | network@example.com | 网络问题 |
| 紧急联系 | +1-xxx-xxx-xxxx | 7x24小时 |