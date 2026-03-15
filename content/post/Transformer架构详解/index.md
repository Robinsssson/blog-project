---
title: "Transformer架构详解：从原理到实现"
date: 2026-03-15T03:53:00+08:00
draft: false
categories: ["深度学习"]
tags: ["Transformer", "深度学习", "NLP", "注意力机制", "神经网络"]
math: true
---

## 概述

Transformer 是 2017 年 Google 在论文《Attention Is All You Need》中提出的神经网络架构。它彻底改变了自然语言处理领域，成为 GPT、BERT、LLaMA 等大语言模型的基础架构。

### 为什么需要 Transformer？

传统序列模型（RNN、LSTM、GRU）存在的问题：
- **顺序计算**：无法并行处理，训练效率低
- **长距离依赖**：信息需要逐步传递，远距离关系难以捕捉
- **梯度消失**：长序列中梯度难以传播

Transformer 的解决方案：
- **自注意力机制**：直接建模任意位置的关系
- **并行计算**：所有位置同时处理
- **位置编码**：保留序列顺序信息

---

## 一、Transformer 整体架构

### 1.1 架构图

```
                    编码器侧                          解码器侧
┌─────────────────────────────────┐    ┌─────────────────────────────────┐
│                                 │    │                                 │
│  输入嵌入 + 位置编码             │    │  输出嵌入 + 位置编码             │
│         │                       │    │         │                       │
│         ▼                       │    │         ▼                       │
│  ┌─────────────────────────┐    │    │  ┌─────────────────────────┐    │
│  │    多头自注意力          │    │    │  │    掩码多头自注意力      │    │
│  │    (Multi-Head Self-    │    │    │  │    (Masked Multi-Head   │    │
│  │     Attention)          │    │    │  │     Self-Attention)     │    │
│  └───────────┬─────────────┘    │    │  └───────────┬─────────────┘    │
│              │                  │    │              │                  │
│    Add & Norm                  │    │    Add & Norm                  │
│              │                  │    │              │                  │
│              ▼                  │    │              ▼                  │
│  ┌─────────────────────────┐    │    │  ┌─────────────────────────┐    │
│  │    前馈神经网络          │◄───┼────┼──│    编码器-解码器注意力   │    │
│  │    (Feed Forward)       │    │    │  │    (Cross Attention)    │    │
│  └───────────┬─────────────┘    │    │  └───────────┬─────────────┘    │
│              │                  │    │              │                  │
│    Add & Norm                  │    │    Add & Norm                  │
│              │                  │    │              │                  │
│              ▼                  │    │              ▼                  │
│         N × 堆叠                │    │  ┌─────────────────────────┐    │
│                                 │    │  │    前馈神经网络          │    │
│                                 │    │  └───────────┬─────────────┘    │
│                                 │    │              │                  │
│                                 │    │    Add & Norm                  │
│                                 │    │              │                  │
│                                 │    │         N × 堆叠                │
│                                 │    │              │                  │
│                                 │    │              ▼                  │
│                                 │    │         线性层 + Softmax        │
└─────────────────────────────────┘    └─────────────────────────────────┘
```

### 1.2 核心组件

Transformer 由以下核心组件构成：

| 组件 | 功能 |
|------|------|
| **自注意力机制** | 建模序列内部的关系 |
| **多头注意力** | 并行学习多种表示 |
| **位置编码** | 注入位置信息 |
| **前馈网络** | 非线性变换 |
| **层归一化** | 稳定训练 |
| **残差连接** | 缓解梯度消失 |

---

## 二、自注意力机制(Self-Attention)

### 2.1 核心思想

自注意力让序列中的每个位置都能直接关注到其他所有位置，从而捕捉全局依赖关系。

```
输入序列: "我 爱 你"
           │  │  │
           ▼  ▼  ▼
        ┌──────────┐
        │ 自注意力  │
        │  机制    │
        └──────────┘
           │  │  │
           ▼  ▼  ▼
每个词都能直接看到其他所有词
```

