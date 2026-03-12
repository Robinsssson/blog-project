+++
date = 2026-03-12T13:38:00+08:00
draft = false
title = 'SPI与I2C通信协议'
categories = ['信息技术']
tags = ['嵌入式', '通信协议', 'SPI', 'I2C', '单片机']
math = true
+++

## 概述

SPI（Serial Peripheral Interface）和 I2C（Inter-Integrated Circuit）是嵌入式系统中最常用的两种串行通信协议。它们都用于短距离通信，主要连接单片机与各种外设芯片，如传感器、存储器、显示屏等。

---

## 一、SPI协议

### 1.1 简介

SPI是由Motorola公司开发的高速同步串行通信协议，具有以下特点：

| 特性 | 描述 |
|------|------|
| 全双工 | 同时发送和接收 |
| 同步 | 需要时钟信号 |
| 主从模式 | 一个主机，多个从机 |
| 高速 | 可达数十MHz |
| 四线制 | SCK, MOSI, MISO, SS |

### 1.2 信号线定义

| 信号 | 名称 | 方向 | 功能 |
|------|------|------|------|
| SCK | Serial Clock | 主→从 | 时钟信号 |
| MOSI | Master Out Slave In | 主→从 | 主机发送数据 |
| MISO | Master In Slave Out | 从→主 | 从机发送数据 |
| SS/CS | Slave Select | 主→从 | 片选信号（低有效） |

### 1.3 工作模式

SPI有4种工作模式，由时钟极性（CPOL）和时钟相位（CPHA）决定：

| 模式 | CPOL | CPHA | 空闲电平 | 采样边沿 |
|------|------|------|----------|----------|
| Mode 0 | 0 | 0 | 低 | 上升沿 |
| Mode 1 | 0 | 1 | 低 | 下降沿 |
| Mode 2 | 1 | 0 | 高 | 下降沿 |
| Mode 3 | 1 | 1 | 高 | 上升沿 |

```
Mode 0 (CPOL=0, CPHA=0):
    ___     ___     ___     ___
___|   |___|   |___|   |___|   |___  SCK
      ↑       ↑       ↑       ↑
    采样    采样    采样    采样

Mode 1 (CPOL=0, CPHA=1):
    ___     ___     ___     ___
___|   |___|   |___|   |___|   |___  SCK
          ↓       ↓       ↓       ↓
        采样    采样    采样    采样
```

### 1.4 数据传输时序

```
        ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐
SCK  ───┘   └───┘   └───┘   └───┘   └───┘   └───┘   └───┘   └───┘   └───
        
SS   ─────┐                                                       ─────
          └─────────────────────────────────────────────────────────┘
        
MOSI ─────┤ D7  ├───┤ D6  ├───┤ D5  ├───┤ D4  ├───┤ D3  ├───┤ D2  ├───┤ ...
        
MISO ─────┤ D7  ├───┤ D6  ├───┤ D5  ├───┤ D4  ├───┤ D3  ├───┤ D2  ├───┤ ...
```

### 1.5 多从机连接

**方式一：独立片选**

```
        ┌────────┐
        │ Master │
        └───┬────┘
    ┌───────┼───────┬───────┐
   SCK    MOSI    MISO     │
    │       │       │       │
    ├───────┼───────┼───┐   │
    │       │       │   │   │
    │      ┌┴───────┴┐  │   │
    │      │ Slave 1 │  │   │
    │      └─────────┘  │   │
    │       ↑ SS1       │   │
    │                   │   │
    ├───────┼───────┼───┼───┤
    │      ┌┴───────┴┐  │   │
    │      │ Slave 2 │  │   │
    │      └─────────┘  │   │
    │       ↑ SS2       │   │
    └────────────────────┴───┘
```

**方式二：菊花链**

```
        ┌────────┐
        │ Master │
        └───┬────┘
            │
    ┌───────┴───────┐
   SCK    MOSI    MISO
    │       │       │
    │   ┌───┴───┐   │
    │   │Slave 1│   │
    │   └───┬───┘   │
    │       │       │
    │   ┌───┴───┐   │
    │   │Slave 2│   │
    │   └───┬───┘   │
    │       │       │
    └───────┴───────┘
```

### 1.6 优缺点

**优点：**
- 速度快，可达50MHz以上
- 全双工通信
- 协议简单，硬件实现容易
- 无需从机地址

**缺点：**
- 需要更多引脚（至少4根）
- 没有确认机制
- 没有标准帧格式
- 传输距离有限

---

## 二、I2C协议

### 2.1 简介

I2C是由Philips公司开发的两线式串行通信协议：

| 特性 | 描述 |
|------|------|
| 半双工 | 不能同时收发 |
| 同步 | 需要时钟信号 |
| 多主机 | 支持多主机模式 |
| 寻址 | 7位/10位地址 |
| 低速 | 标准模式100kHz，快速模式400kHz |

