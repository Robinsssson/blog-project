---
title: "ARM指令集与STM32软件设计详解"
date: 2026-03-15T14:15:00+08:00
draft: false
categories: ["嵌入式开发"]
tags: ["ARM", "STM32", "汇编", "嵌入式", "Cortex-M"]
math: true
toc: true
readingTime: true
description: "深入讲解ARM指令集架构基础、Cortex-M内核原理以及在STM32上的软件设计实践"
---

## 概述

ARM（Advanced RISC Machine）是目前嵌入式领域最流行的处理器架构之一。STM32系列微控制器基于ARM Cortex-M内核，在工业控制、消费电子、物联网等领域广泛应用。理解ARM指令集对于编写高效的嵌入式软件至关重要。

### ARM 架构家族

| 架构 | 内核系列 | 典型应用 |
|------|----------|----------|
| ARMv7-A | Cortex-A | 手机、平板（运行Linux/Android） |
| ARMv7-R | Cortex-R | 实时控制系统、汽车电子 |
| ARMv7-M | Cortex-M3/M4 | 微控制器、嵌入式系统 |
| ARMv8-M | Cortex-M33/M55 | 安全物联网设备 |

**STM32系列内核对照：**

| STM32系列 | 内核 | 架构 | 特点 |
|-----------|------|------|------|
| STM32F1 | Cortex-M3 | ARMv7-M | 经典系列，性价比高 |
| STM32F4 | Cortex-M4 | ARMv7E-M | 带DSP和FPU |
| STM32F7 | Cortex-M7 | ARMv7E-M | 高性能，双发射 |
| STM32H7 | Cortex-M7 | ARMv7E-M | 超高性能 |
| STM32G4 | Cortex-M4 | ARMv7E-M | 模拟集成度高 |

---

## 一、ARM指令集基础

### 1.1 RISC 设计哲学

ARM 采用精简指令集（RISC）设计，核心原则：

- **指令长度固定**：Thumb指令16位，ARM指令32位
- **Load/Store架构**：数据处理只在寄存器间进行
- **大量通用寄存器**：减少内存访问
- **流水线执行**：提高指令吞吐量

### 1.2 寄存器组织

Cortex-M3/M4 有以下寄存器：

```
┌─────────────────────────────────────────────────────────┐
│                    通用寄存器 (R0-R12)                   │
├─────────┬───────────────────────────────────────────────┤
│   R0    │  参数传递/返回值 (函数调用约定)                │
│   R1    │  参数传递/临时寄存器                           │
│   R2    │  参数传递/临时寄存器                           │
│   R3    │  参数传递/临时寄存器                           │
│   R4    │  被调用者保存 (callee-saved)                   │
│   R5    │  被调用者保存                                  │
│   R6    │  被调用者保存                                  │
│   R7    │  被调用者保存 / Frame Pointer (可选)          │
│   R8    │  被调用者保存                                  │
│   R9    │  被调用者保存                                  │
│   R10   │  被调用者保存                                  │
│   R11   │  被调用者保存 / Frame Pointer                  │
│   R12   │  过程调用临时寄存器 (IP)                       │
├─────────┼───────────────────────────────────────────────┤
│   SP    │  R13, 堆栈指针                                 │
│   LR    │  R14, 链接寄存器 (返回地址)                    │
│   PC    │  R15, 程序计数器                               │
├─────────┼───────────────────────────────────────────────┤
│  xPSR   │  程序状态寄存器                                │
├─────────┼───────────────────────────────────────────────┤
│ PRIMASK │  中断屏蔽寄存器                                │
│ CONTROL │  控制寄存器                                    │
└─────────┴───────────────────────────────────────────────┘
```

### 1.3 程序状态寄存器 (xPSR)

```
┌─────────────────────────────────────────────────────────────┐
│  N  │  Z  │  C  │  V  │  Q  │   GE[3:0]   │   IT/ICI   │
├─────┴─────┴─────┴─────┴─────┴──────────────┴────────────┤
│                       条件标志位                          │
├───────────────────────────────────────────────────────────┤
│  N (Negative): 结果为负                                  │
│  Z (Zero): 结果为零                                      │
│  C (Carry): 进位/借位                                    │
│  V (Overflow): 溢出                                      │
│  Q (Saturation): 饱和                                    │
└───────────────────────────────────────────────────────────┘
```

