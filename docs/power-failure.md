# OTA断电保护设计文档

## 1. 概述

### 1.1 文档目的
定义机器人端**OTA升级过程中断电保护**的完整设计方案，确保设备在任何断电场景下均能恢复至可用状态，避免变砖风险。

### 1.2 设计目标
- 零变砖风险：任何时刻断电，设备均能恢复至可用状态
- 状态可恢复：断电后重新上电，能准确识别当前状态并继续或回滚
- 数据完整性：Flash写入过程中断电，不产生损坏的分区数据
- 原子性操作：关键状态切换必须原子化完成
- 快速恢复：断电恢复时间不超过30秒

### 1.3 断电场景分类

| 场景 | 断电时机 | 风险等级 | 恢复策略 |
|------|---------|---------|---------|
| 场景A | 固件下载中 | 低 | 断点续传，重新下载 |
| 场景B | 固件校验中 | 低 | 重新校验 |
| 场景C | Flash写入中 | 高 | 标记分区损坏，重新写入 |
| 场景D | 分区切换前 | 中 | 继续切换流程 |
| 场景E | 分区切换中 | 高 | 回退至原分区 |
| 场景F | 新固件启动中 | 高 | 回退至原分区 |
| 场景G | 升级确认前 | 中 | 重新确认或回退 |

---

## 2. 系统架构

### 2.1 双分区架构

```
┌─────────────────────────────────────────────────┐
│                  Flash存储布局                    │
├─────────────────────────────────────────────────┤
│  U-Boot A (4MB)    │  U-Boot B (4MB)            │
├─────────────────────────────────────────────────┤
│  Kernel A (16MB)   │  Kernel B (16MB)           │
├─────────────────────────────────────────────────┤
│  Rootfs A (1.5GB)  │  Rootfs B (1.5GB)          │
├─────────────────────────────────────────────────┤
│  数据分区 (512MB)   │  配置分区 (128MB)           │
├─────────────────────────────────────────────────┤
│  RPMB (512KB)     │  U-Boot环境变量 (128KB)      │
└─────────────────────────────────────────────────┘
```

### 2.2 状态存储架构

```
┌──────────────────────────────────────────────┐
│              状态存储层次结构                   │
├──────────────────────────────────────────────┤
│  RPMB（防篡改安全存储）                         │
│  ├── boot_partition：当前启动分区              │
│  ├── boot_retry_count：启动重试计数             │
│  ├── boot_commit：升级确认标记                  │
│  ├── rollback_version：最低允许版本             │
│  └── upgrade_state：升级状态机状态              │
├───────────────────────────────────────────── ┤
│  U-Boot环境变量（配置分区）                      │
│  ├── upgrade_active：是否有升级进行中            │
│  ├── target_partition：目标分区                 │
│  ├── target_version：目标版本                   │
│  └── upgrade_stage：升级阶段                    │
├─────────────────────────────────────────────┤
│  文件系统标记（各分区内）                        │
│  ├── /etc/ota/upgrade.lock：升级锁文件          │
│  ├── /etc/ota/upgrade.status：升级状态文件      │
│  └── /etc/ota/version：当前版本文件             │
└──────────────────────────────────────────────┘
```

---

## 3. 升级状态机

### 3.1 状态定义

```
                    ┌─────────┐
                    │  IDLE   │
                    └────┬────┘
                         │ 收到升级指令
                         ▼
                    ┌─────────┐
                    │PRECHECK │
                    └────┬────┘
                         │ 预检通过
                         ▼
                    ┌─────────┐
                    │DOWNLOAD │◄──── 断点续传
                    └────┬────┘
                         │ 下载完成
                         ▼
                    ┌─────────┐
                    │ VERIFY  │
                    └────┬────┘
                         │ 校验通过
                         ▼
                    ┌─────────┐
                    │ FLASH   │◄──── 重新写入
                    └────┬────┘
                         │ 写入完成
                         ▼
                    ┌─────────┐
                    │ SWITCH  │
                    └────┬────┘
                         │ 切换成功
                         ▼
                    ┌─────────┐
                    │ REBOOT  │
                    └────┬────┘
                         │ 启动新固件
                         ▼
                    ┌─────────┐
                    │ COMMIT  │──── 失败 ────→ 回滚
                    └────┬────┘
                         │ 确认成功
                         ▼
                    ┌─────────┐
                    │  IDLE   │
                    └─────────┘
```

