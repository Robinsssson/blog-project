---
title: "基于Transformer的航迹预测方法"
date: 2026-03-15T13:50:00+08:00
draft: false
categories: ["深度学习", "航迹规划"]
tags: ["Transformer", "航迹预测", "序列预测", "时间序列", "航空"]
math: true
toc: true
readingTime: true
description: "深入讲解如何利用Transformer架构进行飞行器航迹预测，包括模型设计、数据处理和完整代码实现"
---

## 概述

航迹预测是空中交通管理和飞行器导航中的核心问题。准确预测飞行器未来的航迹对于冲突检测、流量管理和路径规划至关重要。传统方法主要依赖物理模型和运动学方程，而近年来深度学习，尤其是Transformer架构，为航迹预测提供了新的解决方案。

### 为什么选择 Transformer？

传统方法（如卡尔曼滤波、隐马尔可夫模型）的局限性：

- **线性假设**：难以捕捉复杂的非线性运动模式
- **特征工程**：需要大量领域知识设计特征
- **长距离依赖**：难以有效利用长历史信息

Transformer 的优势：

- **自注意力机制**：直接建模任意时刻之间的关系
- **并行计算**：训练效率高，可处理大规模数据
- **多尺度特征**：多头注意力可学习不同尺度的运动模式

---

## 一、问题定义

### 1.1 航迹预测任务

给定飞行器过去 $T$ 个时刻的状态序列，预测未来 $H$ 个时刻的状态：

$$
X = \{(p_t, v_t, a_t)\}_{t=1}^{T} \rightarrow Y = \{(p_t, v_t)\}_{t=T+1}^{T+H}
$$

其中：
- $p_t \in \mathbb{R}^3$：三维位置（经度、纬度、高度）
- $v_t \in \mathbb{R}^3$：速度向量
- $a_t \in \mathbb{R}^3$：加速度向量

### 1.2 数据表示

一条完整的航迹可以表示为时间序列：

$$
\mathbf{S} = [s_1, s_2, \ldots, s_n], \quad s_t = [lon_t, lat_t, alt_t, speed_t, heading_t, climb\_rate_t]
$$

常用特征包括：

| 特征 | 说明 | 单位 |
|------|------|------|
| longitude | 经度 | 度 |
| latitude | 纬度 | 度 |
| altitude | 高度 | 米 |
| speed | 地速 | m/s |
| heading | 航向角 | 度 |
| climb_rate | 爬升率 | m/s |

---

## 二、模型架构

### 2.1 整体架构

```
输入序列 (T × D)
      │
      ▼
┌─────────────┐
│  输入嵌入层  │  线性投影 + 位置编码
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ Transformer │  N 层 Encoder
│   Encoder   │  自注意力 + FFN
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  解码头     │  预测未来 H 步
└──────┬──────┘
       │
       ▼
输出序列 (H × D)
```

### 2.2 输入嵌入层

将原始特征投影到高维空间，并加入位置信息：

$$
z_t = W_e x_t + b_e + PE(t)
$$

其中 $PE(t)$ 为位置编码：

$$
PE_{(t, 2i)} = \sin(t / 10000^{2i/d})
$$

$$
PE_{(t, 2i+1)} = \cos(t / 10000^{2i/d})
$$

### 2.3 时序自注意力

航迹预测中，我们使用因果自注意力，每个时刻只能关注过去的信息：

$$
\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}} + M\right)V
$$

其中 $M$ 为下三角掩码矩阵：

$$
M_{ij} = \begin{cases}
0, & i \geq j \\
-\infty, & i < j
\end{cases}
$$

### 2.4 多尺度注意力

飞行器运动具有多时间尺度特性：

- **短期**：风场扰动、机动调整（秒级）
- **中期**：航路点切换、爬升/下降（分钟级）
- **长期**：航线结构、目的地约束（小时级）

使用多头注意力捕捉不同尺度的模式：

$$
\text{MultiHead}(X) = \text{Concat}(head_1, \ldots, head_h)W^O
$$

---

## 三、模型实现

### 3.1 完整代码实现

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
import math
import numpy as np

