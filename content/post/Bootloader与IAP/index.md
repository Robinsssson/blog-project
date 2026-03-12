+++
date = 2026-03-12T13:38:00+08:00
draft = false
title = 'Bootloader与IAP'
categories = ['信息技术']
tags = ['嵌入式', 'Bootloader', 'IAP', 'STM32', '固件升级']
math = true
+++

## 概述

Bootloader是嵌入式系统启动时运行的第一段程序，负责初始化硬件并加载主应用程序。IAP（In-Application Programming）是一种在线升级技术，允许设备在运行状态下更新固件，无需外部编程器。

---

## 一、Bootloader基础

### 1.1 什么是Bootloader

Bootloader是一段固化在微控制器Flash起始地址的程序，主要功能：

| 功能 | 描述 |
|------|------|
| 硬件初始化 | 时钟、GPIO、外设等 |
| 应用程序加载 | 跳转到主程序 |
| 固件升级 | 接收新固件并写入Flash |
| 安全验证 | 校验固件完整性 |
| 恢复模式 | 异常情况下的恢复机制 |

### 1.2 启动流程

```
上电/复位
    │
    ▼
┌─────────────────┐
│   Bootloader    │
│  (0x08000000)   │
└────────┬────────┘
         │
         ├──── 检测升级模式？
         │           │
         │           ▼
         │     ┌──────────┐
         │     │ 接收新固件 │
         │     │ 写入Flash  │
         │     └──────────┘
         │
         ▼
┌─────────────────┐
│   应用程序      │
│  (0x08008000)  │
└─────────────────┘
```

### 1.3 内存布局

**典型STM32内存分配：**

```
Flash: 0x08000000 - 0x0807FFFF (512KB)
┌──────────────────────────────┐ 0x08000000
│         Bootloader           │   16KB
│        (Sector 0)            │
├──────────────────────────────┤ 0x08004000
│         App标志区            │    4KB
├──────────────────────────────┤ 0x08005000
│         应用程序              │  492KB
│                              │
├──────────────────────────────┤
│         参数存储区            │    4KB
└──────────────────────────────┘ 0x0807FFFF
```

---

## 二、Bootloader设计

### 2.1 核心功能模块

```
┌────────────────────────────────────────────┐
│               Bootloader                   │
├────────────────────────────────────────────┤
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ 硬件初始化 │  │ 通信接口  │  │ Flash操作 │  │
│  │          │  │ UART/USB │  │ 擦/写/读  │  │
│  └──────────┘  └──────────┘  └──────────┘  │
│                                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ 协议解析  │  │ 固件校验  │  │ 跳转管理  │  │
│  │ YMODEM等 │  │ CRC/MD5  │  │ MSP/VTOR │  │
│  └──────────┘  └──────────┘  └──────────┘  │
└────────────────────────────────────────────┘
```

### 2.2 启动检测逻辑

```c
typedef struct {
    uint32_t magic;        // 魔数，标识有效App
    uint32_t app_addr;     // App起始地址
    uint32_t app_size;     // App大小
    uint32_t app_crc;      // App校验值
    uint32_t boot_flag;    // 启动标志（用于触发升级）
} app_header_t;

#define APP_MAGIC      0x41505031  // "APP1"
#define BOOT_FLAG_UPG  0x55504752  // "UPGR"
#define APP_ADDR       0x08005000
#define HEADER_ADDR    0x08004000

bool check_app_valid(void) {
    app_header_t *header = (app_header_t *)HEADER_ADDR;
    
    // 检查魔数
    if (header->magic != APP_MAGIC) {
        return false;
    }
    
    // 检查栈指针是否在RAM范围内
    uint32_t sp = *(uint32_t *)header->app_addr;
    if ((sp < 0x20000000) || (sp > 0x20020000)) {
        return false;
    }
    
    // 检查复位向量是否在Flash范围内
    uint32_t pc = *(uint32_t *)(header->app_addr + 4);
    if ((pc < APP_ADDR) || (pc > 0x0807FFFF)) {
        return false;
    }
    
    return true;
}
```

