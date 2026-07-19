# OTA通信协议设计文档

## 1. 概述

### 1.1 文档目的
定义机器人端**OTA升级系统的完整通信协议规范**，包括MQTT指令协议、HTTPS下载协议、数据格式定义及交互时序，确保云端与设备端通信的标准化和可靠性。

### 1.2 协议架构

```
┌─────────────────────────────────────────┐
│               云端OTA服务                 │
│  ┌─────────┐  ┌──────────┐  ┌─────────┐  │
│  │MQTT Broker││HTTPS CDN │  │ 业务服务 │  │
│  └────┬────┘  └────┬─────┘  └────┬────┘  │
└───────┼────────────┼─────────────┼───────┘
        │            │             │
   MQTT │       HTTPS│        HTTP │
   (8883)│      (443)│       (443) │
        │            │             │
┌───────┼────────────┼─────────────┼──────┐
│  ┌────▼────┐  ┌────▼─────┐  ┌───▼─────┐ │
│  │OTA Daemon│ │下载管理器  │  │状态上报  │ │
│  └─────────┘  └──────────┘  └─────────┘ │
│              机器人端OTA客户端            │
└─────────────────────────────────────────┘
```

### 1.3 通信协议选型

| 通信场景 | 协议 | 端口 | 说明 |
|---------|------|------|------|
| 升级指令下发 | MQTT over TLS | 8883 | 长连接，低延迟，双向通信 |
| 状态上报 | MQTT over TLS | 8883 | 实时上报升级进度和状态 |
| 固件下载 | HTTPS | 443 | 大文件传输，支持断点续传 |
| 版本查询 | MQTT Request/Response | 8883 | 请求-响应模式 |
| 证书管理 | HTTPS | 443 | 证书更新和吊销列表查询 |

---

## 2. MQTT协议规范

### 2.1 连接配置

```
连接参数：
- Broker地址：mqtts://ota-broker.example.com:8883
- 协议版本：MQTT 5.0
- TLS版本：1.3
- 认证方式：X.509客户端证书（ATECC608签名）
- Keep Alive：60秒
- Clean Session：false
- Session Expiry Interval：7200秒（2小时）
- 最大报文大小：256KB
```

### 2.2 Topic定义

#### 2.2.1 Topic命名规范

```
格式：ota/{device_id}/{direction}/{category}[/{sub_category}]

device_id：设备唯一标识
direction：down（云端→设备）/ up（设备→云端）
category：功能分类
sub_category：子分类（可选）
```

#### 2.2.2 云端→设备 Topic

| Topic | QoS | 说明 | 保留消息 |
|-------|-----|------|---------|
| `ota/{device_id}/down/cmd` | 1 | 升级指令 | 否 |
| `ota/{device_id}/down/cancel` | 1 | 取消升级 | 否 |
| `ota/{device_id}/down/rollback` | 1 | 强制回滚 | 否 |
| `ota/{device_id}/down/config` | 1 | 配置更新 | 是 |
| `ota/{device_id}/down/cert` | 1 | 证书更新 | 否 |

#### 2.2.3 设备→云端 Topic

| Topic | QoS | 说明 | 保留消息 |
|-------|-----|------|---------|
| `ota/{device_id}/up/status` | 1 | 状态上报 | 否 |
| `ota/{device_id}/up/progress` | 1 | 进度上报 | 否 |
| `ota/{device_id}/up/event` | 1 | 事件上报 | 否 |
| `ota/{device_id}/up/version` | 1 | 版本上报 | 是 |
| `ota/{device_id}/up/heartbeat` | 0 | 心跳 | 否 |

#### 2.2.4 请求响应 Topic

| Topic | QoS | 说明 |
|-------|-----|------|
| `ota/{device_id}/down/request` | 1 | 云端请求 |
| `ota/{device_id}/up/response` | 1 | 设备响应 |

### 2.3 消息格式

#### 2.3.1 通用消息结构

