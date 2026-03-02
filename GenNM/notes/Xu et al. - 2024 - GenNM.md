# Unleashing the Power of Generative Model in Recovering Variable Names from Stripped Binary

**Authors:** Xiangzhe Xu, Zhuo Zhang, Zian Su, Ziyang Huang, Shiwei Feng, Yapeng Ye, Nan Jiang, Danning Xie, Siyuan Cheng, Lin Tan, Xiangyu Zhang  
**Venue:** NDSS 2025 (Network and Distributed System Security Symposium)  
**PDF:** Xu et al. - 2024 - Symbol Preference Aware Generative Models for Recovering Variable Names from Stripped Binary.pdf

## 1. Problem Statement

反编译的目标是从二进制可执行文件中恢复源代码形式，这对于恶意软件分析、漏洞检测、代码加固等安全应用至关重要。其中一个突出挑战是**从完全剥离符号的二进制程序中恢复变量名**。

现有方法（如 VarBERT）将此问题建模为**分类任务**，存在三个主要局限：
1. **封闭词汇问题**：分类模型只能预测训练集中出现过的名称，无法"发明"新名称
2. **长尾分布问题**：变量名分布极度不平衡（50%的名称只出现1次，而0.1%的名称出现超过1000次），导致模型偏向高频名称
3. **缺失上下文信息**：现有方法每次只分析单个函数，缺少调用上下文信息

> **Human marks**: 1. 词汇封闭，是指只有固定的词汇表？2.长尾分布固然存在，但是分类方法应该早就有解决办法？ 3. 本方法的上下文信息怎么retrieve的。

**Response:**

1. **词汇封闭问题**：是的，分类方法在训练阶段会收集一个变量名集合作为词汇表（vocabulary）。在推理阶段，模型只能从这个词汇表中选择名称，无法"发明"新名称。论文原文："A classification model can only predict names within its training vocabulary. It cannot 'invent' new names based on program contexts." 实验显示，VarBERT 在未见名称上仅达到 8.5% precision。

2. **长尾分布问题**：确实，机器学习领域有很多处理长尾分布的方法（如过采样、类别权重调整、focal loss 等）。但论文认为这些通用方法**不够适用于符号恢复任务**，原因在于：
   - 变量名的"偏好"是**上下文相关的**（context-dependent）。例如在网络编程场景中，`packet` 比 `array` 更合适，尽管 `array` 在整体数据集中更频繁。
   - 论文引用了 [25] 和 [29] 表明"在偏斜分布上训练好的分类器很有挑战性"。
   - SymPO 的创新在于：不是简单地重新加权，而是通过**对比学习**的方式，让模型学会在特定程序上下文中偏好特定名称。

3. **上下文信息的获取方式**：通过**程序调用图（call graph）传播**：
   - **来自调用者（caller）的上下文**：识别调用当前函数 f 的调用点（callsite），用调用者函数中已预测的变量名重命名调用表达式，然后附加到 f 的查询中。
   - **来自被调用者（callee）的上下文**：重命名被调用函数的签名（函数名 + 参数列表），附加到当前函数的查询中。
   - 关键设计：**只传播函数签名，不传播完整函数体**，避免 token 过多导致性能下降。
   - 例如：`<CallSites> <0> log(0, err_msg) </0> </CallSites> <Callees> gee: fflush(FILE *fp) </Callees>`

## 2. Key Methods & Techniques

### 2.1 核心思想：生成式模型 + 符号偏好优化

GENNM 采用生成式代码语言模型（如 CodeGemma-2B、CodeLlama-7B/34B），通过微调使其能够**组合 token 生成变量名**，而非从固定词汇表中选择。

### 2.2 训练流程（三阶段）

#### Stage 1: Context-aware Fine-tuning（上下文感知微调）
- 使用两种数据集微调预训练模型：
  - **局部数据集 (Dloc)**：单独的反编译函数 + 真实变量名
  - **上下文数据集 (Dctx)**：函数体 + 沿调用图传播的上下文信息
- 上下文传播算法：收集调用者函数的调用点信息和被调用函数的签名信息
- 使用因果语言建模（CLM）损失函数

