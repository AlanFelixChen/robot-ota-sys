# MCU固件升级技术规范

## 1. 文档概述

### 1.1 文档目的
定义MCU（微控制器）固件升级的完整技术规范，包括**升级架构、通信协议、存储分区、安全机制、异常处理和实现要求**，确保MCU固件升级的可靠性、安全性和可维护性。

### 1.2 适用范围
本规范适用于所有基于MCU的嵌入式设备固件升级功能的设计、开发和测试。

### 1.3 术语定义

| 术语 | 定义 |
|------|------|
| MCU | Microcontroller Unit，微控制器 |
| Bootloader | 引导加载程序，负责固件升级和启动管理 |
| Application | 应用程序，设备正常运行的功能固件 |
| OTA | Over-The-Air，空中升级 |
| DFU | Device Firmware Update，设备固件升级 |
| CRC | Cyclic Redundancy Check，循环冗余校验 |
| SHA256 | Secure Hash Algorithm 256-bit，安全哈希算法 |
| AES | Advanced Encryption Standard，高级加密标准 |
| Flash | 闪存，用于存储固件的非易失性存储器 |
| Partition | 分区，Flash存储的逻辑划分 |

---

## 2. 系统架构

### 2.1 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                        MCU Flash Layout                      │
├───────────┬───────────┬───────────┬───────────┬─────────────┤
│ Bootloader│  App A    │  App B    │  Data     │  Config     │
│  (32KB)   │  (256KB)  │  (256KB)  │  (128KB)  │  (16KB)     │
├───────────┴───────────┴───────────┴───────────┴─────────────┤
│                    Communication Module                       │
│              (UART / SPI / I2C / CAN / USB)                  │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 双分区架构（A/B分区）

```
┌──────────────────────────────────────────────────────────────┐
│  Active Partition (运行分区)    │  Backup Partition (备份分区) │
│  ┌──────────────────────────┐  │  ┌──────────────────────────┐│
│  │  Vector Table            │  │  │  Vector Table            ││
│  │  Application Code        │  │  │  Application Code        ││
│  │  Application Data        │  │  │  Application Data        ││
│  │  Version Info            │  │  │  Version Info            ││
│  └──────────────────────────┘  │  └──────────────────────────┘│
└──────────────────────────────────────────────────────────────┘
```

### 2.3 内存映射

```c
/* MCU Flash Memory Map */
define FLASH_BASE_ADDRESS          0x08000000
define FLASH_TOTAL_SIZE            0x00080000  /* 512KB */

/* Bootloader Section */
define BOOTLOADER_ADDRESS          FLASH_BASE_ADDRESS
define BOOTLOADER_SIZE             0x00008000  /* 32KB */

/* Application A Section */
define APP_A_ADDRESS               (BOOTLOADER_ADDRESS + BOOTLOADER_SIZE)
define APP_A_SIZE                  0x00040000  /* 256KB */

/* Application B Section */
define APP_B_ADDRESS               (APP_A_ADDRESS + APP_A_SIZE)
define APP_B_SIZE                  0x00040000  /* 256KB */

/* Data Section */
define DATA_ADDRESS                (APP_B_ADDRESS + APP_B_SIZE)
define DATA_SIZE                   0x00020000  /* 128KB */

/* Configuration Section */
define CONFIG_ADDRESS              (DATA_ADDRESS + DATA_SIZE)
define CONFIG_SIZE                 0x00004000  /* 16KB */
```

---

## 3. 固件格式规范

### 3.1 固件包结构

```
┌─────────────────────────────────────────────────────────────┐
│                     Firmware Package                        │
├─────────────┬───────────────────────────────────────────────┤
│   Header    │                   Payload                     │
│  (256Byte)  │              (Variable Size)                  │
├─────────────┴───────────────────────────────────────────────┤
│                       Signature                             │
│                      (256Byte)                              │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 固件头定义

```c
/* Firmware Header Structure */
typedef struct __attribute__((packed)) {
    uint32_t magic_number;          /* 0x4F544146 ("OTAF") */
    uint32_t header_version;        /* Header format version */
    uint32_t firmware_size;         /* Total firmware size */
    uint32_t payload_size;          /* Payload size */
    uint32_t payload_offset;        /* Payload offset */
    uint8_t  firmware_version;      /* Version: major.minor.patch.build */
    uint32_t firmware_crc;          /* CRC32 of payload */
    uint8_t  device_type;           /* Target device type */
    uint8_t  hardware_rev;          /* Hardware revision */
    uint32_t timestamp;             /* Build timestamp */
    uint8_t  encryption_type;       /* 0=None, 1=AES-128, 2=AES-256 */
    uint8_t  compression_type;      /* 0=None, 1=ZLIB, 2=LZ4 */
    uint8_t  reserved;              /* Reserved for future use */
    uint8_t  header_crc;            /* CRC32 of header */
} firmware_header_t;