---

## 二、ARM指令详解

### 2.1 数据处理指令

#### 算术运算

```asm
; 加法
ADD  R0, R1, R2      ; R0 = R1 + R2
ADD  R0, R1, #100    ; R0 = R1 + 100
ADC  R0, R1, R2      ; R0 = R1 + R2 + C (带进位加)

; 减法
SUB  R0, R1, R2      ; R0 = R1 - R2
SBC  R0, R1, R2      ; R0 = R1 - R2 - !C (带借位减)

; 乘法
MUL  R0, R1, R2      ; R0 = R1 × R2 (低32位)
MLA  R0, R1, R2, R3  ; R0 = R1 × R2 + R3
UMULL R0, R1, R2, R3 ; R0:R1 = R2 × R3 (64位无符号)
```

#### 逻辑运算

```asm
AND  R0, R1, R2      ; R0 = R1 & R2
ORR  R0, R1, R2      ; R0 = R1 | R2
EOR  R0, R1, R2      ; R0 = R1 ^ R2
BIC  R0, R1, R2      ; R0 = R1 & ~R2 (位清除)
```

#### 移位操作

```asm
LSL  R0, R1, #2      ; 逻辑左移: R0 = R1 << 2
LSR  R0, R1, #3      ; 逻辑右移: R0 = R1 >> 3
ASR  R0, R1, #4      ; 算术右移: R0 = R1 >> 4 (保留符号位)
ROR  R0, R1, #5      ; 循环右移
```

### 2.2 Load/Store 指令

```asm
; 加载指令
LDR   R0, [R1]           ; R0 = *R1
LDR   R0, [R1, #4]       ; R0 = *(R1 + 4)
LDR   R0, [R1, R2]       ; R0 = *(R1 + R2)
LDR   R0, [R1, #4]!      ; 预索引: R1 += 4, R0 = *R1
LDR   R0, [R1], #4       ; 后索引: R0 = *R1, R1 += 4

; 存储指令
STR   R0, [R1]           ; *R1 = R0
STR   R0, [R1, #4]       ; *(R1 + 4) = R0

; 批量加载/存储
LDMIA R0!, {R1-R4}       ; 加载多个寄存器，递增后索引
STMDB R0!, {R1-R4}       ; 存储多个寄存器，递减前索引

; 常用缩写:
; IA: Increment After  (先加载/存储，后递增)
; IB: Increment Before (先递增，后加载/存储)
; DA: Decrement After
; DB: Decrement Before
```

**PUSH/POP 实现：**

```asm
; 函数入口保存寄存器
PUSH  {R4-R11, LR}       ; 等价于 STMDB SP!, {R4-R11, LR}

; 函数出口恢复寄存器
POP   {R4-R11, PC}       ; 等价于 LDMIA SP!, {R4-R11, PC}
```

### 2.3 分支指令

```asm
B     label          ; 无条件跳转
BL    function       ; 带链接跳转 (调用函数)
BX    R0             ; 间接跳转 (切换状态)

; 条件分支
BEQ   label          ; Z=1 时跳转 (相等)
BNE   label          ; Z=0 时跳转 (不等)
BLT   label          ; N!=V 时跳转 (小于)
BGT   label          ; Z=0 && N=V 时跳转 (大于)
BLE   label          ; Z=1 || N!=V 时跳转 (小于等于)
BGE   label          ; N=V 时跳转 (大于等于)
```

**条件码后缀：**

| 后缀 | 条件 | 标志位 |
|------|------|--------|
| EQ | Equal | Z=1 |
| NE | Not Equal | Z=0 |
| CS/HS | Carry Set/Unsigned Higher or Same | C=1 |
| CC/LO | Carry Clear/Unsigned Lower | C=0 |
| MI | Minus/Negative | N=1 |
| PL | Plus/Positive or Zero | N=0 |
| VS | Overflow | V=1 |
| VC | No Overflow | V=0 |
| HI | Unsigned Higher | C=1 && Z=0 |
| LS | Unsigned Lower or Same | C=0 \|\| Z=1 |
| GE | Signed Greater or Equal | N=V |
| LT | Signed Less Than | N≠V |
| GT | Signed Greater Than | Z=0 && N=V |
| LE | Signed Less or Equal | Z=1 \|\| N≠V |

