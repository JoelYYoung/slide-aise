# C to Safe Rust Translation: Research Summary

> 本文档综合比较九篇C到Safe Rust翻译的前沿研究论文，分析其共性、差异及发展趋势。

---

## 开源代码/模型

| 论文 | 开源状态 | 链接 | 说明 |
|------|----------|------|------|
| **C2SAFERRUST** | ✅ 已开源 | https://github.com/vikramnitin9/c2saferrust | 含静态分析工具链和Rust编译器Pass |
| **Syzygy** | ⚠️ 仅项目主页 | https://syzygy-project.github.io/ | 无代码仓库，仅论文和演示 |
| **EvoC2Rust** | ✅ 已开源 | https://github.com/bbzswcf/EvoC2rust | 含Feature Mapping和骨架引导框架 |
| **Rustine** | ⚠️ 仓库为空 | https://github.com/Intelligent-CAT-Lab/Rustine | **注意：仓库内容为空，无实际代码** |
| **RustMap** | ✅ 已开源 | https://github.com/Cxm211/RustMap | 含依赖引导翻译框架 |
| **Laertes** | ✅ 已开源 | https://zenodo.org/records/5442253 | 含指针分析和转换工具 |
| **CRustS** | ✅ 已开源 | https://github.com/yijunyu/crusts | 含220条TXL变换规则 |
| **CRUST-Bench** | ✅ 已开源 | https://github.com/AnirudhKhatry/CRUST-bench | 100个C仓库 + 手写Rust接口 + 测试 |

### 开源分析总结
- **5/9论文有可用开源代码**：C2SAFERRUST、EvoC2Rust、RustMap、Laertes、CRustS
- **1个标准基准**：CRUST-Bench提供首个标准化C-to-Safe-Rust评估基准
- **Syzygy缺乏代码**：仅有项目主页，实际代码未公开
- **Rustine仓库为空**：虽声称开源但实际仓库无内容，可复现性存疑

---

## Baseline分析：已发表的C to Safe Rust工作

### 主要Baseline方法

