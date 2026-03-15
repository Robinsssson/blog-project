---
title: "MAPPO多智能体强化学习算法详解"
date: 2026-03-15T21:31:00+08:00
draft: false
categories: ["强化学习"]
tags: ["MAPPO", "多智能体", "PPO", "强化学习", "MARL"]
math: true
toc: true
readingTime: true
description: "深入讲解MAPPO算法原理、架构设计及完整PyTorch代码实现"
---

## 概述

MAPPO（Multi-Agent Proximal Policy Optimization）是一种高效的多智能体强化学习算法，由Google Research在2021年提出。它将单智能体PPO算法成功扩展到多智能体场景，在多个基准测试中取得了优异的性能。

### 为什么需要MAPPO？

多智能体强化学习面临的挑战：

| 挑战 | 描述 |
|------|------|
| **非平稳性** | 其他智能体的策略在变化，环境对每个智能体来说是非平稳的 |
| **信用分配** | 团队奖励难以分配到个体贡献 |
| **可扩展性** | 智能体数量增加时状态空间指数增长 |
| **部分可观测** | 每个智能体只能观测局部信息 |

MAPPO的解决方案：

- **集中训练分布执行（CTDE）**：训练时利用全局信息，执行时只用局部观测
- **值函数分解**：集中式Critic学习全局价值
- **简单有效**：基于PPO，无需复杂的价值分解网络

---

## 一、背景知识

### 1.1 PPO回顾

PPO是一种on-policy策略梯度算法，核心思想是通过限制策略更新幅度来保证训练稳定性：

$$
L^{CLIP}(\theta) = \mathbb{E}_t \left[ \min\left( r_t(\theta) \hat{A}_t, \text{clip}(r_t(\theta), 1-\epsilon, 1+\epsilon) \hat{A}_t \right) \right]
$$

其中：
- $r_t(\theta) = \frac{\pi_\theta(a_t|s_t)}{\pi_{\theta_{old}}(a_t|s_t)}$ 为重要性采样比率
- $\hat{A}_t$ 为优势函数估计
- $\epsilon$ 为裁剪参数

### 1.2 多智能体设置

考虑一个部分可观测的马尔可夫博弈（POSG）：

$$
\mathcal{G} = \langle \mathcal{N}, \mathcal{S}, \{\mathcal{A}_i\}, \{\mathcal{O}_i\}, P, \{R_i\}, \gamma \rangle
$$