### 2.4 Thumb 指令集

Cortex-M 处理器只支持 Thumb-2 指令集，是 16 位和 32 位指令的混合：

```asm
; 16位Thumb指令
MOVS  R0, #100       ; 16位：立即数范围受限
ADDS  R0, R1, R2     ; 16位：默认更新标志位
PUSH  {R4, R5}       ; 16位

; 32位Thumb-2指令
MOVW  R0, #0x5678    ; 32位：加载16位立即数
MOVT  R0, #0x1234    ; 32位：加载高16位
; 结果: R0 = 0x12345678
```

---

## 三、STM32 软件设计

### 3.1 启动流程

STM32 启动过程：

```
┌──────────────────┐
│   上电复位       │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 从 0x00000000    │
│ 读取 MSP 初始值  │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 从 0x00000004    │
│ 读取 Reset_Handler│
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 执行启动代码     │
│ (startup_xxx.s)  │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 初始化 .data 段  │
│ 清零 .bss 段     │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 调用 SystemInit()│
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 调用 main()      │
└──────────────────┘
```

**启动文件分析 (startup_stm32f407xx.s)：**

```asm
; 向量表
__Vectors       DCD     __initial_sp          ; Top of Stack
                DCD     Reset_Handler         ; Reset Handler
                DCD     NMI_Handler           ; NMI Handler
                DCD     HardFault_Handler     ; Hard Fault Handler
                DCD     MemManage_Handler     ; MPU Fault Handler
                DCD     BusFault_Handler      ; Bus Fault Handler
                DCD     UsageFault_Handler    ; Usage Fault Handler
                ; ... 其他异常和中断

; 复位处理
Reset_Handler   PROC
                EXPORT  Reset_Handler   [WEAK]
                IMPORT  SystemInit
                IMPORT  __main
                
                LDR     R0, =SystemInit
                BLX     R0
                LDR     R0, =__main
                BX      R0
                ENDP
```

### 3.2 内存布局

**链接脚本 (Linker Script) 定义的内存区域：**

```
STM32F407 内存布局:
┌─────────────────────────────────────────┐ 0x1FFF FFFF
│           System Memory (Bootloader)    │ 60KB
├─────────────────────────────────────────┤ 0x1FFF 0000
│           Reserved                      │
├─────────────────────────────────────────┤ 0x2003 FFFF
│           SRAM2 (16KB)                  │
├─────────────────────────────────────────┤ 0x2002 C000
│           SRAM1 (112KB)                 │
├─────────────────────────────────────────┤ 0x2000 0000
│           Peripherals                   │
├─────────────────────────────────────────┤ 0x4000 0000
│           FSMC Bank1/2                  │
├─────────────────────────────────────────┤ 0x6000 0000
│           External RAM                  │
├─────────────────────────────────────────┤ 0xA000 0000
│           CCM RAM (64KB)                │
├─────────────────────────────────────────┤ 0x1000 0000
│           Flash (512KB/1MB)             │
│   ┌─────────────────────────────────┐   │
│   │ .isr_vector (中断向量表)         │   │
│   ├─────────────────────────────────┤   │
│   │ .text (代码段)                   │   │
│   ├─────────────────────────────────┤   │
│   │ .rodata (只读数据)               │   │
│   ├─────────────────────────────────┤   │
│   │ .data (初始化数据，运行时复制)   │   │
│   ├─────────────────────────────────┤   │
│   │ .bss (未初始化数据，运行时清零)  │   │
│   └─────────────────────────────────┘   │
└─────────────────────────────────────────┘ 0x0800 0000
```

### 3.3 中断处理

**NVIC (Nested Vectored Interrupt Controller)：**