class PositionalEncoding(nn.Module):
    """位置编码层"""
    def __init__(self, d_model, max_len=5000, dropout=0.1):
        super().__init__()
        self.dropout = nn.Dropout(p=dropout)
        
        pe = torch.zeros(max_len, d_model)
        position = torch.arange(0, max_len, dtype=torch.float).unsqueeze(1)
        
        div_term = torch.exp(
            torch.arange(0, d_model, 2).float() * (-math.log(10000.0) / d_model)
        )
        
        pe[:, 0::2] = torch.sin(position * div_term)
        pe[:, 1::2] = torch.cos(position * div_term)
        pe = pe.unsqueeze(0)
        
        self.register_buffer('pe', pe)
    
    def forward(self, x):
        # x: (batch, seq_len, d_model)
        x = x + self.pe[:, :x.size(1), :]
        return self.dropout(x)


class TrajectoryTransformer(nn.Module):
    """基于Transformer的航迹预测模型"""
    
    def __init__(self, 
                 input_dim=6,      # 输入特征维度
                 d_model=128,      # 模型维度
                 n_heads=8,        # 注意力头数
                 n_layers=6,       # Transformer层数
                 d_ff=512,         # FFN隐藏维度
                 output_dim=6,     # 输出特征维度
                 pred_len=30,      # 预测长度
                 dropout=0.1):
        super().__init__()
        
        self.d_model = d_model
        self.pred_len = pred_len
        
        # 输入嵌入
        self.input_embedding = nn.Linear(input_dim, d_model)
        self.pos_encoding = PositionalEncoding(d_model, dropout=dropout)
        
        # Transformer编码器
        encoder_layer = nn.TransformerEncoderLayer(
            d_model=d_model,
            nhead=n_heads,
            dim_feedforward=d_ff,
            dropout=dropout,
            batch_first=True
        )
        self.transformer_encoder = nn.TransformerEncoder(
            encoder_layer, 
            num_layers=n_layers
        )
        
        # 预测头：预测未来 pred_len 步
        self.predictor = nn.Sequential(
            nn.Linear(d_model, d_ff),
            nn.ReLU(),
            nn.Dropout(dropout),
            nn.Linear(d_ff, output_dim * pred_len)
        )
        
        self.output_dim = output_dim
    
    def forward(self, x, src_mask=None):
        """
        Args:
            x: (batch, seq_len, input_dim) 输入航迹序列
            src_mask: 可选的注意力掩码
        Returns:
            pred: (batch, pred_len, output_dim) 预测的航迹
        """
        batch_size, seq_len, _ = x.shape
        
        # 输入嵌入 + 位置编码
        x = self.input_embedding(x) * math.sqrt(self.d_model)
        x = self.pos_encoding(x)
        
        # 因果掩码（自回归预测）
        if src_mask is None:
            src_mask = self._generate_square_subsequent_mask(seq_len).to(x.device)
        
        # Transformer编码
        encoded = self.transformer_encoder(x, mask=src_mask)
        
        # 取最后一个时刻的表示进行预测
        last_hidden = encoded[:, -1, :]  # (batch, d_model)
        
        # 预测未来轨迹
        pred = self.predictor(last_hidden)  # (batch, pred_len * output_dim)
        pred = pred.view(batch_size, self.pred_len, self.output_dim)
        
        return pred
    
    def _generate_square_subsequent_mask(self, sz):
        """生成因果掩码"""
        mask = torch.triu(torch.ones(sz, sz), diagonal=1)
        mask = mask.masked_fill(mask == 1, float('-inf'))
        return mask
    
    def predict_trajectory(self, x, steps=None):
        """
        自回归预测（逐步预测）
        Args:
            x: (batch, seq_len, input_dim)
            steps: 预测步数，默认使用 pred_len
        """
        if steps is None:
            steps = self.pred_len
        
        self.eval()
        predictions = []
        
        with torch.no_grad():
            current_input = x.clone()
            
            for _ in range(steps):
                # 预测下一步
                pred = self.forward(current_input)
                next_step = pred[:, 0:1, :]  # 取第一步预测
                
                predictions.append(next_step)
                
                # 滑动窗口：添加新预测，移除最旧的
                current_input = torch.cat([current_input[:, 1:, :], next_step], dim=1)
        
        return torch.cat(predictions, dim=1)