/* Firmware Signature Structure */
typedef struct __attribute__((packed)) {
    uint8_t  signature_type;        /* 0=RSA, 1=ECDSA */
    uint8_t  signature_size;        /* Signature size */
    uint8_t  signature_data;        /* Signature data */
} firmware_signature_t;
```

### 3.3 固件包常量定义

```c
/* Firmware Package Constants */
define FIRMWARE_MAGIC_NUMBER       0x4F544146
define FIRMWARE_HEADER_VERSION     0x00000001
define FIRMWARE_HEADER_SIZE        256
define FIRMWARE_SIGNATURE_SIZE     256
define FIRMWARE_MAX_SIZE           (256 * 1024)  /* 256KB */

/* Version Encoding */
define VERSION_ENCODE(major, minor, patch, build) \
    ((uint32_t)(((major) << 24) | ((minor) << 16) | ((patch) << 8) | (build)))

/* Encryption Types */
define ENCRYPTION_NONE             0x00
define ENCRYPTION_AES128           0x01
define ENCRYPTION_AES256           0x02

/* Compression Types */
define COMPRESSION_NONE            0x00
define COMPRESSION_ZLIB            0x01
define COMPRESSION_LZ4             0x02
```

---

## 4. Bootloader设计

### 4.1 Bootloader状态机

```
                    ┌─────────────┐
                    │   RESET     │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
              ┌─────│  CHECK      │─────┐
              │     │  CONDITION  │     │
              │     └──────┬──────┘     │
              │            │            │
     ┌────────▼───┐  ┌────▼─────┐  ┌───▼────────┐
     │  UPGRADE   │  │  VALIDATE│  │  NORMAL     │
     │  MODE      │  │  APP     │  │  BOOT       │
     └──────┬─────┘  └────┬─────┘  └───┬────────┘
            │             │            │
     ┌──────▼─────┐  ┌────▼─────┐      │
     │  RECEIVE   │  │  BOOT    │      │
     │  FIRMWARE  │  │  APP     │      │
     └──────┬─────┘  └──────────┘      │
            │                          │
     ┌──────▼─────┐                    │
     │  VERIFY    │                    │
     │  FIRMWARE  │                    │
     └──────┬─────┘                    │
            │                          │
     ┌──────▼─────┐                    │
     │  INSTALL   │                    │
     │  FIRMWARE  │                    │
     └──────┬─────┘                    │
            │                          │
     ┌──────▼─────┐                    │
     │  SET       │                    │
     │  ACTIVE    │                    │
     └──────┬─────┘                    │
            │                          │
            └──────────┬───────────────┘
                       │
                ┌──────▼──────┐
                │   REBOOT    │
                └─────────────┘
```

### 4.2 Bootloader核心代码

```c
/* Bootloader Main Flow */
void bootloader_main(void) {
    bootloader_init();
    
    /* Check boot reason */
    boot_reason_t reason = get_boot_reason();
    
    /* Check if upgrade mode requested */
    if (is_upgrade_requested() || reason == BOOT_UPGRADE_MODE) {
        enter_upgrade_mode();
        return;
    }
    
    /* Validate and boot application */
    partition_t active_partition = get_active_partition();
    
    if (validate_application(active_partition) == VALIDATION_OK) {
        boot_application(active_partition);
    } else {
        /* Try backup partition */
        partition_t backup_partition = get_backup_partition();
        
        if (validate_application(backup_partition) == VALIDATION_OK) {
            set_active_partition(backup_partition);
            boot_application(backup_partition);
        } else {
            /* No valid application, enter upgrade mode */
            enter_upgrade_mode();
        }
    }
}