```c
// 中断优先级配置
void NVIC_Configuration(void)
{
    NVIC_InitTypeDef NVIC_InitStruct;
    
    // 配置 USART1 中断
    NVIC_InitStruct.NVIC_IRQChannel = USART1_IRQn;
    NVIC_InitStruct.NVIC_IRQChannelPreemptionPriority = 1;  // 抢占优先级
    NVIC_InitStruct.NVIC_IRQChannelSubPriority = 0;         // 子优先级
    NVIC_InitStruct.NVIC_IRQChannelCmd = ENABLE;
    NVIC_Init(&NVIC_InitStruct);
    
    // 优先级分组
    NVIC_PriorityGroupConfig(NVIC_PriorityGroup_4);  // 4位抢占优先级，0位子优先级
}
```

**中断服务程序：**

```c
// 在 stm32f4xx_it.c 中
void USART1_IRQHandler(void)
{
    // 检查接收中断标志
    if (USART_GetITStatus(USART1, USART_IT_RXNE) != RESET)
    {
        uint8_t data = USART_ReceiveData(USART1);
        // 处理接收数据...
    }
    
    // 检查发送完成中断
    if (USART_GetITStatus(USART1, USART_IT_TC) != RESET)
    {
        USART_ClearITPendingBit(USART1, USART_IT_TC);
        // 发送完成处理...
    }
}
```

**中断嵌套原则：**

```
优先级规则:
- 抢占优先级高的可以打断抢占优先级低的
- 抢占优先级相同，子优先级高的不能打断低的
- 数值越小优先级越高

示例 (NVIC_PriorityGroup_2):
┌────────────────────────────────────────┐
│ 中断A: 抢占=0, 子=1  (最高优先级)       │
│ 中断B: 抢占=0, 子=2                     │
│ 中断C: 抢占=1, 子=0  (最低优先级)       │
└────────────────────────────────────────┘

- A 可以打断 B 和 C
- B 不能打断 A（抢占相同，子优先级低）
- B 可以打断 C（抢占优先级更高）
```

---

## 四、外设编程

### 4.1 GPIO 操作

**寄存器直接操作：**

```c
// GPIO 寄存器结构
typedef struct
{
    volatile uint32_t MODER;    // 模式寄存器
    volatile uint32_t OTYPER;   // 输出类型寄存器
    volatile uint32_t OSPEEDR;  // 输出速度寄存器
    volatile uint32_t PUPDR;    // 上拉/下拉寄存器
    volatile uint32_t IDR;      // 输入数据寄存器
    volatile uint32_t ODR;      // 输出数据寄存器
    volatile uint32_t BSRR;     // 置位/复位寄存器
    volatile uint32_t LCKR;     // 配置锁定寄存器
    volatile uint32_t AFR[2];   // 复用功能寄存器
} GPIO_TypeDef;

// GPIO 初始化
void GPIO_Init(void)
{
    // 使能 GPIOA 时钟
    RCC->AHB1ENR |= RCC_AHB1ENR_GPIOAEN;
    
    // 配置 PA5 为输出模式 (LED)
    GPIOA->MODER &= ~(3U << (5 * 2));   // 清除位
    GPIOA->MODER |= (1U << (5 * 2));    // 通用输出模式
    
    GPIOA->OTYPER &= ~(1U << 5);        // 推挽输出
    GPIOA->OSPEEDR |= (3U << (5 * 2));  // 高速
    GPIOA->PUPDR &= ~(3U << (5 * 2));   // 无上拉下拉
}

// GPIO 输出控制
#define LED_ON()   GPIOA->BSRR = GPIO_BSRR_BS_5   // 置位
#define LED_OFF()  GPIOA->BSRR = GPIO_BSRR_BR_5   // 复位
#define LED_TOGGLE() GPIOA->ODR ^= GPIO_ODR_OD_5  // 翻转
```

**位带操作 (Bit-Banding)：**