| Baseline | 类型 | 首发 | 方法概述 |
|----------|------|------|----------|
| **C2Rust** | 规则 | 2018 | Immunant的transpiler，基于Clang AST，生成unsafe Rust |
| **Corrode** | 规则 | 2017 | 语义保持的C到Rust翻译 (https://github.com/jameysharp/corrode) |
| **Laertes** | 类型分析 | 2021 | 首个学术化unsafe消除工作 (Emre et al., OOPSLA'21) |
| **Aliasing Limits** | 分析 | 2023 | Laertes后续，分析别名对转换的限制 (Emre et al., OOPSLA'23) |
| **CROWN** | 规则+LLM | 2023 | 在C2Rust基础上使用LLM优化 |
| **CRustS** | 规则 | 2023 | 220条TXL源码变换规则 (华为) |
| **VERT** | LLM | 2024 | LLM采样翻译 |
| **RustMap** | LLM+分析 | 2025 | 依赖引导的项目级C到Rust迁移 |
| **FLOURINE** | LLM | 2024 | 使用fuzz testing验证语义等价 |
| **Tymcrat** | LLM | 2025 | 类型推断辅助签名翻译 |

### 标准评估基准

| 基准 | 来源 | 规模 | 特点 |
|------|------|------|------|
| **CRUST-Bench** | UT Austin 2025 | 100个C仓库 | 手写Rust接口+测试，功能正确性验证 |
| **Laertes Corpus** | UCSB 2021 | 17个C程序 | bzip2, gzip, json-c等，unsafe分析 |
| **Rosetta Code** | 社区 | 126+程序 | 多语言算法，常用于函数级翻译 |

### 各论文使用的Baseline

| 论文 | 使用的Baseline | 对比维度 |
|------|----------------|----------|
| **C2SAFERRUST** | C2Rust, Laertes, GPT-4 Direct | unsafe代码量、raw pointer数量 |
| **Syzygy** | C2Rust, VERT, CROWN, Shiraishi | 编译成功率、safe Rust比例、验证方式 |
| **EvoC2Rust** | C2Rust, C2SaferRust, Self-Repair, Tymcrat, Direct Prompting | 编译率、行接受率、安全率、测试通过率 |
| **Rustine** | C2Rust, C2SaferRust, CROWN, Laertes, RustMap, Syzygy | 编译率、功能等价性、安全性指标、可读性 |

### 关键发现

#### 1. 规则方法的局限性
- **C2Rust** (baseline of baselines): 几乎100%编译成功，但**0%安全率**（全是unsafe）
- **Laertes**: OOPSLA 2021，专注指针安全，有学术数据集可用
- **CROWN**: 结合规则和LLM，但安全率仍然有限（~48-60%）

#### 2. 已发表的C to Safe Rust学术工作

| 工作 | 发表 | 核心贡献 | 局限 |
|------|------|----------|------|
| **Laertes** | OOPSLA 2021 | 首次系统分析unsafe来源，定义7类unsafe | 仅11%指针可转换 |
| **Aliasing Limits** | OOPSLA 2023 | 形式化类型等价性障碍，伪安全技术 | 改进有限(12%→21%) |
| **CRUST-Bench** | arXiv 2025 | 首个标准化基准，100仓库+测试 | 规模仍有限 |
| **FLOURINE** | arXiv 2024 | Fuzz testing验证语义等价 | 未处理项目级翻译 |

#### 3. 时间线演进

```
2017: Corrode (规则)
2018: C2Rust (规则，成为事实标准)
2021: Laertes (OOPSLA，首个学术化unsafe分析，7类unsafe分类)
2023: CROWN, Aliasing Limits (OOPSLA), CRustS (TXL规则)
2024: FLOURINE, VERT, Syzygy (LLM时代开始)
2025: C2SaferRust, EvoC2Rust, Rustine, RustMap, CRUST-Bench, Tymcrat (项目级、基准化)
```

#### 4. CRUST-Bench揭示的现实

**最强LLM (OpenAI o1) 也仅15%测试通过率**，说明C-to-Safe-Rust远未解决：
| 模型 | 编译通过 | 测试通过 |
|------|----------|----------|
| OpenAI o1 | 47% | 15% |
| Claude 3.5 Sonnet | 43% | 12% |
| GPT-4o | 39% | 10% |

#### 5. Baseline质量问题
- 某些工具的artifacts有质量问题
- 指标计算方式不一致
- 部分声称开源的仓库实际为空

---

## 论文概览

### 近期工作（2024-2025）

| 论文 | 机构 | 核心方法 | 最大测试规模 | 成本 |
|------|------|----------|-------------|------|
| **C2SAFERRUST** | Columbia + IBM | C2Rust桥接 + LLM增量翻译 | ~14K LoC | 未报告 |
| **Syzygy** | UC Berkeley | 动态分析 + 双重翻译(代码+测试) | ~3K LoC (Zopfli) | ~$2500 |
| **EvoC2Rust** | 上交 + 华为 | 骨架引导 + Feature Mapping | ~91K LoC | 未报告 |
| **Rustine** | UIUC + 上交 | 预处理重构 + 两级LLM | ~13K LoC | $0.48 (存疑) |
| **RustMap** | SMU + Open Univ. | 依赖引导 + GPT-4o翻译 | ~7K LoC (bzip2) | 高(商业API) |

### 基础工作（2021-2023）

| 论文 | 机构 | 核心方法 | 贡献 |
|------|------|----------|------|
| **Laertes** | UCSB | 原始指针→引用转换 | 首个学术工作，定义7类unsafe |
| **Aliasing Limits** | UCSB | 伪安全+别名分析 | 形式化转换障碍 |
| **CRustS** | 华为 | TXL源码变换 | 220条可审计规则 |

### 评估基准

| 基准 | 机构 | 规模 | 用途 |
|------|------|------|------|
| **CRUST-Bench** | UT Austin | 100仓库 | 功能正确性评估，附带Rust接口+测试 |

---

## 共性分析

### 1. 增量翻译策略
所有方法都采用增量翻译而非一次性翻译整个项目：

```
共识：大项目 → 分解为翻译单元 → 按依赖顺序翻译 → 逐步集成
```

- **C2SAFERRUST**: 基于AST的translation units
- **Syzygy**: 基于依赖图的translation units
- **EvoC2Rust**: 模块级分解 + 骨架占位
- **Rustine**: 依赖图 + 大函数分解

### 2. 编译器反馈循环
所有方法都使用编译错误作为LLM的反馈信号：

```
LLM生成 → 编译 → 失败 → 错误反馈 → 重新生成 → 重复N次
```

### 3. 静态分析辅助
都依赖某种形式的静态分析：

| 分析类型 | 使用者 |
|----------|--------|
| 调用图 | 全部 |
| 依赖关系 | 全部 |
| 数据流 | C2SAFERRUST, Rustine |
| 类型分析 | Syzygy (动态), Rustine |

### 4. 测试驱动验证
都使用测试来验证翻译正确性：

| 方法 | 验证方式 |
|------|----------|
| C2SAFERRUST | 端到端系统测试 |
| Syzygy | 等价性测试(C-Rust I/O比较) |
| EvoC2Rust | 填空编译 + 单元测试 |
| Rustine | 断言级功能等价性 |

---

## 差异分析

### 1. 起点选择

| 方法 | 起点 | 优势 | 劣势 |
|------|------|------|------|
| **C2SAFERRUST** | C2Rust输出 | 保证语义正确 | 继承非惯用代码 |
| **Syzygy** | C源码 | 可生成惯用代码 | 需动态分析支持 |
| **EvoC2Rust** | C源码 | 骨架保证可编译 | 依赖Feature Mapping |
| **Rustine** | 重构后C源码 | 降低翻译难度 | 重构可能失败 |

### 2. 处理C-Rust语义差异的策略

#### 指针问题

| 方法 | 策略 |
|------|------|
| **C2SAFERRUST** | 依赖LLM自动处理 |
| **Syzygy** | 动态分析推断类型/别名/nullability |
| **EvoC2Rust** | Feature Mapping定义Ptr<T>等抽象 |
| **Rustine** | **预处理重构**将指针算术转为数组索引 |

#### 全局变量

| 方法 | 策略 |
|------|------|
| **C2SAFERRUST** | 作为上下文提供给LLM |
| **Syzygy** | 手动处理 |
| **EvoC2Rust** | Global<T>包装器 + Mutex |
| **Rustine** | 集中到globals.rs模块 |

### 3. LLM使用策略

| 方法 | 模型 | 策略 |
|------|------|------|
| **C2SAFERRUST** | GPT-4o-mini | 单模型 + 5次反馈 |
| **Syzygy** | O1-preview/mini | 采样+过滤 |
| **EvoC2Rust** | DeepSeek-V3/Qwen3 | Feature Mapping增强提示 |
| **Rustine** | DeepSeek-V3 + R1 | 两级策略(便宜→推理) |

### 4. 动态分析使用

| 方法 | 使用程度 | 收集信息 |
|------|----------|----------|
| **C2SAFERRUST** | ❌ 不使用 | - |
| **Syzygy** | ⭐⭐⭐ 核心 | 类型、分配大小、nullability、别名、I/O |
| **EvoC2Rust** | ❌ 不使用 | - |
| **Rustine** | ❌ 不使用 | - |

### 5. 成本效率

| 方法 | 成本 | 原因 |
|------|------|------|
| **C2SAFERRUST** | 中等 | GPT-4o-mini相对便宜 |
| **Syzygy** | **极高** (~$2500) | 依赖O1模型，大量采样 |
| **EvoC2Rust** | 未报告 | DeepSeek相对便宜 |
| **Rustine** | **极低** ($0.48) | 两级策略 + 预处理减少LLM负担 |

---

## 技术创新点对比

| 创新点 | C2SAFERRUST | Syzygy | EvoC2Rust | Rustine |
|--------|-------------|--------|-----------|---------|
| 桥接C2Rust | ✅ | ❌ | ❌ | ❌ |
| 动态分析 | ❌ | ✅ | ❌ | ❌ |
| 双重翻译(代码+测试) | ❌ | ✅ | ❌ | ❌ |
| 骨架引导 | ❌ | ❌ | ✅ | ✅ |
| Feature Mapping | ❌ | ❌ | ✅ | ❌ |
| 源码重构 | ❌ | ❌ | ❌ | ✅ |
| 指针算术处理 | ❌ | ❌ | ❌ | ✅ |
| 两级LLM策略 | ❌ | ❌ | ❌ | ✅ |
| 自适应ICL | ❌ | ❌ | ❌ | ✅ |

---

## 评估指标对比

### 安全性指标

| 指标 | C2SAFERRUST | Syzygy | EvoC2Rust | Rustine |
|------|-------------|--------|-----------|---------|
| Raw Pointer声明 | ✅ | - | - | ✅ |
| Raw Pointer解引用 | ✅ | - | - | ✅ |
| Unsafe代码行 | ✅ | - | - | ✅ |
| Unsafe类型转换 | ✅ | - | - | ✅ |
| 指针算术 | ❌ | ❌ | ❌ | ✅ |
| 代码安全率 | ❌ | ❌ | ✅ | - |

### 质量指标

| 指标 | C2SAFERRUST | Syzygy | EvoC2Rust | Rustine |
|------|-------------|--------|-----------|---------|
| 编译通过率 | ✅ | ✅ | ✅ | ✅ |
| 测试通过率 | ✅ | ✅ | ✅ | ✅ |
| 断言通过率 | ❌ | ❌ | ❌ | ✅ |
| 行接受率 | ❌ | ❌ | ✅ | ❌ |
| Clippy警告 | ❌ | ❌ | ❌ | ✅ |
| 可读性指标 | ❌ | ❌ | ❌ | ✅ |

---

## 限制与挑战

### 共同限制

1. **测试依赖**: 所有方法都需要测试用例来验证正确性
2. **部分unsafe保留**: 无法完全消除所有unsafe代码
3. **性能差距**: Rust翻译通常比C原版慢
4. **人工干预**: 复杂情况仍需人工介入

### 特定限制

| 方法 | 特定限制 |
|------|----------|
| **C2SAFERRUST** | 不处理结构体/枚举；依赖FFI |
| **Syzygy** | 需手动翻译结构体；高成本 |
| **EvoC2Rust** | 仅支持ISO C标准库 |
| **Rustine** | 某些宏展开可能导致问题 |

---

## 发展趋势

### 技术演进路线

```
规则方法(C2Rust) → 规则+LLM混合(C2SAFERRUST) → 纯LLM+分析(Syzygy/EvoC2Rust) → 预处理+高效LLM(Rustine)
```

### 关键趋势

1. **从后处理到预处理**: 
   - 早期：翻译后优化C2Rust输出
   - 现在：预处理C代码降低翻译难度

2. **从动态到静态分析**:
   - Syzygy：依赖动态分析收集运行时信息
   - 后续工作：更多使用静态分析和LLM能力

3. **成本敏感**:
   - 早期不关注成本
   - Rustine：显式优化成本（$0.48 vs $2500）

4. **评估全面化**:
   - 从编译通过率到断言通过率
   - 引入惯用性、可读性等质量指标

---

## 研究空白与未来方向

### 1. 未解决问题

- [ ] 循环数据结构的翻译
- [ ] 多线程代码的翻译
- [ ] 类型双关(type punning)的处理
- [ ] 完全自动化（无人工干预）
- [ ] 性能等价（不仅功能等价）

### 2. 潜在研究方向

1. **混合验证**: 结合形式化验证和测试
2. **增量翻译**: 支持代码库演进后的增量更新
3. **性能优化**: 生成高性能Rust代码
4. **多语言支持**: 扩展到C++, Go等语言
5. **自学习**: 从翻译错误中学习改进策略

---

## 推荐阅读顺序

**入门**: C2SAFERRUST (概念清晰，方法直观)
**深入**: Syzygy (动态分析思想) → Rustine (工程优化)
**应用**: EvoC2Rust (工业实践)

---

## 参考文献

### 近期工作
1. Nitin et al. "C2SAFERRUST: Transforming C Projects into Safer Rust with NeuroSymbolic Techniques" arXiv 2025
2. Shetty et al. "Syzygy: Dual Code-Test C to (safe) Rust Translation using LLMs and Dynamic Analysis" arXiv 2024
3. Wang et al. "EvoC2Rust: A Skeleton-guided Framework for Project-Level C-to-Rust Translation" ICSE-SEIP 2026
4. Dehghan et al. "Rustine: Translating Large-Scale C Repositories to Idiomatic Rust" arXiv 2025
5. Cai et al. "RustMap: Towards Project-Scale C-to-Rust Migration via LLM and Dependency-Guided Transpilation" arXiv 2025

### 基础工作
6. Emre et al. "Translating C to Safer Rust" OOPSLA 2021 (Laertes)
7. Emre et al. "Aliasing Limits on Translating C to Safe Rust" OOPSLA 2023
8. Yu et al. "CRustS: C to Rust via Source Transformation" 2023

### 评估基准
9. Khatry et al. "CRUST-Bench: A Comprehensive Benchmark for C-to-safe-Rust Transpilation" arXiv 2025