/* Application Validation */
validation_result_t validate_application(partition_t partition) {
    uint32_t app_address = get_partition_address(partition);
    
    /* Check magic number */
    if (*(uint32_t *)app_address != APP_MAGIC_NUMBER) {
        return VALIDATION_INVALID_MAGIC;
    }
    
    /* Check CRC */
    uint32_t app_size = get_application_size(app_address);
    uint32_t calculated_crc = crc32_calculate(
        (uint8_t *)(app_address + APP_HEADER_SIZE),
        app_size - APP_HEADER_SIZE
    );
    uint32_t stored_crc = get_stored_crc(app_address);
    
    if (calculated_crc != stored_crc) {
        return VALIDATION_CRC_ERROR;
    }
    
    /* Check version */
    if (!is_version_valid(app_address)) {
        return VALIDATION_VERSION_ERROR;
    }
    
    return VALIDATION_OK;
}
```

### 4.3 升级模式实现

```c
/* Upgrade Mode Handler */
void enter_upgrade_mode(void) {
    upgrade_state_t state = UPGRADE_STATE_IDLE;
    firmware_packet_t packet;
    
    /* Initialize communication interface */
    comm_interface_init();
    
    while (1) {
        switch (state) {
            case UPGRADE_STATE_IDLE:
                /* Wait for upgrade start command */
                if (receive_upgrade_start(&packet)) {
                    state = UPGRADE_STATE_RECEIVING;
                    init_firmware_receive();
                }
                break;
                
            case UPGRADE_STATE_RECEIVING:
                /* Receive firmware data */
                if (receive_firmware_data(&packet)) {
                    if (write_firmware_data(&packet) != WRITE_OK) {
                        state = UPGRADE_STATE_ERROR;
                        send_error_response(ERROR_FLASH_WRITE);
                    }
                    
                    if (is_firmware_complete()) {
                        state = UPGRADE_STATE_VERIFYING;
                    }
                }
                break;
                
            case UPGRADE_STATE_VERIFYING:
                /* Verify received firmware */
                if (verify_firmware() == VERIFY_OK) {
                    state = UPGRADE_STATE_INSTALLING;
                    send_status_response(STATUS_VERIFY_OK);
                } else {
                    state = UPGRADE_STATE_ERROR;
                    send_error_response(ERROR_VERIFY_FAILED);
                }
                break;
                
            case UPGRADE_STATE_INSTALLING:
                /* Install firmware to backup partition */
                if (install_firmware() == INSTALL_OK) {
                    state = UPGRADE_STATE_COMPLETE;
                    set_boot_partition(BOOT_NEW_FIRMWARE);
                } else {
                    state = UPGRADE_STATE_ERROR;
                    send_error_response(ERROR_INSTALL_FAILED);
                }
                break;
                
            case UPGRADE_STATE_COMPLETE:
                /* Send success and reboot */
                send_status_response(STATUS_UPGRADE_COMPLETE);
                delay_ms(100);
                system_reset();
                break;
                
            case UPGRADE_STATE_ERROR:
                /* Handle error and wait for retry */
                handle_upgrade_error();
                state = UPGRADE_STATE_IDLE;
                break;
        }
    }
}
```

---

## 5. 通信协议

### 5.1 协议帧格式

```
┌──────────┬──────────┬──────────┬──────────┬──────────┬──────────┐
│   SOF    │  Length  │ Command  │  Data    │   CRC    │   EOF    │
│  (1Byte) │ (2Bytes) │ (1Byte)  │ (N Bytes)│ (2Bytes) │ (1Byte)  │
│   0xAA   │          │          │          │          │   0x55   │
└──────────┴──────────┴──────────┴──────────┴──────────┴──────────┘
```

### 5.2 协议定义

```c
/* Protocol Constants */
define PROTOCOL_SOF                0xAA
define PROTOCOL_EOF                0x55
define PROTOCOL_MAX_DATA_SIZE      1024
define PROTOCOL_HEADER_SIZE        5
define PROTOCOL_FOOTER_SIZE        3

/* Command Definitions */
typedef enum {
    CMD_UPGRADE_START       = 0x01,  /* Start upgrade */
    CMD_UPGRADE_DATA        = 0x02,  /* Firmware data */
    CMD_UPGRADE_END         = 0x03,  /* End upgrade */
    CMD_UPGRADE_ABORT       = 0x04,  /* Abort upgrade */
    CMD_STATUS_REQUEST      = 0x05,  /* Request status */
    CMD_STATUS_RESPONSE     = 0x06,  /* Status response */
    CMD_VERSION_REQUEST     = 0x07,  /* Request version */
    CMD_VERSION_RESPONSE    = 0x08,  /* Version response */
    CMD_RESET               = 0x09,  /* Reset device */
    CMD_ACK                 = 0x0A,  /* Acknowledge */
    CMD_NACK                = 0x0B,  /* Negative Acknowledge */
} protocol_command_t;

/* Status Codes */
typedef enum {
    STATUS_OK               = 0x00,  /* Operation successful */
    STATUS_BUSY             = 0x01,  /* Device busy */
    STATUS_ERROR            = 0x02,  /* General error */
    STATUS_INVALID_CMD      = 0x03,  /* Invalid command */
    STATUS_INVALID_PARAM    = 0x04,  /* Invalid parameter */
    STATUS_FLASH_ERROR      = 0x05,  /* Flash operation error */
    STATUS_CRC_ERROR        = 0x06,  /* CRC check error */
    STATUS_VERIFY_ERROR     = 0x07,  /* Verification error */
    STATUS_SPACE_ERROR      = 0x08,  /* Insufficient space */
    STATUS_VERSION_ERROR    = 0x09,  /* Version incompatible */
    STATUS_UPGRADE_DONE     = 0x0A,  /* Upgrade completed */
} protocol_status_t;

