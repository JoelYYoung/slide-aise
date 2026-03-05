# RustMap: Towards Project-Scale C-to-Rust Migration

## 基本信息
- **标题**: RustMap: Towards Project-Scale C-to-Rust Migration via LLM and Dependency-Guided Transpilation
- **作者**: Xuemeng Cai, Jiakun Liu, David Lo (SMU), Liming Nie, Xing Hu, Shanping Li (Open University of China)
- **机构**: Singapore Management University, Open University of China
- **年份**: 2025
- **会议/期刊**: arXiv preprint (待发表)
- **开源代码**: https://github.com/AInnovateLab/RustMap

## 核心问题
现有C-to-Rust工具如C2Rust只能生成包含大量unsafe代码的Rust代码，无法利用Rust的安全特性。而基于LLM的方法只能处理函数级别的翻译，难以扩展到项目级别的复杂C代码。

## 主要贡献
1. **项目脚手架生成**: 分析C项目结构，生成对应的Rust Cargo项目结构
2. **宏到函数转换**: 将C宏转换为函数，以便LLM处理
3. **依赖引导翻译**: 按照函数依赖的拓扑顺序进行翻译
4. **编译+功能双反馈**: 不仅检查编译通过，还验证功能正确性

## 技术方法

### 1. 项目脚手架 (Project Scaffolding)
- 解析C项目结构（.c/.h文件、目录层次）
- 生成对应的Cargo.toml和模块结构
- 保持原有的模块化组织

### 2. 宏处理
- 将函数式宏转换为Rust函数
- 对象式宏使用const或static替代
- 关键创新：利用LLM进行宏语义理解

### 3. 依赖分析与翻译顺序
- 构建函数调用依赖图
- 按拓扑顺序从叶子节点开始翻译
- 确保翻译时已有所有依赖的Rust实现

### 4. LLM翻译流程
- 使用GPT-4o作为翻译模型
- 提供函数签名、被调用函数上下文
- 编译错误时进行修复迭代

### 5. 二分法错误定位
- 当编译通过但测试失败时
- 使用二分搜索定位错误翻译的函数
- 用C实现替换Rust实现来隔离问题

## 实验设置
- **基准**: 从Rosetta Code选择126个程序
- **对比**: C2Rust, Crown
- **指标**: 编译成功率、测试通过率、代码质量

## 主要结果
- 成功翻译bzip2（约7000行代码），达到项目级别
- 在Rosetta Code基准上显著优于C2Rust和Crown
- 生成的代码safe比例显著更高

## 关键局限
1. **成本**: 依赖商业LLM (GPT-4o)，API调用成本高
2. **速度**: 项目级翻译需要大量API交互
3. **可重复性**: LLM输出存在随机性
4. **复杂宏**: 某些复杂的预处理器宏仍难以处理

## 与其他工作对比
- **vs C2Rust**: RustMap生成更安全的代码，但需要LLM支持
- **vs Crown**: RustMap覆盖更完整（包括全局变量、结构体）
- **vs 纯LLM方法**: RustMap通过依赖分析扩展到项目级别

## 评价
**优点**:
- 首个系统性解决项目级C-to-Safe-Rust的工作
- 依赖引导策略有效解决跨函数一致性问题
- 二分法错误定位很实用

**缺点**:
- 过度依赖商业LLM，成本和可复现性存疑
- 7K LoC的bzip2仍然是相对小的项目
- 缺乏与学术baseline的对比（如Laertes）

## 引用关系
- 基于C2Rust的基础翻译能力
- 继承了Crown对unsafe消除的研究
- 结合了近期LLM代码翻译的趋势