```c
// 位带区域映射
// 外设位带区: 0x40000000-0x400FFFFF -> 0x42000000-0x43FFFFFF
// SRAM位带区:  0x20000000-0x200FFFFF -> 0x22000000-0x23FFFFFF

#define BITBAND(addr, bit) \
    ((volatile uint32_t*)(0x42000000 + ((uint32_t)(addr) - 0x40000000) * 32 + (bit) * 4))

// 位带操作宏
#define PA5_OUT  BITBAND(&GPIOA->ODR, 5)

// 使用
*PA5_OUT = 1;  // PA5 输出高电平
*PA5_OUT = 0;  // PA5 输出低电平

// 位带操作的汇编优势:
// 普通操作: 读-改-写 (3条指令)
// 位带操作: 直接写 (1条指令)
```

### 4.2 定时器编程

**基本定时器配置：**

```c
void TIM6_Init(void)
{
    // 使能 TIM6 时钟
    RCC->APB1ENR |= RCC_APB1ENR_TIM6EN;
    
    // 配置预分频器和自动重载值
    // 假设 APB1 时钟 84MHz, 目标 1ms 中断
    TIM6->PSC = 8400 - 1;   // 预分频: 84MHz / 8400 = 10kHz
    TIM6->ARR = 10 - 1;     // 自动重载: 10kHz / 10 = 1kHz (1ms)
    
    // 使能更新中断
    TIM6->DIER |= TIM_DIER_UIE;
    
    // 配置 NVIC
    NVIC_EnableIRQ(TIM6_DAC_IRQn);
    NVIC_SetPriority(TIM6_DAC_IRQn, 2);
    
    // 启动定时器
    TIM6->CR1 |= TIM_CR1_CEN;
}

// 定时器中断服务程序
void TIM6_DAC_IRQHandler(void)
{
    if (TIM6->SR & TIM_SR_UIF)
    {
        TIM6->SR &= ~TIM_SR_UIF;  // 清除中断标志
        // 1ms 定时任务...
        ms_ticks++;
    }
}
```

**PWM 输出：**

```c
void TIM3_PWM_Init(void)
{
    // 使能时钟
    RCC->APB1ENR |= RCC_APB1ENR_TIM3EN;
    RCC->AHB1ENR |= RCC_AHB1ENR_GPIOAEN;
    
    // 配置 PA6 为 TIM3_CH1 复用功能
    GPIOA->MODER &= ~(3U << (6 * 2));
    GPIOA->MODER |= (2U << (6 * 2));      // 复用模式
    GPIOA->AFR[0] |= (2U << (6 * 4));     // AF2 (TIM3)
    
    // TIM3 配置: PWM 模式 1
    TIM3->PSC = 84 - 1;          // 84MHz / 84 = 1MHz
    TIM3->ARR = 1000 - 1;        // PWM 频率: 1MHz / 1000 = 1kHz
    TIM3->CCR1 = 500;            // 占空比 50%
    
    // PWM 模式 1 配置
    TIM3->CCMR1 &= ~TIM_CCMR1_OC1M;
    TIM3->CCMR1 |= (6U << TIM_CCMR1_OC1M_Pos);  // PWM 模式 1
    TIM3->CCMR1 |= TIM_CCMR1_OC1PE;              // 使能预装载
    
    // 使能输出
    TIM3->CCER |= TIM_CCER_CC1E;
    
    // 启动定时器
    TIM3->CR1 |= TIM_CR1_CEN;
}

// 设置 PWM 占空比
void PWM_SetDuty(uint16_t duty)
{
    TIM3->CCR1 = duty;
}
```

### 4.3 DMA 传输

**DMA 配置：**

```c
void DMA_USART_TX_Init(void)
{
    // 使能 DMA2 时钟
    RCC->AHB1ENR |= RCC_AHB1ENR_DMA2EN;
    
    // 配置 DMA2 Stream7 (USART1 TX)
    DMA2_Stream7->CR = 0;  // 先禁用
    
    DMA2_Stream7->PAR = (uint32_t)&USART1->DR;     // 外设地址
    DMA2_Stream7->M0AR = (uint32_t)tx_buffer;       // 内存地址
    DMA2_Stream7->NDTR = TX_BUFFER_SIZE;            // 传输长度
    
    // 配置控制寄存器
    DMA2_Stream7->CR |= (4U << DMA_SxCR_CHSEL_Pos);  // 通道 4
    DMA2_Stream7->CR |= DMA_SxCR_MINC;               // 内存地址递增
    DMA2_Stream7->CR |= DMA_SxCR_DIR_0;              // 内存到外设
    DMA2_Stream7->CR |= DMA_SxCR_TCIE;               // 传输完成中断
    
    // 使能 DMA
    DMA2_Stream7->CR |= DMA_SxCR_EN;
    
    // 使能 USART DMA 发送
    USART1->CR3 |= USART_CR3_DMAT;
}
```

