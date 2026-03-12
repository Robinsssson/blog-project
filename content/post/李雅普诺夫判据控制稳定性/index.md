+++
date = 2026-03-12T13:38:00+08:00
draft = false
title = '李雅普诺夫判据控制稳定性'
categories = ['信息技术']
tags = ['控制理论', '稳定性分析', '李雅普诺夫']
math = true
+++

## 概述

李雅普诺夫稳定性理论是分析非线性系统稳定性的核心工具，由俄国数学家亚历山大·李雅普诺夫于1892年提出。它提供了判断系统稳定性的充分条件，是控制理论中最基础且最重要的理论之一。

---

## 一、稳定性定义

### 1.1 平衡点

考虑非线性系统：

$$
\dot{x} = f(x), \quad x \in \mathbb{R}^n
$$

若 $x_e$ 满足 $f(x_e) = 0$，则称 $x_e$ 为系统的**平衡点**。

通常通过坐标变换将平衡点移至原点，即研究 $x_e = 0$ 的稳定性。

### 1.2 李雅普诺夫稳定性定义

设 $x_e = 0$ 是平衡点：

| 稳定性类型 | 定义 |
|-----------|------|
| **李雅普诺夫稳定** | $\forall \epsilon > 0, \exists \delta > 0$，当 $\|x(0)\| < \delta$ 时，$\|x(t)\| < \epsilon, \forall t \geq 0$ |
| **渐近稳定** | 李雅普诺夫稳定 + $\lim_{t \to \infty} x(t) = 0$ |
| **指数稳定** | 存在 $\alpha, \lambda > 0$，使 $\|x(t)\| \leq \alpha \|x(0)\| e^{-\lambda t}$ |
| **全局渐近稳定** | 渐近稳定 + 对任意初始条件成立 |
| **不稳定** | 不满足李雅普诺夫稳定 |

### 1.3 几何解释

- **李雅普诺夫稳定**：从平衡点附近出发，轨迹始终在平衡点附近
- **渐近稳定**：轨迹最终收敛到平衡点
- **指数稳定**：收敛速度有指数下界

---

## 二、李雅普诺夫第一方法（间接法）

### 2.1 基本思想

将非线性系统在平衡点附近线性化，通过线性化系统的稳定性推断原系统的稳定性。

### 2.2 线性化

对于系统 $\dot{x} = f(x)$，在平衡点 $x_e$ 处线性化：

$$
\dot{\tilde{x}} = A\tilde{x}, \quad A = \left.\frac{\partial f}{\partial x}\right|_{x=x_e}
$$

其中 $\tilde{x} = x - x_e$ 为偏差变量。

### 2.3 判据

**定理（李雅普诺夫间接法）**：
- 若 $A$ 的所有特征值具有负实部，则原系统在 $x_e$ 处**渐近稳定**
- 若 $A$ 至少有一个特征值具有正实部，则原系统在 $x_e$ 处**不稳定**
- 若 $A$ 有零实部特征值，无法通过线性化判断（临界情况）

### 2.4 示例：单摆

单摆方程：

$$
\ddot{\theta} + \frac{g}{L}\sin\theta = 0
$$

状态方程：$x_1 = \theta, x_2 = \dot{\theta}$

$$
\begin{cases}
\dot{x}_1 = x_2 \\
\dot{x}_2 = -\frac{g}{L}\sin x_1
\end{cases}
$$

平衡点：$(0, 0)$ 和 $(\pi, 0)$

雅可比矩阵：

$$
A = \begin{bmatrix} 0 & 1 \\ -\frac{g}{L}\cos x_1 & 0 \end{bmatrix}
$$

**在 $(0, 0)$ 处**：

$$
A = \begin{bmatrix} 0 & 1 \\ -\frac{g}{L} & 0 \end{bmatrix}, \quad \lambda = \pm j\sqrt{\frac{g}{L}}
$$

特征值为纯虚数 → 临界情况（第一方法失效）

**在 $(\pi, 0)$ 处**：

$$
A = \begin{bmatrix} 0 & 1 \\ \frac{g}{L} & 0 \end{bmatrix}, \quad \lambda = \pm \sqrt{\frac{g}{L}}
$$

一个特征值为正 → **不稳定**

---

## 三、李雅普诺夫第二方法（直接法）

### 3.1 基本思想

不求解微分方程，构造一个"能量函数"（李雅普诺夫函数），通过该函数的性质判断稳定性。

### 3.2 李雅普诺夫函数

函数 $V(x): \mathbb{R}^n \to \mathbb{R}$ 称为李雅普诺夫函数候选者，若：

1. $V(0) = 0$
2. $V(x) > 0, \forall x \neq 0$（正定）
3. $V(x) \to \infty$ 当 $\|x\| \to \infty$（径向无界，全局稳定需要）