### 2.2 信号线定义

| 信号 | 名称 | 功能 |
|------|------|------|
| SDA | Serial Data | 数据线（双向） |
| SCL | Serial Clock | 时钟线（主机控制） |

**重要特性：开漏输出 + 上拉电阻**

```
        VCC
         │
        ┌┴┐
        │ │ R_p (上拉电阻)
        └┬┘
         │
    ─────┼───── SDA/SCL
         │
    ┌────┴────┐
    │  开漏   │
    │  输出   │
    └─────────┘
```

### 2.3 信号时序

**起始条件和停止条件**

```
起始条件(S)：SCL高电平时，SDA下降沿
停止条件(P)：SCL高电平时，SDA上升沿

        ┌───┐               ┌───┐
SCL  ───┘   └───────────────┘   └───
            │               │
SDA  ───────┐           ┌───┘
            │           │
            S           P
```

**数据传输**

```
数据在SCL低电平时改变，高电平时稳定

        ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐
SCL  ───┘   └───┘   └───┘   └───┘   └───┘   └───┘   └───┘   └───┘   └───
        
SDA  ────┤ D7  ├───────┤ D6  ├───────┤ D5  ├───────┤ D4  ├─── ...
            ↑               ↑               ↑
          采样            采样            采样
```

### 2.4 数据帧格式

**标准写操作**

```
┌───┬─────────┬───┬─────────┬───┬─────────┬───┐
│ S │ 7位地址 │ W │ ACK │ 8位数据 │ ACK │ P │
└───┴─────────┴───┴─────────┴───┴─────────┴───┘
    └────┬────┘
      主机发送
              └────┬────┘
                从机应答
```

**标准读操作**

```
┌───┬─────────┬───┬───┬─────────┬───┬───┐
│ S │ 7位地址 │ R │ ACK │ 8位数据 │NACK│ P │
└───┴─────────┴───┴─────────┴───┴───┘
    └────┬────┘          └────┬────┘
      主机发送             从机发送
              └────┬────┘
                主机应答
```

**ACK/NACK规则：**
- ACK：接收方拉低SDA
- NACK：最后一个字节后，主机不拉低SDA

### 2.5 多主机仲裁

I2C支持多主机，使用仲裁机制：

```
主机A发送: 1 0 1 1 ...
主机B发送: 1 0 0 1 ...
                ↑
            主机B检测到SDA为高（自己发0，但线为1）
            主机B失去仲裁，释放总线
```

**仲裁原则**：发送0的主机优先级高。

### 2.6 时序参数

| 参数 | 标准模式 | 快速模式 | 快速模式+ |
|------|----------|----------|-----------|
| 最高频率 | 100 kHz | 400 kHz | 1 MHz |
| SCL低电平时间 | 4.7 μs | 1.3 μs | 0.5 μs |
| SCL高电平时间 | 4.0 μs | 0.6 μs | 0.26 μs |
| 起始条件保持时间 | 4.0 μs | 0.6 μs | 0.26 μs |
| 停止条件建立时间 | 4.0 μs | 0.6 μs | 0.26 μs |

### 2.7 优缺点

**优点：**
- 只需2根线
- 支持多主机
- 内置应答机制
- 有标准帧格式

**缺点：**
- 速度慢
- 半双工
- 需要上拉电阻
- 地址冲突问题

---

## 三、SPI vs I2C 对比

### 3.1 总体对比

| 特性 | SPI | I2C |
|------|-----|-----|
| 信号线数量 | 4+ | 2 |
| 全双工/半双工 | 全双工 | 半双工 |
| 最大速度 | 50+ MHz | 1 MHz（高速模式） |
| 多主机支持 | 不支持 | 支持 |
| 地址机制 | 片选线 | 地址位 |
| 应答机制 | 无 | 有 |
| 传输距离 | 短 | 短 |
| 复杂度 | 简单 | 中等 |

### 3.2 选型建议

| 场景 | 推荐 | 原因 |
|------|------|------|
| 高速数据采集 | SPI | 速度快，全双工 |
| 多传感器系统 | I2C | 引脚少，易扩展 |
| 大容量Flash | SPI | 速度快 |
| EEPROM配置存储 | I2C | 简单，够用 |
| OLED显示屏 | SPI/I2C | 取决于刷新率要求 |
| IMU传感器 | SPI | 高速数据流 |

---

## 四、代码实现

### 4.1 STM32 HAL库 SPI