```json
{
  "msg_id": "uuid-v4",
  "msg_type": "消息类型",
  "timestamp": 1234567890,
  "version": "1.0",
  "data": {
    // 具体消息内容
  }
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| msg_id | string | 是 | 消息唯一ID，UUID v4格式 |
| msg_type | string | 是 | 消息类型标识 |
| timestamp | int64 | 是 | 消息发送时间戳（毫秒） |
| version | string | 是 | 协议版本号 |
| data | object | 是 | 消息体，根据msg_type变化 |

#### 2.3.2 升级指令（cmd）

```json
{
  "msg_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "msg_type": "upgrade_cmd",
  "timestamp": 1690000000000,
  "version": "1.0",
  "data": {
    "upgrade_id": "upg-20260719-001",
    "target_version": "2.0.0",
    "firmware_url": "https://ota-cdn.example.com/firmware/v2.0.0.img",
    "firmware_size": 1572864000,
    "firmware_sha256": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
    "signature": "MEUCIQD...base64_encoded_signature",
    "delta_url": "https://ota-cdn.example.com/delta/v1.0.0_to_v2.0.0.zck",
    "delta_size": 251658240,
    "delta_sha256": "a1b2c3d4e5f6...",
    "delta_signature": "MEQCID...base64_encoded_signature",
    "upgrade_type": "delta",
    "priority": "normal",
    "schedule": {
      "start_time": 1690086400,
      "deadline": 1690172800,
      "install_policy": "immediate"
    },
    "mcu_firmwares": [
      {
        "mcu_id": "power_mcu",
        "version": "1.2.0",
        "url": "https://ota-cdn.example.com/mcu/power_v1.2.0.bin",
        "size": 65536,
        "sha256": "b2c3d4e5f6..."
      },
      {
        "mcu_id": "motor_mcu",
        "version": "2.1.0",
        "url": "https://ota-cdn.example.com/mcu/motor_v2.1.0.bin",
        "size": 131072,
        "sha256": "c3d4e5f6a7..."
      }
    ]
  }
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| upgrade_id | string | 是 | 升级任务唯一ID |
| target_version | string | 是 | 目标版本号 |
| firmware_url | string | 是 | 全量固件下载地址 |
| firmware_size | int64 | 是 | 全量固件大小（字节） |
| firmware_sha256 | string | 是 | 全量固件SHA256 |
| signature | string | 是 | 全量固件ECDSA签名 |
| delta_url | string | 否 | 差分包下载地址 |
| delta_size | int64 | 否 | 差分包大小 |
| delta_sha256 | string | 否 | 差分包SHA256 |
| delta_signature | string | 否 | 差分包签名 |
| upgrade_type | string | 是 | 升级类型：full/delta |
| priority | string | 是 | 优先级：low/normal/high/critical |
| schedule | object | 是 | 调度策略 |
| mcu_firmwares | array | 否 | MCU固件列表 |

#### 2.3.3 调度策略

```json
{
  "schedule": {
    "start_time": 1690086400,
    "deadline": 1690172800,
    "install_policy": "immediate"
  }
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| start_time | int64 | 是 | 允许开始升级时间戳 |
| deadline | int64 | 是 | 升级截止时间戳 |
| install_policy | string | 是 | 安装策略 |

安装策略：
| 策略 | 说明 |
|------|------|
| immediate | 立即下载安装 |
| idle_only | 仅空闲时安装 |
| scheduled | 按start_time指定时间安装 |
| user_confirm | 需用户确认后安装 |

#### 2.3.4 状态上报（status）

```json
{
  "msg_id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
  "msg_type": "status_report",
  "timestamp": 1690000001000,
  "version": "1.0",
  "data": {
    "upgrade_id": "upg-20260719-001",
    "device_id": "robot-001",
    "status": "DOWNLOADING",
    "status_code": 2,
    "current_version": "1.0.0",
    "target_version": "2.0.0",
    "progress": 45,
    "error_code": 0,
    "error_msg": "",
    "detail": {
      "downloaded_bytes": 707788800,
      "total_bytes": 1572864000,
      "speed_bps": 5242880
    }
  }
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| upgrade_id | string | 是 | 升级任务ID |
| device_id | string | 是 | 设备ID |
| status | string | 是 | 当前状态枚举值 |
| status_code | int | 是 | 状态码 |
| current_version | string | 是 | 当前固件版本 |
| target_version | string | 是 | 目标固件版本 |
| progress | int | 是 | 进度百分比（0-100） |
| error_code | int | 否 | 错误码，0表示无错误 |
| error_msg | string | 否 | 错误描述 |
| detail | object | 否 | 状态详情 |

#### 2.3.5 进度上报（progress）

```json
{
  "msg_id": "c3d4e5f6-a7b8-9012-cdef-234567890123",
  "msg_type": "progress_report",
  "timestamp": 1690000002000,
  "version": "1.0",
  "data": {
    "upgrade_id": "upg-20260719-001",
    "stage": "DOWNLOADING",
    "progress": 45,
    "stage_progress": 90,
    "detail": {
      "downloaded_bytes": 707788800,
      "total_bytes": 1572864000,
      "speed_bps": 5242880,
      "eta_seconds": 165
    }
  }
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| upgrade_id | string | 是 | 升级任务ID |
| stage | string | 是 | 当前阶段 |
| progress | int | 是 | 总体进度（0-100） |
| stage_progress | int | 是 | 当前阶段进度（0-100） |
| detail | object | 否 | 进度详情 |

阶段定义：
| 阶段 | 说明 | 总体进度范围 |
|------|------|-------------|
| PRECHECK | 预检 | 0-5% |
| DOWNLOADING | 下载固件 | 5-50% |
| VERIFYING | 校验固件 | 50-55% |
| FLASHING | 写入固件 | 55-90% |
| ACTIVATING | 激活新固件 | 90-95% |
| COMMITTING | 确认升级 | 95-100% |

#### 2.3.6 事件上报（event）

```json
{
  "msg_id": "d4e5f6a7-b8c9-0123-defa-345678901234",
  "msg_type": "event_report",
  "timestamp": 1690000003000,
  "version": "1.0",
  "data": {
    "upgrade_id": "upg-20260719-001",
    "event_type": "SECURITY",
    "event_name": "SIGNATURE_VERIFY_FAIL",
    "event_level": "CRITICAL",
    "detail": {
      "expected_hash": "e3b0c442...",
      "actual_hash": "a1b2c3d4...",
      "file": "firmware.img"
    }
  }
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| upgrade_id | string | 是 | 升级任务ID |
| event_type | string | 是 | 事件类型 |
| event_name | string | 是 | 事件名称 |
| event_level | string | 是 | 事件级别 |
| detail | object | 否 | 事件详情 |

事件类型：
| 类型 | 说明 |
|------|------|
| SECURITY | 安全事件 |
| NETWORK | 网络事件 |
| STORAGE | 存储事件 |
| SYSTEM | 系统事件 |
| MCU | MCU事件 |

事件级别：
| 级别 | 说明 |
|------|------|
| CRITICAL | 严重，需立即处理 |
| ERROR | 错误，升级中止 |
| WARNING | 警告，可继续但需关注 |
| INFO | 信息，正常记录 |

#### 2.3.7 版本上报（version）

```json
{
  "msg_id": "e5f6a7b8-c9d0-1234-efab-456789012345",
  "msg_type": "version_report",
  "timestamp": 1690000000000,
  "version": "1.0",
  "data": {
    "device_id": "robot-001",
    "linux_version": "2.0.0",
    "uboot_version": "2024.01",
    "kernel_version": "5.15.120",
    "mcu_versions": [
      {"mcu_id": "power_mcu", "version": "1.2.0"},
      {"mcu_id": "sensor_mcu", "version": "2.0.1"},
      {"mcu_id": "motor_mcu", "version": "2.1.0"},
      {"mcu_id": "hand_mcu", "version": "1.0.5"}
    ],
    "hardware_revision": "rev-c",
    "boot_partition": "A",
    "last_upgrade_time": 1689000000,
    "last_upgrade_status": "SUCCESS"
  }
}
```

#### 2.3.8 心跳（heartbeat）

```json
{
  "msg_id": "f6a7b8c9-d0e1-2345-fabc-567890123456",
  "msg_type": "heartbeat",
  "timestamp": 1690000000000,
  "version": "1.0",
  "data": {
    "device_id": "robot-001",
    "status": "IDLE",
    "uptime": 86400,
    "cpu_usage": 35,
    "memory_usage": 60,
    "disk_usage": 45,
    "battery_level": 85,
    "charging": true
  }
}
```

#### 2.3.9 取消升级（cancel）

```json
{
  "msg_id": "a7b8c9d0-e1f2-3456-abcd-678901234567",
  "msg_type": "cancel_upgrade",
  "timestamp": 1690000004000,
  "version": "1.0",
  "data": {
    "upgrade_id": "upg-20260719-001",
    "reason": "VERSION_BUG_FOUND"
  }
}
```

#### 2.3.10 强制回滚（rollback）

```json
{
  "msg_id": "b8c9d0e1-f2a3-4567-bcde-789012345678",
  "msg_type": "force_rollback",
  "timestamp": 1690000005000,
  "version": "1.0",
  "data": {
    "upgrade_id": "upg-20260719-001",
    "target_version": "1.0.0",
    "reason": "CRITICAL_BUG_DETECTED"
  }
}
```

---

## 3. HTTPS下载协议

### 3.1 下载请求

```
GET /firmware/v2.0.0.img HTTP/1.1
Host: ota-cdn.example.com
Authorization: Bearer {download_token}
Range: bytes=0-1048575
Accept: application/octet-stream
X-Device-ID: robot-001
X-Firmware-Version: 2.0.0
```

| 请求头 | 必填 | 说明 |
|--------|------|------|
| Authorization | 是 | 下载令牌，Bearer格式 |
| Range | 否 | 断点续传范围，格式：bytes=start-end |
| X-Device-ID | 是 | 设备唯一标识 |
| X-Firmware-Version | 是 | 请求的固件版本 |

### 3.2 下载响应

```
HTTP/1.1 206 Partial Content
Content-Type: application/octet-stream
Content-Length: 1048576
Content-Range: bytes 0-1048575/1572864000
Content-SHA256: e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
ETag: "e3b0c44298fc1c149afbf4c8996fb924"
Last-Modified: Wed, 19 Jul 2026 10:00:00 GMT
Cache-Control: public, max-age=86400
X-Download-Token-Expires: 1690086400

[固件二进制数据]
```

| 响应头 | 必填 | 说明 |
|--------|------|------|
| Content-Range | 是 | 返回的数据范围 |
| Content-SHA256 | 是 | 完整文件的SHA256哈希 |
| ETag | 是 | 文件版本标识 |
| X-Download-Token-Expires | 是 | 下载令牌过期时间戳 |

### 3.3 下载令牌

```
令牌获取流程：
1. 设备通过MQTT收到升级指令（含download_token）
2. 使用download_token作为Bearer Token发起HTTPS下载
3. 令牌有效期：24小时
4. 令牌过期后需通过MQTT重新获取

令牌格式（JWT）：
{
  "header": {
    "alg": "ES256",
    "typ": "JWT"
  },
  "payload": {
    "sub": "robot-001",
    "firmware": "v2.0.0",
    "exp": 1690086400,
    "iat": 1690000000,
    "jti": "unique-token-id"
  },
  "signature": "..."
}
```

### 3.4 断点续传

```
初始请求：
GET /firmware/v2.0.0.img HTTP/1.1
Range: bytes=0-

中断后恢复：
GET /firmware/v2.0.0.img HTTP/1.1
Range: bytes=707788800-

服务器响应：
HTTP/1.1 206 Partial Content
Content-Range: bytes 707788800-1572863999/1572864000
```

---

## 4. 交互时序

### 4.1 正常升级流程

```
设备端                         云端
  │                             │
  │──── MQTT CONNECT ──────────→│
  │←─── MQTT CONNACK ───────────│
  │                             │
  │──── version_report ────────→│
  │                             │
  │←─── upgrade_cmd ────────────│  (MQTT)
  │                             │
  │──── status: PRECHECK ──────→│
  │                             │
  │──── status: DOWNLOADING ───→│
  │                             │
  │──── HTTPS GET (Range) ─────→│  (HTTPS)
  │←─── 206 Partial Content ────│
  │──── progress: 25% ─────────→│
  │──── HTTPS GET (Range续传) ─→│
  │←─── 206 Partial Content ────│
  │──── progress: 50% ─────────→│
  │                             │
  │──── status: VERIFYING ─────→│
  │──── status: FLASHING ──────→│
  │──── progress: 75% ─────────→│
  │──── status: ACTIVATING ────→│
  │                             │
  │  [设备重启]                  │
  │                             │
  │──── MQTT CONNECT ──────────→│
  │──── status: COMMITTING ────→│
  │──── status: IDLE ──────────→│
  │──── version_report ────────→│
```

### 4.2 异常回滚流程

```
设备端                         云端
  │                             │
  │  [升级后启动]                 │
  │──── status: COMMITTING ────→│
  │                             │
  │  [健康检查失败]               │
  │──── event: HEALTH_FAIL ────→│
  │──── status: ROLLBACK ──────→│
  │                             │
  │  [设备重启至旧版本]            │
  │                             │
  │──── MQTT CONNECT ──────────→│
  │──── status: IDLE ──────────→│
  │──── event: ROLLBACK_DONE ──→│
```

### 4.3 取消升级流程

```
设备端                         云端
  │                             │
  │  [正在下载中]                 │
  │←─── cancel_upgrade ─────────│  (MQTT)
  │                             │
  │  [停止下载，清理临时文件]       │
  │──── status: IDLE ──────────→│
  │──── event: UPGRADE_CANCEL ─→│
```

---

## 5. 错误处理

### 5.1 MQTT错误处理

| 场景 | 处理方式 |
|------|---------|
| 连接断开 | 指数退避重连，初始1秒，最大60秒 |
| 消息发送失败 | QoS 1自动重试，3次失败后标记离线 |
| 消息格式错误 | 记录错误日志，忽略该消息 |
| Topic未授权 | 断开连接，检查证书权限 |

### 5.2 HTTPS错误处理

| HTTP状态码 | 处理方式 |
|-----------|---------|
| 401 Unauthorized | 令牌过期，通过MQTT重新获取 |
| 403 Forbidden | 设备无权限，上报事件 |
| 404 Not Found | 固件不存在，上报事件 |
| 416 Range Not Satisfiable | Range请求无效，重新从0开始下载 |
| 429 Too Many Requests | 限流，等待Retry-After时间后重试 |
| 5xx Server Error | 指数退避重试，最多3次 |

### 5.3 超时配置

| 操作 | 超时时间 | 重试策略 |
|------|---------|---------|
| MQTT连接 | 30秒 | 重试3次 |
| MQTT消息响应 | 60秒 | 重试1次 |
| HTTPS连接 | 15秒 | 重试3次 |
| HTTPS下载空闲 | 30秒 | 断开重连 |
| 下载令牌刷新 | 过期前1小时 | 自动刷新 |

---

## 6. 协议版本兼容

### 6.1 版本协商

```
设备端上报支持的协议版本：
- 在MQTT CONNECT报文的User Property中携带
- 格式：protocol_version=1.0,1.1

云端选择兼容版本：
- 优先选择最高共同支持的版本
- 在CONNACK中确认使用的版本
```

### 6.2 向后兼容原则

- 新增字段必须设置默认值，老版本设备忽略未知字段
- 废弃字段保留至少2个大版本周期
- 消息类型不可删除，废弃时标记为deprecated
- 枚举值只增不减，废弃值保留但标记

---

## 7. 安全约束

### 7.1 消息安全

| 约束 | 说明 |
|------|------|
| 消息完整性 | 所有消息必须包含timestamp，防重放攻击 |
| 消息大小 | 单条MQTT消息不超过256KB |
| 敏感信息 | 不在日志中打印Token、签名等敏感数据 |
| 频率限制 | 单设备上报频率不超过10条/秒 |

### 7.2 下载安全

| 约束 | 说明 |
|------|------|
| 下载地址 | 仅允许HTTPS，拒绝HTTP |
| 证书校验 | 严格校验服务器证书链 |
| 域名白名单 | 仅允许下载白名单域名下的固件 |
| 文件校验 | 下载完成后必须校验SHA256 |