### 3.3 判据定理

**定理（李雅普诺夫稳定性定理）**：

设 $x_e = 0$ 是平衡点，$V(x)$ 是连续可微函数。

| 条件 | 结论 |
|------|------|
| $V(x)$ 正定，$\dot{V}(x) \leq 0$ | 李雅普诺夫稳定 |
| $V(x)$ 正定，$\dot{V}(x)$ 负定 | 渐近稳定 |
| $V(x)$ 正定且径向无界，$\dot{V}(x)$ 负定 | 全局渐近稳定 |
| $\dot{V}(x) > 0$（在某区域内） | 不稳定 |

### 3.4 物理意义

李雅普诺夫函数可视为"广义能量"：
- 系统能量有界 → 轨迹有界（稳定）
- 系统能量持续衰减 → 轨迹收敛（渐近稳定）

### 3.5 示例：单摆（重访）

取李雅普诺夫函数为机械能：

$$
V(x) = \frac{1}{2}mL^2 x_2^2 + mgL(1 - \cos x_1)
$$

验证：
- $V(0) = 0$
- $V(x) > 0$ 当 $x \neq 0$（在 $|x_1| < 2\pi$ 范围内）
- $\dot{V} = mL^2 x_2 \dot{x}_2 + mgL\sin x_1 \cdot x_1 = 0$（无阻尼，能量守恒）

结论：$\dot{V} = 0$ → **李雅普诺夫稳定**（非渐近稳定）

若加入阻尼：$\dot{x}_2 = -\frac{g}{L}\sin x_1 - \frac{c}{m}x_2$

$$
\dot{V} = -cL x_2^2 \leq 0
$$

由LaSalle不变集定理可证渐近稳定。

---

## 四、LaSalle不变集定理

### 4.1 问题背景

当 $\dot{V}(x) \leq 0$（非负定）时，李雅普诺夫定理只能判断稳定，无法判断渐近稳定。LaSalle定理解决了这个问题。

### 4.2 不变集定义

集合 $M$ 称为不变集，若从 $M$ 中出发的轨迹始终停留在 $M$ 中。

### 4.3 LaSalle定理

**定理**：设 $\Omega$ 为有界正不变集，$V(x)$ 在 $\Omega$ 上连续可微，$\dot{V}(x) \leq 0$ 在 $\Omega$ 内成立。

令 $E = \{x \in \Omega : \dot{V}(x) = 0\}$，$M$ 为 $E$ 中最大不变集。

则从 $\Omega$ 中出发的每条轨迹都趋于 $M$。

### 4.4 应用示例

带阻尼的单摆系统，$E = \{x_2 = 0\}$。

在 $E$ 上，$\dot{x}_2 = -\frac{g}{L}\sin x_1$。

要使轨迹停留在 $E$，需 $\dot{x}_2 = 0$，即 $\sin x_1 = 0$，即 $x_1 = 0, \pi, 2\pi, ...$

最大不变集 $M = \{(0, 0), (\pi, 0), ...\}$

在原点附近，唯一可能的不变点是 $(0, 0)$，故局部渐近稳定。

---

## 五、线性系统的李雅普诺夫分析

### 5.1 线性系统的李雅普诺夫方程

对于线性系统 $\dot{x} = Ax$，构造二次型李雅普诺夫函数：

$$
V(x) = x^T P x
$$

其中 $P$ 为正定矩阵。

导数：

$$
\dot{V} = x^T(A^TP + PA)x
$$

### 5.2 李雅普诺夫方程

系统渐近稳定 $\Leftrightarrow$ 对于任意正定 $Q$，方程

$$
A^TP + PA = -Q
$$

有唯一正定解 $P$。

### 5.3 求解方法

对于给定的稳定 $A$ 和正定 $Q$：

```python
import numpy as np
from scipy import linalg

A = np.array([[-1, 1], [-1, -1]])
Q = np.eye(2)

# 求解李雅普诺夫方程 A^T P + P A = -Q
P = linalg.solve_continuous_lyapunov(A, Q)

# 验证 P 正定
print("P =")
print(P)
print("\nP的特征值:", np.linalg.eigvals(P))
print("P是否正定:", np.all(np.linalg.eigvals(P) > 0))
```

---

## 六、构造李雅普诺夫函数的方法

### 6.1 能量函数法

对于机械系统，直接取机械能：

$$
V = E_k + E_p = \frac{1}{2}\dot{q}^T M(q) \dot{q} + P(q)
$$

### 6.2 二次型法

对于线性系统或线性化系统：

$$
V = x^T P x
$$

通过求解李雅普诺夫方程确定 $P$。

### 6.3 变量梯度法

假设：

$$
V = \int_0^x \nabla V \cdot dx
$$

