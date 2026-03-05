# Syzygy: Dual Code-Test C to (safe) Rust Translation using LLMs and Dynamic Analysis

**Authors:** Manish Shetty, Naman Jain, Adwait Godbole, Sanjit A. Seshia, Koushik Sen  
**Venue:** arXiv, 2024 (UC Berkeley)  
**PDF:** c2rust-test.pdf

## 1. Problem Statement

C到safe Rust的翻译面临以下挑战：
- **规则方法**: 产生不可解释的目标代码，难以扩展到多函数代码库
- **LLM方法**: 无法精确推断语义执行信息（如别名、分配大小）
- **测试挑战**: 中间函数的可靠测试困难（缺少I/O样例）

目标：实现中大型C代码库到等价safe Rust程序的自动翻译。

## 2. Key Methods & Techniques

### 核心理念：Syzygy（双重协同）

**两层双重性:**
1. **LLM + 动态分析**: LLM的生成能力 + 动态分析的语义执行信息
2. **代码 + 测试生成**: 不仅翻译代码，还翻译测试

### Pipeline架构:

```
C Codebase → Slicer → CodeGenerator → ArgTranslator → EqTester → Rust Program
                ↑                           ↑
            SpecMiner ←──── Dynamic Analysis (LLVM)
```

### 关键技术组件:

#### 1) SpecMiner (动态规范挖掘)
通过LLVM插桩收集函数参数的属性：

| 属性 | 说明 | 用途 |
|------|------|------|
| Types | 追踪指针类型（包括void*的bitcast） | 指导类型转换 |
| Allocation Sizes | 使用区间树记录分配信息 | 序列化I/O样例 |
| Nullability | 参数是否可能为NULL | 决定Option类型使用 |
| Aliasing | 参数间的别名关系 | 决定智能指针类型 |

#### 2) CodeGenerator
- 按依赖顺序增量翻译（叶子节点优先）
- **手动结构体定义**: 结构体选择会影响上游函数签名
- 使用动态分析信息指导LLM生成签名
- 采样多个候选翻译，通过编译过滤

#### 3) ArgTranslator
- 将C测试I/O转换为Rust I/O
- 提供API工具函数处理：
  - 原始类型/数组 → Vec
  - 字符串 → String
  - 结构体 → Rc<RefCell<Struct>>
  - 保持别名关系的转换

#### 4) EqTester
- 生成等价性测试：验证C和Rust函数在相同输入上产生等价输出
- 基于值的等价性检查
- 多轮采样 + 错误反馈修复

### 采样与过滤流程:
```
CodeGen(N) → Compile Filter(N') → TranslateArgs(M) → Execute Filter(M') → EqTest(S) → Final(S')
```

## 3. Primary Results & Findings

### 案例研究:

**1) UrlParser (~400 LoC)**
- 先前方法(Flourine, VERT)在程序分解上遇到困难
- Syzygy通过增量翻译+测试过滤成功翻译

**2) Zopfli (~3000 LoC, 98函数, 10结构体)**
- 翻译成本: ~$2500 (使用O1模型)
- 生成~7000行Rust代码
- 测试：100万个压缩输入，95%行覆盖率，83%分支覆盖率
- Pass率: 编译>75%, 执行>75% (平均)

### 性能对比:
| 配置 | C | Rust | 减速比 |
|------|---|------|--------|
| Default编译 | 0.17s | 1.56s | 8.9x |
| 优化编译 | 0.04s | 0.059s | 1.47x |

## 4. Limitations & Open Questions

### 输入C程序限制:
1. **无环数据结构**: 避免Rust中的内存泄漏
2. **无多线程**: 需要Arc包装
3. **无类型双关**: 影响类型分析

### 其他限制:
1. **手动结构体翻译**: 结构体字段选择需人工指定
2. **高成本**: O1模型费用昂贵（~$2500/项目）
3. **性能差距**: Rust翻译比C慢（未优化时8-13x，优化后1.5-3.7x）
4. **依赖顺序翻译**: 无法回溯修正早期决策

## 5. Novel Contributions

1. **双重翻译方法**: 同时翻译代码和测试，实现可靠的等价性验证
2. **动态分析辅助LLM**: 首次将动态分析信息（别名、nullability、分配大小）融入LLM翻译
3. **I/O规范挖掘**: 通过执行顶层测试收集中间函数的I/O样例
4. **最大规模验证翻译**: Zopfli是目前最大的测试验证C到safe Rust翻译

## 6. Potential Applications & Extensions

1. **降低成本**: 混合使用便宜模型(GPT-4o)和强力模型(O1)
2. **性能优化**: 分析并优化Rust翻译的运行时性能
3. **扩展类型支持**: 支持循环数据结构和多线程代码
4. **自动化结构体翻译**: 减少人工干预

---

## Session Notes

### 关键洞察:
- C指针隐藏了对象结构和使用模式信息，这是翻译的核心挑战
- 动态分析比静态分析更适合收集C程序的语义信息
- 中间函数测试是保证翻译质量的关键
- LLM倾向于"优化"代码逻辑，常导致错误

### 实现细节:
- 使用LLVM-14实现动态分析插桩
- 使用Clang-17实现程序切片
- 使用bindgen生成FFI绑定
- 使用O1-preview和O1-mini作为主要LLM
