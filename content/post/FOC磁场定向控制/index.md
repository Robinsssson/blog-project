+++
date = 2026-03-12T13:38:00+08:00
draft = false
title = 'FOC磁场定向控制'
categories = ['信息技术']
tags = ['电机控制', 'FOC', '矢量控制', 'PMSM', 'BLDC']
math = true
+++

## 概述

FOC（Field-Oriented Control，磁场定向控制），又称矢量控制，是一种高性能交流电机控制技术。它通过坐标变换，将交流电机的多变量、强耦合系统解耦为类似直流电机的独立控制结构，实现对转矩和磁通的独立控制。

---

## 一、为什么需要FOC

### 1.1 传统控制的问题

| 控制方式 | 问题 |
|----------|------|
| 标量控制（V/f） | 动态响应慢，转矩控制精度低 |
| BLDC方波驱动 | 转矩脉动大，噪音大 |
| 直接转矩控制 | 转矩脉动，低速性能差 |

### 1.2 FOC的优势

- **转矩控制精度高**：独立控制转矩和磁通
- **动态响应快**：毫秒级响应
- **效率高**：最优电流矢量控制
- **低速性能好**：平滑运行
- **噪音低**：正弦波驱动

---

## 二、电机模型基础

### 2.1 PMSM结构

永磁同步电机（PMSM）结构：
- **定子**：三相绕组，空间分布120°
- **转子**：永磁体产生转子磁场

### 2.2 三相坐标系（abc）

定子三相电流：

$$
i_a = I_m \cos(\omega t)
$$

$$
i_b = I_m \cos(\omega t - \frac{2\pi}{3})
$$

$$
i_c = I_m \cos(\omega t + \frac{2\pi}{3})
$$

三相坐标系下的电机方程：

$$
\begin{bmatrix} u_a \\ u_b \\ u_c \end{bmatrix} = R_s \begin{bmatrix} i_a \\ i_b \\ i_c \end{bmatrix} + \frac{d}{dt} \begin{bmatrix} \psi_a \\ \psi_b \\ \psi_c \end{bmatrix}
$$

### 2.3 问题描述

三相系统的问题：
- 三相电流相互耦合
- 时变参数（电感随转子位置变化）
- 控制复杂

---

## 三、坐标变换

FOC的核心是坐标变换，将三相交流量转换为直流量。

### 3.1 Clarke变换（abc → αβ）

将三相静止坐标系转换为两相静止坐标系：

$$
\begin{bmatrix} i_\alpha \\ i_\beta \\ i_0 \end{bmatrix} = \frac{2}{3} \begin{bmatrix} 1 & -\frac{1}{2} & -\frac{1}{2} \\ 0 & \frac{\sqrt{3}}{2} & -\frac{\sqrt{3}}{2} \\ \frac{1}{2} & \frac{1}{2} & \frac{1}{2} \end{bmatrix} \begin{bmatrix} i_a \\ i_b \\ i_c \end{bmatrix}
$$

对于平衡系统，$i_0 = 0$，简化为：

$$
i_\alpha = i_a
$$

$$
i_\beta = \frac{1}{\sqrt{3}}(i_a + 2i_b)
$$

### 3.2 Park变换（αβ → dq）

将两相静止坐标系转换为两相旋转坐标系：

$$
\begin{bmatrix} i_d \\ i_q \end{bmatrix} = \begin{bmatrix} \cos\theta & \sin\theta \\ -\sin\theta & \cos\theta \end{bmatrix} \begin{bmatrix} i_\alpha \\ i_\beta \end{bmatrix}
$$

其中 $\theta$ 是转子电角度。

### 3.3 反变换

**逆Park变换（dq → αβ）**：

$$
\begin{bmatrix} i_\alpha \\ i_\beta \end{bmatrix} = \begin{bmatrix} \cos\theta & -\sin\theta \\ \sin\theta & \cos\theta \end{bmatrix} \begin{bmatrix} i_d \\ i_q \end{bmatrix}
$$

**逆Clarke变换（αβ → abc）**：

$$
\begin{bmatrix} i_a \\ i_b \\ i_c \end{bmatrix} = \begin{bmatrix} 1 & 0 \\ -\frac{1}{2} & \frac{\sqrt{3}}{2} \\ -\frac{1}{2} & -\frac{\sqrt{3}}{2} \end{bmatrix} \begin{bmatrix} i_\alpha \\ i_\beta \end{bmatrix}
$$

### 3.4 变换的意义

| 坐标系 | 特点 |
|--------|------|
| abc | 三相静止，交流量，耦合 |
| αβ | 两相静止，交流量，解耦 |
| dq | 两相旋转，直流量，解耦 |

dq坐标系下，$i_d$ 和 $i_q$ 为直流分量，便于PI控制！

---

## 四、dq坐标系下的电机模型

### 4.1 电压方程

$$
u_d = R_s i_d + L_d \frac{di_d}{dt} - \omega_e L_q i_q
$$

