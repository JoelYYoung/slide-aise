# EvoC2Rust: A Skeleton-guided Framework for Project-Level C-to-Rust Translation

**Authors:** Chaofan Wang, Tingrui Yu, Beijun Shen, et al.  
**Venue:** ICSE-SEIP 2026 (上海交通大学 & 华为)  
**PDF:** evoc2rust.pdf

## 1. Problem Statement

### 核心挑战:

**Challenge-1: 语言差异导致的安全问题**
- C允许松散类型检查、任意指针算术、手动内存管理
- Rust强制编译时类型安全、所有权保证、借用规则
- 现有方法无法可靠满足Rust的安全要求

**Challenge-2: 项目级代码依赖**
- 翻译完整项目需保持跨模块依赖和项目结构一致性
- LLM难以处理大规模上下文，导致引用断裂、API不一致

目标：将完整C项目自动转换为等价的安全Rust代码。

## 2. Key Methods & Techniques

### 核心策略：骨架引导翻译 (Skeleton-guided Translation)

**三阶段Pipeline:**

```
C Project → Skeleton Construction → Module Translation → Post-Generation Repair → Rust Project
```

### Stage 1: Project Skeleton Construction

1. **项目分解**: 使用Tree-sitter解析C代码，提取元数据
2. **元数据转换**: 将C的include转为Rust的use imports
3. **骨架生成**: 
   - 自动转换定义、宏、函数签名
   - 为每个函数生成带`unimplemented!()`的占位函数
   - 确保项目可编译（即使函数体未翻译）

### Stage 2: Module Translation with Feature Mapping

**7类安全保持映射:**

| 类别 | C构造 | Rust等价物 |
|------|-------|-----------|
| Type | `int*` | `Ptr<i32>` |
| Array | `int[N]` | `Array<i32, N>` |
| String | `char*` | `cstr!()` macro |
| Function Pointer | `(*func)` | `FuncPtr<fn(...)->T>` |
| File | `FILE*` | `FilePtr` |
| Variadic | `va_list` | `VaList` |

**其他映射:**
- **Type Conversion**: `CastIntoTyped` trait处理类型转换
- **Macro/Function**: 为malloc, free, memcpy等提供安全抽象
- **Syntax Structure**: `c_for`, `c_do`, `c_switch`宏模拟C控制流
- **Operator**: 处理++, --, &, sizeof
- **Global Variable**: `Global<T>`包装器 + Mutex保证线程安全

### Stage 3: Post-Generation Repair

**三步级联修复:**

1. **Bracket Repair**: LLM修复括号不匹配等语法错误
2. **Rule-based Repair**: 正则表达式处理常见语法问题
3. **LLM Refinement**: 修复残留的语义错误

## 3. Primary Results & Findings

### 评估数据集:
- **Vivo-Bench**: 19个开源算法项目 (80-917 LoC)
- **C2R-Bench**: 6个华为工业项目 (280-3724 LoC)

### 项目级结果 (DeepSeek-V3):

| 数据集 | ICompRate | AccRate Precision | AccRate Recall | SafeRate |
|--------|-----------|-------------------|----------------|----------|
| Vivo-Bench | 100% | 99.83% | 99.86% | 98% |
| C2R-Bench | 93.84% | 97.56% | 97.34% | 97.41% |

### 与基线对比:
- 比最强LLM基线提升 **17.24%** 语法准确率
- 比最强LLM基线提升 **14.32%** 语义准确率  
- 比最佳规则工具提升 **43.59%** 代码安全率

### 模块级结果:
- **FCompRate**: 92.25-100%
- **TestRate**: 89.53-99.07%

### 可扩展性 (RepoTransBench 大型项目):
- 10个项目 (9,973 - 91,588 LoC)
- ICompRate: 75.61%
- SafeRate: 96.20%

## 4. Limitations & Open Questions

1. **第三方库支持**: 当前仅支持ISO C标准库依赖
2. **复杂控制流**: 复杂项目（如cmptlz）测试通过率较低
3. **人工模板**: Feature mapping需要专家手工定义
4. **模型依赖**: 性能与LLM能力强相关

## 5. Novel Contributions

1. **骨架引导策略**: 首创通过可编译骨架解决跨模块依赖问题
2. **安全保持映射**: 系统定义7类C-Rust语言特性映射
3. **级联修复链**: 结合规则和LLM的混合修复策略
4. **工业验证**: 在华为真实项目上验证有效性

## 6. Potential Applications & Extensions

1. **扩展第三方库支持**: 自动生成外部库的Rust绑定
2. **自适应Feature Mapping**: 自动学习新的映射规则
3. **并行翻译**: 利用骨架的独立性并行翻译模块
4. **增量更新**: 支持C代码修改后的增量翻译

---

## Session Notes

### 技术亮点:
- 骨架保证任何时刻项目都可编译，避免级联依赖错误
- Feature mapping是领域专家知识的结晶，提升翻译质量
- 括号修复优先于其他修复，因为基础语法错误会干扰后续分析

### 实现细节:
- 基础模型: DeepSeek-V3, Qwen3-32B
- 解析工具: Tree-sitter v0.22.3
- 嵌入模型: BGE-M3
- 迭代限制: Bracket repair 5轮, LLM refinement 3轮

### 对比特点:
- 与C2SaferRust不同：从头翻译而非后处理C2Rust输出
- 与Syzygy不同：不依赖动态分析，使用Feature Mapping