class TrajectoryLoss(nn.Module):
    """航迹预测损失函数"""
    
    def __init__(self, w_position=1.0, w_velocity=0.5, w_heading=0.3):
        super().__init__()
        self.w_position = w_position
        self.w_velocity = w_velocity
        self.w_heading = w_heading
        self.mse = nn.MSELoss()
    
    def forward(self, pred, target):
        """
        Args:
            pred: (batch, pred_len, 6) 预测轨迹
            target: (batch, pred_len, 6) 真实轨迹
        """
        # 位置损失 (lon, lat, alt)
        pos_loss = self.mse(pred[:, :, :3], target[:, :, :3])
        
        # 速度损失
        vel_loss = self.mse(pred[:, :, 3:4], target[:, :, 3:4])
        
        # 航向损失（考虑周期性）
        heading_pred = pred[:, :, 4:5]
        heading_target = target[:, :, 4:5]
        heading_loss = self._circular_loss(heading_pred, heading_target)
        
        # 综合损失
        total_loss = (self.w_position * pos_loss + 
                      self.w_velocity * vel_loss + 
                      self.w_heading * heading_loss)
        
        return total_loss, {
            'position_loss': pos_loss.item(),
            'velocity_loss': vel_loss.item(),
            'heading_loss': heading_loss.item()
        }
    
    def _circular_loss(self, pred, target):
        """处理角度的周期性"""
        diff = torch.abs(pred - target)
        diff = torch.min(diff, 360 - diff)  # 考虑360度周期
        return diff.mean()


class TrajectoryDataset(torch.utils.data.Dataset):
    """航迹数据集"""
    
    def __init__(self, trajectories, hist_len=60, pred_len=30, normalize=True):
        """
        Args:
            trajectories: list of trajectories, each is (T, D) array
            hist_len: 历史序列长度
            pred_len: 预测序列长度
        """
        self.hist_len = hist_len
        self.pred_len = pred_len
        self.samples = []
        
        # 构建样本
        for traj in trajectories:
            T = len(traj)
            if T < hist_len + pred_len:
                continue
            
            # 滑动窗口切分
            for i in range(T - hist_len - pred_len + 1):
                hist = traj[i:i+hist_len]
                future = traj[i+hist_len:i+hist_len+pred_len]
                self.samples.append((hist, future))
        
        # 标准化
        if normalize:
            self._compute_normalization(trajectories)
        
        print(f"Created {len(self.samples)} samples from {len(trajectories)} trajectories")
    
    def _compute_normalization(self, trajectories):
        """计算标准化参数"""
        all_data = np.concatenate(trajectories, axis=0)
        self.mean = all_data.mean(axis=0)
        self.std = all_data.std(axis=0) + 1e-8
    
    def normalize(self, data):
        return (data - self.mean) / self.std
    
    def denormalize(self, data):
        return data * self.std + self.mean
    
    def __len__(self):
        return len(self.samples)
    
    def __getitem__(self, idx):
        hist, future = self.samples[idx]
        hist = self.normalize(hist)
        future = self.normalize(future)
        return (
            torch.FloatTensor(hist),
            torch.FloatTensor(future)
        )


def train_model(model, train_loader, val_loader, epochs=100, lr=1e-4, device='cuda'):
    """训练函数"""
    
    optimizer = torch.optim.AdamW(model.parameters(), lr=lr, weight_decay=1e-5)
    scheduler = torch.optim.lr_scheduler.CosineAnnealingLR(optimizer, T_max=epochs)
    criterion = TrajectoryLoss()
    
    best_val_loss = float('inf')
    
    for epoch in range(epochs):
        # 训练
        model.train()
        train_loss = 0
        
        for batch_idx, (hist, future) in enumerate(train_loader):
            hist = hist.to(device)
            future = future.to(device)
            
            optimizer.zero_grad()
            pred = model(hist)
            loss, loss_dict = criterion(pred, future)
            loss.backward()
            
            # 梯度裁剪
            torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
            
            optimizer.step()
            train_loss += loss.item()
        
        # 验证
        model.eval()
        val_loss = 0
        
        with torch.no_grad():
            for hist, future in val_loader:
                hist = hist.to(device)
                future = future.to(device)
                pred = model(hist)
                loss, _ = criterion(pred, future)
                val_loss += loss.item()
        
        train_loss /= len(train_loader)
        val_loss /= len(val_loader)
        
        scheduler.step()
        
        print(f"Epoch {epoch+1}/{epochs} - Train Loss: {train_loss:.4f}, Val Loss: {val_loss:.4f}")
        
        # 保存最佳模型
        if val_loss < best_val_loss:
            best_val_loss = val_loss
            torch.save(model.state_dict(), 'best_trajectory_model.pt')
    
    return model


