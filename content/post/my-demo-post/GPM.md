+++
date = '2025-06-17'
draft = true
title = '高斯伪谱法'
categories = ['轨迹优化']
tags = ['算法']
+++

## 1 计算原理

### 1.1 最优控制问题

在现代控制系统中，常常面临如下问题
$$
\dot{X} = f(X, U, t) \\\\
g(X(0),U(0),t)<0 \ or \ g(X(t_{f}), U(t_{f}), t_{f})<0 \\\\
c(X,U,t) < 0 \\\\
s.t. \ min \ J(X,U,t)
$$

- 第一个方程表示控制系统的状态转移方程，状态量的微分与当前状态和控制变量之间的关系
- 第二个方程表示初和终的约束条件
- 第三个方程为过程约束，即在整个控制过程中存在的强约束
- 第四个为控制系统的设计目标，最小化某个优化指标。

### 1.2 优化指标

从变分学的角度来看，控制系统的本质就是求取一个泛函$X(\cdot),U(\cdot)$使得指标泛函$J(\cdot)$最小。指标泛函有如下表现形式

#### 1.2.1 Mayer型泛函

我们只对最后状态$t=t_{f}$感兴趣，可以定义指标泛函
$$
J_M(X(\cdot), U(\cdot))=h(X(t_f))
$$

- 该指标泛函只与最终状态有关

#### 1.2.2 Lagrange型泛函

假设我们对$t \in [0,\ t_f]$期间的状态和控制感兴趣，则可以定义指标泛函
$$
J_L(X(\cdot), \ U(\cdot))= \int^{t_f}_{0}f^0(t, X(t), U(t)) dt
$$

- 该指标泛函定义了整个时间内的控制状态，在0至$t_f$时间内进行控制

#### 1.2.3 Bolza型泛函

假设我们对$t \in [0,\ t_f]$期间的状态和控制感兴趣也对最后状态$t=t_{f}$感兴趣，则可以定义指标泛函
$$
J_L(X(\cdot), \ U(\cdot))= h(X(t_f)) + \int^{t_f}_{0}f^0(t, X(t), U(t)) dt
$$

## 2 Gauss积分回顾

### 2.1 介绍

数值分析中，有这么一类特殊的积分方式，叫做Gauss积分。通过选取不均匀的积分点实现提高积分精度。通过$n+1$个Gauss积分点，有$2n+1$的积分精度。

不同于传统积分方式，如Newton-Cotes积分法-梯形法则
$$
\int^{x_1}_{x_0}f(x)dx = \frac{h}{2}(f(x_0)+f(x_1))-\frac{h^3}{12}f^{''}(x)
$$
选取两个点只有1阶精度，在数值积分中对该类积分的精度评价是$n+1$个点最多只有$n+1$阶精度。



高斯积分法可通过如下方式表示：
$$
\int_{-1}^{1}f(x)dx=\sum_{i=1}^{n}{c_if(x_i)}
$$
其中$c_i$，$x_i$为gauss积分的点。

### 2.2 Gauss积分点的选取

**定义：**一组在区间$[a,\ b]$上的非0函数$\{ p_0, \cdots, p_n \}$在该区间**正交**，当且仅当：

