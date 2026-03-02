# SymPO (Symbol Preference Optimization) 公式详解

本文档旨在为深度学习初学者详细解释 GenNM 论文中提出的 **SymPO (Symbol Preference Optimization)** 方法的核心公式。

---

## 1. 背景：为什么需要 SymPO？

在训练生成模型恢复变量名时，存在一个问题：**数据分布不均衡**。

- 有些变量名（如 `buffer`、`data`）在训练数据中出现了上万次
- 而大多数变量名只出现几次甚至一次

这导致模型会**偏向预测高频名称**，即使在某些特定上下文中，低频名称更合适。

例如：在网络编程中，变量应该叫 `packet` 而不是更常见的 `buffer`。

**SymPO 的核心思想**：让模型学会在特定程序上下文中，**更倾向于选择合适的名称，而非仅仅选择高频名称**。

---

## 2. 核心概念解释

### 2.1 符号偏好 (Symbol Preference)

**符号偏好**指的是：给定某个程序上下文，开发者会更倾向于选择某个名称而非另一个。

用数学符号表示：$r_b \succ_r r_w$

- $r_b$：better name（更好的名称）
- $r_w$：worse name（较差的名称）  
- $\succ_r$：表示"在ground-truth名称 $r$ 的参照下，$r_b$ 比 $r_w$ 更好"

### 2.2 涉及的模型

SymPO 涉及两个模型：

| 符号 | 模型名称 | 说明 |
|------|---------|------|
| $\Theta$ | GenNM-SymPO | 正在被优化的模型（目标模型） |
| $\Theta_{ctx}$ | GenNM-Ctx | 参考模型，权重**冻结**不更新 |

---

## 3. SymPO 损失函数详解

### 3.1 主损失函数

$$
\mathcal{L}_{\text{SymPO}}(\Theta, \Theta_{ctx}) := -\mathbb{E}_{(q,b,w) \sim \mathcal{D}_{prf}} \left[ \log \sigma \left( \beta \log \frac{P(b|q; \Theta)}{P(b|q; \Theta_{ctx})} - \beta \log \frac{P(w|q; \Theta)}{P(w|q; \Theta_{ctx})} \right) \right]
$$

**公式 (9)**

#### 逐部分解释：

| 符号 | 含义 |
|------|------|
| $\mathcal{L}_{\text{SymPO}}$ | SymPO的损失函数，需要最小化 |
| $\Theta$ | GenNM-SymPO 模型的参数（要优化的） |
| $\Theta_{ctx}$ | GenNM-Ctx 模型的参数（冻结的参考模型） |
| $\mathbb{E}_{(q,b,w) \sim \mathcal{D}_{prf}}$ | 对偏好数据集 $\mathcal{D}_{prf}$ 中的样本取**期望**（平均值） |
| $(q, b, w)$ | 一个训练样本，包含：<br>• $q$: query（查询的反编译函数）<br>• $b$: better name（更好的名称）<br>• $w$: worse name（较差的名称） |
| $\sigma(\cdot)$ | **Sigmoid 函数**：$\sigma(x) = \frac{1}{1+e^{-x}}$，将任意实数映射到 $(0,1)$ |
| $\beta$ | **超参数**，控制损失函数对概率差距的敏感度 |
| $P(b\|q; \Theta)$ | 给定查询 $q$，模型 $\Theta$ 生成名称 $b$ 的**概率** |
| $\log$ | 自然对数函数 |

#### 直观理解：

这个损失函数的目标是让：

$$
\frac{P(b|q; \Theta)}{P(b|q; \Theta_{ctx})} > \frac{P(w|q; \Theta)}{P(w|q; \Theta_{ctx})}
$$

用人话说：**相比参考模型，目标模型应该更多地提升"好名称"的概率，同时降低"差名称"的概率**。

---

### 3.2 损失函数的梯度形式