# 使用示例
if __name__ == "__main__":
    # 模拟数据
    np.random.seed(42)
    num_trajectories = 100
    trajectories = []
    
    for _ in range(num_trajectories):
        T = np.random.randint(200, 500)
        traj = np.cumsum(np.random.randn(T, 6) * 0.1, axis=0)
        trajectories.append(traj)
    
    # 创建数据集
    dataset = TrajectoryDataset(trajectories, hist_len=60, pred_len=30)
    
    # 划分训练/验证集
    train_size = int(0.8 * len(dataset))
    val_size = len(dataset) - train_size
    train_dataset, val_dataset = torch.utils.data.random_split(
        dataset, [train_size, val_size]
    )
    
    train_loader = torch.utils.data.DataLoader(
        train_dataset, batch_size=32, shuffle=True
    )
    val_loader = torch.utils.data.DataLoader(
        val_dataset, batch_size=32
    )
    
    # 创建模型
    device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
    model = TrajectoryTransformer(
        input_dim=6,
        d_model=128,
        n_heads=8,
        n_layers=4,
        pred_len=30
    ).to(device)
    
    # 训练
    model = train_model(model, train_loader, val_loader, epochs=50, device=device)
    
    # 预测示例
    model.eval()
    with torch.no_grad():
        hist, future = next(iter(val_loader))
        hist = hist.to(device)
        pred = model(hist)
        print(f"Input shape: {hist.shape}")
        print(f"Prediction shape: {pred.shape}")
        print(f"Ground truth shape: {future.shape}")
```

### 3.2 关键设计说明

**1. 因果掩码**

航迹预测是时间序列任务，需要确保模型不能"看到未来"：

```python
# 因果掩码示例
seq_len = 5
mask = torch.triu(torch.ones(seq_len, seq_len), diagonal=1)
mask = mask.masked_fill(mask == 1, float('-inf'))
# 结果:
# [[0, -inf, -inf, -inf, -inf],
#  [0,    0, -inf, -inf, -inf],
#  [0,    0,    0, -inf, -inf],
#  [0,    0,    0,    0, -inf],
#  [0,    0,    0,    0,    0]]
```

**2. 位置编码的重要性**

飞行器运动与时间强相关，位置编码帮助模型理解时序关系：

$$
\text{Attention}(t_1, t_2) \propto \exp\left(\frac{q_{t_1} \cdot k_{t_2}}{\sqrt{d}}\right)
$$

**3. 多步预测策略**

- **单步预测**：直接预测所有未来步（速度快）
- **自回归预测**：逐步预测，每步将预测加入输入（精度高）

---

## 四、数据处理

### 4.1 数据清洗

```python
def clean_trajectory(traj, max_speed=500, max_altitude=15000):
    """
    清洗航迹数据
    Args:
        traj: (T, D) 航迹数组
        max_speed: 最大合理速度 (m/s)
        max_altitude: 最大合理高度 (m)
    """
    cleaned = traj.copy()
    
    # 1. 去除异常值
    speed_mask = (cleaned[:, 3] > 0) & (cleaned[:, 3] < max_speed)
    alt_mask = (cleaned[:, 2] > 0) & (cleaned[:, 2] < max_altitude)
    valid_mask = speed_mask & alt_mask
    
    # 2. 线性插值填补缺失
    for col in range(cleaned.shape[1]):
        valid_idx = np.where(valid_mask)[0]
        cleaned[~valid_mask, col] = np.interp(
            np.where(~valid_mask)[0],
            valid_idx,
            cleaned[valid_idx, col]
        )
    
    # 3. 平滑滤波
    from scipy.ndimage import uniform_filter1d
    cleaned = uniform_filter1d(cleaned, size=3, axis=0)
    
    return cleaned
```

### 4.2 特征工程

```python
def extract_features(raw_data):
    """
    提取航迹特征
    Args:
        raw_data: 包含 lon, lat, alt, timestamp 的原始数据
    """
    features = []
    
    # 基础特征
    features.append(raw_data[:, :3])  # 位置
    
    # 计算速度
    dt = np.diff(raw_data[:, -1], prepend=raw_data[0, -1])
    dx = np.diff(raw_data[:, 0], prepend=raw_data[0, 0])
    dy = np.diff(raw_data[:, 1], prepend=raw_data[0, 1])
    speed = np.sqrt(dx**2 + dy**2) / (dt + 1e-8)
    features.append(speed.reshape(-1, 1))
    
    # 航向角
    heading = np.arctan2(dy, dx) * 180 / np.pi
    heading = (heading + 360) % 360  # 归一化到 0-360
    features.append(heading.reshape(-1, 1))
    
    # 爬升率
    dz = np.diff(raw_data[:, 2], prepend=raw_data[0, 2])
    climb_rate = dz / (dt + 1e-8)
    features.append(climb_rate.reshape(-1, 1))
    
    return np.concatenate(features, axis=1)