<!-- $$
\int^b_ap_j(x)p_k(x)dx= \left\{ 
\begin{array}{**lr**} 
0， j \neq k & \\\\
\neq 0, \ j = k & \\\\
\end{array}
\right.
$$ -->

选取一组正交基函数后，对任何一个函数的n阶插值近似，有拉格朗日多项式实现：
$$
Q(x) = \sum_{i=1}^{n}L_i(x)f(x_i)
$$
其中插值点的选取有该正交基的根决定。

有如下**定理**：如果$\{ p_0, \cdots, p_n \}$在$[a,\ b]$区间上是多项式的正交基，并且有$deg \ p_i=i$，则$p_i$在区间$[a, b]$上有i个不同的根。

例如Legendre多项式集为
$$
p_i(x)=\frac{1}{2^ii!}\frac{d^i}{dx^i}[(x^2-1)^i]
$$
就是一组在$[-1, 1]$上的正交多项式。

对拉格朗日插值多项式进行积分，得到如下表达：
$$
\int^1_{-1}\sum_{i=1}^{n}L_i(x)f(x_i)dx=\sum_{i=1}^{n}{c_if(x_i)}
$$
得到：
$$
c_i=\int^1_{-1}L_i(x)dx
$$
其中：
$$
x = root(p_n) \ as \ vector
$$
至于为什么这样选点进行积分就可以实现精度提升，这就是数值分析讨论的问题，这里目前就不研究了。

我们将采用Gauss-Legendre积分法，来求解最优控制问题。

## 3 NLP问题

NLP(Nonlinear programming)问题，即非线性规划问题。

其问题通式有如下表示：
$$
minimize \ f(x) \\\\
subject \ to \ g_i(x) \le 0 \ for \ each \ i \in \{1, \cdots , m\} \\\\
 h_j(x) = 0 \ for \ each \ j \in \{1, \cdots , p\} \\\\
 x \in X.
$$
例如：
$$
最小化\ f(x)=x_1x_2 \\\\
其中满足 \ g_1(x)=-x_1, g_2(x_2) = -x_2 \\\\
x \in R^2
$$
这就是一个简单的非线性规划问题，针对这类问题有多种求解方式，如`MATLAB`中的`fmincon`函数，`IPOPT`求解器，`SNOPT`求解器都是对该类问题进行求解的方式。

而高斯伪谱法就是将最优控制问题转化为求解NLP问题的方法。

## 4 Gauss伪谱法

在这一节进入主题，即如何将最优控制问题转化为NLP问题。

### 4.1 时间区间的变化

原始最优控制问题通常定义在时间区间$t \in [t_0, t_f]$上。

而Gauss-Legendre节点则是属于$\tau \in [-1, 1]$上，因而存在如下时间变换：
$$
t = \frac{(t_f - t_0)τ + (t_f + t_0)}{2} \\\\
\tau = \frac{2t-t_f-t_0}{t_f-t_0}
$$

### 4.1 状态方程的转化

回看这个函数
$$
\dot{X} = f(X, U, t)
$$
修改为：
$$
\frac{dX}{dt}=f(X(t), U(t), t)
$$
经过时间变换
$$
\frac{dX}{d\tau} \frac{d\tau}{dt}=f(X(t), U(t), t) \\\\
\frac{d\tau}{dt}=\frac{2}{t_f - t_0} \\\\
then, \ \frac{dX}{d\tau} = \frac{t_f-t_0}{2}f(X(\tau), U(\tau), \tau)
$$
使用拉格朗日插值多项式表示各个控制状态：
$$
X(\tau) \approx \sum_{i=0}^{n}L_i(\tau)X(\tau_i) \\
\dot{X}(\tau) \approx \sum_{i=0}^{n}\dot{L}_i(\tau)X(\tau_i)\\
U(\tau) \approx \sum_{i=1}^{n}L_i^*(\tau)U(\tau_i) \\
$$
其中，拉格朗日插值多项式有如下表示：
$$
L_i(\tau)= \prod_{j=0,j\neq i}^{n}\frac{\tau-\tau_j}{\tau_i-\tau_j} \\\\
L^*_i(\tau)= \prod_{j=1,j\neq i}^{n}\frac{\tau-\tau_j}{\tau_i-\tau_j} \\\\
D_{ki}=\dot{L}_i(\tau_k)=\sum_{i=0}^N\frac{\prod_{j=0,j\neq i}^{N}\tau_k-\tau_j}{\prod_{j=0,j\neq i}^{N}\tau_i-\tau_j}
$$
其中$\tau_0 = -1$。

从而状态方程表示为：
$$
\sum_{i=0}^{N}D_{ki}X_{i}-\frac{\tau_f-\tau_0}{2}F(X_k,U_k,\tau_k;t_0,t_f)=0
$$
将状态方程表示为上述的代数约束。

## 5 计算例程

设如下最优控制问题：
$$
\dot{x}=-x^2+u \\\\
x(0) = 1 \\\\
x(2) = 0.5 \\\\
J=0.5\int^2_0u^2(t)dt \\\\
t \in [0, 2]
$$
进行如下时间关系变换。
$$
t = \frac{(t_f - t_0)τ + (t_f + t_0)}{2} = \tau+1 \\\\
\frac{dx}{d\tau}=1 \cdot \dot{x} ⇒ \frac{dx}{d\tau}=-x^2+u
$$ { }
选择N=2的Gauss-Legendre节点，则
$$
\tau_0=-1, \tau_1=-\frac{1}{\sqrt3}, \tau_2=\frac{1}{\sqrt3}，\tau_f=1
$$
对状态变量进行离散化:
$$
X(\tau) \approx \sum_{i=0}^{2}L_i(\tau)X(\tau_i) = \frac{(\tau-\tau_1)(\tau-\tau_2)}{(\tau_0-\tau_1)(\tau_0-\tau_2)}X_0+\frac{(\tau-\tau_0)(\tau-\tau_2)}{(\tau_1-\tau_0)(\tau_1-\tau_2)}X_1+\frac{(\tau-\tau_0)(\tau-\tau_1)}{(\tau_2-\tau_0)(\tau_2-\tau_1)}X_2 \\\\
U(\tau) \approx \sum_{i=1}^{n}L_i^*(\tau)U(\tau_i)= \frac{\tau-\tau_2}{\tau_1-\tau_2}U_1 + \frac{\tau-\tau_1}{\tau_2-\tau_1}U_2 \\\\
$$
NLP的决策变量为:
$$
Z=[X_0, X_1,X_2,U_1,U_2]^T
$$
在对微分约束进行离散化， 在配点$\tau_k$的导数为：

$$
\frac{dX}{d\tau} = \sum^2_{j=0}D_{kj}X_j, D_{kj}=\frac{dL_j}{d\tau} \\\\
D = \begin{bmatrix}
\frac{dL_0}{d\tau} & \frac{dL_1}{d\tau} & \frac{dL_2}{d\tau} \\\\
\frac{dL_0}{d\tau} & \frac{dL_1}{d\tau} & \frac{dL_2}{d\tau}
\end{bmatrix}=
\begin{bmatrix}
\frac{(\tau_1-\tau_1) +(\tau_1-\tau_2)}{(\tau_0-\tau_1)(\tau_0-\tau_2)} &
\frac{(\tau_1-\tau_0) +(\tau_1-\tau_2)}{(\tau_1-\tau_0)(\tau_1-\tau_2)} &
\frac{(\tau_1-\tau_0) +(\tau_1-\tau_1)}{(\tau_2-\tau_0)(\tau_2-\tau_1)} \\\\
\frac{(\tau_2-\tau_1) +(\tau_2-\tau_2)}{(\tau_0-\tau_1)(\tau_0-\tau_2)} &
\frac{(\tau_2-\tau_0) +(\tau_2-\tau_2)}{(\tau_1-\tau_0)(\tau_1-\tau_2)} &
\frac{(\tau_2-\tau_0) +(\tau_2-\tau_1)}{(\tau_2-\tau_0)(\tau_2-\tau_1)}
\end{bmatrix}
$$

则约束变成

$$
\left\{ 
\begin{array}{**lr**}
D_{00}X_0+D_{01}X_1 +D_{02}X_2 = -X_1^2 +U_1 \\\\
D_{10}X_0+D_{11}X_1 +D_{12}X_2 = -X_2^2 +U_2
\end{array}
\right.
$$


约束处理：

初始条件：
$$
X_0=1
$$
终端条件：
$$
X_f=X_0+\int^1_{-1}\frac{dX}{d\tau}d\tau\approx X_0+\sum_{i=1}^2c_i(-X_i^2+U_i)
$$
目标函数：
$$
minimize \ J=\frac{1}{2}(U^2_1+U^2_2)
$$


```matlab
clear, clc
[x,c] = lgwt(2, -1, 1); 
% 该函数返回LG节点的值，在N等于2，区间[-1, 1]上，x=[1/sqrt(3); -1/sqrt(3)]
% c = [1; 1]
tau = [-1;flip(x);1];
D = [
(tau(2) - tau(2) + tau(2) - tau(3))/((tau(1)-tau(2))*(tau(1)-tau(3))), ...
(tau(2) - tau(1) + tau(2) - tau(3))/((tau(2)-tau(1))*(tau(2)-tau(3))), ...
(tau(2) - tau(1) + tau(2) - tau(2))/((tau(3)-tau(1))*(tau(3)-tau(2)));
(tau(3) - tau(2) + tau(3) - tau(3))/((tau(1)-tau(2))*(tau(1)-tau(3))), ...
(tau(3) - tau(1) + tau(3) - tau(3))/((tau(2)-tau(1))*(tau(2)-tau(3))), ...
(tau(3) - tau(1) + tau(3) - tau(2))/((tau(3)-tau(1))*(tau(3)-tau(2)));
];

x = zeros(5, 1);
x = fmincon(@myfun, x, [], [], [], [], [], [], @(x)mycon(x, D));

X0 = x(1);
X1 = x(2);
X2 = x(3);
U1 = x(4);
U2 = x(5);
L0 = @(t) (t - tau(2)).*(t - tau(3))./((tau(1) - tau(2))*(tau(1) - tau(3)));
L1 = @(t) (t - tau(1))*(t - tau(3))./((tau(2) - tau(1))*(tau(2) - tau(3)));
L2 = @(t) (t - tau(1)).*(t - tau(2))./((tau(3) - tau(1))*(tau(3) - tau(2)));
LU1 = @(t)(t-tau(3))./(tau(2)-tau(3));
LU2 = @(t)(t-tau(2))./(tau(3)-tau(2));
fx = @(t)L0(t).*X0+L1(t).*X1+L2(t).*X2;
fu = @(t)LU1(t).*U1+LU2(t).*U2;

figure;
subplot(2, 1, 1);
fplot(fx, [-1, 1]);
subplot(2, 1, 2);
fplot(fu, [-1, 1]);

function  f = myfun(x)
U1 = x(4);
U2 = x(5);
f = 0.5 * (U1.^2 + U2.^2);
end

function [c, ceq] = mycon(x, D)
% x = {X0, X1, X2, U1, U2}
% c <= 0
% ceq = 0
X0 = x(1);
X1 = x(2);
X2 = x(3);
U1 = x(4);
U2 = x(5);
c = 0;
ceq(1) = X0 - 1;
ceq(2) = 0.5 + U1 + U2 - X1 .^ 2 - X2 .^ 2;
ceq(3) = D(1, 1) * X0 + D(1, 2) * X1 + D(1, 3) * X2 + X1^2 - U1;
ceq(4) = D(2, 1) * X0 + D(2, 2) * X1 + D(2, 3) * X2 + X2^2 - U2;
end
```