---

## 五、实时操作系统集成

### 5.1 FreeRTOS 移植

**FreeRTOS 配置 (FreeRTOSConfig.h)：**

```c
#ifndef FREERTOS_CONFIG_H
#define FREERTOS_CONFIG_H

// 内核配置
#define configUSE_PREEMPTION                    1
#define configUSE_IDLE_HOOK                     0
#define configUSE_TICK_HOOK                     0
#define configCPU_CLOCK_HZ                      (168000000UL)
#define configTICK_RATE_HZ                      ((TickType_t)1000)
#define configMAX_PRIORITIES                    (5)
#define configMINIMAL_STACK_SIZE                ((uint16_t)128)
#define configTOTAL_HEAP_SIZE                   ((size_t)(20 * 1024))
#define configMAX_TASK_NAME_LEN                 (16)
#define configUSE_16_BIT_TICKS                  0
#define configIDLE_SHOULD_YIELD                 1

// 内存管理
#define configSUPPORT_STATIC_ALLOCATION         0
#define configSUPPORT_DYNAMIC_ALLOCATION        1

// 互斥量
#define configUSE_MUTEXES                       1
#define configUSE_RECURSIVE_MUTEXES             1
#define configUSE_COUNTING_SEMAPHORES           1

// Cortex-M 特定配置
#define configPRIO_BITS                         4
#define configLIBRARY_LOWEST_INTERRUPT_PRIORITY 15
#define configKERNEL_INTERRUPT_PRIORITY         (configLIBRARY_LOWEST_INTERRUPT_PRIORITY << (8 - configPRIO_BITS))
#define configMAX_SYSCALL_INTERRUPT_PRIORITY    5

#endif
```

**任务创建：**

```c
#include "FreeRTOS.h"
#include "task.h"
#include "queue.h"

// 任务句柄
TaskHandle_t led_task_handle;
TaskHandle_t sensor_task_handle;

// 任务函数
void LED_Task(void *pvParameters)
{
    while (1)
    {
        LED_TOGGLE();
        vTaskDelay(pdMS_TO_TICKS(500));  // 延时 500ms
    }
}

void Sensor_Task(void *pvParameters)
{
    while (1)
    {
        // 读取传感器数据
        float temp = Read_Temperature();
        float humidity = Read_Humidity();
        
        // 发送到队列或处理...
        
        vTaskDelay(pdMS_TO_TICKS(1000));  // 延时 1s
    }
}

int main(void)
{
    // 硬件初始化
    HAL_Init();
    SystemClock_Config();
    GPIO_Init();
    
    // 创建任务
    xTaskCreate(LED_Task, "LED", 128, NULL, 1, &led_task_handle);
    xTaskCreate(Sensor_Task, "Sensor", 256, NULL, 2, &sensor_task_handle);
    
    // 启动调度器
    vTaskStartScheduler();
    
    while (1);
}
```

### 5.2 中断与RTOS

**安全的中断处理：**

```c
// 在中断中使用 FreeRTOS API
void EXTI0_IRQHandler(void)
{
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    
    if (EXTI_GetITStatus(EXTI_Line0) != RESET)
    {
        // 清除中断标志
        EXTI_ClearITPendingBit(EXTI_Line0);
        
        // 给出信号量 (使用 FromISR 版本)
        xSemaphoreGiveFromISR(xButtonSemaphore, &xHigherPriorityTaskWoken);
    }
    
    // 如果唤醒了更高优先级的任务，请求上下文切换
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

---

## 六、调试技巧

### 6.1 使用 GDB 调试

**启动调试会话：**

```bash
# 启动 OpenOCD
openocd -f interface/stlink-v2.cfg -f target/stm32f4x.cfg