### 2.2 Query、Key、Value

自注意力借鉴了信息检索的思想：

- **Query (Q)**：查询向量，"我想找什么"
- **Key (K)**：键向量，"我是什么特征"
- **Value (V)**：值向量，"我的实际内容"

```python
import torch
import torch.nn as nn
import math

class SelfAttention(nn.Module):
    def __init__(self, embed_dim):
        super().__init__()
        self.embed_dim = embed_dim
        
        # Q, K, V 的线性变换
        self.query = nn.Linear(embed_dim, embed_dim)
        self.key = nn.Linear(embed_dim, embed_dim)
        self.value = nn.Linear(embed_dim, embed_dim)
        
        # 缩放因子
        self.scale = math.sqrt(embed_dim)
    
    def forward(self, x):
        """
        x: (batch_size, seq_len, embed_dim)
        """
        # 计算 Q, K, V
        Q = self.query(x)  # (batch, seq_len, embed_dim)
        K = self.key(x)
        V = self.value(x)
        
        # 计算注意力分数: Q @ K^T / sqrt(d_k)
        scores = torch.matmul(Q, K.transpose(-2, -1)) / self.scale
        # scores: (batch, seq_len, seq_len)
        
        # Softmax 归一化
        attention_weights = torch.softmax(scores, dim=-1)
        
        # 加权求和
        output = torch.matmul(attention_weights, V)
        # output: (batch, seq_len, embed_dim)
        
        return output, attention_weights
```

### 2.3 注意力分数计算

对于输入序列 $X = [x_1, x_2, ..., x_n]$：

$$
\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V
$$

**为什么需要缩放因子 $\sqrt{d_k}$？**

当维度较大时，点积结果会很大，导致 softmax 进入饱和区，梯度变得很小。缩放可以稳定梯度。

```python
# 演示缩放的重要性
import torch.nn.functional as F

d_k = 512
q = torch.randn(1, 10, d_k)
k = torch.randn(1, 10, d_k)

# 不缩放
scores_no_scale = torch.matmul(q, k.transpose(-2, -1))
print("无缩放的最大值:", scores_no_scale.max().item())  # 可能很大
print("无缩放的Softmax:", F.softmax(scores_no_scale, dim=-1)[0, 0])  # 接近one-hot

# 缩放后
scores_scaled = scores_no_scale / math.sqrt(d_k)
print("缩放后的最大值:", scores_scaled.max().item())
print("缩放后的Softmax:", F.softmax(scores_scaled, dim=-1)[0, 0])  # 更平滑
```

### 2.4 注意力权重可视化

```
输入: "The cat sat on the mat"

注意力权重矩阵 (示例):
           The  cat  sat  on  the  mat
The       [0.1  0.3  0.2  0.1  0.2  0.1]
cat       [0.2  0.1  0.3  0.1  0.1  0.2]
sat       [0.1  0.4  0.1  0.2  0.1  0.1]
on        [0.1  0.1  0.3  0.1  0.2  0.2]
the       [0.3  0.1  0.1  0.1  0.1  0.3]
mat       [0.1  0.2  0.1  0.1  0.3  0.2]

解读:
- "cat" 对 "sat" 关注度高 (0.3)
- "mat" 对 "the" 关注度高 (0.3)
```

---

## 三、多头注意力(Multi-Head Attention)

### 3.1 为什么需要多头？

单头注意力只能学习一种关系。多头注意力允许模型并行学习多种不同的表示：

- 有的头关注语法关系
- 有的头关注语义关系
- 有的头关注长距离依赖

### 3.2 多头注意力实现