- $\mathcal{N} = \{1, 2, \ldots, n\}$：智能体集合
- $\mathcal{S}$：全局状态空间
- $\mathcal{A}_i$：智能体 $i$ 的动作空间
- $\mathcal{O}_i$：智能体 $i$ 的观测空间
- $P(s'|s, \mathbf{a})$：状态转移概率
- $R_i(s, \mathbf{a})$：智能体 $i$ 的奖励函数

**CTDE范式：**

```
训练阶段 (Centralized):
┌─────────────────────────────────────────┐
│              全局状态 s                  │
│    ┌─────┬─────┬─────┬─────┐           │
│    │ o_1 │ o_2 │ ... │ o_n │           │
│    └─────┴─────┴─────┴─────┘           │
│              ↓                          │
│    ┌─────────────────────────┐          │
│    │    集中式 Critic        │          │
│    │    V(s) 或 Q(s,a)      │          │
│    └─────────────────────────┘          │
└─────────────────────────────────────────┘

执行阶段 (Decentralized):
┌───────┐   ┌───────┐       ┌───────┐
│ Agent1│   │ Agent2│  ...  │ Agentn│
│ π(a|o)│   │ π(a|o)│       │ π(a|o)│
└───────┘   └───────┘       └───────┘
```

---

## 二、MAPPO算法设计

### 2.1 核心思想

MAPPO的核心设计：

1. **独立Actor**：每个智能体维护独立的策略网络 $\pi_{\theta_i}(a_i|o_i)$
2. **集中Critic**：训练时使用全局状态 $s$ 学习值函数 $V_\phi(s)$
3. **PPO目标**：使用PPO的目标函数进行策略优化

### 2.2 算法架构

```
┌──────────────────────────────────────────────────────────────┐
│                        MAPPO 架构                             │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│   Environment                                                │
│       │                                                      │
│       ▼                                                      │
│   ┌───────────────────────────────────────┐                  │
│   │ 全局状态 s = [o_1, o_2, ..., o_n]     │                  │
│   └───────────────┬───────────────────────┘                  │
│                   │                                          │
│         ┌────────┴────────┐                                  │
│         ▼                 ▼                                  │
│   ┌───────────┐     ┌───────────┐                           │
│   │  Actor 1  │     │  Critic   │  (Centralized)            │
│   │ π(o_1)    │     │  V(s)     │                           │
│   └─────┬─────┘     └─────┬─────┘                           │
│         │                 │                                  │
│   ┌───────────┐           │                                  │
│   │  Actor 2  │           │                                  │
│   │ π(o_2)    │           │                                  │
│   └─────┬─────┘           │                                  │
│         │                 │                                  │
│        ...               ...                                 │
│         │                 │                                  │
│   ┌───────────┐           │                                  │
│   │  Actor n  │           │                                  │
│   │ π(o_n)    │           │                                  │
│   └─────┬─────┘           │                                  │
│         │                 │                                  │
│         └────────┬────────┘                                  │
│                  ▼                                           │
│   ┌──────────────────────────────────┐                       │
│   │         PPO Update               │                       │
│   │  L = L_CLIP + c1·L_VF - c2·L_ENT │                       │
│   └──────────────────────────────────┘                       │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 2.3 参数共享策略

MAPPO支持两种参数配置：

**1. 独立参数（Individual Parameters）**

每个智能体有独立的Actor和Critic：

$$
\theta_1, \theta_2, \ldots, \theta_n, \quad \phi_1, \phi_2, \ldots, \phi_n
$$

优点：可以学习异构策略
缺点：参数量大，数据效率低

**2. 参数共享（Parameter Sharing）**

所有智能体共享同一个网络：

$$
\theta_1 = \theta_2 = \ldots = \theta_n = \theta
$$

优点：参数效率高，数据共享
缺点：假设智能体同构

**建议**：同构智能体使用共享参数，异构智能体使用独立参数或添加智能体ID编码。

---

## 三、核心组件实现

### 3.1 Actor网络

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
from torch.distributions import Categorical, Normal
import numpy as np

class ActorNetwork(nn.Module):
    """策略网络 (Actor)"""
    
    def __init__(self, obs_dim, action_dim, hidden_dim=64, 
                 use_agent_id=False, n_agents=1):
        super().__init__()
        
        self.use_agent_id = use_agent_id
        input_dim = obs_dim
        
        # 智能体ID编码
        if use_agent_id:
            self.agent_id_embedding = nn.Embedding(n_agents, hidden_dim // 2)
            input_dim = obs_dim + hidden_dim // 2
        
        # 网络结构
        self.fc1 = nn.Linear(input_dim, hidden_dim)
        self.fc2 = nn.Linear(hidden_dim, hidden_dim)
        self.fc3 = nn.Linear(hidden_dim, action_dim)
        
        # 初始化
        self._init_weights()
    
    def _init_weights(self):
        for m in [self.fc1, self.fc2, self.fc3]:
            nn.init.orthogonal_(m.weight, gain=np.sqrt(2))
            nn.init.constant_(m.bias, 0)
        # 输出层使用更小的初始化
        nn.init.orthogonal_(self.fc3.weight, gain=0.01)
    
    def forward(self, obs, agent_id=None):
        """
        Args:
            obs: (batch, obs_dim) 或 (batch, n_agents, obs_dim)
            agent_id: 智能体ID索引
        Returns:
            action_probs: (batch, action_dim) 或 (batch, n_agents, action_dim)
        """
        if self.use_agent_id and agent_id is not None:
            agent_embed = self.agent_id_embedding(agent_id)
            obs = torch.cat([obs, agent_embed], dim=-1)
        
        x = F.relu(self.fc1(obs))
        x = F.relu(self.fc2(x))
        action_logits = self.fc3(x)
        
        return action_logits
    
    def get_action(self, obs, agent_id=None, deterministic=False):
        """采样动作"""
        action_logits = self.forward(obs, agent_id)
        probs = F.softmax(action_logits, dim=-1)
        
        if deterministic:
            action = torch.argmax(probs, dim=-1)
        else:
            dist = Categorical(probs)
            action = dist.sample()
        
        log_prob = self.get_log_prob(obs, action, agent_id)
        
        return action, log_prob
    
    def get_log_prob(self, obs, action, agent_id=None):
        """计算动作的对数概率"""
        action_logits = self.forward(obs, agent_id)
        dist = Categorical(logits=action_logits)
        return dist.log_prob(action)
    
    def evaluate_actions(self, obs, actions, agent_id=None):
        """评估动作（用于训练）"""
        action_logits = self.forward(obs, agent_id)
        dist = Categorical(logits=action_logits)
        
        log_prob = dist.log_prob(actions)
        entropy = dist.entropy()
        
        return log_prob, entropy
```

### 3.2 Critic网络

```python
class CriticNetwork(nn.Module):
    """价值网络 (Critic) - 集中式"""
    
    def __init__(self, state_dim, hidden_dim=64):
        super().__init__()
        
        self.fc1 = nn.Linear(state_dim, hidden_dim)
        self.fc2 = nn.Linear(hidden_dim, hidden_dim)
        self.fc3 = nn.Linear(hidden_dim, 1)
        
        self._init_weights()
    
    def _init_weights(self):
        for m in [self.fc1, self.fc2, self.fc3]:
            nn.init.orthogonal_(m.weight, gain=np.sqrt(2))
            nn.init.constant_(m.bias, 0)
    
    def forward(self, state):
        """
        Args:
            state: (batch, state_dim) 全局状态
        Returns:
            value: (batch, 1) 状态价值
        """
        x = F.relu(self.fc1(state))
        x = F.relu(self.fc2(x))
        value = self.fc3(x)
        
        return value


class RunningMeanStd:
    """用于值归一化的统计量"""
    
    def __init__(self, shape=(), epsilon=1e-4):
        self.mean = np.zeros(shape, dtype=np.float32)
        self.var = np.ones(shape, dtype=np.float32)
        self.count = epsilon
    
    def update(self, x):
        batch_mean = np.mean(x, axis=0)
        batch_var = np.var(x, axis=0)
        batch_count = x.shape[0]
        
        self.update_from_moments(batch_mean, batch_var, batch_count)
    
    def update_from_moments(self, batch_mean, batch_var, batch_count):
        delta = batch_mean - self.mean
        total_count = self.count + batch_count
        
        new_mean = self.mean + delta * batch_count / total_count
        m_a = self.var * self.count
        m_b = batch_var * batch_count
        M2 = m_a + m_b + np.square(delta) * self.count * batch_count / total_count
        
        self.mean = new_mean
        self.var = M2 / total_count
        self.count = total_count


class NormalizedCritic(nn.Module):
    """带归一化的Critic"""
    
    def __init__(self, state_dim, hidden_dim=64, use_normalization=True):
        super().__init__()
        
        self.critic = CriticNetwork(state_dim, hidden_dim)
        self.use_normalization = use_normalization
        
        if use_normalization:
            self.value_normalizer = RunningMeanStd(shape=())
    
    def forward(self, state, update_normalizer=True):
        value = self.critic(state)
        
        if self.use_normalization:
            if update_normalizer and self.training:
                self.value_normalizer.update(value.detach().cpu().numpy())
            value = value / (np.sqrt(self.value_normalizer.var) + 1e-8)
        
        return value
    
    def denormalize(self, value):
        """反归一化，用于计算优势函数"""
        if self.use_normalization:
            return value * np.sqrt(self.value_normalizer.var) + self.value_normalizer.mean
        return value
```

### 3.3 MAPPO Agent

```python
class MAPPOAgent:
    """MAPPO智能体"""
    
    def __init__(self, 
                 obs_dim, 
                 action_dim, 
                 state_dim,
                 n_agents=1,
                 hidden_dim=64,
                 lr_actor=3e-4,
                 lr_critic=1e-3,
                 gamma=0.99,
                 gae_lambda=0.95,
                 clip_epsilon=0.2,
                 value_loss_coef=0.5,
                 entropy_coef=0.01,
                 max_grad_norm=0.5,
                 use_agent_id=False,
                 use_value_normalization=True,
                 share_actor=True):
        
        self.n_agents = n_agents
        self.gamma = gamma
        self.gae_lambda = gae_lambda
        self.clip_epsilon = clip_epsilon
        self.value_loss_coef = value_loss_coef
        self.entropy_coef = entropy_coef
        self.max_grad_norm = max_grad_norm
        self.share_actor = share_actor
        
        # Actor网络
        if share_actor:
            self.actor = ActorNetwork(obs_dim, action_dim, hidden_dim, 
                                       use_agent_id, n_agents)
            self.actor_optimizer = torch.optim.Adam(self.actor.parameters(), lr=lr_actor)
        else:
            self.actors = nn.ModuleList([
                ActorNetwork(obs_dim, action_dim, hidden_dim)
                for _ in range(n_agents)
            ])
            self.actor_optimizer = torch.optim.Adam(self.actors.parameters(), lr=lr_actor)
        
        # Critic网络（集中式）
        self.critic = NormalizedCritic(state_dim, hidden_dim, use_value_normalization)
        self.critic_optimizer = torch.optim.Adam(self.critic.parameters(), lr=lr_critic)
    
    def get_actions(self, observations, deterministic=False):
        """
        获取所有智能体的动作
        Args:
            observations: (n_agents, obs_dim) 或 (batch, n_agents, obs_dim)
        Returns:
            actions: (n_agents,) 或 (batch, n_agents)
            log_probs: (n_agents,) 或 (batch, n_agents)
        """
        if observations.dim() == 2:
            observations = observations.unsqueeze(0)
        
        batch_size = observations.shape[0]
        actions = []
        log_probs = []
        
        for i in range(self.n_agents):
            obs_i = observations[:, i, :]
            
            if self.share_actor:
                agent_id = torch.tensor([i] * batch_size, device=observations.device)
                action, log_prob = self.actor.get_action(obs_i, agent_id, deterministic)
            else:
                action, log_prob = self.actors[i].get_action(obs_i, deterministic=False)
            
            actions.append(action)
            log_probs.append(log_prob)
        
        actions = torch.stack(actions, dim=-1)
        log_probs = torch.stack(log_probs, dim=-1)
        
        return actions, log_probs
    
    def compute_gae(self, rewards, values, dones, next_value):
        """
        计算广义优势估计 (GAE)
        Args:
            rewards: (T, batch, n_agents) 或 (T, batch)
            values: (T+1, batch) 需要包含最后一步的value
            dones: (T, batch)
            next_value: (batch,)
        """
        T = rewards.shape[0]
        advantages = torch.zeros_like(rewards)
        last_gae = 0
        
        for t in reversed(range(T)):
            if t == T - 1:
                next_val = next_value
            else:
                next_val = values[t + 1]
            
            delta = rewards[t] + self.gamma * next_val * (1 - dones[t]) - values[t]
            advantages[t] = last_gae = delta + self.gamma * self.gae_lambda * (1 - dones[t]) * last_gae
        
        returns = advantages + values[:-1]
        
        return advantages, returns
    
    def update(self, batch):
        """
        PPO更新
        Args:
            batch: dict containing:
                - observations: (T, batch, n_agents, obs_dim)
                - actions: (T, batch, n_agents)
                - old_log_probs: (T, batch, n_agents)
                - advantages: (T, batch, n_agents) 或 (T, batch)
                - returns: (T, batch)
                - states: (T, batch, state_dim)
        """
        observations = batch['observations']
        actions = batch['actions']
        old_log_probs = batch['old_log_probs']
        advantages = batch['advantages']
        returns = batch['returns']
        states = batch['states']
        
        T, batch_size = states.shape[0], states.shape[1]
        
        # 展平batch维度
        observations = observations.view(-1, self.n_agents, -1)
        actions = actions.view(-1, self.n_agents)
        old_log_probs = old_log_probs.view(-1, self.n_agents)
        advantages = advantages.view(-1)
        returns = returns.view(-1)
        states = states.view(-1, states.shape[-1])
        
        # 标准化优势
        advantages = (advantages - advantages.mean()) / (advantages.std() + 1e-8)
        
        # 计算当前策略的log_prob和熵
        new_log_probs = []
        entropies = []
        
        for i in range(self.n_agents):
            obs_i = observations[:, i, :]
            action_i = actions[:, i]
            
            if self.share_actor:
                agent_id = torch.tensor([i] * observations.shape[0], device=observations.device)
                log_prob, entropy = self.actor.evaluate_actions(obs_i, action_i, agent_id)
            else:
                log_prob, entropy = self.actors[i].evaluate_actions(obs_i, action_i)
            
            new_log_probs.append(log_prob)
            entropies.append(entropy)
        
        new_log_probs = torch.stack(new_log_probs, dim=-1)
        entropies = torch.stack(entropies, dim=-1)
        
        # 计算重要性采样比率
        ratio = torch.exp(new_log_probs - old_log_probs)
        
        # PPO裁剪目标
        if advantages.dim() == 1:
            advantages = advantages.unsqueeze(-1).expand(-1, self.n_agents)
        
        surr1 = ratio * advantages
        surr2 = torch.clamp(ratio, 1 - self.clip_epsilon, 1 + self.clip_epsilon) * advantages
        policy_loss = -torch.min(surr1, surr2).mean()
        
        # 价值损失
        values = self.critic(states).squeeze(-1)
        value_loss = F.mse_loss(values, returns)
        
        # 熵奖励
        entropy_loss = -entropies.mean()
        
        # 总损失
        loss = policy_loss + self.value_loss_coef * value_loss + self.entropy_coef * entropy_loss
        
        # 更新网络
        self.actor_optimizer.zero_grad()
        self.critic_optimizer.zero_grad()
        loss.backward()
        
        # 梯度裁剪
        nn.utils.clip_grad_norm_(self.actor.parameters(), self.max_grad_norm)
        nn.utils.clip_grad_norm_(self.critic.parameters(), self.max_grad_norm)
        
        self.actor_optimizer.step()
        self.critic_optimizer.step()
        
        return {
            'policy_loss': policy_loss.item(),
            'value_loss': value_loss.item(),
            'entropy': -entropy_loss.item()
        }
```

---

## 四、训练流程

### 4.1 经验收集

```python
class RolloutBuffer:
    """经验回放缓冲区"""
    
    def __init__(self, buffer_size, n_agents, obs_dim, action_dim, state_dim):
        self.buffer_size = buffer_size
        self.n_agents = n_agents
        
        # 存储空间
        self.observations = np.zeros((buffer_size, n_agents, obs_dim), dtype=np.float32)
        self.actions = np.zeros((buffer_size, n_agents), dtype=np.int64)
        self.rewards = np.zeros((buffer_size,), dtype=np.float32)
        self.dones = np.zeros((buffer_size,), dtype=np.float32)
        self.log_probs = np.zeros((buffer_size, n_agents), dtype=np.float32)
        self.states = np.zeros((buffer_size, state_dim), dtype=np.float32)
        self.values = np.zeros((buffer_size,), dtype=np.float32)
        
        self.ptr = 0
        self.size = 0
    
    def add(self, obs, actions, reward, done, log_probs, state, value):
        """添加一条经验"""
        self.observations[self.ptr] = obs
        self.actions[self.ptr] = actions
        self.rewards[self.ptr] = reward
        self.dones[self.ptr] = done
        self.log_probs[self.ptr] = log_probs
        self.states[self.ptr] = state
        self.values[self.ptr] = value
        
        self.ptr = (self.ptr + 1) % self.buffer_size
        self.size = min(self.size + 1, self.buffer_size)
    
    def get(self):
        """获取所有数据"""
        return {
            'observations': self.observations[:self.size],
            'actions': self.actions[:self.size],
            'rewards': self.rewards[:self.size],
            'dones': self.dones[:self.size],
            'log_probs': self.log_probs[:self.size],
            'states': self.states[:self.size],
            'values': self.values[:self.size]
        }
    
    def clear(self):
        """清空缓冲区"""
        self.ptr = 0
        self.size = 0


def collect_trajectories(env, agent, n_steps, device='cpu'):
    """
    收集训练数据
    Args:
        env: 多智能体环境
        agent: MAPPO智能体
        n_steps: 收集步数
    """
    buffer = RolloutBuffer(
        n_steps,
        agent.n_agents,
        env.observation_space[0].shape[0],
        env.action_space[0].n,
        env.state_space.shape[0]
    )
    
    obs = env.reset()
    
    for step in range(n_steps):
        # 转换为tensor
        obs_tensor = torch.FloatTensor(obs).to(device)
        state_tensor = torch.FloatTensor(env.state()).to(device)
        
        # 获取动作
        with torch.no_grad():
            actions, log_probs = agent.get_actions(obs_tensor)
            value = agent.critic(state_tensor.unsqueeze(0)).squeeze()
        
        actions_np = actions.cpu().numpy().flatten()
        log_probs_np = log_probs.cpu().numpy().flatten()
        value_np = value.cpu().item()
        
        # 环境交互
        next_obs, rewards, dones, info = env.step(actions_np)
        
        # 存储经验
        buffer.add(
            obs=obs,
            actions=actions_np,
            reward=rewards[0] if isinstance(rewards, (list, np.ndarray)) else rewards,
            done=dones[0] if isinstance(dones, (list, np.ndarray)) else dones,
            log_probs=log_probs_np,
            state=env.state(),
            value=value_np
        )
        
        obs = next_obs
        
        if all(dones) if isinstance(dones, (list, np.ndarray)) else dones:
            obs = env.reset()
    
    # 计算最后一步的value
    with torch.no_grad():
        last_state = torch.FloatTensor(env.state()).to(device)
        last_value = agent.critic(last_state.unsqueeze(0)).squeeze().cpu().item()
    
    return buffer, last_value
```

### 4.2 完整训练循环

```python
def train_mappo(env, agent, total_timesteps, n_steps=128, n_epochs=4, 
                batch_size=32, device='cpu', log_interval=10):
    """
    MAPPO训练主循环
    """
    timesteps_collected = 0
    episode_rewards = []
    episode_lengths = []
    
    while timesteps_collected < total_timesteps:
        # 1. 收集经验
        buffer, last_value = collect_trajectories(env, agent, n_steps, device)
        timesteps_collected += n_steps
        
        # 2. 计算优势函数
        data = buffer.get()
        rewards = torch.FloatTensor(data['rewards'])
        dones = torch.FloatTensor(data['dones'])
        values = torch.FloatTensor(data['values'])
        
        # 添加最后一步的value
        values_with_last = torch.cat([values, torch.tensor([last_value])])
        
        # GAE计算
        advantages, returns = agent.compute_gae(
            rewards.unsqueeze(-1).expand(-1, agent.n_agents),
            values_with_last,
            dones,
            last_value
        )
        
        # 3. PPO更新（多轮）
        for epoch in range(n_epochs):
            # 随机打乱数据
            indices = np.random.permutation(n_steps)
            
            for start in range(0, n_steps, batch_size):
                end = start + batch_size
                batch_indices = indices[start:end]
                
                batch = {
                    'observations': torch.FloatTensor(data['observations'][batch_indices]),
                    'actions': torch.LongTensor(data['actions'][batch_indices]),
                    'old_log_probs': torch.FloatTensor(data['log_probs'][batch_indices]),
                    'advantages': advantages[batch_indices],
                    'returns': returns[batch_indices],
                    'states': torch.FloatTensor(data['states'][batch_indices])
                }
                
                losses = agent.update(batch)
        
        # 4. 记录统计信息
        if 'episode' in data:
            episode_rewards.extend(data.get('episode_rewards', []))
        
        if timesteps_collected % (log_interval * n_steps) == 0:
            mean_reward = np.mean(episode_rewards[-100:]) if episode_rewards else 0
            print(f"Timesteps: {timesteps_collected}/{total_timesteps}, "
                  f"Mean Reward: {mean_reward:.2f}, "
                  f"Policy Loss: {losses['policy_loss']:.4f}, "
                  f"Value Loss: {losses['value_loss']:.4f}")
    
    return agent
```

---

## 五、环境接口示例

### 5.1 兼容OpenAI Gym的接口

```python
import gym
from gym import spaces
import numpy as np

class MultiAgentEnvWrapper:
    """
    多智能体环境包装器
    遵循OpenAI Gym接口
    """
    
    def __init__(self, env):
        self.env = env
        self.n_agents = env.n_agents
        
        # 定义空间
        self.observation_space = [
            spaces.Box(low=-np.inf, high=np.inf, shape=(env.obs_dim,), dtype=np.float32)
            for _ in range(self.n_agents)
        ]
        self.action_space = [
            spaces.Discrete(env.action_dim)
            for _ in range(self.n_agents)
        ]
        self.state_space = spaces.Box(
            low=-np.inf, high=np.inf, shape=(env.state_dim,), dtype=np.float32
        )
    
    def reset(self):
        """重置环境"""
        return self.env.reset()
    
    def step(self, actions):
        """
        执行动作
        Args:
            actions: (n_agents,) 每个智能体的动作
        Returns:
            observations: (n_agents, obs_dim)
            rewards: (n_agents,) 或 标量
            dones: (n_agents,) 或 标量
            info: dict
        """
        return self.env.step(actions)
    
    def state(self):
        """获取全局状态"""
        return self.env.get_state()
    
    def render(self):
        self.env.render()
    
    def close(self):
        self.env.close()


# 示例：简单协作环境
class SimpleSpreadEnv:
    """简单的多智能体协作导航环境"""
    
    def __init__(self, n_agents=3, world_size=1.0):
        self.n_agents = n_agents
        self.world_size = world_size
        self.obs_dim = 4 + n_agents - 1  # 位置 + 其他智能体相对位置
        self.action_dim = 5  # 上、下、左、右、不动
        self.state_dim = n_agents * 2 + n_agents * 2  # 所有位置 + 目标位置
        
        self.agents_pos = None
        self.targets = None
        self.max_steps = 25
        self.step_count = 0
    
    def reset(self):
        """重置环境"""
        # 随机初始化智能体位置
        self.agents_pos = np.random.rand(self.n_agents, 2) * self.world_size
        
        # 随机初始化目标位置
        self.targets = np.random.rand(self.n_agents, 2) * self.world_size
        
        self.step_count = 0
        return self._get_observations()
    
    def _get_observations(self):
        """获取各智能体的局部观测"""
        observations = []
        
        for i in range(self.n_agents):
            # 自身位置
            obs = [self.agents_pos[i]]
            
            # 其他智能体的相对位置
            for j in range(self.n_agents):
                if i != j:
                    rel_pos = self.agents_pos[j] - self.agents_pos[i]
                    obs.append(rel_pos)
            
            # 目标相对位置
            target_rel = self.targets[i] - self.agents_pos[i]
            obs.append(target_rel)
            
            observations.append(np.concatenate(obs))
        
        return np.array(observations)
    
    def get_state(self):
        """获取全局状态"""
        return np.concatenate([
            self.agents_pos.flatten(),
            self.targets.flatten()
        ])
    
    def step(self, actions):
        """执行动作"""
        self.step_count += 1
        
        # 动作映射
        action_effects = np.array([
            [0, 0.1],    # 上
            [0, -0.1],   # 下
            [-0.1, 0],   # 左
            [0.1, 0],    # 右
            [0, 0]       # 不动
        ])
        
        # 更新位置
        for i, action in enumerate(actions):
            self.agents_pos[i] += action_effects[action]
            self.agents_pos[i] = np.clip(self.agents_pos[i], 0, self.world_size)
        
        # 计算奖励
        distances = np.linalg.norm(self.agents_pos - self.targets, axis=1)
        reward = -distances.sum()  # 负距离作为奖励
        
        # 检查终止条件
        done = (self.step_count >= self.max_steps) or (distances.max() < 0.1)
        
        return self._get_observations(), reward, done, {}
```

---

## 六、训练技巧与调优

### 6.1 MAPPO关键超参数

| 超参数 | 推荐值 | 说明 |
|--------|--------|------|
| `clip_epsilon` | 0.2 | PPO裁剪参数 |
| `gae_lambda` | 0.95 | GAE平滑参数 |
| `lr_actor` | 3e-4 | Actor学习率 |
| `lr_critic` | 1e-3 | Critic学习率 |
| `entropy_coef` | 0.01 | 熵正则化系数 |
| `value_loss_coef` | 0.5 | 价值损失系数 |
| `n_epochs` | 5-10 | 每次收集后的更新轮数 |
| `n_steps` | 128-256 | 每次收集的步数 |
| `batch_size` | 32-64 | 小批量大小 |

### 6.2 值归一化

值归一化对MAPPO的性能至关重要：

```python
# 不使用归一化：值函数可能跨越很大范围，训练不稳定
# 使用归一化：值函数被归一化到接近标准正态分布

# 实现要点：
# 1. 训练时使用归一化的值
# 2. 计算优势函数时反归一化
# 3. 使用Running Mean-Std统计
```

### 6.3 PopArt

更高级的值归一化技术：

```python
class PopArt(nn.Module):
    """PopArt值归一化"""
    
    def __init__(self, input_dim, output_dim, beta=0.0001):
        super().__init__()
        
        self.linear = nn.Linear(input_dim, output_dim)
        self.beta = beta
        
        # 可学习的统计量
        self.register_buffer('mu', torch.zeros(output_dim))
        self.register_buffer('sigma', torch.ones(output_dim))
    
    def forward(self, x):
        # 归一化的输出
        y = self.linear(x)
        
        # 反归一化得到真实值
        normalized_y = (y - self.mu) / self.sigma
        
        return normalized_y, y
    
    def update(self, targets):
        """更新统计量"""
        old_mu = self.mu.clone()
        old_sigma = self.sigma.clone()
        
        # 计算新的统计量
        new_mu = targets.mean(dim=0)
        new_sigma = targets.std(dim=0)
        
        # 指数移动平均
        self.mu = (1 - self.beta) * old_mu + self.beta * new_mu
        self.sigma = (1 - self.beta) * old_sigma + self.beta * new_sigma
        
        # 调整权重以保持输出不变
        self.linear.weight.data = self.linear.weight * old_sigma / self.sigma
        self.linear.bias.data = (self.linear.bias * old_sigma + old_mu - self.mu) / self.sigma
```

### 6.4 网络架构选择

```python
# 简单任务
hidden_dim = 64

# 中等复杂度任务
hidden_dim = 128
n_layers = 2

# 复杂任务（如SMAC）
hidden_dim = 256
n_layers = 3
use_layer_norm = True
use_orthogonal_init = True

# 使用RNN处理部分可观测性
class GRUActor(nn.Module):
    def __init__(self, obs_dim, action_dim, hidden_dim=64):
        super().__init__()
        self.fc1 = nn.Linear(obs_dim, hidden_dim)
        self.gru = nn.GRU(hidden_dim, hidden_dim, batch_first=True)
        self.fc2 = nn.Linear(hidden_dim, action_dim)
        self.hidden = None
    
    def forward(self, obs, hidden=None):
        x = F.relu(self.fc1(obs))
        if x.dim() == 2:
            x = x.unsqueeze(1)
        x, self.hidden = self.gru(x, hidden)
        return self.fc2(x.squeeze(1)), self.hidden
```

---

## 七、实验结果

### 7.1 基准测试

MAPPO在多个基准上取得了优秀性能：

**StarCraft II Micromanagement (SMAC):**

| 环境 | MAPPO | QMIX | MADDPG |
|------|-------|------|--------|
| 3m | 100% | 100% | 95% |
| 8m | 100% | 98% | 88% |
| 2s3z | 97% | 88% | 72% |
| 5m_vs_6m | 85% | 78% | 62% |
| corridor | 100% | 100% | 85% |

**Multi-Agent Particle Environment (MPE):**

| 环境 | MAPPO | MADDPG |
|------|-------|--------|
| Spread | -400 | -380 |
| Reference | -45 | -50 |
| Predator-Prey | 0.85 | 0.80 |

### 7.2 与其他算法比较

| 算法 | 类型 | 优点 | 缺点 |
|------|------|------|------|
| **MAPPO** | Policy Gradient | 简单有效，稳定 | On-policy，数据效率低 |
| **QMIX** | Value Decomposition | 可扩展，off-policy | 需要价值分解假设 |
| **MADDPG** | Actor-Critic | Off-policy，连续动作 | 训练不稳定 |
| **COMA** | Actor-Critic | Counterfactual baseline | 计算复杂 |

---

## 八、完整示例代码

### 8.1 运行脚本

```python
import torch
import numpy as np
from mappo import MAPPOAgent, train_mappo
from env_wrapper import SimpleSpreadEnv, MultiAgentEnvWrapper

def main():
    # 设置随机种子
    torch.manual_seed(42)
    np.random.seed(42)
    
    # 创建环境
    env = SimpleSpreadEnv(n_agents=3)
    env = MultiAgentEnvWrapper(env)
    
    device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
    
    # 创建智能体
    agent = MAPPOAgent(
        obs_dim=env.observation_space[0].shape[0],
        action_dim=env.action_space[0].n,
        state_dim=env.state_space.shape[0],
        n_agents=env.n_agents,
        hidden_dim=64,
        lr_actor=3e-4,
        lr_critic=1e-3,
        use_agent_id=True,
        use_value_normalization=True,
        share_actor=True
    ).to(device)
    
    # 训练
    trained_agent = train_mappo(
        env=env,
        agent=agent,
        total_timesteps=100000,
        n_steps=128,
        n_epochs=5,
        batch_size=32,
        device=device,
        log_interval=10
    )
    
    # 保存模型
    torch.save({
        'actor': trained_agent.actor.state_dict(),
        'critic': trained_agent.critic.state_dict(),
    }, 'mappo_model.pt')
    
    print("Training completed!")

if __name__ == '__main__':
    main()
```

---

## 九、总结

### 9.1 MAPPO优势

1. **简单**：基于成熟的PPO算法，易于实现和调试
2. **有效**：在多个基准上达到最优性能
3. **稳定**：PPO的裁剪机制保证训练稳定性
4. **灵活**：支持参数共享和独立参数

### 9.2 使用建议

- **值归一化**：必须使用，显著提升性能
- **参数共享**：同构智能体推荐使用
- **Agent ID**：异构智能体需要添加ID编码
- **超参数**：从默认值开始，根据任务调整

### 9.3 扩展方向

- **RNN**：处理部分可观测环境
- **Transformer**：更好的长序列建模
- **分层**：处理复杂任务分解
- **离线RL**：从固定数据集学习

---

## 参考资料

- Yu et al. "The Surprising Effectiveness of PPO in Cooperative Multi-Agent Games", NeurIPS 2021
- Schulman et al. "Proximal Policy Optimization Algorithms", arXiv 2017
- Rashid et al. "QMIX: Monotonic Value Function Factorisation", ICML 2018
- Lowe et al. "Multi-Agent Actor-Critic for Mixed Cooperative-Competitive Environments", NIPS 2017
- SMAC: https://github.com/oxwhirl/smac
- MPE: https://github.com/openai/multiagent-particle-envs