# 在另一个终端启动 GDB
arm-none-eabi-gdb firmware.elf

# GDB 命令
(gdb) target remote localhost:3333
(gdb) monitor reset halt
(gdb) load
(gdb) break main
(gdb) continue
```

**常用 GDB 命令：**

```bash
# 断点
break main           # 在 main 函数设置断点
break file.c:100     # 在特定行设置断点
break *0x08001000    # 在特定地址设置断点
info breakpoints     # 查看所有断点
delete 1             # 删除 1 号断点

# 执行控制
step                 # 单步执行 (进入函数)
next                 # 单步执行 (不进入函数)
continue             # 继续执行
finish               # 执行到函数返回

# 查看变量和内存
print variable       # 打印变量值
print/x variable     # 以十六进制打印
print *pointer       # 打印指针指向的值
x/10x 0x20000000     # 查看内存 (10个十六进制值)

# 寄存器
info registers       # 显示所有寄存器
print $sp            # 打印堆栈指针
print $pc            # 打印程序计数器

# 堆栈
backtrace            # 显示调用栈
frame 2              # 切换到第 2 层栈帧
```

### 6.2 HardFault 调试

**HardFault 处理程序：**

```c
// 确定故障原因
void HardFault_Handler(void)
{
    __asm volatile
    (
        "TST LR, #4                                    \n"
        "ITE EQ                                        \n"
        "MRSEQ R0, MSP                                 \n"
        "MRSNE R0, PSP                                 \n"
        "B HardFault_C_Handler                         \n"
    );
}

void HardFault_C_Handler(uint32_t *stack)
{
    volatile uint32_t cfsr  = SCB->CFSR;   // 配置故障状态寄存器
    volatile uint32_t hfsr  = SCB->HFSR;   // 硬件故障状态寄存器
    volatile uint32_t mmfar = SCB->MMFAR;  // 内存管理故障地址
    volatile uint32_t bfar  = SCB->BFAR;   // 总线故障地址
    
    volatile uint32_t r0  = stack[0];
    volatile uint32_t r1  = stack[1];
    volatile uint32_t r2  = stack[2];
    volatile uint32_t r3  = stack[3];
    volatile uint32_t r12 = stack[4];
    volatile uint32_t lr  = stack[5];  // 返回地址
    volatile uint32_t pc  = stack[6];  // 故障地址
    volatile uint32_t psr = stack[7];
    
    // 在此处设置断点查看变量
    while (1);
}
```

**故障状态寄存器解析：**

```c
// CFSR (Configurable Fault Status Register) 位定义
// [31:24] MFSR - 内存管理故障状态
// [23:16] BFSR - 总线故障状态  
// [15:8]  UFSR - 用法故障状态

// 常见故障原因:
// MMARVALID (CFSR[7]): 内存管理故障地址有效
// BFARVALID (CFSR[15]): 总线故障地址有效
// INVSTATE (CFSR[17]): 无效状态 (可能是 Thumb 位问题)
// UNDEFINSTR (CFSR[16]): 未定义指令
// INVPC (CFSR[18]): 无效的 PC 加载
```

### 6.3 性能分析

**使用 DWT 周期计数器：**

```c
// 启用 DWT
#define DWT_CYCCNT  (*((volatile uint32_t *)0xE0001004))
#define DWT_CONTROL (*((volatile uint32_t *)0xE0001000))

void DWT_Init(void)
{
    CoreDebug->DEMCR |= CoreDebug_DEMCR_TRCENA_Msk;
    DWT_CONTROL |= 1;  // 使能周期计数器
    DWT_CYCCNT = 0;
}

// 测量代码执行时间
uint32_t start, end, cycles;

DWT_Init();
start = DWT_CYCCNT;

// 要测量的代码
Process_Data();

end = DWT_CYCCNT;
cycles = end - start;

// 转换为时间 (假设 168MHz)
float time_us = (float)cycles / 168.0f;
```

---

## 七、优化技巧

### 7.1 内存优化

**使用 CCM RAM (Core-Coupled Memory)：**

```c
// 在链接脚本中定义 CCM RAM 区域
// .ccmram (NOLOAD) :
// {
//     . = ALIGN(4);
//     *(.ccmram)
//     . = ALIGN(4);
// } >CCM_RAM