/* Protocol Frame Structure */
typedef struct __attribute__((packed)) {
    uint8_t  sof;                   /* Start of frame */
    uint16_t length;                /* Data length */
    uint8_t  command;               /* Command */
    uint8_t  data[PROTOCOL_MAX_DATA_SIZE]; /* Data */
    uint16_t crc;                   /* CRC16 */
    uint8_t  eof;                   /* End of frame */
} protocol_frame_t;
```

### 5.3 协议实现

```c
/* Frame Builder */
uint16_t build_frame(protocol_frame_t *frame, uint8_t cmd, 
                     uint8_t *data, uint16_t data_len) {
    frame->sof = PROTOCOL_SOF;
    frame->length = data_len;
    frame->command = cmd;
    
    if (data != NULL && data_len > 0) {
        memcpy(frame->data, data, data_len);
    }
    
    /* Calculate CRC over command and data */
    uint16_t crc = crc16_calculate((uint8_t *)&frame->command, 1);
    crc = crc16_update(crc, data, data_len);
    frame->crc = crc;
    
    frame->eof = PROTOCOL_EOF;
    
    return PROTOCOL_HEADER_SIZE + data_len + PROTOCOL_FOOTER_SIZE;
}

/* Frame Parser */
parse_result_t parse_frame(uint8_t *buffer, uint16_t buffer_len,
                           protocol_frame_t *frame) {
    if (buffer_len < PROTOCOL_HEADER_SIZE + PROTOCOL_FOOTER_SIZE) {
        return PARSE_INCOMPLETE;
    }
    
    if (buffer != PROTOCOL_SOF) {
        return PARSE_INVALID_SOF;
    }
    
    uint16_t data_len = (buffer << 8) | buffer;
    uint16_t total_len = PROTOCOL_HEADER_SIZE + data_len + PROTOCOL_FOOTER_SIZE;
    
    if (buffer_len < total_len) {
        return PARSE_INCOMPLETE;
    }
    
    if (buffer[total_len - 1] != PROTOCOL_EOF) {
        return PARSE_INVALID_EOF;
    }
    
    /* Verify CRC */
    uint16_t received_crc = (buffer[total_len - 3] << 8) | buffer[total_len - 2];
    uint16_t calculated_crc = crc16_calculate(&buffer, 1 + data_len);
    
    if (received_crc != calculated_crc) {
        return PARSE_CRC_ERROR;
    }
    
    /* Fill frame structure */
    frame->sof = buffer;
    frame->length = data_len;
    frame->command = buffer;
    memcpy(frame->data, &buffer, data_len);
    frame->crc = received_crc;
    frame->eof = buffer[total_len - 1];
    
    return PARSE_OK;
}
```

---

## 6. Flash操作

### 6.1 Flash驱动接口

```c
/* Flash Driver Interface */
typedef struct {
    /* Initialize flash driver */
    int (*init)(void);
    
    /* Erase flash sector */
    int (*erase)(uint32_t address, uint32_t size);
    
    /* Write data to flash */
    int (*write)(uint32_t address, uint8_t *data, uint32_t size);
    
    /* Read data from flash */
    int (*read)(uint32_t address, uint8_t *data, uint32_t size);
    
    /* Get sector size */
    uint32_t (*get_sector_size)(uint32_t address);
    
    /* Get flash information */
    void (*get_info)(flash_info_t *info);
} flash_driver_t;

/* Flash Operations */
typedef enum {
    FLASH_OK            = 0,
    FLASH_ERROR_ERASE   = -1,
    FLASH_ERROR_WRITE   = -2,
    FLASH_ERROR_READ    = -3,
    FLASH_ERROR_ADDRESS = -4,
    FLASH_ERROR_SIZE    = -5,
    FLASH_ERROR_BUSY    = -6,
    FLASH_ERROR_TIMEOUT = -7,
} flash_result_t;
```

### 6.2 Flash操作实现

```c
/* Flash Write with Verification */
flash_result_t flash_write_verified(uint32_t address, uint8_t *data, 
                                     uint32_t size) {
    flash_result_t result;
    uint8_t *verify_buffer;
    
    /* Allocate verification buffer */
    verify_buffer = malloc(size);
    if (verify_buffer == NULL) {
        return FLASH_ERROR_WRITE;
    }
    
    /* Erase sector if needed */
    if (is_sector_boundary(address)) {
        result = flash_driver.erase(address, size);
        if (result != FLASH_OK) {
            free(verify_buffer);
            return result;
        }
    }
    
    /* Write data */
    result = flash_driver.write(address, data, size);
    if (result != FLASH_OK) {
        free(verify_buffer);
        return result;
    }
    
    /* Read back and verify */
    result = flash_driver.read(address, verify_buffer, size);
    if (result != FLASH_OK) {
        free(verify_buffer);
        return FLASH_ERROR_READ;
    }
    
    if (memcmp(data, verify_buffer, size) != 0) {
        free(verify_buffer);
        return FLASH_ERROR_WRITE;
    }
    
    free(verify_buffer);
    return FLASH_OK;
}