### 3.2 状态持久化

```c
// 升级状态枚举
typedef enum {
    OTA_STATE_IDLE = 0,        // 空闲
    OTA_STATE_PRECHECK,        // 预检中
    OTA_STATE_DOWNLOADING,     // 下载中
    OTA_STATE_DOWNLOADED,      // 下载完成
    OTA_STATE_VERIFYING,       // 校验中
    OTA_STATE_VERIFIED,        // 校验通过
    OTA_STATE_FLASHING,        // 写入中
    OTA_STATE_FLASHED,         // 写入完成
    OTA_STATE_SWITCHING,       // 分区切换中
    OTA_STATE_SWITCHED,        // 切换完成
    OTA_STATE_REBOOTING,       // 重启中
    OTA_STATE_COMMITTING,      // 确认中
    OTA_STATE_ROLLBACK,        // 回滚中
    OTA_STATE_FAILED           // 升级失败
} ota_state_t;

// 状态存储结构
typedef struct {
    ota_state_t state;         // 当前状态
    char target_version;   // 目标版本
    char target_partition;     // 目标分区 'A'/'B'
    char source_partition;     // 源分区
    uint32_t download_offset;  // 下载偏移量
    uint32_t flash_offset;     // 写入偏移量
    uint32_t retry_count;      // 重试次数
    uint32_t state_timestamp;  // 状态变更时间戳
    uint8_t checksum;      // 结构体校验和
} ota_persistent_state_t;
```

### 3.3 状态写入策略

```
状态持久化原则：
1. 状态变更前先写入RPMB
2. 写入完成后验证校验和
3. 校验通过后才执行实际操作
4. 实际操作完成后更新状态

写入顺序：
┌──────────────┐
│ 1. 写入RPMB   │ ← 先持久化状态
├──────────────┤
│ 2. 验证写入   │ ← 读回校验
├──────────────┤
│ 3. 执行操作   │ ← 实际升级操作
├──────────────┤
│ 4. 更新状态   │ ← 操作完成后更新
└──────────────┘
```

---

## 4. 各阶段断电保护

### 4.1 下载阶段（DOWNLOAD）

```
断电场景：下载固件过程中断电

恢复流程：
1. 上电后读取RPMB状态：OTA_STATE_DOWNLOADING
2. 检查目标分区状态
3. 读取download_offset（已下载字节数）
4. 向云端发起Range请求，从download_offset继续下载
5. 每下载1MB更新一次download_offset到RPMB

保护机制：
- 下载偏移量持久化到RPMB
- 支持HTTP Range断点续传
- 下载临时文件使用原子写入
- 下载完成后校验SHA256

代码示例：
```c
int ota_download_resume(void) {
    ota_persistent_state_t state;
    
    // 读取持久化状态
    if (rpmb_read(OTA_STATE_ADDR, &state, sizeof(state)) != 0) {
        return -1;
    }
    
    if (state.state != OTA_STATE_DOWNLOADING) {
        return -1;
    }
    
    // 从断点继续下载
    uint32_t offset = state.download_offset;
    return http_download_range(FIRMWARE_URL, offset, &state);
}
```

### 4.2 校验阶段（VERIFY）

```
断电场景：固件校验过程中断电

恢复流程：
1. 上电后读取RPMB状态：OTA_STATE_VERIFYING
2. 检查固件文件完整性
3. 重新计算SHA256并与云端签名比对
4. 校验通过 → 进入FLASH状态
5. 校验失败 → 删除固件，回到IDLE