```python
class MultiHeadAttention(nn.Module):
    def __init__(self, embed_dim, num_heads):
        super().__init__()
        self.embed_dim = embed_dim
        self.num_heads = num_heads
        self.head_dim = embed_dim // num_heads
        
        assert embed_dim % num_heads == 0, "embed_dim must be divisible by num_heads"
        
        # Q, K, V 的线性变换
        self.q_proj = nn.Linear(embed_dim, embed_dim)
        self.k_proj = nn.Linear(embed_dim, embed_dim)
        self.v_proj = nn.Linear(embed_dim, embed_dim)
        
        # 输出投影
        self.out_proj = nn.Linear(embed_dim, embed_dim)
        
        self.scale = math.sqrt(self.head_dim)
    
    def forward(self, x, mask=None):
        """
        x: (batch_size, seq_len, embed_dim)
        mask: 可选的注意力掩码
        """
        batch_size, seq_len, _ = x.shape
        
        # 线性变换
        Q = self.q_proj(x)  # (batch, seq_len, embed_dim)
        K = self.k_proj(x)
        V = self.v_proj(x)
        
        # 重塑为多头形式
        # (batch, seq_len, embed_dim) -> (batch, num_heads, seq_len, head_dim)
        Q = Q.view(batch_size, seq_len, self.num_heads, self.head_dim).transpose(1, 2)
        K = K.view(batch_size, seq_len, self.num_heads, self.head_dim).transpose(1, 2)
        V = V.view(batch_size, seq_len, self.num_heads, self.head_dim).transpose(1, 2)
        
        # 计算注意力分数
        scores = torch.matmul(Q, K.transpose(-2, -1)) / self.scale
        # scores: (batch, num_heads, seq_len, seq_len)
        
        # 应用掩码（可选）
        if mask is not None:
            scores = scores.masked_fill(mask == 0, float('-inf'))
        
        # Softmax
        attention_weights = torch.softmax(scores, dim=-1)
        
        # 加权求和
        context = torch.matmul(attention_weights, V)
        # context: (batch, num_heads, seq_len, head_dim)
        
        # 拼接多头
        context = context.transpose(1, 2).contiguous()
        context = context.view(batch_size, seq_len, self.embed_dim)
        
        # 输出投影
        output = self.out_proj(context)
        
        return output, attention_weights
```

### 3.3 多头示意图

```
输入 X (embed_dim = 512, num_heads = 8)
           │
           ▼
    ┌──────┴──────┐
    │  线性变换   │
    │  Q, K, V    │
    └──────┬──────┘
           │
    ┌──────┴──────────────────────────────┐
    │             分割为 8 个头            │
    │  Head1  Head2  Head3  ...  Head8    │
    │  (64d)  (64d)  (64d)       (64d)    │
    └──────┬──────┬──────┬─────┬──────────┘
           │      │      │     │
           ▼      ▼      ▼     ▼
       注意力   注意力  注意力  注意力
       计算     计算    计算    计算
           │      │      │     │
           ▼      ▼      ▼     ▼
        输出1   输出2   输出3   输出8
           │      │      │     │
           └──────┴──────┴─────┘
                  │
                  ▼
             Concat (512d)
                  │
                  ▼
             线性变换 (512d)
                  │
                  ▼
               输出
```

---

## 四、位置编码(Positional Encoding)

### 4.1 为什么需要位置编码？

Transformer 本身不具备处理序列顺序的能力（自注意力是排列不变的），需要显式注入位置信息。

### 4.2 正弦余弦位置编码

原始 Transformer 使用正弦和余弦函数生成位置编码：

$$
PE_{(pos, 2i)} = \sin\left(\frac{pos}{10000^{2i/d_{model}}}\right)
$$

$$
PE_{(pos, 2i+1)} = \cos\left(\frac{pos}{10000^{2i/d_{model}}}\right)
$$