/* Firmware Installation */
install_result_t install_firmware(partition_t target_partition) {
    uint32_t target_address = get_partition_address(target_partition);
    uint32_t firmware_size = get_received_firmware_size();
    uint32_t offset = 0;
    flash_result_t flash_result;
    
    /* Erase target partition */
    flash_result = flash_driver.erase(target_address, 
                                       get_partition_size(target_partition));
    if (flash_result != FLASH_OK) {
        return INSTALL_ERASE_FAILED;
    }
    
    /* Write firmware data */
    while (offset < firmware_size) {
        uint32_t chunk_size = min(FLASH_WRITE_CHUNK_SIZE, 
                                   firmware_size - offset);
        
        flash_result = flash_write_verified(
            target_address + offset,
            &received_firmware_data[offset],
            chunk_size
        );
        
        if (flash_result != FLASH_OK) {
            return INSTALL_WRITE_FAILED;
        }
        
        offset += chunk_size;
        
        /* Report progress */
        report_install_progress(offset * 100 / firmware_size);
    }
    
    /* Verify complete firmware */
    if (verify_installed_firmware(target_address, firmware_size) != VERIFY_OK) {
        return INSTALL_VERIFY_FAILED;
    }
    
    return INSTALL_OK;
}
```

---

## 7. 安全机制

### 7.1 安全架构

```
┌─────────────────────────────────────────────────────────────┐
│                      Security Layers                         │
├─────────────────────────────────────────────────────────────┤
│  1. Firmware Encryption (AES-256-GCM)                       │
│  2. Firmware Signing (ECDSA-P256)                           │
│  3. Secure Boot (Chain of Trust)                            │
│  4. Secure Storage (Encrypted Config)                       │
│  5. Anti-Rollback Protection                                │
│  6. Secure Communication (TLS/DTLS)                         │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 加密实现

```c
/* AES-256-GCM Encryption */
typedef struct {
    uint8_t key;        /* 256-bit key */
    uint8_t iv;         /* 96-bit initialization vector */
    uint8_t tag;        /* 128-bit authentication tag */
} aes_gcm_context_t;

/* Firmware Decryption */
decrypt_result_t decrypt_firmware(uint8_t *encrypted_data, 
                                   uint32_t encrypted_size,
                                   uint8_t *decrypted_data,
                                   uint32_t *decrypted_size) {
    aes_gcm_context_t ctx;
    decrypt_result_t result;
    
    /* Load device key from secure storage */
    if (load_device_key(ctx.key, sizeof(ctx.key)) != KEY_LOAD_OK) {
        return DECRYPT_KEY_ERROR;
    }
    
    /* Extract IV from encrypted data */
    memcpy(ctx.iv, encrypted_data, sizeof(ctx.iv));
    
    /* Extract authentication tag */
    uint32_t tag_offset = encrypted_size - sizeof(ctx.tag);
    memcpy(ctx.tag, encrypted_data + tag_offset, sizeof(ctx.tag));
    
    /* Decrypt firmware */
    uint32_t ciphertext_size = tag_offset - sizeof(ctx.iv);
    result = aes_gcm_decrypt(
        &ctx,
        encrypted_data + sizeof(ctx.iv),
        ciphertext_size,
        decrypted_data,
        decrypted_size
    );
    
    if (result != DECRYPT_OK) {
        return result;
    }
    
    /* Verify authentication tag */
    if (!verify_auth_tag(&ctx)) {
        return DECRYPT_AUTH_FAILED;
    }
    
    return DECRYPT_OK;
}
```

### 7.3 签名验证

```c
/* ECDSA Signature Verification */
typedef struct {
    uint8_t public_key;     /* Public key (x, y) */
    uint8_t signature;      /* Signature (r, s) */
    uint8_t hash;           /* SHA256 hash */
} ecdsa_verify_context_t;

/* Verify Firmware Signature */
verify_result_t verify_firmware_signature(firmware_header_t *header,
                                           firmware_signature_t *signature) {
    ecdsa_verify_context_t ctx;
    
    /* Load public key from secure storage */
    if (load_public_key(ctx.public_key, sizeof(ctx.public_key)) != KEY_LOAD_OK) {
        return VERIFY_KEY_ERROR;
    }
    
    /* Calculate firmware hash */
    sha256_calculate((uint8_t *)header + sizeof(firmware_header_t),
                     header->payload_size,
                     ctx.hash);
    
    /* Extract signature */
    memcpy(ctx.signature, signature->signature_data, sizeof(ctx.signature));
    
    /* Verify signature */
    if (!ecdsa_verify(&ctx)) {
        return VERIFY_SIGNATURE_INVALID;
    }
    
    return VERIFY_OK;
}
```

### 7.4 安全启动