### 2.3 跳转到应用程序

```c
typedef void (*app_func_t)(void);

void jump_to_app(uint32_t app_addr) {
    uint32_t app_stack;
    app_func_t app_entry;
    
    // 获取栈指针和入口地址
    app_stack = *(uint32_t *)app_addr;
    app_entry = (app_func_t)(*(uint32_t *)(app_addr + 4));
    
    // 关闭所有中断
    __disable_irq();
    
    // 清除所有中断挂起标志
    for (int i = 0; i < 8; i++) {
        NVIC->ICER[i] = 0xFFFFFFFF;
        NVIC->ICPR[i] = 0xFFFFFFFF;
    }
    
    // 复位所有外设
    RCC->AHB1RSTR = 0xFFFFFFFF;
    RCC->AHB2RSTR = 0xFFFFFFFF;
    RCC->APB1RSTR = 0xFFFFFFFF;
    RCC->APB2RSTR = 0xFFFFFFFF;
    
    // 设置VTOR（Cortex-M3/M4/M7）
    SCB->VTOR = app_addr;
    
    // 设置栈指针
    __set_MSP(app_stack);
    
    // 跳转到App
    app_entry();
    
    // 不应该执行到这里
    while(1);
}
```

### 2.4 完整Bootloader流程

```c
int main(void) {
    // 硬件初始化
    HAL_Init();
    SystemClock_Config();
    
    // 读取App头信息
    app_header_t *header = (app_header_t *)HEADER_ADDR;
    
    // 检查是否需要升级
    if (header->boot_flag == BOOT_FLAG_UPG) {
        // 进入升级模式
        bootloader_upgrade_mode();
    }
    
    // 检查按键是否按下（强制升级）
    if (check_upgrade_key()) {
        bootloader_upgrade_mode();
    }
    
    // 检查App是否有效
    if (!check_app_valid()) {
        // App无效，进入升级模式
        bootloader_upgrade_mode();
    }
    
    // 跳转到App
    jump_to_app(header->app_addr);
    
    while(1);
}
```

---

## 三、IAP升级实现

### 3.1 IAP升级方式

| 方式 | 描述 | 优点 | 缺点 |
|------|------|------|------|
| UART | 串口传输 | 简单，成本低 | 速度慢 |
| USB | USB传输 | 速度快 | 复杂 |
| SD卡 | 从SD卡读取 | 离线升级 | 需要SD卡槽 |
| 网络 | TCP/HTTP下载 | 远程升级 | 需要网络模块 |
| CAN | CAN总线传输 | 工业场景 | 需要CAN接口 |

### 3.2 升级协议设计

**自定义协议帧格式：**

```
┌────────┬────────┬────────┬────────┬────────┬────────┐
│ 帧头   │ 命令   │ 长度   │ 序号   │ 数据   │ 校验   │
│ 2字节  │ 1字节  │ 2字节  │ 2字节  │ N字节  │ 2字节  │
└────────┴────────┴────────┴────────┴────────┴────────┘
```

**命令定义：**

| 命令码 | 名称 | 描述 |
|--------|------|------|
| 0x01 | CMD_HANDSHAKE | 握手请求 |
| 0x02 | CMD_ERASE | 擦除Flash |
| 0x03 | CMD_WRITE | 写入数据 |
| 0x04 | CMD_VERIFY | 校验固件 |
| 0x05 | CMD_JUMP | 跳转运行 |
| 0x06 | CMD_ABORT | 中止升级 |

### 3.3 协议实现