```python
class PositionalEncoding(nn.Module):
    def __init__(self, embed_dim, max_len=5000, dropout=0.1):
        super().__init__()
        self.dropout = nn.Dropout(p=dropout)
        
        # 预计算位置编码
        pe = torch.zeros(max_len, embed_dim)
        position = torch.arange(0, max_len, dtype=torch.float).unsqueeze(1)
        
        # 计算分母项
        div_term = torch.exp(
            torch.arange(0, embed_dim, 2).float() * 
            (-math.log(10000.0) / embed_dim)
        )
        
        # 偶数维度用 sin，奇数维度用 cos
        pe[:, 0::2] = torch.sin(position * div_term)
        pe[:, 1::2] = torch.cos(position * div_term)
        
        # 添加 batch 维度: (max_len, embed_dim) -> (1, max_len, embed_dim)
        pe = pe.unsqueeze(0)
        
        # 注册为 buffer（不参与训练）
        self.register_buffer('pe', pe)
    
    def forward(self, x):
        """
        x: (batch_size, seq_len, embed_dim)
        """
        # 加上位置编码
        x = x + self.pe[:, :x.size(1), :]
        return self.dropout(x)
```

### 4.3 位置编码可视化

```python
import matplotlib.pyplot as plt

def visualize_positional_encoding():
    embed_dim = 128
    max_len = 100
    
    pe = PositionalEncoding(embed_dim, max_len)
    
    # 获取位置编码矩阵
    pe_matrix = pe.pe[0, :, :].numpy()
    
    plt.figure(figsize=(12, 6))
    plt.imshow(pe_matrix.T, aspect='auto', cmap='RdBu')
    plt.xlabel('Position')
    plt.ylabel('Embedding Dimension')
    plt.title('Positional Encoding Visualization')
    plt.colorbar()
    plt.show()

# 输出特点:
# - 不同位置有不同模式
# - 相邻位置相似
# - 可以外推到训练时未见过的位置
```

### 4.4 其他位置编码方式

| 方式 | 特点 |
|------|------|
| **正弦余弦** | 原始方案，可外推 |
| **可学习位置编码** | 更灵活，但不能外推 |
| **相对位置编码** | 考虑相对距离 |
| **旋转位置编码(RoPE)** | 结合绝对和相对，广泛用于 LLM |

```python
# 可学习位置编码
class LearnablePositionalEncoding(nn.Module):
    def __init__(self, embed_dim, max_len=5000):
        super().__init__()
        self.pos_embedding = nn.Embedding(max_len, embed_dim)
    
    def forward(self, x):
        batch_size, seq_len, _ = x.shape
        positions = torch.arange(seq_len, device=x.device)
        return x + self.pos_embedding(positions)
```

---

## 五、前馈神经网络(Feed Forward Network)

### 5.1 结构

每个 Transformer 层中的前馈网络：

$$
\text{FFN}(x) = \text{ReLU}(xW_1 + b_1)W_2 + b_2
$$

```python
class FeedForward(nn.Module):
    def __init__(self, embed_dim, ffn_dim, dropout=0.1):
        super().__init__()
        self.linear1 = nn.Linear(embed_dim, ffn_dim)
        self.linear2 = nn.Linear(ffn_dim, embed_dim)
        self.dropout = nn.Dropout(dropout)
        self.activation = nn.ReLU()
    
    def forward(self, x):
        """
        x: (batch_size, seq_len, embed_dim)
        """
        # 升维 -> 激活 -> 降维
        x = self.linear1(x)
        x = self.activation(x)
        x = self.dropout(x)
        x = self.linear2(x)
        return x
```

### 5.2 激活函数变体