```

### 4.3 数据增强

```python
def augment_trajectory(traj, noise_level=0.01):
    """
    航迹数据增强
    """
    augmented = traj.copy()
    
    # 1. 添加高斯噪声
    noise = np.random.randn(*traj.shape) * noise_level * traj.std(axis=0)
    augmented += noise
    
    # 2. 时间扰动
    max_shift = 3
    shift = np.random.randint(-max_shift, max_shift + 1)
    if shift > 0:
        augmented = np.pad(augmented[shift:], ((0, shift), (0, 0)), mode='edge')
    elif shift < 0:
        augmented = np.pad(augmented[:shift], ((-shift, 0), (0, 0)), mode='edge')
    
    # 3. 速度缩放
    scale = 1 + np.random.uniform(-0.1, 0.1)
    augmented[:, 3] *= scale
    
    return augmented
```

---

## 五、评估指标

### 5.1 位置误差

**平均位移误差 (ADE)**：
$$
ADE = \frac{1}{H} \sum_{t=T+1}^{T+H} \| \hat{p}_t - p_t \|_2
$$

**终点位移误差 (FDE)**：
$$
FDE = \| \hat{p}_{T+H} - p_{T+H} \|_2
$$

### 5.2 代码实现

```python
def evaluate_metrics(pred, target, dataset):
    """
    计算评估指标
    Args:
        pred: (batch, pred_len, 6) 预测轨迹（归一化）
        target: (batch, pred_len, 6) 真实轨迹（归一化）
        dataset: 数据集对象，用于反归一化
    """
    # 反归一化
    pred = dataset.denormalize(pred.cpu().numpy())
    target = dataset.denormalize(target.cpu().numpy())
    
    # 转换为经纬度坐标系下的距离（米）
    # 简化计算：1度经纬度 ≈ 111km
    pred_pos = pred[:, :, :3].copy()
    target_pos = target[:, :, :3].copy()
    
    # 经纬度转米
    pred_pos[:, :, 0] *= 111000 * np.cos(np.radians(pred_pos[:, :, 1]))
    pred_pos[:, :, 1] *= 111000
    target_pos[:, :, 0] *= 111000 * np.cos(np.radians(target_pos[:, :, 1]))
    target_pos[:, :, 1] *= 111000
    
    # 计算位置误差
    position_error = np.sqrt(np.sum((pred_pos - target_pos)**2, axis=-1))
    
    ade = position_error.mean()
    fde = position_error[:, -1].mean()
    
    # 高度误差
    alt_error = np.abs(pred[:, :, 2] - target[:, :, 2]).mean()
    
    return {
        'ADE': ade,
        'FDE': fde,
        'Altitude_Error': alt_error
    }
```

---

## 六、进阶改进

### 6.1 多模态预测

飞行器运动存在不确定性，可以预测多个可能的轨迹：

```python
class MultiModalTrajectoryPredictor(nn.Module):
    """多模态轨迹预测"""
    
    def __init__(self, num_modes=5, **kwargs):
        super().__init__()
        self.num_modes = num_modes
        
        self.encoder = TrajectoryTransformer(**kwargs)
        
        # 多模态预测头
        self.mode_predictor = nn.Linear(kwargs['d_model'], num_modes)
        self.trajectory_heads = nn.ModuleList([
            nn.Linear(kwargs['d_model'], kwargs['pred_len'] * kwargs['output_dim'])
            for _ in range(num_modes)
        ])
    
    def forward(self, x):
        # 编码
        encoded = self.encoder.transformer_encoder(
            self.encoder.pos_encoding(
                self.encoder.input_embedding(x) * math.sqrt(self.encoder.d_model)
            )
        )
        last_hidden = encoded[:, -1, :]
        
        # 预测各模态概率
        mode_probs = F.softmax(self.mode_predictor(last_hidden), dim=-1)
        
        # 预测各模态轨迹
        trajectories = []
        for head in self.trajectory_heads:
            traj = head(last_hidden).view(-1, self.encoder.pred_len, -1)
            trajectories.append(traj)
        trajectories = torch.stack(trajectories, dim=1)  # (batch, num_modes, pred_len, dim)
        
        return mode_probs, trajectories