$$
u_q = R_s i_q + L_q \frac{di_q}{dt} + \omega_e (L_d i_d + \psi_f)
$$

其中：
- $R_s$：定子电阻
- $L_d, L_q$：d轴、q轴电感（表贴式 $L_d = L_q = L_s$）
- $\omega_e$：电角速度
- $\psi_f$：永磁体磁链

### 4.2 转矩方程

$$
T_e = \frac{3}{2} p \left[ \psi_f i_q + (L_d - L_q) i_d i_q \right]
$$

对于表贴式PMSM（$L_d = L_q$）：

$$
T_e = \frac{3}{2} p \psi_f i_q
$$

**关键结论**：转矩仅与 $i_q$ 成正比！

### 4.3 运动方程

$$
J \frac{d\omega_r}{dt} = T_e - T_L - B \omega_r
$$

其中：
- $J$：转动惯量
- $T_L$：负载转矩
- $B$：阻尼系数
- $\omega_r$：机械角速度，$\omega_e = p \cdot \omega_r$

---

## 五、FOC控制系统结构

### 5.1 系统框图

```
                    ┌─────────────────────────────────────────────────┐
                    │                    FOC 控制器                    │
                    │                                                 │
  θ_ref ──►(+)      │   ┌─────┐   ┌─────┐      ┌──────┐   ┌─────┐   │
            │      │   │速度 │   │  PI  │ id   │      │   │逆变 │   │
            └──►[-]───►│ PI ├───►│控制器├─────►│ SVPWM├──►│   器├───┼──► 电机
            ω_r       │   │     │   │     │      │      │   │     │   │
                      │   └─────┘   └─────┘ iq   │      │   │     │   │
                      │             电流环        │      │   │     │   │
                      │             ┌─────┐       │      │   │     │   │
                      │             │  PI │ ─────►│      │   │     │   │
                      │             │控制器│       │      │   │     │   │
                      │             └─────┘       └──────┘   └─────┘   │
                      │                ▲              ▲                │
                      │                │              │                │
                      │           电流采样      编码器反馈(θ, ω)        │
                      └─────────────────────────────────────────────────┘
```

### 5.2 控制环结构

| 控制环 | 带宽 | 作用 |
|--------|------|------|
| 电流环 | 最高 | 控制 $i_d, i_q$ |
| 速度环 | 中等 | 控制转速 |
| 位置环 | 最低 | 控制位置 |

### 5.3 控制策略

**最大转矩电流比（MTPA）**：
- 表贴式PMSM：$i_d = 0$，$i_q$ 控制转矩
- 内置式PMSM：优化 $i_d, i_q$ 组合

**弱磁控制**：
- 高速时 $i_d < 0$ 减弱磁场
- 扩展速度范围

---

## 六、SVPWM调制

### 6.1 原理

空间矢量PWM（Space Vector PWM）通过合成参考电压矢量，实现逆变器输出控制。

三相逆变器有8种开关状态：

| 状态 | 开关组合 | 空间矢量 |
|------|----------|----------|
| 0 | 000 | V0 (零矢量) |
| 1 | 100 | V1 |
| 2 | 110 | V2 |
| 3 | 010 | V3 |
| 4 | 011 | V4 |
| 5 | 001 | V5 |
| 6 | 101 | V6 |
| 7 | 111 | V7 (零矢量) |

### 6.2 扇区判断

根据参考矢量角度判断所在扇区：

$$
\theta = \arctan\left(\frac{U_\beta}{U_\alpha}\right)
$$

### 6.3 矢量作用时间

设参考矢量 $U_{ref}$ 位于扇区I（V1和V2之间）：

$$
T_1 = \frac{\sqrt{3} T_s}{U_{dc}} U_{ref} \sin\left(\frac{\pi}{3} - \theta\right)
$$

$$
T_2 = \frac{\sqrt{3} T_s}{U_{dc}} U_{ref} \sin\theta
$$

$$
T_0 = T_s - T_1 - T_2
$$

### 6.4 开关序列

七段式开关序列（扇区I）：

```
T0/2 - T1 - T2 - T0 - T2 - T1 - T0/2
 000   100   110  111  110  100   000
```

---

## 七、FOC实现流程

### 7.1 核心步骤

```
1. 电流采样：获取 i_a, i_b（i_c 可计算）
2. 位置检测：编码器获取转子角度 θ
3. Clarke变换：i_abc → i_αβ
4. Park变换：i_αβ → i_dq
5. PI控制：i_d*, i_q* 与实际值比较，输出 u_d, u_q
6. 逆Park变换：u_dq → u_αβ
7. SVPWM：生成PWM驱动逆变器
```

### 7.2 控制周期

| 操作 | 典型时间 |
|------|----------|
| PWM周期 | 50-100 μs (10-20 kHz) |
| 电流采样 | 几 μs |
| 坐标变换 | <1 μs |
| PI计算 | <1 μs |
| SVPWM | 几 μs |