```python
# GELU 激活（GPT、BERT 使用）
class FeedForwardGELU(nn.Module):
    def __init__(self, embed_dim, ffn_dim, dropout=0.1):
        super().__init__()
        self.linear1 = nn.Linear(embed_dim, ffn_dim)
        self.linear2 = nn.Linear(ffn_dim, embed_dim)
        self.dropout = nn.Dropout(dropout)
        self.activation = nn.GELU()
    
    def forward(self, x):
        return self.linear2(self.dropout(self.activation(self.linear1(x))))

# SwiGLU 激活（LLaMA 使用）
class FeedForwardSwiGLU(nn.Module):
    def __init__(self, embed_dim, ffn_dim, dropout=0.1):
        super().__init__()
        self.w1 = nn.Linear(embed_dim, ffn_dim, bias=False)
        self.w2 = nn.Linear(ffn_dim, embed_dim, bias=False)
        self.w3 = nn.Linear(embed_dim, ffn_dim, bias=False)  # 门控
        self.dropout = nn.Dropout(dropout)
    
    def forward(self, x):
        # SwiGLU(x) = Swish(xW1) ⊙ (xW3) * W2
        return self.dropout(self.w2(nn.functional.silu(self.w1(x)) * self.w3(x)))
```

---

## 六、层归一化与残差连接

### 6.1 层归一化(Layer Normalization)

```python
class LayerNorm(nn.Module):
    def __init__(self, embed_dim, eps=1e-6):
        super().__init__()
        self.gamma = nn.Parameter(torch.ones(embed_dim))
        self.beta = nn.Parameter(torch.zeros(embed_dim))
        self.eps = eps
    
    def forward(self, x):
        # 计算均值和方差
        mean = x.mean(dim=-1, keepdim=True)
        var = x.var(dim=-1, keepdim=True, unbiased=False)
        
        # 归一化
        x_norm = (x - mean) / torch.sqrt(var + self.eps)
        
        # 缩放和平移
        return self.gamma * x_norm + self.beta
```

### 6.2 残差连接

```python
# Post-LN (原始 Transformer)
# x = LayerNorm(x + Sublayer(x))

# Pre-LN (更稳定)
# x = x + Sublayer(LayerNorm(x))

class TransformerBlock(nn.Module):
    def __init__(self, embed_dim, num_heads, ffn_dim, dropout=0.1):
        super().__init__()
        self.attention = MultiHeadAttention(embed_dim, num_heads)
        self.ffn = FeedForward(embed_dim, ffn_dim, dropout)
        self.norm1 = nn.LayerNorm(embed_dim)
        self.norm2 = nn.LayerNorm(embed_dim)
        self.dropout = nn.Dropout(dropout)
    
    def forward(self, x, mask=None):
        # Pre-LN 结构
        # 注意力子层
        attn_out, _ = self.attention(self.norm1(x), mask)
        x = x + self.dropout(attn_out)
        
        # 前馈子层
        ffn_out = self.ffn(self.norm2(x))
        x = x + self.dropout(ffn_out)
        
        return x
```

---

## 七、完整 Transformer 实现

### 7.1 编码器

```python
class Encoder(nn.Module):
    def __init__(self, vocab_size, embed_dim, num_heads, ffn_dim, 
                 num_layers, max_len=5000, dropout=0.1):
        super().__init__()
        
        # 词嵌入
        self.token_embedding = nn.Embedding(vocab_size, embed_dim)
        
        # 位置编码
        self.pos_encoding = PositionalEncoding(embed_dim, max_len, dropout)
        
        # Transformer 层
        self.layers = nn.ModuleList([
            TransformerBlock(embed_dim, num_heads, ffn_dim, dropout)
            for _ in range(num_layers)
        ])
        
        self.norm = nn.LayerNorm(embed_dim)
    
    def forward(self, x, mask=None):
        """
        x: (batch_size, seq_len) - token indices
        """
        # 词嵌入 + 位置编码
        x = self.token_embedding(x)
        x = self.pos_encoding(x)
        
        # 通过各层
        for layer in self.layers:
            x = layer(x, mask)
        
        return self.norm(x)
```

### 7.2 解码器