```c
// SPI初始化
SPI_HandleTypeDef hspi1;

void SPI1_Init(void) {
    hspi1.Instance = SPI1;
    hspi1.Init.Mode = SPI_MODE_MASTER;
    hspi1.Init.Direction = SPI_DIRECTION_2LINES;
    hspi1.Init.DataSize = SPI_DATASIZE_8BIT;
    hspi1.Init.CLKPolarity = SPI_POLARITY_LOW;    // CPOL=0
    hspi1.Init.CLKPhase = SPI_PHASE_1EDGE;        // CPHA=0
    hspi1.Init.NSS = SPI_NSS_SOFT;
    hspi1.Init.BaudRatePrescaler = SPI_BAUDRATEPRESCALER_8;
    hspi1.Init.FirstBit = SPI_FIRSTBIT_MSB;
    HAL_SPI_Init(&hspi1);
}

// SPI收发数据
uint8_t SPI_TransmitReceive(uint8_t data) {
    uint8_t rxData;
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_RESET);  // CS拉低
    HAL_SPI_TransmitReceive(&hspi1, &data, &rxData, 1, 100);
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_SET);    // CS拉高
    return rxData;
}

// 读取寄存器
uint8_t SPI_ReadReg(uint8_t reg) {
    uint8_t txData[2] = {reg | 0x80, 0xFF};  // 读命令
    uint8_t rxData[2];
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_RESET);
    HAL_SPI_TransmitReceive(&hspi1, txData, rxData, 2, 100);
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_SET);
    return rxData[1];
}
```

### 4.2 STM32 HAL库 I2C

```c
// I2C初始化
I2C_HandleTypeDef hi2c1;

void I2C1_Init(void) {
    hi2c1.Instance = I2C1;
    hi2c1.Init.ClockSpeed = 100000;  // 100kHz
    hi2c1.Init.DutyCycle = I2C_DUTYCYCLE_2;
    hi2c1.Init.OwnAddress1 = 0;
    hi2c1.Init.AddressingMode = I2C_ADDRESSINGMODE_7BIT;
    hi2c1.Init.DualAddressMode = I2C_DUALADDRESS_DISABLE;
    hi2c1.Init.GeneralCallMode = I2C_GENERALCALL_DISABLE;
    hi2c1.Init.NoStretchMode = I2C_NOSTRETCH_DISABLE;
    HAL_I2C_Init(&hi2c1);
}

// I2C扫描
void I2C_Scan(void) {
    for (uint8_t addr = 0; addr < 127; addr++) {
        if (HAL_I2C_IsDeviceReady(&hi2c1, addr << 1, 1, 10) == HAL_OK) {
            printf("Found device at 0x%02X\n", addr);
        }
    }
}

// I2C写数据
HAL_StatusTypeDef I2C_WriteBytes(uint8_t devAddr, uint8_t reg, 
                                   uint8_t *data, uint16_t len) {
    uint8_t buf[len + 1];
    buf[0] = reg;
    memcpy(&buf[1], data, len);
    return HAL_I2C_Master_Transmit(&hi2c1, devAddr << 1, buf, len + 1, 100);
}

// I2C读数据
HAL_StatusTypeDef I2C_ReadBytes(uint8_t devAddr, uint8_t reg,
                                  uint8_t *data, uint16_t len) {
    HAL_I2C_Master_Transmit(&hi2c1, devAddr << 1, &reg, 1, 100);
    return HAL_I2C_Master_Receive(&hi2c1, devAddr << 1, data, len, 100);
}
```

### 4.3 Arduino 示例

**SPI**

```cpp
#include <SPI.h>

const int CS_PIN = 10;

void setup() {
    pinMode(CS_PIN, OUTPUT);
    digitalWrite(CS_PIN, HIGH);
    SPI.begin();
    SPI.beginTransaction(SPISettings(1000000, MSBFIRST, SPI_MODE0));
}

uint8_t spi_transfer(uint8_t data) {
    digitalWrite(CS_PIN, LOW);
    uint8_t rx = SPI.transfer(data);
    digitalWrite(CS_PIN, HIGH);
    return rx;
}
```

**I2C**

```cpp
#include <Wire.h>

#define DEVICE_ADDR 0x68

void setup() {
    Wire.begin();
    Serial.begin(9600);
}

void i2c_write(uint8_t reg, uint8_t data) {
    Wire.beginTransmission(DEVICE_ADDR);
    Wire.write(reg);
    Wire.write(data);
    Wire.endTransmission();
}

uint8_t i2c_read(uint8_t reg) {
    Wire.beginTransmission(DEVICE_ADDR);
    Wire.write(reg);
    Wire.endTransmission();
    
    Wire.requestFrom(DEVICE_ADDR, 1);
    return Wire.read();
}
```

---

## 五、常见问题与解决

### 5.1 SPI常见问题

