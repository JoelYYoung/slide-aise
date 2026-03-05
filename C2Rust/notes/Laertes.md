# Laertes: Translating C to Safe Rust

## 基本信息
- **标题**: Translating C to Safer Rust
- **作者**: Mehmet Emre, Ryan Schroeder, Kyle Dewey, Ben Hardekopf
- **机构**: UC Santa Barbara
- **年份**: 2021
- **会议/期刊**: OOPSLA 2021
- **开源代码**: https://github.com/pllvm/c2saferust

## 核心问题
C2Rust生成的代码几乎完全是unsafe的，其中原始指针(raw pointer)是主要问题。如何系统性地将原始指针转换为安全的Rust引用(&T, &mut T)？

## 主要贡献
1. **首次学术研究**: 系统分析C2Rust输出中的unsafe来源
2. **分类法**: 定义7类unsafe来源
3. **指针转换方法**: 提出原始指针到引用的转换技术
4. **实证研究**: 在17个真实C程序上进行评估

## Unsafe分类

### 七类unsafe来源
1. **RawDeref**: 原始指针解引用
2. **Global**: 全局可变状态
3. **Union**: 联合类型
4. **Allocation**: 内存分配
5. **Extern**: 外部函数调用
6. **Cast**: 不安全类型转换
7. **InlineAsm**: 内联汇编

### 分布统计
- RawDeref占绝对多数（~95%）
- 其他类型占比很小

## 技术方法

### 核心策略
将`*mut T` / `*const T` 转换为 `&mut T` / `&T`

### 关键步骤
1. **可空性分析**: 判断指针是否可能为NULL
2. **所有权分析**: 判断是unique还是shared
3. **生命周期推断**: 确定引用的有效范围

### 转换规则
```
*mut T → &mut T  (unique ownership, non-null)
*const T → &T    (shared, non-null)
*mut T → Option<&mut T>  (unique, nullable)
```

### 保守策略
- 如果分析不确定，保持原始指针（保证安全）
- 优先正确性，牺牲转换率

## 实验设置

### 测试程序（17个）
- bzip2, gzip, urlparser
- libcsv, json-c, grabc
- genann, lil, RoaringBitmap
- optfuzz, robotfindskitten, xzoom
- 等

### 评估指标
- 转换的指针数量/比例
- 消除的unsafe操作数量
- 编译成功率
- 测试通过率

## 主要结果
- **合格指针转换率**: 93%（对于分析认为可转换的指针）
- **整体转换率**: 仅约11%的原始指针被转换
- **原因**: 大量指针因别名问题无法安全转换

## 关键局限
1. **别名问题**: Rust的借用规则比C更严格，多处别名时无法转换
2. **全局变量**: 静态可变全局变量几乎无法安全化
3. **外部函数**: FFI边界仍需unsafe
4. **覆盖率低**: 89%的指针无法转换

## 重要发现
1. **C和Rust语义差距巨大**: 不仅是语法问题
2. **别名是核心障碍**: C的别名模式与Rust不兼容
3. **需要新方法**: 单纯的类型转换不够

## 评价
**优点**:
- 首次系统性研究，奠定领域基础
- 分类法对后续工作有指导意义
- 方法正确性有保证（保守策略）

**缺点**:
- 转换率太低（11%）实用性受限
- 17个程序规模较小
- 未解决核心的别名问题

## 后续影响
- 直接引出了OOPSLA'23的后续工作（Aliasing Limits）
- 影响了Crown、C2SafeRust等多个后续工作
- 定义的7类unsafe成为领域标准分类

## 引用关系
- 基于C2Rust的输出
- 与LLVM相关的程序分析工作
- 后续被Aliasing Limits扩展