// 将变量放置到 CCM RAM
__attribute__((section(".ccmram"))) uint8_t dma_buffer[1024];

// 注意: CCM RAM 不能被 DMA 访问!
```

**堆栈优化：**

```c
// 在链接脚本中调整堆栈大小
_stack_size = 0x1000;  // 4KB 堆栈

// 检测堆栈溢出
#define STACK_CANARY 0xDEADBEEF

void Stack_Init(void)
{
    extern uint32_t _estack;
    uint32_t *ptr = &_estack - STACK_SIZE/4;
    
    while (ptr < &_estack)
    {
        *ptr++ = STACK_CANARY;
    }
}

uint32_t Stack_Check(void)
{
    extern uint32_t _estack;
    uint32_t *ptr = &_estack - STACK_SIZE/4;
    
    while (ptr < &_estack)
    {
        if (*ptr != STACK_CANARY)
            return (uint32_t)ptr;  // 返回溢出位置
        ptr++;
    }
    return 0;  // 无溢出
}
```

### 7.2 代码优化

**内联关键函数：**

```c
static inline void GPIO_SetHigh(GPIO_TypeDef *GPIO, uint8_t pin)
{
    GPIO->BSRR = (1U << pin);
}

static inline void GPIO_SetLow(GPIO_TypeDef *GPIO, uint8_t pin)
{
    GPIO->BSRR = (1U << (pin + 16));
}
```

**使用 DMA 减少CPU负载：**

```c
// ADC + DMA 连续采样
void ADC_DMA_Init(void)
{
    // 使能时钟
    RCC->AHB1ENR |= RCC_AHB1ENR_GPIOAEN | RCC_AHB1ENR_DMA2EN;
    RCC->APB2ENR |= RCC_APB2ENR_ADC1EN;
    
    // 配置 DMA
    DMA2_Stream0->PAR = (uint32_t)&ADC1->DR;
    DMA2_Stream0->M0AR = (uint32_t)adc_buffer;
    DMA2_Stream0->NDTR = ADC_BUFFER_SIZE;
    DMA2_Stream0->CR = DMA_SxCR_CHSEL_0 | DMA_SxCR_MINC | DMA_SxCR_CIRC;
    DMA2_Stream0->CR |= DMA_SxCR_EN;
    
    // 配置 ADC
    ADC1->CR2 = ADC_CR2_ADON | ADC_CR2_DMA | ADC_CR2_DDS;
    ADC1->CR2 |= ADC_CR2_SWSTART;  // 启动连续转换
}
```

---

## 八、总结

### 8.1 ARM 指令集要点

| 类别 | 重要指令 | 用途 |
|------|----------|------|
| 数据处理 | ADD, SUB, MUL, AND, ORR | 算术逻辑运算 |
| Load/Store | LDR, STR, LDM, STM | 内存访问 |
| 分支 | B, BL, BX, BEQ, BNE | 流程控制 |
| 状态 | MRS, MSR, CPSID, CPSIE | 状态和中断控制 |

### 8.2 STM32 开发流程

1. **理解硬件**：阅读数据手册和参考手册
2. **配置时钟**：SystemInit 或 HAL_RCC_OscConfig
3. **初始化外设**：GPIO, UART, SPI, I2C 等
4. **编写应用**：主循环或 RTOS 任务
5. **调试测试**：使用 GDB、逻辑分析仪
6. **优化完善**：性能、功耗、代码大小

### 8.3 最佳实践

- 使用 CMSIS 标准库提高可移植性
- 合理设置中断优先级避免优先级反转
- 使用 DMA 减少 CPU 负载
- 启用看门狗保证系统可靠性
- 做好错误处理和异常恢复

---

## 参考资料

- ARM Cortex-M4 Technical Reference Manual
- STM32F4xx Reference Manual (RM0090)
- STM32F4xx Programming Manual (PM0214)
- Joseph Yiu, "The Definitive Guide to ARM Cortex-M3 and Cortex-M4 Processors"
- FreeRTOS Documentation