```c
#define FRAME_HEAD   0xAA55
#define CRC16_INIT   0xFFFF

typedef struct {
    uint16_t head;
    uint8_t  cmd;
    uint16_t length;
    uint16_t seq;
    uint8_t  data[256];
    uint16_t crc;
} __packed frame_t;

typedef enum {
    STATE_IDLE,
    STATE_RECEIVING,
    STATE_VERIFY,
    STATE_DONE,
    STATE_ERROR
} upgrade_state_t;

static upgrade_state_t state = STATE_IDLE;
static uint32_t write_addr = APP_ADDR;

// 处理接收数据
void process_frame(frame_t *frame) {
    uint16_t crc = calculate_crc16((uint8_t *)frame, 
                                    sizeof(frame_t) - 2);
    
    if (frame->crc != crc) {
        send_response(frame->cmd, 0xFF);  // CRC错误
        return;
    }
    
    switch (frame->cmd) {
        case CMD_HANDSHAKE:
            handle_handshake(frame);
            break;
        case CMD_ERASE:
            handle_erase(frame);
            break;
        case CMD_WRITE:
            handle_write(frame);
            break;
        case CMD_VERIFY:
            handle_verify(frame);
            break;
        case CMD_JUMP:
            handle_jump(frame);
            break;
        default:
            send_response(frame->cmd, 0xFE);  // 未知命令
            break;
    }
}

// 擦除处理
void handle_erase(frame_t *frame) {
    uint32_t size = *(uint32_t *)frame->data;
    uint32_t pages = (size + FLASH_PAGE_SIZE - 1) / FLASH_PAGE_SIZE;
    
    HAL_FLASH_Unlock();
    
    for (uint32_t i = 0; i < pages; i++) {
        if (flash_erase_page(APP_ADDR + i * FLASH_PAGE_SIZE) != HAL_OK) {
            send_response(CMD_ERASE, 0x01);
            HAL_FLASH_Lock();
            return;
        }
    }
    
    HAL_FLASH_Lock();
    write_addr = APP_ADDR;
    send_response(CMD_ERASE, 0x00);
}

// 写入处理
void handle_write(frame_t *frame) {
    uint16_t len = frame->length;
    
    HAL_FLASH_Unlock();
    
    for (uint16_t i = 0; i < len; i += 4) {
        uint32_t data = *(uint32_t *)&frame->data[i];
        if (HAL_FLASH_Program(FLASH_TYPEPROGRAM_WORD, 
                              write_addr + i, data) != HAL_OK) {
            send_response(CMD_WRITE, 0x01);
            HAL_FLASH_Lock();
            return;
        }
    }
    
    HAL_FLASH_Lock();
    write_addr += len;
    send_response(CMD_WRITE, 0x00);
}
```

### 3.4 YMODEM协议（串口升级）

YMODEM是常用的串口文件传输协议：

```c
// YMODEM帧格式
#define SOH  0x01  // 128字节数据
#define STX  0x02  // 1024字节数据
#define EOT  0x04  // 传输结束
#define ACK  0x06  // 确认
#define NAK  0x15  // 否认
#define CAN  0x18  // 取消

// 接收YMODEM数据包
int ymodem_receive(uint8_t *buf, int *size) {
    uint8_t packet[1024 + 5];
    int packet_num = 0;
    int total_size = 0;
    
    // 发送'C'请求CRC校验
    send_byte('C');
    
    while (1) {
        int len = receive_packet(packet);
        if (len < 0) continue;
        
        switch (packet[0]) {
            case SOH:  // 128字节
            case STX:  // 1024字节
                if (check_crc(packet)) {
                    // 第0包包含文件名和大小
                    if (packet[1] == 0x00 && packet_num == 0) {
                        parse_filename_size(packet, buf, size);
                        send_byte(ACK);
                        send_byte('C');
                        packet_num++;
                    } else {
                        // 数据包
                        memcpy(buf + total_size, packet + 3, len);
                        total_size += len - 5;
                        send_byte(ACK);
                    }
                } else {
                    send_byte(NAK);
                }
                break;
                
            case EOT:  // 结束
                send_byte(ACK);
                return total_size;
                
            case CAN:  // 取消
                return -1;
        }
    }
}
```