$$
\nabla_\Theta \mathcal{L}_{\text{SymPO}}(\Theta, \Theta_{ctx}) := -\beta \mathbb{E}_{(q,b,w) \sim \mathcal{D}_{prf}} \left[ \delta(q,b,w,\Theta,\Theta_{ctx}) \left[ \underbrace{\nabla_\Theta \log P(b|q;\Theta)}_{\text{增加对好名称的偏好}} - \underbrace{\nabla_\Theta \log P(w|q;\Theta)}_{\text{减少对差名称的偏好}} \right] \right]
$$

**公式 (12)**

#### 符号解释：

| 符号 | 含义 |
|------|------|
| $\nabla_\Theta$ | 对参数 $\Theta$ 求**梯度**（偏导数） |
| $\delta(q,b,w,\Theta,\Theta_{ctx})$ | 一个**权重因子**，决定这个样本的更新强度 |

#### 两个关键项的含义：

1. **$\nabla_\Theta \log P(b\|q;\Theta)$**：增加生成"好名称" $b$ 的概率
2. **$-\nabla_\Theta \log P(w\|q;\Theta)$**：减少生成"差名称" $w$ 的概率

---

### 3.3 权重因子 δ 的定义

$$
\delta(q,b,w,\Theta,\Theta_{ctx}) := \sigma\left( \beta \log \frac{P(w|q;\Theta)}{P(w|q;\Theta_{ctx})} - \beta \log \frac{P(b|q;\Theta)}{P(b|q;\Theta_{ctx})} \right)
$$

**公式 (13)**

可以简化为：

$$
\delta = \sigma\left( \beta \left( \log \frac{P(w|q;\Theta)}{P(b|q;\Theta)} - \log \frac{P(w|q;\Theta_{ctx})}{P(b|q;\Theta_{ctx})} \right) \right)
$$

**公式 (14)**

#### 直观理解：

- 当模型 $\Theta$ 还在犯错（给差名称 $w$ 的概率比好名称 $b$ 高）时，$\delta$ 较大，更新幅度大
- 当模型已经学会了正确的偏好时，$\delta$ 较小，更新幅度小

这是一种**自适应机制**：模型在需要改进的地方改进更多。

---

## 4. SymPO 数据集的构建

### 4.1 推理数据集

首先，用已经微调过的模型 GenNM-Ctx 对训练数据进行推理：

$$
\hat{\mathcal{D}} := \left\{ (\text{query}: q, \text{preds}: \hat{r}, \text{gt}: r) \mid (q,r) \in \mathcal{D}_{loc} \cup \mathcal{D}_{ctx} \land \hat{r} = \text{GenNM}_{ctx}(q, \text{top20}) \right\}
$$

**公式 (7)**

| 符号 | 含义 |
|------|------|
| $\hat{\mathcal{D}}$ | 推理得到的数据集 |
| $q$ | 查询（反编译函数） |
| $\hat{r}$ | 模型预测的 top-20 名称 |
| $r$ | ground-truth（真实名称） |
| $\mathcal{D}_{loc}$ | 只含局部信息的训练数据 |
| $\mathcal{D}_{ctx}$ | 含上下文信息的训练数据 |

### 4.2 偏好数据集

$$
\mathcal{D}_{prf} := \left\{ (\text{query}: q, \text{better}: r_b, \text{worse}: r_w) \mid (q, \hat{r}, r) \in \hat{\mathcal{D}} \land r_w = \text{sample}(\hat{r}) \land r_b = \text{best}(\hat{r}, r) \land r_b \succ_r r_w \right\}
$$

**公式 (8)**

| 符号 | 含义 |
|------|------|
| $\mathcal{D}_{prf}$ | 偏好数据集 |
| $r_b$ | 更好的名称（从 top-20 中选出最接近 ground-truth 的） |
| $r_w$ | 较差的名称（从 top-20 中随机采样） |
| $\text{best}(\hat{r}, r)$ | 从预测中选出与 ground-truth 最相似的名称 |
| $\text{sample}(\hat{r})$ | 从预测中随机采样一个名称 |

---

## 5. 相关的微调损失函数