选择 $\nabla V$ 的形式，利用 $\frac{\partial^2 V}{\partial x_i \partial x_j} = \frac{\partial^2 V}{\partial x_j \partial x_i}$ 确定系数。

### 6.4 Krasovskii方法

对于系统 $\dot{x} = f(x)$，取：

$$
V = f^T f
$$

适用于 $f(0) = 0$ 且 $f$ 在平衡点附近满足某些条件的情况。

### 6.5 反推法（Backstepping）

适用于严格反馈形式的系统，逐步构造李雅普诺夫函数。

---

## 七、应用实例

### 7.1 倒立摆平衡控制

状态方程：

$$
\begin{cases}
\dot{x}_1 = x_2 \\
\dot{x}_2 = \frac{g}{L}\sin x_1 - \frac{u}{mL^2}\cos x_1
\end{cases}
$$

控制目标：$x_1 \to 0$

李雅普诺夫函数：

$$
V = \frac{1}{2}x_1^2 + \frac{1}{2}x_2^2
$$

设计控制律使 $\dot{V} < 0$。

### 7.2 机器人轨迹跟踪

跟踪误差动力学：

$$
\dot{e} = f(e, u)
$$

构造 $V(e)$，设计控制 $u$ 使 $\dot{V} < 0$。

### 7.3 电力系统稳定性

发电机摇摆方程：

$$
M\ddot{\delta} + D\dot{\delta} = P_m - P_e\sin\delta
$$

能量函数法判断暂态稳定性。

---

## 八、李雅普诺夫方法的局限性

| 局限性 | 说明 |
|--------|------|
| 保守性 | 只是充分条件，找到李雅普诺夫函数不能保证全局最优 |
| 构造困难 | 没有通用的构造方法，依赖经验 |
| 临界情况 | 第一方法在边界情况失效 |
| 非局部结论 | 第二方法通常只给出局部结论 |

---

## 九、稳定性分析流程

```
1. 确定平衡点
2. 线性化（若可能）
3. 计算雅可比矩阵特征值
4. 若特征值有正实部 → 不稳定
5. 若特征值全负实部 → 渐近稳定
6. 若有零实部特征值 → 使用直接法
7. 构造李雅普诺夫函数
8. 验证 V 正定，V̇ 负定
9. 若 V̇ ≤ 0，使用 LaSalle 定理
```

---

## 十、Python工具

### 10.1 判断线性系统稳定性

```python
import numpy as np
from scipy import linalg

def is_stable(A):
    """判断线性系统 dx = Ax 的稳定性"""
    eigvals = np.linalg.eigvals(A)
    real_parts = np.real(eigvals)
    
    if np.all(real_parts < 0):
        return "渐近稳定"
    elif np.any(real_parts > 0):
        return "不稳定"
    else:
        return "临界情况"

# 示例
A = np.array([[-1, 0], [0, -2]])
print(f"系统: {is_stable(A)}")
```

### 10.2 求解李雅普诺夫方程

```python
# 连续时间
P = linalg.solve_continuous_lyapunov(A, Q)

# 离散时间
P = linalg.solve_discrete_lyapunov(A, Q)
```

### 10.3 绘制相轨迹

```python
import matplotlib.pyplot as plt

def phase_portrait(f, x_range, y_range, grid=20):
    """绘制相轨迹"""
    x = np.linspace(x_range[0], x_range[1], grid)
    y = np.linspace(y_range[0], y_range[1], grid)
    X, Y = np.meshgrid(x, y)
    
    U, V = np.zeros_like(X), np.zeros_like(Y)
    for i in range(grid):
        for j in range(grid):
            dx = f(np.array([X[i,j], Y[i,j]]))
            U[i,j] = dx[0]
            V[i,j] = dx[1]
    
    plt.quiver(X, Y, U, V)
    plt.xlabel('x1')
    plt.ylabel('x2')
    plt.show()
```

---

## 总结

| 方法 | 适用范围 | 优点 | 缺点 |
|------|----------|------|------|
| 第一方法（间接法） | 可线性化系统 | 计算简单 | 临界情况失效，仅局部 |
| 第二方法（直接法） | 一般非线性系统 | 适用范围广 | 构造函数困难 |
| LaSalle定理 | $\dot{V} \leq 0$ 情况 | 可证渐近稳定 | 需分析不变集 |

**核心思想**：稳定性 = 能量有界且衰减

---

## 参考资料

- 《Nonlinear Systems》- Hassan K. Khalil
- 《应用非线性控制》- Slotine & Li
- 《自动控制原理》- 胡寿松
- [李雅普诺夫稳定性理论](https://en.wikipedia.org/wiki/Lyapunov_stability)

---

> 🎯 李雅普诺夫理论是非线性控制的基石，掌握它是理解现代控制理论的关键！