保护机制：
- 校验是幂等操作，可安全重试
- 校验失败自动清理，不留残留数据
- 校验结果写入RPMB后才进入下一阶段
```

### 4.3 写入阶段（FLASH）

```
断电场景：固件写入Flash过程中断电

恢复流程：
1. 上电后读取RPMB状态：OTA_STATE_FLASHING
2. 检查目标分区完整性
3. 分区损坏 → 擦除目标分区，从头写入
4. 分区完整 → 验证已写入数据，从断点继续

保护机制：
- 写入前标记目标分区为"写入中"
- 使用原子写入，每块写入后更新flash_offset
- 写入完成后验证分区完整性
- 分区标记存储在RPMB中

分区状态标记：
```c
typedef enum {
    PART_STATE_EMPTY = 0,      // 空分区
    PART_STATE_WRITING,        // 写入中
    PART_STATE_WRITTEN,        // 写入完成
    PART_STATE_VERIFIED,       // 验证通过
    PART_STATE_ACTIVE,         // 当前运行
    PART_STATE_DAMAGED         // 已损坏
} partition_state_t;
```

### 4.4 分区切换阶段（SWITCH）

```
断电场景：分区切换过程中断电

这是最危险的断电场景，需要特殊保护：

恢复流程：
1. 上电后读取RPMB状态：OTA_STATE_SWITCHING
2. 检查boot_partition值
3. 如果boot_partition已更新 → 尝试启动新分区
4. 如果boot_partition未更新 → 继续启动原分区
5. 无论哪种情况，设备都能正常启动

保护机制：
- 分区切换使用原子操作
- RPMB写入是原子性的，要么成功要么失败
- 切换前确保目标分区已验证通过
- 切换后设置boot_retry_count

原子切换实现：
```c
int ota_switch_partition(void) {
    ota_persistent_state_t state;
    uint8_t target_partition;
    
    // 读取当前状态
    rpmb_read(OTA_STATE_ADDR, &state, sizeof(state));
    target_partition = state.target_partition;
    
    // 原子切换：一次性写入所有关键信息
    struct {
        uint8_t boot_partition;
        uint8_t boot_retry_count;
        uint8_t boot_commit;
    } boot_config = {
        .boot_partition = target_partition,
        .boot_retry_count = 3,      // 允许3次启动重试
        .boot_commit = 0            // 未确认
    };
    
    // RPMB原子写入
    return rpmb_atomic_write(BOOT_CONFIG_ADDR, &boot_config, sizeof(boot_config));
}
```

### 4.5 启动阶段（REBOOT）

```
断电场景：重启后新固件启动过程中断电

恢复流程：
1. U-Boot读取RPMB中的boot_config
2. 检查boot_commit标记
3. boot_commit == 0（未确认）→ 检查boot_retry_count
4. boot_retry_count > 0 → 减1，尝试启动
5. boot_retry_count == 0 → 回退至原分区

U-Boot启动逻辑：
```c
int boot_select_partition(void) {
    boot_config_t config;
    
    rpmb_read(BOOT_CONFIG_ADDR, &config, sizeof(config));
    
    if (config.boot_commit == 1) {
        // 已确认，正常启动
        return config.boot_partition;
    }
    
    if (config.boot_retry_count > 0) {
        // 还有重试次数，尝试启动新分区
        config.boot_retry_count--;
        rpmb_write(BOOT_CONFIG_ADDR, &config, sizeof(config));
        return config.boot_partition;
    }
    
    // 重试次数耗尽，回退
    ota_rollback();
    return config.boot_partition == 'A' ? 'B' : 'A';
}
```

### 4.6 确认阶段（COMMIT）