在 SymPO 之前，模型首先通过**因果语言建模 (CLM)** 进行微调：

$$
\mathcal{L}_{ft}(\Theta, \mathcal{D}_{loc}, \mathcal{D}_{ctx}) := -\mathbb{E}_{(q,r) \sim \mathcal{D}_{loc} \cup \mathcal{D}_{ctx}} \left[ \sum_{i=1}^{\text{len}(q)+\text{len}(r)} \log P(x_i | x_{<i}; \Theta) \right]
$$

**公式 (6)**

| 符号 | 含义 |
|------|------|
| $\mathcal{L}_{ft}$ | 微调的损失函数 |
| $x$ | query $q$ 和 response $r$ 拼接后的 token 序列 |
| $x_i$ | 第 $i$ 个 token |
| $x_{<i}$ | 第 $i$ 个 token 之前的所有 token |
| $P(x_i \| x_{<i}; \Theta)$ | 给定前面的 token，模型预测第 $i$ 个 token 的概率 |

这是标准的**自回归语言模型**训练目标：预测下一个 token。

---

## 6. 训练流程总结

```
┌─────────────────────────────────────────────────────────────┐
│                     GENNM 训练流程                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Step 1: 微调 (Fine-tuning)                                 │
│  ┌─────────────┐      CLM Loss       ┌──────────────┐      │
│  │ Pre-trained │  ───────────────▶  │ GenNM-Ctx    │      │
│  │ Model       │     公式 (6)        │ (Θ_ctx)      │      │
│  └─────────────┘                     └──────────────┘      │
│                                             │               │
│  Step 2: 构建偏好数据集                       ▼               │
│  ┌──────────────┐    推理 top-20    ┌──────────────┐      │
│  │ 训练数据      │ ◀──────────────── │ GenNM-Ctx    │      │
│  └──────────────┘     公式 (7)(8)    └──────────────┘      │
│         │                                                   │
│         ▼                                                   │
│  ┌──────────────┐                                          │
│  │ 偏好数据集    │  (query, better, worse)                  │
│  │ D_prf        │                                          │
│  └──────────────┘                                          │
│         │                                                   │
│  Step 3: SymPO 优化                                         │
│         ▼                                                   │
│  ┌──────────────┐    SymPO Loss     ┌──────────────┐      │
│  │ GenNM-Ctx    │  ───────────────▶ │ GenNM-SymPO  │      │
│  │ (冻结)       │     公式 (9)       │ (Θ)          │      │
│  └──────────────┘                    └──────────────┘      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. 数学符号速查表

| 符号 | 读法 | 含义 |
|------|------|------|
| $\Theta$ | Theta | 模型参数 |
| $\nabla$ | Nabla / Del | 梯度算子 |
| $\mathbb{E}$ | Expectation | 期望（平均值） |
| $\sigma$ | Sigma | Sigmoid 函数 |
| $\beta$ | Beta | 超参数 |
| $\sim$ | "服从于" | 从某分布中采样 |
| $\log$ | Log | 自然对数 |
| $P(\cdot\|\cdot)$ | Probability | 条件概率 |
| $\succ$ | "优于" | 偏好关系 |
| $\mathcal{D}$ | 花体 D | 数据集 |
| $\mathcal{L}$ | 花体 L | 损失函数 |

---

## 8. 关键洞察

1. **为什么用模型的预测而不是 ground-truth 作为"好名称"？**
   - 如果用 ground-truth，模型可能根本不知道如何生成它
   - 用模型自己的 top-20 预测中最好的，保证模型"有能力"生成

2. **为什么需要参考模型 $\Theta_{ctx}$？**
   - 防止目标模型偏离原始分布太远
   - 类似于 PPO/DPO 中的 KL 散度约束

3. **SymPO vs 传统分类损失**
   - 传统：只优化 ground-truth 的概率
   - SymPO：明确告诉模型"A 比 B 好"，学习相对偏好

---

## Session Notes

*本笔记创建于用户首次阅读 SymPO 相关内容时，专门为深度学习初学者编写。*