```c
/* Secure Boot Chain */
typedef enum {
    BOOT_STAGE_ROM      = 0,    /* ROM Bootloader */
    BOOT_STAGE_BOOT     = 1,    /* Main Bootloader */
    BOOT_STAGE_APP      = 2,    /* Application */
} boot_stage_t;

/* Secure Boot Implementation */
secure_boot_result_t secure_boot_sequence(void) {
    secure_boot_result_t result;
    
    /* Stage 1: ROM Bootloader (immutable) */
    result = verify_rom_bootloader();
    if (result != SECURE_BOOT_OK) {
        return SECURE_BOOT_ROM_FAILED;
    }
    
    /* Stage 2: Main Bootloader */
    result = verify_and_boot_stage(BOOT_STAGE_BOOT);
    if (result != SECURE_BOOT_OK) {
        return SECURE_BOOT_BOOT_FAILED;
    }
    
    /* Stage 3: Application */
    result = verify_and_boot_stage(BOOT_STAGE_APP);
    if (result != SECURE_BOOT_OK) {
        return SECURE_BOOT_APP_FAILED;
    }
    
    return SECURE_BOOT_OK;
}

/* Anti-Rollback Protection */
anti_rollback_result_t check_anti_rollback(uint32_t new_version) {
    uint32_t current_version = get_current_version();
    uint32_t min_version = get_min_allowed_version();
    
    /* Check if new version is allowed */
    if (new_version < min_version) {
        return ROLLBACK_REJECTED;
    }
    
    /* Update version counter in secure storage */
    if (new_version > current_version) {
        update_version_counter(new_version);
    }
    
    return ROLLBACK_ALLOWED;
}
```

---

## 8. 异常处理

### 8.1 异常分类

```c
/* Exception Types */
typedef enum {
    EXCEPTION_NONE              = 0x00,
    EXCEPTION_WATCHDOG          = 0x01,  /* Watchdog timeout */
    EXCEPTION_HARD_FAULT        = 0x02,  /* Hard fault */
    EXCEPTION_BUS_FAULT         = 0x03,  /* Bus fault */
    EXCEPTION_USAGE_FAULT       = 0x04,  /* Usage fault */
    EXCEPTION_MEM_FAULT         = 0x05,  /* Memory fault */
    EXCEPTION_NMI               = 0x06,  /* Non-maskable interrupt */
    EXCEPTION_POWER_LOSS        = 0x07,  /* Power loss */
    EXCEPTION_FLASH_ERROR       = 0x08,  /* Flash error */
    EXCEPTION_COMM_ERROR        = 0x09,  /* Communication error */
    EXCEPTION_UPGRADE_TIMEOUT   = 0x0A,  /* Upgrade timeout */
    EXCEPTION_UPGRADE_ABORT     = 0x0B,  /* Upgrade aborted */
} exception_type_t;

/* Exception Handler Structure */
typedef struct {
    exception_type_t type;
    uint32_t timestamp;
    uint32_t error_code;
    uint32_t address;
    uint8_t context;    /* CPU context */
} exception_info_t;
```

### 8.2 异常处理实现

```c
/* Exception Handler */
void exception_handler(exception_type_t type, uint32_t error_code) {
    exception_info_t info;
    
    /* Save exception information */
    info.type = type;
    info.timestamp = get_system_tick();
    info.error_code = error_code;
    info.address = get_fault_address();
    save_cpu_context(info.context);
    
    /* Save to non-volatile storage */
    save_exception_info(&info);
    
    /* Handle specific exceptions */
    switch (type) {
        case EXCEPTION_WATCHDOG:
            handle_watchdog_reset();
            break;
            
        case EXCEPTION_POWER_LOSS:
            handle_power_loss();
            break;
            
        case EXCEPTION_UPGRADE_TIMEOUT:
            handle_upgrade_timeout();
            break;
            
        case EXCEPTION_UPGRADE_ABORT:
            handle_upgrade_abort();
            break;
            
        default:
            handle_general_exception();
            break;
    }
    
    /* System reset */
    system_reset();
}

/* Upgrade Error Recovery */
void handle_upgrade_error(void) {
    upgrade_state_t current_state = get_upgrade_state();
    
    switch (current_state) {
        case UPGRADE_STATE_RECEIVING:
            /* Clear partial firmware data */
            clear_received_firmware();
            break;
            
        case UPGRADE_STATE_INSTALLING:
            /* Restore backup partition if needed */
            if (is_backup_corrupted()) {
                restore_backup_partition();
            }
            break;
            
        case UPGRADE_STATE_VERIFYING:
            /* Re-download firmware */
            request_firmware_retransmit();
            break;
            
        default:
            break;
    }
    
    /* Reset upgrade state */
    reset_upgrade_state();
}
```

### 8.3 看门狗管理

```c
/* Watchdog Configuration */
define WATCHDOG_TIMEOUT_MS         5000    /* 5 seconds */
define WATCHDOG_FEED_INTERVAL_MS   1000    /* 1 second */

/* Watchdog Management */
void watchdog_init(void) {
    /* Configure watchdog timer */
    watchdog_config_t config = {
        .timeout_ms = WATCHDOG_TIMEOUT_MS,
        .window_ms = 0,             /* No window */
        .early_warning_ms = 500,    /* 500ms early warning */
    };
    
    watchdog_configure(&config);
    watchdog_enable();
}

/* Feed watchdog during upgrade */
void upgrade_watchdog_task(void) {
    static uint32_t last_feed = 0;
    uint32_t current_time = get_system_tick();
    
    if (current_time - last_feed >= WATCHDOG_FEED_INTERVAL_MS) {
        watchdog_feed();
        last_feed = current_time;
    }
}
```