---

## 四、Flash操作

### 4.1 STM32 Flash编程

```c
// Flash解锁
void flash_unlock(void) {
    FLASH->KEYR = 0x45670123;
    FLASH->KEYR = 0xCDEF89AB;
}

// Flash上锁
void flash_lock(void) {
    FLASH->CR |= FLASH_CR_LOCK;
}

// 擦除页
HAL_StatusTypeDef flash_erase_page(uint32_t addr) {
    // 等待操作完成
    while (FLASH->SR & FLASH_SR_BSY);
    
    // 解锁
    flash_unlock();
    
    // 设置页擦除
    FLASH->CR |= FLASH_CR_PER;
    FLASH->AR = addr;
    FLASH->CR |= FLASH_CR_STRT;
    
    // 等待完成
    while (FLASH->SR & FLASH_SR_BSY);
    
    // 清除标志
    FLASH->SR |= FLASH_SR_EOP;
    FLASH->CR &= ~FLASH_CR_PER;
    
    flash_lock();
    
    return HAL_OK;
}

// 写入数据（按字）
HAL_StatusTypeDef flash_write_word(uint32_t addr, uint32_t data) {
    while (FLASH->SR & FLASH_SR_BSY);
    
    flash_unlock();
    
    FLASH->CR |= FLASH_CR_PG;
    *(__IO uint32_t *)addr = data;
    
    while (FLASH->SR & FLASH_SR_BSY);
    
    FLASH->SR |= FLASH_SR_EOP;
    FLASH->CR &= ~FLASH_CR_PG;
    
    flash_lock();
    
    return HAL_OK;
}
```

### 4.2 Flash写入优化

```c
// 按页写入（更高效）
HAL_StatusTypeDef flash_write_page(uint32_t addr, 
                                     uint8_t *data, 
                                     uint16_t len) {
    flash_unlock();
    
    // 按字（4字节）写入
    for (uint16_t i = 0; i < len; i += 4) {
        uint32_t word = *(uint32_t *)&data[i];
        if (HAL_FLASH_Program(FLASH_TYPEPROGRAM_WORD, 
                              addr + i, word) != HAL_OK) {
            flash_lock();
            return HAL_ERROR;
        }
    }
    
    flash_lock();
    return HAL_OK;
}
```

---

## 五、固件校验

### 5.1 CRC校验

```c
// CRC32计算
uint32_t crc32_table[256];

void crc32_init(void) {
    for (uint32_t i = 0; i < 256; i++) {
        uint32_t crc = i;
        for (int j = 0; j < 8; j++) {
            if (crc & 1)
                crc = (crc >> 1) ^ 0xEDB88320;
            else
                crc >>= 1;
        }
        crc32_table[i] = crc;
    }
}

uint32_t crc32_calc(uint8_t *data, uint32_t len) {
    uint32_t crc = 0xFFFFFFFF;
    for (uint32_t i = 0; i < len; i++) {
        crc = crc32_table[(crc ^ data[i]) & 0xFF] ^ (crc >> 8);
    }
    return crc ^ 0xFFFFFFFF;
}

// 校验固件
bool verify_firmware(uint32_t addr, uint32_t size, uint32_t expected_crc) {
    uint32_t calc_crc = crc32_calc((uint8_t *)addr, size);
    return (calc_crc == expected_crc);
}
```

### 5.2 数字签名验证（安全升级）

```c
#include "mbedtls/rsa.h"
#include "mbedtls/sha256.h"

bool verify_signature(uint8_t *firmware, uint32_t size,
                      uint8_t *signature, uint32_t sig_len) {
    uint8_t hash[32];
    mbedtls_sha256_context sha_ctx;
    mbedtls_rsa_context rsa_ctx;
    
    // 计算固件哈希
    mbedtls_sha256_init(&sha_ctx);
    mbedtls_sha256_starts(&sha_ctx, 0);
    mbedtls_sha256_update(&sha_ctx, firmware, size);
    mbedtls_sha256_finish(&sha_ctx, hash);
    
    // 加载公钥
    mbedtls_rsa_init(&rsa_ctx);
    // ... 加载公钥 ...
    
    // 验证签名
    int ret = mbedtls_rsa_pkcs1_verify(&rsa_ctx, 
                                         MBEDTLS_MD_SHA256, 
                                         32, hash, signature);
    
    mbedtls_rsa_free(&rsa_ctx);
    
    return (ret == 0);
}
```