```
断电场景：升级确认过程中断电

恢复流程：
1. 新固件启动后，OTA守护进程检查boot_commit
2. boot_commit == 0 → 执行健康检查
3. 健康检查通过 → 设置boot_commit = 1
4. 健康检查失败 → 触发回滚

健康检查项：
- 关键进程运行状态
- 网络连接状态
- 传感器数据读取
- 电机控制响应
- 电池管理状态

确认流程：
```c
int ota_commit_upgrade(void) {
    boot_config_t config;
    
    // 执行健康检查
    if (health_check() != 0) {
        // 健康检查失败，触发回滚
        ota_trigger_rollback();
        return -1;
    }
    
    // 健康检查通过，确认升级
    rpmb_read(BOOT_CONFIG_ADDR, &config, sizeof(config));
    config.boot_commit = 1;
    config.boot_retry_count = 0;
    rpmb_write(BOOT_CONFIG_ADDR, &config, sizeof(config));
    
    // 更新状态为IDLE
    ota_set_state(OTA_STATE_IDLE);
    
    return 0;
}
```

---

## 5. 回滚机制

### 5.1 自动回滚触发条件

| 触发条件 | 检测方式 | 回滚动作 |
|---------|---------|---------|
| 启动重试耗尽 | U-Boot检测boot_retry_count=0 | 切换至原分区启动 |
| 健康检查失败 | OTA守护进程检测 | 设置回滚标记并重启 |
| 看门狗超时 | 硬件看门狗触发 | 系统重启，U-Boot检测回滚 |
| 内核Panic | Kernel检测 | 重启后U-Boot检测 |
| 用户手动触发 | MQTT指令 | 设置回滚标记并重启 |

### 5.2 回滚流程

```
回滚执行流程：

1. 设置回滚标记
   └── rpmb_write(ROLLBACK_FLAG, 1)

2. 系统重启
   └── reboot(RB_AUTOBOOT)

3. U-Boot检测回滚标记
   ├── 回滚标记=1 → 执行回滚
   └── 回滚标记=0 → 正常启动

4. 执行回滚
   ├── 切换boot_partition至原分区
   ├── 清除boot_retry_count
   ├── 设置boot_commit=1（原分区已确认）
   └── 启动原分区

5. 原分区启动后
   ├── 上报回滚事件
   ├── 清理升级临时文件
   └── 恢复IDLE状态
```

### 5.3 回滚安全保证

```
回滚原子性保证：
- 回滚标记写入RPMB是原子操作
- 分区切换在U-Boot中完成，不受系统状态影响
- 原分区始终保持完整，不会被升级过程破坏
- 回滚后原分区直接启动，无需额外验证

回滚后状态清理：
```c
int ota_rollback_cleanup(void) {
    // 清理目标分区
    erase_partition(TARGET_PARTITION);
    
    // 清理升级临时文件
    unlink("/data/ota/firmware.img");
    unlink("/data/ota/firmware.sig");
    unlink("/data/ota/delta.zck");
    
    // 清理状态文件
    unlink("/etc/ota/upgrade.lock");
    unlink("/etc/ota/upgrade.status");
    
    // 重置RPMB状态
    ota_set_state(OTA_STATE_IDLE);
    
    // 上报回滚事件
    ota_report_event(EVENT_ROLLBACK_COMPLETE);
    
    return 0;
}
```

---

## 6. 看门狗保护

### 6.1 硬件看门狗配置

```
看门狗参数：
- 芯片：SoC内置看门狗定时器
- 超时时间：60秒
- 喂狗间隔：30秒
- 启动阶段超时：120秒（给予启动足够时间）

看门狗在升级各阶段的行为：
| 阶段 | 超时时间 | 喂狗策略 |
|------|---------|---------|
| 下载 | 120秒 | 每下载10MB喂狗 |
| 校验 | 120秒 | 每校验100MB喂狗 |
| 写入 | 120秒 | 每写入50MB喂狗 |
| 切换 | 30秒  | 操作前喂狗 |
| 重启 | 120秒 | U-Boot阶段不喂狗 |
```

### 6.2 看门狗与断电保护协同

```
看门狗作为断电保护的最后防线：

场景：系统在升级过程中卡死（非断电）
1. 看门狗超时触发系统复位
2. 复位后U-Boot读取RPMB状态
3. 根据状态机执行恢复流程
4. 效果等同于断电恢复

协同机制：
- 看门狗复位模拟断电场景
- 状态机保证恢复的一致性
- 双重保护确保设备安全
```