> **Human mark**: Why using CLM (shouldn't)? cause this is not a generation task, but a understanding task.

**Response:**

这是一个很好的问题。虽然变量名恢复从语义上看是一个"理解"任务，但论文选择 CLM（因果语言建模）有以下原因：

1. **任务建模为生成任务**：论文将问题重新建模为**序列到序列的生成任务**，而非分类任务。输入是反编译代码，输出是变量名映射。CLM 正是训练生成模型的标准方法。

2. **理解与生成的统一**：现代 LLM 的设计理念是通过"预测下一个 token"来学习语言理解。论文原文："Our fine-tuning stage calculates the CLM loss for tokens in **both the query and the response** to help the model understand the distribution of the decompiled code in the query." 即 CLM 损失同时作用于输入（理解反编译代码）和输出（生成变量名）。

3. **利用预训练知识**：CodeGemma/CodeLlama 等预训练模型本身就是用 CLM 训练的。使用相同的损失函数进行微调可以更好地保留预训练知识，同时适应新领域（反编译代码）。

4. **为什么不用 MLM（掩码语言建模）**？MLM 更适合纯理解任务（如 BERT），但无法直接生成新序列。论文需要模型能够**组合 token 生成新名称**（如 `ip_hdrlen` 由 `ip` + `hdr` + `len` 组成），CLM 更适合这种开放式生成。

5. **对比基线**：VarBERT 使用类似 BERT 的架构（理解型），但受限于封闭词汇表。GENNM 使用 CLM 则能够泛化到未见名称。

#### Stage 2: Building Symbol Preference Dataset（构建符号偏好数据集）
- 用训练后的模型 GENNMCtx 对训练集进行推理
- 收集模型犯错但 top-20 预测中有更好选项的案例
- 构建成对数据：(query, better_name, worse_name)

#### Stage 3: Symbol Preference Optimization (SymPO)（符号偏好优化）
- 核心洞察：开发者的符号偏好隐含在真实名称中，可用于缓解训练数据中的偏差
- 改编自 DPO（Direct Preference Optimization）的损失函数：
  - 引导模型对"更好的名称"产生更高概率
  - 防止模型过度偏离原始分布
- 使用静态代码特征的启发式方法过滤低质量数据（减少60%数据量）

> **Human marks**: 我想知道符号偏好优化是不是训练模型过程中解决长尾分布问题的常用方法？

**Response:**

**不是常用方法，这是本文的核心创新之一。** 让我解释一下：

1. **传统长尾分布解决方法**：
   - **重采样**（oversampling/undersampling）
   - **类别权重调整**（class-weighted loss）
   - **Focal Loss**（降低易分类样本的权重）
   - **数据增强**（对少数类进行增强）
   - **两阶段训练**（先学习表示，再平衡分类器）

2. **SymPO 的独特之处**：
   - 借鉴自 NLP 领域的 **DPO（Direct Preference Optimization）**[Rafailov et al., 2023]，原本用于将 LLM 与人类偏好对齐（如 RLHF 的替代方案）。
   - **本文首次将偏好优化思想应用于符号恢复任务**。
   - 关键区别：不是简单地增加罕见名称的权重，而是通过**对比学习**让模型在特定上下文中学会"偏好"正确名称。

3. **为什么更有效**：
   - 传统方法假设所有样本的"正确性"是独立于上下文的。
   - SymPO 认识到：同一个高频名称（如 `buffer`）在某些上下文中是正确的，在另一些上下文中应被 `packet` 替代。
   - 通过构建 (better_name, worse_name) 对，模型学习**上下文相关的偏好**。

4. **消融研究验证**：SymPO 贡献了 5.4% 的改进（见 Fig. 13），虽然不是最大贡献因素，但对罕见名称的提升更显著。

### 2.3 推理流程（迭代式）

1. **初始推理**：基于局部上下文为每个函数生成变量名
2. **上下文传播**：沿调用图传播预测的名称
3. **名称验证**：使用程序数据流分析和多数投票选择最一致的名称
4. **迭代更新**：重复上述过程直到收敛或达到预算

> **Human marks**: 详细讲一下怎么进行迭代式推理。

**Response:**

迭代式推理的设计灵感来自人类逆向工程师的工作方式：先广度优先地浏览所有函数，然后逐步细化。具体流程如下：

**迭代推理流程（对应 Fig. 7）：**

```
Round 0 (初始化):
├── 对每个函数 f，仅基于函数体（局部上下文）生成变量名
├── 输出：每个变量的 top-k 候选名称
└── 存入候选名称映射表

Round 1, 2, ... (迭代细化):
├── Step 1: 上下文传播
│   ├── 沿调用图收集上下文：
│   │   - CallerCtx: 调用者中调用当前函数的表达式（已重命名）
│   │   - CalleeCtx: 被调用函数的签名（已重命名）
│   └── 将上下文附加到查询 prompt 中
│
├── Step 2: 模型推理
│   ├── 输入：函数体 + 上下文信息
│   └── 输出：新的 top-k 候选名称，加入候选池
│
├── Step 3: 名称验证（Name Validation, Algorithm 1）
│   ├── 收集关联名称：通过数据流分析找到直接赋值/传参关系的变量
│   ├── 语义投票：将所有候选名称和关联名称编码为向量
│   └── 选择与关联名称语义最相似的候选名称
│
├── Step 4: 更新名称映射
│   └── 将选中的名称作为当前最佳预测
│
└── 终止条件：名称不再更新 或 达到预设迭代次数
```

**具体例子（Fig. 4 的 motivating example）：**

1. **Round 0**：分析函数 `sub_401430`，仅看函数体
   - `v29` → 候选：`ip_hdr`, `tcp_hdr`, `udp_hdr`（无法确定具体是哪种）
   - `n` → 候选：`duk_len`, `buf_len`, `ip_hdrlen`

2. **Round 1**：加入上下文
   - 从被调用函数 `sub_40197A` 传播上下文：`sub_40197A(ip_hdr, ip_hdrlen, ...)`
   - 模型看到被调用函数的参数名是 `ip_hdr` 和 `ip_hdrlen`
   - 结合数据流（`v29` 传给第一个参数，`n` 传给第二个参数）
   - `v29` → 高置信度选择 `ip_hdr`
   - `n` → 高置信度选择 `ip_hdrlen`

3. **名称验证**：
   - `s` 与 `v29` 有数据流关系（`v29 = s`）
   - `v29` 选择了 `ip_hdr`
   - `s` 的候选中 `packet` 与 `ip_hdr` 语义最相似 → 选择 `packet`

**关键设计决策**：
- 只传播**函数签名**而非完整函数体，控制 token 数量
- 使用**多数投票 + 语义相似度**而非简单取 top-1
- 迭代允许信息在调用图中**多跳传播**

### 2.4 技术细节

- **上下文传播**：不是直接附加调用者/被调用者的完整函数体（会导致 token 过多且性能下降），而是使用重命名后的函数签名作为上下文摘要
- **名称验证**：沿程序数据流传播名称，计算候选名称与相关变量名称的语义相似度

## 3. Primary Results & Findings

### 3.1 主要性能提升

| 数据集 | 相比 VarBERT | 相比 ReSym |
|--------|-------------|-----------|
| DIRTY | Precision +8.5pp, Recall +8.5pp | Precision +5.6pp, Recall +4.6pp |
| VarCorpus | Precision +11.4pp, Recall +11.0pp | Precision +9.5pp, Recall +6.2pp |

### 3.2 关键发现

1. **未见名称的泛化能力**：
   - VarBERT 在未见名称上仅 8.5% precision
   - GENNM 达到 22.8%，提升 168%
   
2. **罕见名称的鲁棒性**：
   - 95% 的罕见名称由常见 token 组成
   - GENNM 在罕见变量上达到 27.1%，VarBERT 仅 13.5%

3. **对比黑盒 LLM**：
   - GENNM (42.7% precision) 优于 GPT-4 3-shot (31.4%) 11.3 个百分点
   - 证明了在此任务上微调的必要性

4. **编译优化级别泛化**：
   - GENNM 在所有优化级别（O0-O3）上都优于基线
   - 在低优化级别（O0, O1）上改进更显著

### 3.3 消融研究

各组件对改进的贡献：
- Context-aware Fine-tuning: 62.2%
- Context Propagation: 18.9%
- Name Validation: 13.5%
- SymPO: 5.4%

## 4. Limitations & Open Questions

1. **高优化级别程序**：在 O2/O3 编译的程序上改进幅度较小，因为这些程序更长且偏离源代码分布更远

2. **语义准确性 vs 上下文相关性**：GPT4Evaluator 显示模型在上下文相关性上表现优于语义准确性，表明准确理解程序语义仍是挑战

3. **计算资源**：34B 模型需要使用 LoRA 微调，全量微调资源需求高

4. **反编译器依赖**：依赖现有反编译技术（如 IDA Pro）的质量

## 5. Novel Contributions

1. **首次将符号偏好优化应用于变量名恢复**：设计 SymPO 方法对齐模型输出与开发者偏好

2. **上下文感知微调范式**：教导模型如何利用调用上下文信息

3. **迭代推理与上下文传播**：模拟人类逆向工程师的分析方式

4. **生成式方法解决封闭词汇问题**：相比分类方法可组合 token 生成新名称

## 6. Potential Applications & Extensions

1. **恶意软件分析增强**：如论文案例所示，帮助安全分析师理解 C&C 通信和命令分发逻辑

2. **二进制代码摘要**：结合 ChatGPT 等 LLM，先用 GENNM 恢复变量名再生成代码摘要

3. **扩展到其他符号恢复任务**：
   - 函数名恢复
   - 类型恢复
   - 数据结构恢复

4. **跨架构/跨编译器泛化研究**：评估模型在不同 CPU 架构和编译器上的泛化能力

---

## Session Notes

### Session 1 (Initial Reading)
- 完成论文初次阅读和结构化笔记
- 关键技术点：生成式模型 + SymPO + 上下文感知训练/推理
- 核心创新：将 DPO 思想应用到符号恢复任务