---

## 六、应用程序适配

### 6.1 链接脚本修改

**STM32 GCC链接脚本（ld文件）：**

```ld
/* 原始配置 */
MEMORY {
    FLASH (rx) : ORIGIN = 0x08000000, LENGTH = 512K
    RAM (rwx) : ORIGIN = 0x20000000, LENGTH = 128K
}

/* IAP配置 - App起始地址改变 */
MEMORY {
    FLASH (rx) : ORIGIN = 0x08005000, LENGTH = 492K
    RAM (rwx) : ORIGIN = 0x20000000, LENGTH = 128K
}
```

### 6.2 启动代码修改

```c
// 在SystemInit中设置VTOR
void SystemInit(void) {
    // 设置中断向量表偏移
    SCB->VTOR = 0x08005000;  // App起始地址
    
    // ... 其他初始化 ...
}

// 或在main函数开头设置
int main(void) {
    SCB->VTOR = 0x08005000;
    
    HAL_Init();
    SystemClock_Config();
    
    // ...
}
```

### 6.3 Keil配置

在Keil MDK中配置IAP：

```
Options for Target -> Target:
  - ROM (IROM1): Start: 0x08005000, Size: 0x0007B000

Options for Target -> Debug:
  - Initialization File: 
    FUNC void Setup(void) {
      SP = _RDWORD(0x08005000);
      PC = _RDWORD(0x08005004);
    }
    Setup();
```

---

## 七、实际项目示例

### 7.1 完整Bootloader框架

```c
// main.c - Bootloader
int main(void) {
    // HAL初始化
    HAL_Init();
    SystemClock_Config();
    
    // 外设初始化
    MX_GPIO_Init();
    MX_USART1_UART_Init();
    MX_FLASH_Init();
    
    // 启动指示
    LED_ON();
    HAL_Delay(100);
    LED_OFF();
    
    // 读取App头信息
    app_header_t *header = (app_header_t *)HEADER_ADDR;
    
    // 检查升级请求标志
    if (header->boot_flag == BOOT_FLAG_UPG) {
        printf("Enter upgrade mode (flag)\n");
        goto upgrade_mode;
    }
    
    // 检查按键（强制升级）
    if (HAL_GPIO_ReadPin(KEY_GPIO, KEY_PIN) == GPIO_PIN_RESET) {
        printf("Enter upgrade mode (key)\n");
        goto upgrade_mode;
    }
    
    // 检查App有效性
    if (!check_app_valid()) {
        printf("App invalid, enter upgrade mode\n");
        goto upgrade_mode;
    }
    
    // 校验App
    if (!verify_firmware(header->app_addr, header->app_size, 
                          header->app_crc)) {
        printf("CRC check failed\n");
        goto upgrade_mode;
    }
    
    // 关闭用到的外设
    HAL_UART_DeInit(&huart1);
    HAL_RCC_DeInit();
    
    // 跳转到App
    printf("Jump to app @ 0x%08X\n", header->app_addr);
    jump_to_app(header->app_addr);
    
upgrade_mode:
    // 升级模式主循环
    bootloader_main();
    
    while (1);
}
```

### 7.2 App中触发升级

