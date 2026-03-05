# C2SAFERRUST: Transforming C Projects into Safer Rust with NeuroSymbolic Techniques

**Authors:** Vikram Nitin, Rahul Krishna, Luiz Lemos do Valle, Baishakhi Ray  
**Venue:** arXiv, 2025  
**PDF:** C2SAFERRUST.pdf

## 1. Problem Statement

现有的C到Rust转换方法面临两难困境：
- **C2Rust transpiler**: 产生功能正确但非惯用的unsafe Rust代码，大量使用原始指针和unsafe块
- **LLM方法**: 生成惯用代码但容易出错，且受限于上下文窗口长度

研究目标是将完整的C项目转换为更安全、更惯用的Rust代码，同时保持功能正确性。

## 2. Key Methods & Techniques

### 核心架构：神经符号混合方法 (NeuroSymbolic)

**Pipeline流程:**
1. **C2Rust转译** → 将C代码转为unsafe Rust（作为中间桥梁）
2. **程序分解** → 将代码分解为translation units
3. **静态分析** → 提取依赖信息和上下文
4. **LLM翻译** → 将unsafe Rust转为safe Rust
5. **验证与修复** → 编译测试和反馈修复

### 关键技术组件:

#### 1) Translation Orchestrator
- **程序分解**: 遍历AST，将函数分解为≤L行的translation units
  - Root translation unit: 包含函数签名的完整函数
  - Inner translation units: 嵌套在root中的代码块
- **翻译顺序优化**: 构建调用图，按反向拓扑排序（被调用者优先于调用者）

#### 2) Static Analyzer
- **调用图分析**: 确定函数间依赖关系
- **数据流分析**: 计算每个unit的live-in/live-out变量
- **程序切片**: 为LLM提供上下文信息（调用点、全局变量、模块导入）

#### 3) Translation Agent
- 为root unit提供：函数代码 + 调用点 + 全局变量 + 模块导入
- 为inner unit提供：代码 + live-in/live-out变量 + 全局变量 + 模块导入

#### 4) Validation & Repair
- 编译检查 + 端到端测试
- 失败时提供编译器/测试反馈给LLM重新生成
- 最多N次尝试，失败则保留原unsafe版本

## 3. Primary Results & Findings

### 评估数据集:
- **自建数据集**: 7个GNU Coreutils程序 (split, pwd, cat, truncate, uniq, tail, head)
- **Laertes数据集**: 10个程序

### 主要结果:
| 指标 | 平均改进 | 最佳改进 |
|------|----------|----------|
| Raw Pointer Declarations | ↓25% | ↓38% (cat) |
| Raw Pointer Dereferences | ↓22% | ↓27% (uniq) |
| Unsafe Lines of Code | ↓24% | ↓28% (uniq) |
| Unsafe Type Casts | ↓30% | ↓36% (tail) |
| Unsafe Call Expressions | ↓8% | ↓14% (head) |

### 与基线比较:
- 优于Laertes和CROWN在大多数程序和指标上的表现

## 4. Limitations & Open Questions

1. **FFI依赖**: 仍需依赖C API的FFI调用，限制了代码安全性提升的上限
2. **结构体/枚举转换**: 不处理函数外的结构体和枚举定义转换
3. **测试依赖**: 需要端到端测试用例来验证正确性
4. **部分转换**: 无法在N次尝试内完成的unit保持unsafe状态

## 5. Novel Contributions

1. **桥接方法**: 首次使用C2Rust输出作为C和LLM生成代码之间的桥梁
2. **增量翻译**: 将大程序分解为可管理的translation units进行增量翻译
3. **Rust静态分析工具**: 实现了基于Rust编译器的程序切片和调用图分析
4. **Coreutils基准**: 贡献了7个带测试用例的GNU Coreutils程序基准

## 6. Potential Applications & Extensions

1. **扩展到更多语言特性**: 支持结构体、枚举等复杂类型的转换
2. **自动测试生成**: 结合模糊测试自动生成验证用例
3. **混合翻译策略**: 根据代码复杂度选择规则或LLM翻译

---

## Session Notes

### 技术实现细节:
- LLM: GPT-4o-mini
- 最大chunk大小: 150行
- 反馈迭代次数: 5次
- 实现为cargo子命令
- 基于Rust编译器的HIR和MIR进行分析

### 人类笔记:
- 该流程按 unit 独立翻译与修复，若某次修复需要**跨多个 translation unit 协同修改**（如签名与调用处同时变更），现有策略可能无法回溯已翻译的 unit，从而导致失败或退化为保留 unsafe 版本。