```

### 6.2 条件预测

引入外部条件信息（如天气、航线）：

```python
class ConditionalTrajectoryPredictor(nn.Module):
    """条件轨迹预测"""
    
    def __init__(self, condition_dim=32, **kwargs):
        super().__init__()
        
        self.encoder = TrajectoryTransformer(**kwargs)
        
        # 条件编码
        self.condition_encoder = nn.Sequential(
            nn.Linear(condition_dim, kwargs['d_model']),
            nn.ReLU(),
            nn.Linear(kwargs['d_model'], kwargs['d_model'])
        )
        
        # 条件注入（通过cross-attention）
        self.cross_attention = nn.MultiheadAttention(
            embed_dim=kwargs['d_model'],
            num_heads=kwargs['n_heads']
        )
    
    def forward(self, x, condition):
        # 编码航迹
        traj_encoded = self.encoder.pos_encoding(
            self.encoder.input_embedding(x) * math.sqrt(self.encoder.d_model)
        )
        
        # 编码条件
        cond_encoded = self.condition_encoder(condition).unsqueeze(1)
        
        # 交叉注意力融合
        fused, _ = self.cross_attention(
            traj_encoded, cond_encoded, cond_encoded
        )
        traj_encoded = traj_encoded + fused
        
        # 预测
        encoded = self.encoder.transformer_encoder(traj_encoded)
        last_hidden = encoded[:, -1, :]
        pred = self.encoder.predictor(last_hidden)
        
        return pred.view(-1, self.encoder.pred_len, -1)
```

---

## 七、实际应用考虑

### 7.1 实时性要求

航迹预测通常要求实时响应，需要优化推理速度：

```python
# 1. 模型量化
quantized_model = torch.quantization.quantize_dynamic(
    model, {nn.Linear}, dtype=torch.qint8
)

# 2. 模型剪枝
from torch.nn.utils import prune
for name, module in model.named_modules():
    if isinstance(module, nn.Linear):
        prune.l1_unstructured(module, name='weight', amount=0.3)

# 3. ONNX 导出
torch.onnx.export(
    model,
    torch.randn(1, 60, 6),
    "trajectory_predictor.onnx",
    opset_version=11
)
```

### 7.2 不确定性量化

```python
class UncertaintyEstimator:
    """不确定性估计"""
    
    def __init__(self, model, num_samples=10):
        self.model = model
        self.num_samples = num_samples
    
    def predict_with_uncertainty(self, x):
        """
        使用MC Dropout估计不确定性
        """
        self.model.train()  # 启用dropout
        
        predictions = []
        with torch.no_grad():
            for _ in range(self.num_samples):
                pred = self.model(x)
                predictions.append(pred)
        
        predictions = torch.stack(predictions)
        
        mean = predictions.mean(dim=0)
        std = predictions.std(dim=0)
        
        return mean, std
```

---

## 八、总结

### 8.1 方法对比

| 方法 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| 卡尔曼滤波 | 计算简单，实时性好 | 线性假设，精度有限 | 短期预测 |
| LSTM | 序列建模能力强 | 顺序计算，长序列效果差 | 中等长度预测 |
| Transformer | 并行计算，长距离依赖 | 计算量大 | 长期预测 |

### 8.2 最佳实践

1. **数据质量**：清洗异常值，处理缺失数据
2. **特征选择**：位置、速度、航向是核心特征
3. **模型规模**：根据数据量选择合适的模型大小
4. **训练策略**：使用学习率预热和余弦退火
5. **评估指标**：ADE/FDE 是标准指标

### 8.3 未来方向

- **图神经网络**：建模空域结构
- **物理约束**：融入运动学方程
- **多源数据融合**：结合天气、空域限制等
- **强化学习**：考虑飞行意图

---

## 参考资料

- 《Attention Is All You Need》- Vaswani et al., 2017
- 《Social GAN: Socially Acceptable Trajectories with Generative Adversarial Networks》
- 《TNT: Target-driveN Trajectory Prediction》
- Eurocontrol ATM Data Archive