---

## 八、代码实现

### 8.1 坐标变换

```python
import numpy as np

# Clarke变换
def clarke_transform(i_a, i_b, i_c):
    i_alpha = i_a
    i_beta = (i_a + 2 * i_b) / np.sqrt(3)
    return i_alpha, i_beta

# Park变换
def park_transform(i_alpha, i_beta, theta):
    cos_theta = np.cos(theta)
    sin_theta = np.sin(theta)
    i_d = i_alpha * cos_theta + i_beta * sin_theta
    i_q = -i_alpha * sin_theta + i_beta * cos_theta
    return i_d, i_q

# 逆Park变换
def inverse_park(u_d, u_q, theta):
    cos_theta = np.cos(theta)
    sin_theta = np.sin(theta)
    u_alpha = u_d * cos_theta - u_q * sin_theta
    u_beta = u_d * sin_theta + u_q * cos_theta
    return u_alpha, u_beta
```

### 8.2 PI控制器

```python
class PIController:
    def __init__(self, kp, ki, output_limit):
        self.kp = kp
        self.ki = ki
        self.output_limit = output_limit
        self.integral = 0
        self.prev_error = 0
    
    def update(self, setpoint, measured, dt):
        error = setpoint - measured
        self.integral += error * dt
        
        # 抗饱和
        self.integral = np.clip(self.integral, -self.output_limit, self.output_limit)
        
        output = self.kp * error + self.ki * self.integral
        output = np.clip(output, -self.output_limit, self.output_limit)
        
        return output
```

### 8.3 SVPWM

```python
def svpwm(u_alpha, u_beta, u_dc, ts):
    """SVPWM调制"""
    # 计算参考矢量幅值和角度
    u_ref = np.sqrt(u_alpha**2 + u_beta**2)
    theta = np.arctan2(u_beta, u_alpha)
    
    # 扇区判断
    sector = int(theta / (np.pi / 3)) % 6
    
    # 矢量作用时间
    theta_sector = theta - sector * np.pi / 3
    
    t1 = np.sqrt(3) * ts * u_ref / u_dc * np.sin(np.pi / 3 - theta_sector)
    t2 = np.sqrt(3) * ts * u_ref / u_dc * np.sin(theta_sector)
    t0 = ts - t1 - t2
    
    # 占空比计算
    # ... 根据扇区计算各相占空比
    
    return t0, t1, t2, sector
```

---

## 九、无感FOC

### 9.1 挑战

- 低速时反电动势小，估计困难
- 参数变化影响估计精度
- 启动问题

### 9.2 位置估计方法

| 方法 | 原理 | 适用场景 |
|------|------|----------|
| 反电动势观测器 | 利用反电动势估计 | 中高速 |
| 滑模观测器 | 鲁棒性好 | 中高速 |
| 模型参考自适应 | MRAS结构 | 中高速 |
| 高频注入 | 利用凸极效应 | 低速、零速 |
| 扩展卡尔曼滤波 | EKF估计 | 全速范围 |

### 9.3 反电动势观测器

$$
\hat{e}_\alpha = (u_\alpha - R_s i_\alpha) - L_s \frac{di_\alpha}{dt}
$$

$$
\hat{\theta} = -\arctan\left(\frac{\hat{e}_\alpha}{\hat{e}_\beta}\right)
$$

---

## 十、调试要点

### 10.1 参数整定

**电流环PI参数**：
- $K_p = L_s \cdot \omega_c$
- $K_i = R_s \cdot \omega_c$

其中 $\omega_c$ 为电流环带宽（典型值：电角频率的5-10倍）

**速度环PI参数**：
- 根据速度环带宽整定
- 典型带宽：电流环带宽的1/10

### 10.2 常见问题

| 问题 | 可能原因 | 解决方法 |
|------|----------|----------|
| 电流震荡 | PI参数不当 | 减小增益，增加带宽 |
| 转矩脉动 | 角度误差 | 校准编码器，检查对齐 |
| 过流保护 | 启动冲击 | 软启动，限幅 |
| 运行噪音 | PWM频率低 | 提高PWM频率 |

### 10.3 调试工具

- 示波器：观察相电流波形
- 电流探头：检查电流畸变
- 上位机：实时参数调整
- 数据记录：分析动态响应

---

## 总结

| 关键技术 | 作用 |
|----------|------|
| Clarke变换 | 三相→两相静止 |
| Park变换 | 静止→旋转坐标系 |
| PI控制 | 直流量控制 |
| SVPWM | 高效调制 |

**FOC的本质**：通过坐标变换，将交流电机控制转化为直流电机控制问题。

---

## 参考资料

- 《交流电机数学模型及调速系统》- 李华德
- 《永磁同步电机矢量控制》- 王成元
- Texas Instruments: FOC Application Notes
- STMicroelectronics: STM32 Motor Control SDK

---

> 🎯 FOC是现代电机控制的核心技术，掌握它就能驱动高性能电机系统！