```python
class DecoderBlock(nn.Module):
    def __init__(self, embed_dim, num_heads, ffn_dim, dropout=0.1):
        super().__init__()
        
        # 掩码自注意力
        self.self_attention = MultiHeadAttention(embed_dim, num_heads)
        
        # 编码器-解码器注意力
        self.cross_attention = MultiHeadAttention(embed_dim, num_heads)
        
        # 前馈网络
        self.ffn = FeedForward(embed_dim, ffn_dim, dropout)
        
        self.norm1 = nn.LayerNorm(embed_dim)
        self.norm2 = nn.LayerNorm(embed_dim)
        self.norm3 = nn.LayerNorm(embed_dim)
        
        self.dropout = nn.Dropout(dropout)
    
    def forward(self, x, encoder_output, tgt_mask=None, src_mask=None):
        # 掩码自注意力
        attn_out, _ = self.self_attention(self.norm1(x), tgt_mask)
        x = x + self.dropout(attn_out)
        
        # 编码器-解码器注意力
        cross_out, _ = self.cross_attention(self.norm2(x), src_mask)
        # Q 来自解码器，K,V 来自编码器
        x = x + self.dropout(cross_out)
        
        # 前馈网络
        ffn_out = self.ffn(self.norm3(x))
        x = x + self.dropout(ffn_out)
        
        return x

class Decoder(nn.Module):
    def __init__(self, vocab_size, embed_dim, num_heads, ffn_dim,
                 num_layers, max_len=5000, dropout=0.1):
        super().__init__()
        
        self.token_embedding = nn.Embedding(vocab_size, embed_dim)
        self.pos_encoding = PositionalEncoding(embed_dim, max_len, dropout)
        
        self.layers = nn.ModuleList([
            DecoderBlock(embed_dim, num_heads, ffn_dim, dropout)
            for _ in range(num_layers)
        ])
        
        self.norm = nn.LayerNorm(embed_dim)
        self.output_proj = nn.Linear(embed_dim, vocab_size)
    
    def forward(self, x, encoder_output, tgt_mask=None, src_mask=None):
        x = self.token_embedding(x)
        x = self.pos_encoding(x)
        
        for layer in self.layers:
            x = layer(x, encoder_output, tgt_mask, src_mask)
        
        x = self.norm(x)
        logits = self.output_proj(x)
        
        return logits
```

### 7.3 完整 Transformer

```python
class Transformer(nn.Module):
    def __init__(self, src_vocab_size, tgt_vocab_size, embed_dim=512,
                 num_heads=8, ffn_dim=2048, num_layers=6, 
                 max_len=5000, dropout=0.1):
        super().__init__()
        
        self.encoder = Encoder(src_vocab_size, embed_dim, num_heads,
                               ffn_dim, num_layers, max_len, dropout)
        self.decoder = Decoder(tgt_vocab_size, embed_dim, num_heads,
                               ffn_dim, num_layers, max_len, dropout)
    
    def make_src_mask(self, src):
        # 处理 padding
        return (src != 0).unsqueeze(1).unsqueeze(2)
    
    def make_tgt_mask(self, tgt):
        seq_len = tgt.size(1)
        
        # 下三角掩码（防止看到未来的词）
        subsequent_mask = torch.tril(torch.ones(seq_len, seq_len)).unsqueeze(0).unsqueeze(0)
        
        # Padding 掩码
        padding_mask = (tgt != 0).unsqueeze(1).unsqueeze(2)
        
        return subsequent_mask & padding_mask
    
    def forward(self, src, tgt):
        src_mask = self.make_src_mask(src)
        tgt_mask = self.make_tgt_mask(tgt)
        
        encoder_output = self.encoder(src, src_mask)
        decoder_output = self.decoder(tgt, encoder_output, tgt_mask, src_mask)
        
        return decoder_output
    
    def encode(self, src):
        return self.encoder(src)
    
    def decode(self, tgt, encoder_output):
        tgt_mask = self.make_tgt_mask(tgt)
        return self.decoder(tgt, encoder_output, tgt_mask)
```