---

## 9. 测试规范

### 9.1 单元测试

```c
/* Flash Driver Unit Test */
void test_flash_write_read(void) {
    uint8_t write_data;
    uint8_t read_data;
    flash_result_t result;
    
    /* Fill test data */
    for (int i = 0; i < sizeof(write_data); i++) {
        write_data[i] = i & 0xFF;
    }
    
    /* Erase test sector */
    result = flash_driver.erase(TEST_FLASH_ADDRESS, FLASH_SECTOR_SIZE);
    TEST_ASSERT_EQUAL(FLASH_OK, result);
    
    /* Write test data */
    result = flash_driver.write(TEST_FLASH_ADDRESS, write_data, sizeof(write_data));
    TEST_ASSERT_EQUAL(FLASH_OK, result);
    
    /* Read back and verify */
    result = flash_driver.read(TEST_FLASH_ADDRESS, read_data, sizeof(read_data));
    TEST_ASSERT_EQUAL(FLASH_OK, result);
    TEST_ASSERT_EQUAL_MEMORY(write_data, read_data, sizeof(write_data));
}

/* Protocol Unit Test */
void test_protocol_frame_build_parse(void) {
    protocol_frame_t tx_frame, rx_frame;
    uint8_t buffer;
    uint16_t frame_len;
    parse_result_t parse_result;
    
    /* Build test frame */
    uint8_t test_data[] = {0x01, 0x02, 0x03, 0x04};
    frame_len = build_frame(&tx_frame, CMD_UPGRADE_START, 
                            test_data, sizeof(test_data));
    
    /* Parse frame */
    parse_result = parse_frame(buffer, frame_len, &rx_frame);
    TEST_ASSERT_EQUAL(PARSE_OK, parse_result);
    TEST_ASSERT_EQUAL(tx_frame.command, rx_frame.command);
    TEST_ASSERT_EQUAL_MEMORY(tx_frame.data, rx_frame.data, tx_frame.length);
}
```

### 9.2 集成测试

```c
/* Full Upgrade Integration Test */
void test_full_upgrade_flow(void) {
    firmware_package_t package;
    upgrade_result_t result;
    
    /* Prepare test firmware */
    create_test_firmware(&package, TEST_VERSION_1_1_0);
    
    /* Start upgrade */
    result = start_upgrade(&package);
    TEST_ASSERT_EQUAL(UPGRADE_OK, result);
    
    /* Transfer firmware */
    result = transfer_firmware(&package);
    TEST_ASSERT_EQUAL(UPGRADE_OK, result);
    
    /* Verify firmware */
    result = verify_firmware(&package);
    TEST_ASSERT_EQUAL(UPGRADE_OK, result);
    
    /* Install firmware */
    result = install_firmware(&package);
    TEST_ASSERT_EQUAL(UPGRADE_OK, result);
    
    /* Verify new version */
    uint32_t new_version = get_current_version();
    TEST_ASSERT_EQUAL(TEST_VERSION_1_1_0, new_version);
}

/* Rollback Integration Test */
void test_upgrade_rollback(void) {
    firmware_package_t package;
    upgrade_result_t result;
    
    /* Install corrupted firmware */
    create_corrupted_firmware(&package);
    result = start_upgrade(&package);
    TEST_ASSERT_EQUAL(UPGRADE_OK, result);
    
    result = transfer_firmware(&package);
    TEST_ASSERT_EQUAL(UPGRADE_OK, result);
    
    /* Verification should fail */
    result = verify_firmware(&package);
    TEST_ASSERT_EQUAL(UPGRADE_VERIFY_FAILED, result);
    
    /* Device should still run old version */
    uint32_t current_version = get_current_version();
    TEST_ASSERT_EQUAL(TEST_VERSION_1_0_0, current_version);
}
```

---

## 10. 性能指标

### 10.1 性能要求

| 指标 | 要求 | 备注 |
|------|------|------|
| Bootloader启动时间 | < 100ms | 从复位到应用启动 |
| 固件传输速率 | > 100KB/s | UART 921600bps |
| Flash写入速度 | > 50KB/s | 含擦除时间 |
| 固件验证时间 | < 2s | 256KB固件 |
| 升级总时间 | < 30s | 256KB固件 |
| 内存占用 | < 16KB RAM | Bootloader |
| Flash占用 | < 32KB | Bootloader |

### 10.2 资源占用

```c
/* Bootloader Memory Map */
define BOOTLOADER_STACK_SIZE       4096    /* 4KB stack */
define BOOTLOADER_HEAP_SIZE        8192    /* 8KB heap */
define BOOTLOADER_BSS_SIZE         2048    /* 2KB BSS */
define BOOTLOADER_DATA_SIZE        1024    /* 1KB data */

/* Firmware Buffer */
define FIRMWARE_BUFFER_SIZE        4096    /* 4KB receive buffer */
define FIRMWARE_CACHE_SIZE         1024    /* 1KB flash cache */
```

