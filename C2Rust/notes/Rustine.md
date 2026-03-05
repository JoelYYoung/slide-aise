# Rustine: Translating Large-Scale C Repositories to Idiomatic Rust

**Authors:** Saman Dehghan, Tianran Sun, Tianxiang Wu, Zihan Li, Reyhaneh Jabbarvand  
**Venue:** arXiv, 2025 (UIUC & 上海交通大学)  
**PDF:** rustine.pdf

## 1. Problem Statement

### 现有方法的局限:

| 问题 | 描述 |
|------|------|
| **非惯用输出** | C2Rust等工具产生大量unsafe块，代码不"纯粹" |
| **可扩展性-质量权衡** | LLM方法依赖frontier模型，成本高昂（如Syzygy翻译zopfli花费~$800） |
| **验证不足** | 多数工作仅报告编译成功，未充分测试功能等价性 |
| **指标不全** | 缺少对指针算术、惯用性、可读性的评估 |
| **调试支持差** | 缺乏可操作的诊断信息和工具支持 |

目标：低成本、全自动地将大型C仓库翻译为可编译、功能等价、惯用的Rust。

## 2. Key Methods & Techniques

### 五阶段架构:

```
Preprocessing → Refactoring → Analysis → Translation → Validation
```

### Stage 1: Preprocessing
- 使用C预处理器展开头文件、宏定义、条件编译
- 标准化代码结构，让LLM专注于C语法翻译

### Stage 2: Automated Refactoring (核心创新)

#### 2.1 一元运算符重构
- 检测所有前缀/后缀的++/--运算符
- 独立表达式：展开为复合赋值 (`*var++` → `var+=1`)
- 复合表达式：分离为两条语句，保持求值顺序
- 多个一元操作在同一语句：跳过重构避免语义改变

#### 2.2 指针操作重构 (Algorithm 1)
```
原始: while(scan < safe_end && *++scan == *++match)
重构: while(&scan[scan_idx] < safe_end && scan[++scan_idx] == match[++match_idx])
```
- 为每个指针创建整数索引变量
- 指针内容访问 → 数组索引操作
- 指针算术 → 索引变换 + &取地址
- 更新函数参数的指针以保持别名语义

#### 2.3 Constness最大化
- 数据流分析识别未修改的函数参数
- 添加const限定符指导LLM生成immutable引用

#### 2.4 大函数分解 (Algorithm 2)
- 超过LLM上下文窗口50%的函数进行分解
- 识别控制流结构中的基本块
- 提取为helper函数，参数按需按引用传递

### Stage 3: Dependence Graph Construction

**定义:** DG = (V := F ⊎ S ⊎ U ⊎ G ⊎ T, E)
- F: 函数, S: 结构体, U: 联合体, G: 全局变量, T: 类型定义/枚举
- 边表示use-def或caller-callee关系
- 环形依赖通过Tarjan算法检测并压缩为super unit

**骨架创建:**
- src/目录对应C模块
- 占位函数带todo!()和C签名注释
- 全局变量统一放在globals.rs

**API翻译增强:**
- 核心库：从Linux文档检索API描述，帮助找到Rust等价物
- 外部库：使用bindgen自动生成FFI

### Stage 4: Translation (Algorithm 3)

**两级模型策略:**
1. **基础模型 (DeepSeek-V3)**: 预算A_base次重新提示
2. **推理模型 (DeepSeek-R1)**: 编译失败后切换，预算A_reason次

**自适应ICL生成:**
- ICL池按Rust错误码(E0308等)索引
- 存在相似样例：按嵌入相似度检索
- 不存在：使用o4-mini生成相似错误代码+修复

### Stage 5: Validation & Debugging

- 执行翻译后的测试用例
- 失败时：禁用失败断言，逐个启用定位问题
- 使用栈追踪的top-n函数作为候选罪犯
- 提示LLM修复语义不匹配

## 3. Primary Results & Findings

### 评估规模:
- **23个C程序**: 27-13,200 LoC
- **测试套件**: 1,221,192个断言, 平均74.7%函数覆盖率, 72.2%行覆盖率

### 主要结果:

| 指标 | Rustine结果 |
|------|-------------|
| 编译成功率 | **100%** (所有23个项目) |
| 功能等价率 | **87%** (1,063,099/1,221,192断言通过) |
| 自动完成项目 | 10/23 |
| 人工修复时间 | 平均4.5小时 (其余13个项目) |
| 平均翻译时间 | 3.92小时 |
| 平均成本 | **$0.48** |

### 与6种方法对比:
- **更安全**: 更少raw pointers、pointer arithmetic、unsafe构造
- **更惯用**: 更少Clippy警告
- **更可读**: 更好的Cognitive Complexity、Halstead、SEI指数

### 指针算术重构效果:
- 重构前: 452处指针算术/比较
- 重构后: 7处 (↓98.5%)

## 4. Limitations & Open Questions

1. **不完美的调试**: 启发式方法和候选子集可能遗漏真正的罪犯
2. **语义等价定义**: 某些unexploitable的unsafe行为是否应该允许
3. **宏处理**: 预处理器展开可能丢失宏的可读性
4. **性能差距**: Rust翻译通常比C原版慢

## 5. Novel Contributions

1. **从头翻译**: 不依赖C2Rust，生成更惯用的代码
2. **指针操作重构**: 首次系统性处理指针算术的翻译
3. **两级LLM策略**: 平衡成本和质量，使用便宜模型+推理模型组合
4. **自适应ICL**: 根据错误类型动态生成/检索学习样例
5. **综合评估指标**: 引入pointer arithmetic、idiomaticity、readability等新指标
6. **测试套件增强**: 为现有基准添加更多测试和断言
7. **人类研究**: 首次评估翻译的可维护性/可用性

## 6. Potential Applications & Extensions

1. **多语言扩展**: 将框架扩展到其他语言对（如C++到Rust）
2. **增量翻译**: 支持代码库更新后的增量翻译
3. **性能优化**: 分析并减少翻译代码的运行时开销
4. **自动化程度提升**: 减少人工干预的调试时间

---

## Session Notes

### 技术亮点:
- **重构优先于翻译**: 通过预处理C代码降低LLM翻译难度
- **指针算术是关键**: 这是C/Rust语义差异最大的地方
- **成本控制**: $0.48远低于Syzygy的$2500

### 实现细节:
- LLM: DeepSeek-V3 (主) + DeepSeek-R1 (推理)
- 重新提示预算: 基础模型8次 + 推理模型2次
- 分解阈值: 上下文窗口的50%
- ICL池索引: Rust官方500种错误码

### 评估创新:
- 使用pycparser和rustc_hir计算精确指标（非正则表达式）
- 报告断言通过率而非测试通过率
- 引入Translation Size Inflation归一化Clippy警告