---

## 八、掩码机制

### 8.1 Padding 掩码

处理变长序列，忽略 padding 位置：

```python
def create_padding_mask(seq, pad_idx=0):
    """
    seq: (batch_size, seq_len)
    return: (batch_size, 1, 1, seq_len)
    """
    return (seq != pad_idx).unsqueeze(1).unsqueeze(2)

# 示例
seq = torch.tensor([[1, 2, 3, 0, 0], [4, 5, 0, 0, 0]])
mask = create_padding_mask(seq)
# mask[0]: [[[True, True, True, False, False]]]
# mask[1]: [[[True, True, False, False, False]]]
```

### 8.2 因果掩码(Look-ahead Mask)

防止解码器看到未来的词：

```python
def create_causal_mask(seq_len):
    """
    创建下三角掩码
    """
    mask = torch.tril(torch.ones(seq_len, seq_len))
    return mask.unsqueeze(0).unsqueeze(0)

# 示例 (seq_len=5)
# [[[1, 0, 0, 0, 0],
#   [1, 1, 0, 0, 0],
#   [1, 1, 1, 0, 0],
#   [1, 1, 1, 1, 0],
#   [1, 1, 1, 1, 1]]]
```

### 8.3 掩码在注意力中的应用

```python
# 在注意力分数上应用掩码
scores = torch.matmul(Q, K.transpose(-2, -1)) / self.scale

if mask is not None:
    # 将 mask=0 的位置设为负无穷
    scores = scores.masked_fill(mask == 0, float('-inf'))

attention_weights = torch.softmax(scores, dim=-1)
# 负无穷位置 softmax 后为 0
```

---

## 九、训练与推理

### 9.1 训练过程

```python
def train_step(model, src, tgt, criterion, optimizer):
    model.train()
    optimizer.zero_grad()
    
    # 教师强制：输入 tgt[:-1]，预测 tgt[1:]
    tgt_input = tgt[:, :-1]
    tgt_output = tgt[:, 1:]
    
    # 前向传播
    logits = model(src, tgt_input)
    
    # 计算损失
    loss = criterion(
        logits.reshape(-1, logits.size(-1)),
        tgt_output.reshape(-1)
    )
    
    # 反向传播
    loss.backward()
    optimizer.step()
    
    return loss.item()
```

### 9.2 推理过程（贪婪解码）

```python
def greedy_decode(model, src, max_len, start_token, end_token):
    model.eval()
    
    with torch.no_grad():
        # 编码
        encoder_output = model.encode(src)
        
        # 初始化解码输入
        ys = torch.ones(1, 1).fill_(start_token).long().to(src.device)
        
        for _ in range(max_len - 1):
            # 解码
            logits = model.decode(ys, encoder_output)
            
            # 取最后一个位置的预测
            next_token = logits[:, -1, :].argmax(dim=-1, keepdim=True)
            
            # 拼接
            ys = torch.cat([ys, next_token], dim=1)
            
            # 遇到结束符停止
            if next_token.item() == end_token:
                break
        
        return ys
```

### 9.3 束搜索(Beam Search)

```python
def beam_search_decode(model, src, max_len, start_token, end_token, 
                       beam_size=5):
    model.eval()
    
    with torch.no_grad():
        encoder_output = model.encode(src)
        
        # 初始化 beams: [(序列, 分数)]
        beams = [([start_token], 0.0)]
        
        for _ in range(max_len - 1):
            candidates = []
            
            for seq, score in beams:
                if seq[-1] == end_token:
                    candidates.append((seq, score))
                    continue
                
                # 解码
                tgt_tensor = torch.tensor([seq]).to(src.device)
                logits = model.decode(tgt_tensor, encoder_output)
                
                # 取 top-k
                log_probs = torch.log_softmax(logits[:, -1, :], dim=-1)
                topk_probs, topk_indices = log_probs.topk(beam_size)
                
                for i in range(beam_size):
                    new_seq = seq + [topk_indices[0, i].item()]
                    new_score = score + topk_probs[0, i].item()
                    candidates.append((new_seq, new_score))
            
            # 保留 top-k beams
            beams = sorted(candidates, key=lambda x: x[1], reverse=True)[:beam_size]
            
            # 检查是否所有 beam 都结束
            if all(seq[-1] == end_token for seq, _ in beams):
                break
        
        return beams[0][0]
```