---

## 附录

A. 硬件抽象层接口

```c
/* HAL Interface */
typedef struct {
    /* System */
    void (*system_reset)(void);
    uint32_t (*get_system_tick)(void);
    void (*delay_ms)(uint32_t ms);
    
    /* Flash */
    flash_result_t (*flash_init)(void);
    flash_result_t (*flash_erase)(uint32_t addr, uint32_t size);
    flash_result_t (*flash_write)(uint32_t addr, uint8_t *data, uint32_t size);
    flash_result_t (*flash_read)(uint32_t addr, uint8_t *data, uint32_t size);
    
    /* Communication */
    void (*comm_init)(void);
    int (*comm_send)(uint8_t *data, uint32_t size);
    int (*comm_receive)(uint8_t *data, uint32_t size, uint32_t timeout);
    
    /* Crypto */
    void (*crypto_init)(void);
    int (*sha256)(uint8_t *data, uint32_t size, uint8_t *hash);
    int (*aes_gcm_decrypt)(aes_gcm_context_t *ctx, uint8_t *in, uint32_t in_len,
                           uint8_t *out, uint32_t *out_len);
    int (*ecdsa_verify)(ecdsa_verify_context_t *ctx);
    
    /* Watchdog */
    void (*watchdog_init)(uint32_t timeout_ms);
    void (*watchdog_feed)(void);
    
    /* Power */
    uint32_t (*get_battery_level)(void);
    bool (*is_charging)(void);
} hal_interface_t;
```

B. 配置文件格式

```c
/* Device Configuration */
typedef struct __attribute__((packed)) {
    uint32_t magic;                     /* Configuration magic */
    uint32_t version;                   /* Configuration version */
    uint8_t device_id;             /* Unique device ID */
    uint8_t device_type;           /* Device type */
    uint8_t hardware_rev;           /* Hardware revision */
    uint32_t active_partition;         /* Active partition flag */
    uint32_t boot_count;               /* Boot counter */
    uint32_t upgrade_count;            /* Upgrade counter */
    uint32_t last_upgrade_time;        /* Last upgrade timestamp */
    uint8_t last_upgrade_version;   /* Last upgrade version */
    uint32_t min_allowed_version;      /* Anti-rollback version */
    uint8_t encryption_key;        /* Device encryption key */
    uint8_t signature_key;         /* Device signature key */
    uint8_t reserved;             /* Reserved */
    uint32_t crc;                      /* Configuration CRC */
} device_config_t;
```

C. 错误码定义

```c
/* Error Code Definitions */
typedef enum {
    /* General Errors (0x00 - 0x0F) */
    ERROR_NONE                  = 0x00,
    ERROR_GENERAL               = 0x01,
    ERROR_INVALID_PARAM         = 0x02,
    ERROR_NOT_SUPPORTED         = 0x03,
    ERROR_TIMEOUT               = 0x04,
    ERROR_BUSY                  = 0x05,
    
    /* Flash Errors (0x10 - 0x1F) */
    ERROR_FLASH_INIT            = 0x10,
    ERROR_FLASH_ERASE           = 0x11,
    ERROR_FLASH_WRITE           = 0x12,
    ERROR_FLASH_READ            = 0x13,
    ERROR_FLASH_VERIFY          = 0x14,
    ERROR_FLASH_FULL            = 0x15,
    
    /* Communication Errors (0x20 - 0x2F) */
    ERROR_COMM_INIT             = 0x20,
    ERROR_COMM_SEND             = 0x21,
    ERROR_COMM_RECEIVE          = 0x22,
    ERROR_COMM_TIMEOUT          = 0x23,
    ERROR_COMM_CRC              = 0x24,
    ERROR_COMM_PROTOCOL         = 0x25,
    
    /* Firmware Errors (0x30 - 0x3F) */
    ERROR_FW_INVALID            = 0x30,
    ERROR_FW_SIZE               = 0x31,
    ERROR_FW_VERSION            = 0x32,
    ERROR_FW_CRC                = 0x33,
    ERROR_FW_SIGNATURE          = 0x34,
    ERROR_FW_ENCRYPTION         = 0x35,
    ERROR_FW_DECOMPRESS         = 0x36,
    
    /* Security Errors (0x40 - 0x4F) */
    ERROR_SEC_KEY_LOAD          = 0x40,
    ERROR_SEC_KEY_INVALID       = 0x41,
    ERROR_SEC_CERT_EXPIRED      = 0x42,
    ERROR_SEC_ROLLBACK          = 0x43,
    ERROR_SEC_BOOT              = 0x44,
    
    /* System Errors (0x50 - 0x5F) */
    ERROR_SYS_WATCHDOG          = 0x50,
    ERROR_SYS_POWER             = 0x51,
    ERROR_SYS_MEMORY            = 0x52,
    ERROR_SYS_HARD_FAULT        = 0x53,
} error_code_t;
```