---

## 7. 文件系统保护

### 7.1 UBIFS原子写入

```
UBIFS特性利用：
- 写时复制（Copy-on-Write）
- 原子性LEB更新
- 日志式写入

升级文件写入策略：
1. 固件写入使用O_SYNC标志
2. 每写入一个LEB后调用fsync()
3. 写入完成后验证文件完整性
4. 使用rename()原子替换文件
```

### 7.2 临时文件管理

```
临时文件目录结构：
/data/ota/
├── firmware.img        下载的固件文件
├── firmware.sig        签名文件
├── firmware.sha256     哈希文件
├── delta.zck           差分包文件
├── download.progress   下载进度
└── flash.progress      写入进度

清理策略：
- 升级成功后清理所有临时文件
- 升级失败后清理目标分区临时文件
- 回滚后清理所有临时文件
- 磁盘空间不足时优先清理旧临时文件
```

---

## 8. 测试验证

### 8.1 断电测试用例

| 测试用例 | 断电时机 | 预期结果 |
|---------|---------|---------|
| TC-01 | 下载进度10% | 恢复后从10%继续下载 |
| TC-02 | 下载进度90% | 恢复后从90%继续下载 |
| TC-03 | 校验开始    | 重新校验，结果一致 |
| TC-04 | Flash写入5% | 擦除分区，重新写入 |
| TC-05 | Flash写入95% | 擦除分区，重新写入 |
| TC-06 | 分区切换前 | 继续切换流程 |
| TC-07 | 分区切换中 | 回退至原分区 |
| TC-08 | 新固件首次启动 | 回退至原分区 |
| TC-09 | 健康检查中 | 回退至原分区 |
| TC-10 | 升级确认前 | 重新确认或回退 |

### 8.2 自动化测试方案

```
测试环境：
- 程控电源：可编程控制断电时序
- 测试脚本：自动化执行升级并随机断电
- 日志采集：记录每次断电恢复过程

测试流程：
1. 设备上电，建立MQTT连接
2. 下发升级指令
3. 随机时间点切断电源
4. 等待1-5秒后恢复供电
5. 检查设备是否正常启动
6. 检查升级状态是否正确恢复
7. 重复1000次，统计成功率

通过标准：成功率 ≥ 99.9%
```

---

## 9. 监控与告警

### 9.1 断电恢复监控

```json
{
  "event_type": "POWER_RECOVERY",
  "event_name": "POWER_LOSS_DETECTED",
  "event_level": "WARNING",
  "detail": {
    "last_state": "OTA_STATE_FLASHING",
    "recovery_action": "ERASE_AND_REFLASH",
    "recovery_time_ms": 15000,
    "recovery_result": "SUCCESS"
  }
}
```

### 9.2 告警规则

| 告警 | 触发条件 | 级别 |
|------|---------|------|
| 频繁断电 | 1小时内断电恢复超过3次 | 高危 |
| 回滚发生 | 自动回滚触发 | 中危 |
| 恢复失败 | 断电恢复失败 | 严重 |
| 分区损坏 | 检测到分区数据损坏 | 高危 |

---

## 10. 设计约束与限制

### 10.1 硬件约束

| 约束项 | 要求 |
|--------|------|
| RPMB存储 | 至少512KB，支持原子写入 |
| 看门狗 | 硬件看门狗，不可软件禁用 |
| 电源检测 | 支持断电检测中断 |
| Flash | 双分区架构，至少保留一个完整备份 |

### 10.2 软件约束

| 约束项 | 要求 |
|--------|------|
| 状态持久化 | 状态变更前必须写入RPMB |
| 原子操作 | 分区切换必须原子完成 |
| 启动保护 | U-Boot必须支持启动重试和回退 |
| 健康检查 | 新固件必须通过健康检查才确认 |