| 问题 | 原因 | 解决方法 |
|------|------|----------|
| 读出全0或全1 | 模式不匹配 | 检查CPOL/CPHA设置 |
| 数据错位 | 时序问题 | 检查波特率和边沿 |
| 通信不稳定 | 线路干扰 | 缩短线长，加屏蔽 |
| 从机不响应 | CS未拉低 | 检查片选逻辑 |

### 5.2 I2C常见问题

| 问题 | 原因 | 解决方法 |
|------|------|----------|
| 找不到设备 | 地址错误 | I2C扫描确认地址 |
| 总线挂死 | 无ACK | 添加超时和恢复机制 |
| 通信失败 | 上拉电阻不当 | 检查上拉电阻值（2.2k-10k） |
| 数据错误 | 时序问题 | 降低时钟频率 |

### 5.3 I2C总线恢复

```c
// I2C总线恢复（时钟脉冲法）
void I2C_BusRecovery(void) {
    // 将SDA/SCL设为GPIO输出
    // SCL输出9个时钟脉冲
    for (int i = 0; i < 9; i++) {
        HAL_GPIO_WritePin(GPIOB, SCL_PIN, GPIO_PIN_RESET);
        HAL_Delay(1);
        HAL_GPIO_WritePin(GPIOB, SCL_PIN, GPIO_PIN_SET);
        HAL_Delay(1);
    }
    // 发送停止条件
    HAL_GPIO_WritePin(GPIOB, SDA_PIN, GPIO_PIN_RESET);
    HAL_Delay(1);
    HAL_GPIO_WritePin(GPIOB, SCL_PIN, GPIO_PIN_SET);
    HAL_Delay(1);
    HAL_GPIO_WritePin(GPIOB, SDA_PIN, GPIO_PIN_SET);
    // 重新初始化I2C
    HAL_I2C_DeInit(&hi2c1);
    I2C1_Init();
}
```

---

## 六、实际应用示例

### 6.1 SPI驱动TFT显示屏

```c
// 发送命令
void TFT_WriteCommand(uint8_t cmd) {
    DC_LOW();    // 数据/命令选择
    CS_LOW();
    SPI_Transmit(cmd);
    CS_HIGH();
}

// 发送数据
void TFT_WriteData(uint8_t data) {
    DC_HIGH();
    CS_LOW();
    SPI_Transmit(data);
    CS_HIGH();
}

// 设置窗口
void TFT_SetWindow(uint16_t x1, uint16_t y1, uint16_t x2, uint16_t y2) {
    TFT_WriteCommand(0x2A);  // CASET
    TFT_WriteData(x1 >> 8);
    TFT_WriteData(x1 & 0xFF);
    TFT_WriteData(x2 >> 8);
    TFT_WriteData(x2 & 0xFF);
    
    TFT_WriteCommand(0x2B);  // RASET
    TFT_WriteData(y1 >> 8);
    TFT_WriteData(y1 & 0xFF);
    TFT_WriteData(y2 >> 8);
    TFT_WriteData(y2 & 0xFF);
    
    TFT_WriteCommand(0x2C);  // RAMWR
}
```

### 6.2 I2C驱动MPU6050

```c
#define MPU6050_ADDR 0x68

// 初始化
void MPU6050_Init(void) {
    I2C_WriteBytes(MPU6050_ADDR, 0x6B, 0x00);  // 唤醒
    I2C_WriteBytes(MPU6050_ADDR, 0x19, 0x07);  // 采样率分频
    I2C_WriteBytes(MPU6050_ADDR, 0x1A, 0x00);  // 配置
    I2C_WriteBytes(MPU6050_ADDR, 0x1B, 0x00);  // 陀螺仪量程
    I2C_WriteBytes(MPU6050_ADDR, 0x1C, 0x00);  // 加速度量程
}

// 读取加速度
void MPU6050_ReadAccel(int16_t *ax, int16_t *ay, int16_t *az) {
    uint8_t buf[6];
    I2C_ReadBytes(MPU6050_ADDR, 0x3B, buf, 6);
    *ax = (buf[0] << 8) | buf[1];
    *ay = (buf[2] << 8) | buf[3];
    *az = (buf[4] << 8) | buf[5];
}
```

---

## 总结

**SPI选择场景：**
- 需要高速传输
- 数据量大
- 引脚资源充足
- 单主机多从机

**I2C选择场景：**
- 引脚资源紧张
- 多传感器系统
- 低速配置读写
- 需要多主机

---

## 参考资料

- [SPI Specification](https://www.analog.com/media/en/analog-dialogue/Volume-52/Number-3/spi-a-synchronous-serial-interface.pdf)
- [I2C Specification](https://www.nxp.com/docs/en/user-guide/UM10204.pdf)
- STM32 Reference Manual
- 《嵌入式系统设计》

---

> 🎯 SPI和I2C是嵌入式开发必备技能，理解它们的区别和应用场景非常重要！