---

## 十、Transformer 变体

### 10.1 Encoder-Only (BERT)

```python
class BERT(nn.Module):
    """只使用编码器，用于理解任务"""
    def __init__(self, vocab_size, embed_dim, num_heads, ffn_dim,
                 num_layers, dropout=0.1):
        super().__init__()
        self.encoder = Encoder(vocab_size, embed_dim, num_heads,
                               ffn_dim, num_layers, dropout=dropout)
    
    def forward(self, x):
        return self.encoder(x)
```

**应用**：文本分类、命名实体识别、问答系统

### 10.2 Decoder-Only (GPT)

```python
class GPT(nn.Module):
    """只使用解码器，用于生成任务"""
    def __init__(self, vocab_size, embed_dim, num_heads, ffn_dim,
                 num_layers, dropout=0.1):
        super().__init__()
        
        self.token_embedding = nn.Embedding(vocab_size, embed_dim)
        self.pos_encoding = PositionalEncoding(embed_dim)
        
        # 只用解码器块（无交叉注意力）
        self.layers = nn.ModuleList([
            TransformerBlock(embed_dim, num_heads, ffn_dim, dropout)
            for _ in range(num_layers)
        ])
        
        self.norm = nn.LayerNorm(embed_dim)
        self.lm_head = nn.Linear(embed_dim, vocab_size)
    
    def forward(self, x):
        x = self.token_embedding(x)
        x = self.pos_encoding(x)
        
        # 创建因果掩码
        causal_mask = create_causal_mask(x.size(1)).to(x.device)
        
        for layer in self.layers:
            x = layer(x, causal_mask)
        
        x = self.norm(x)
        logits = self.lm_head(x)
        
        return logits
```

**应用**：文本生成、代码生成、对话系统

### 10.3 Encoder-Decoder (T5、BART)

保留完整结构，适用于序列到序列任务。

**应用**：机器翻译、文本摘要、问答生成

---

## 十一、总结

### 11.1 Transformer 核心优势

| 特性 | 优势 |
|------|------|
| 自注意力 | 全局依赖建模 |
| 并行计算 | 训练效率高 |
| 多头机制 | 多种表示学习 |
| 残差连接 | 深层网络可训练 |
| 位置编码 | 保留序列信息 |

### 11.2 计算复杂度分析

| 操作 | 复杂度 | 说明 |
|------|--------|------|
| 自注意力 | O(n²d) | n 为序列长度 |
| 前馈网络 | O(nd²) | d 为隐藏维度 |
| 总复杂度 | O(n²d + nd²) | |

### 11.3 Transformer 家族

```
                    Transformer (2017)
                          │
           ┌──────────────┼──────────────┐
           │              │              │
        Encoder-Only   Decoder-Only  Encoder-Decoder
           │              │              │
         BERT            GPT            T5
        (2018)          (2018)        (2019)
           │              │              │
         RoBERTa        GPT-2          BART
        (2019)          (2019)        (2019)
           │              │
         ALBERT         GPT-3
        (2019)          (2020)
                          │
                        LLaMA
                        (2023)
```

---

## 参考资料

- 《Attention Is All You Need》- Vaswani et al., 2017
- 《The Annotated Transformer》
- 《BERT: Pre-training of Deep Bidirectional Transformers》
- 《Language Models are Few-Shot Learners》(GPT-3)
- PyTorch 官方 Transformer 教程