```c
// app_upgrade.c - 在App中触发升级
#include "app_upgrade.h"

#define BOOT_FLAG_ADDR  0x08004000
#define BOOT_FLAG_UPG   0x55504752

void trigger_upgrade(void) {
    // 清除标志区
    HAL_FLASH_Unlock();
    flash_erase_page(BOOT_FLAG_ADDR);
    
    // 写入升级标志
    HAL_FLASH_Program(FLASH_TYPEPROGRAM_WORD, 
                      BOOT_FLAG_ADDR + 16, BOOT_FLAG_UPG);
    
    HAL_FLASH_Lock();
    
    // 复位进入Bootloader
    NVIC_SystemReset();
}

// 接收到升级命令
void on_upgrade_command(void) {
    printf("Preparing for upgrade...\n");
    trigger_upgrade();
}
```

---

## 八、注意事项

### 8.1 中断向量表

**问题：** Bootloader和App都使用中断，需要正确设置VTOR。

**解决：**
- Bootloader中使用默认VTOR（0x08000000）
- App启动时设置SCB->VTOR = APP_ADDR

### 8.2 栈指针

**问题：** 跳转前需要正确设置栈指针。

**解决：**
```c
__set_MSP(*(uint32_t *)APP_ADDR);
```

### 8.3 时钟配置

**问题：** Bootloader配置的时钟可能影响App。

**解决：**
- 跳转前调用HAL_RCC_DeInit()
- App重新配置时钟

### 8.4 Flash保护

**问题：** 意外写入导致Bootloader损坏。

**解决：**
- 启用Flash写保护
- Bootloader区域设置为只读

```c
// 启用Flash保护（HAL库）
void enable_flash_protection(void) {
    FLASH_OBProgramInitTypeDef ob_config;
    
    HAL_FLASHEx_OBGetConfig(&ob_config);
    
    ob_config.WRPPage = 0x0001;  // 保护前16KB
    ob_config.WRPState = OB_WRPSTATE_ENABLE;
    
    HAL_FLASHEx_OBProgram(&ob_config);
    HAL_FLASH_OB_Launch();
}
```

---

## 九、调试技巧

### 9.1 串口调试

```c
#define DEBUG_PRINT(fmt, ...) \
    printf("[%s:%d] " fmt "\n", __func__, __LINE__, ##__VA_ARGS__)

// 跟踪升级过程
DEBUG_PRINT("Erase flash: addr=0x%08X, pages=%d", addr, pages);
DEBUG_PRINT("Write flash: addr=0x%08X, len=%d", write_addr, len);
DEBUG_PRINT("Verify CRC: calc=0x%08X, expected=0x%08X", calc, expected);
```

### 9.2 LED状态指示

```c
void set_upgrade_status(status_t status) {
    switch (status) {
        case STATUS_IDLE:
            LED_OFF();
            break;
        case STATUS_RECEIVING:
            LED_TOGGLE();  // 快闪
            break;
        case STATUS_WRITING:
            LED_ON();
            break;
        case STATUS_ERROR:
            // 错误闪烁模式
            for (int i = 0; i < 3; i++) {
                LED_ON(); HAL_Delay(100);
                LED_OFF(); HAL_Delay(100);
            }
            break;
    }
}
```

---

## 总结

| 组件 | 功能 | 注意事项 |
|------|------|----------|
| Bootloader | 启动引导、升级 | 放置在Flash起始位置 |
| IAP协议 | 固件传输 | 选择合适的协议 |
| Flash操作 | 擦写存储 | 注意保护和校验 |
| 跳转机制 | 进入App | 设置VTOR和栈指针 |
| 固件校验 | 完整性验证 | CRC或数字签名 |

**设计原则：**
- Bootloader尽可能简单稳定
- 支持异常恢复机制
- 完善的错误处理
- 详细的日志记录

---

## 参考资料

- STM32 Programming Manual (PM0075)
- AN3155: USART protocol used in the STM32 bootloader
- YMODEM Protocol Specification
- 《嵌入式系统Bootloader开发》

---

> 🎯 Bootloader和IAP是嵌入式产品远程升级的基础，掌握它们对产品维护至关重要！