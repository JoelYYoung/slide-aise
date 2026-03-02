# AI, Software Engineering, and Compilers — 演讲指南

> **相关课程**: Stanford CS 343D: The Modern Software Development Stack  
> 🔗 https://themodernsoftware.dev/

---

## 📊 Slides 整体结构

| Section | 主题 | 核心内容 |
|---------|------|---------|
| 1 | Introduction | 软件工程的变革、覆盖的论文 |
| 2 | AI for SE: Tasks | AI 在 SE 中的任务分类学 |
| 3 | AI Coding Agents | AI 编程代理的定义和架构 |
| 4 | LLMs in Compilers | LLM 与编译器的集成方式（含 Compiler.next） |
| 5 | Key Challenges | 五大核心挑战 |
| 6 | Future & Conclusion | 未来方向和总结 |

---

## 🎤 讲解建议（逐页）

### Section 1: Introduction (3 slides)

**Slide: The Changing Landscape of Software Engineering**
- 对比传统 SE 和 AI-Native SE
- 强调核心转变：**从"人写代码"到"人表达意图"**
- **讲法**：先问听众"大家觉得 AI 会如何改变编程？"，然后揭示这个对比表

**Slide: Papers We Cover Today**
- 列出 4 篇论文
- **讲法**：快速过一遍，告诉听众这是一个 survey-of-surveys，综合了不同视角

---

### Section 2: AI for SE Tasks (2 slides)

**Slide: Task Taxonomy**
- 四大任务类别：代码生成、代码转换、测试分析、维护
- **讲法**：强调"大家关注的是代码生成，但真正的 SE 还有很多其他任务"

**Slide: Three Measures**
- Scope / Complexity / Human Intervention 三个维度
- **讲法**：用 HumanEval vs SWE-Bench 做对比，让听众理解不同级别的难度

---

### Section 3: AI Coding Agents (3 slides)

**Slide: What is AI Agentic Programming?**
- 定义 + 流程图
- **讲法**：强调 Agent 的四个特性（自主、交互、迭代、目标导向），用流程图说明 feedback loop

**Slide: Agent Behavior Dimensions**
- 四个维度：Reactivity, Multi-turn, Tool-augmented, Adaptive
- **讲法**：从 Copilot（灰色/reactive）到完全自主 agent（绿色/proactive）的演进

**Slide: Agent System Categories**
- 表格对比不同系统
- **讲法**：指出趋势是从 IDE 助手向多 agent 系统发展

---

### Section 4: LLMs in Compilers (4 slides)

**Slide: The New Compiler Stack - Integration**
- Selector / Translator / Generator 三种模式
- **讲法**：用例子说明
  - **Selector**: LLM 选择编译器优化 flag
  - **Translator**: LLM 做代码翻译（如 C→Rust）
  - **Generator**: LLM 生成编译器代码本身

**Slide: LLM-Compiler Applications: Success Stories**
- 四大应用领域：Code Transpilation, IR Optimization, Hardware Synthesis, Automated Repair
- **讲法**：强调 LLM 擅长需要**跨领域知识**的任务

**Slide: Code Abstraction Levels & Tasks**
- 展示 NL → PL → IR 的层次图
- **讲法**：指着图说明 LLM 可以在每个层次之间做转换（Code Gen, Transpile, Compile, Decompile）

**Slide: Compiler.next: Intent → Optimized FMware**
- 意图编译的核心思想：将人类意图编译为优化的 FMware
- **讲法**：这是 LLM-Compiler 的一个前沿案例，强调多目标优化（accuracy vs cost vs latency）和结果（+47% 准确率，-42% 延迟）

---

### Section 5: Key Challenges (5 slides)

这是最重要的部分之一，建议花时间讲清楚：

1. **Correctness & Verification** - LLM 是概率性的，如何验证正确性？
2. **Scalability & Long Context** - 真实代码库有百万行，LLM context 有限
3. **Evaluation & Benchmarks** - 现有 benchmark 太简单/有数据污染
4. **Tool Integration** - 现有工具是为人设计的，不适合 agent
5. **More Challenges** - 可复现性、成本、人机协作、小众领域

**讲法建议**：每个挑战用 1-2 分钟，给出具体例子

---

### Section 6: Future & Conclusion (3 slides)

**Slide: Research Directions**
- AI for SE（数据策展、RL优化、工具接口、人机协作）+ New Compiler Stack（混合系统、自改进编译器、跨层转换、可复现 FM 编译）
- **讲法**：强调共同主题——弥合 LLM 灵活性与传统编译器可靠性的差距

**Slide: The Hybrid Future**
- 传统编译器 + LLM → 混合系统
- **讲法**：这是核心结论！强调"最有前景的是混合方案"

**Slide: Key Takeaways**
- 4 个要点总结：AI 改变 SE → LLMs in Compilers（模式 + Compiler.next）→ 挑战 → 混合系统
- **讲法**：快速回顾，强调最后的 vision："人关注做什么和权衡，AI 处理日常开发"

---

## ⏱️ 时间分配建议

假设 30 分钟报告：

| Section | 建议时间 |
|---------|---------|
| Introduction | 3 min |
| AI for SE Tasks | 3 min |
| AI Coding Agents | 5 min |
| LLMs in Compilers | 8 min |
| Key Challenges | 7 min |
| Future & Conclusion | 4 min |

---

## 💡 讲解技巧

1. **开场**：以问题开始 — "大家觉得 5 年后程序员还需要写代码吗？"
2. **转场**：每个 section 开始时说明"刚才讲了 X，现在我们看 Y"
3. **强调重点**：LLM-Compiler 集成模式（Selector/Translator/Generator）+ Intent Compilation
4. **结尾**：回到开场的问题，给出你的观点

---

## 📚 References

### 论文
- Gu et al., "Challenges and Paths Towards AI for Software Engineering," 2025
- Wang et al., "AI Agentic Programming: A Survey," 2025
- Zhang et al., "The New Compiler Stack: A Survey on LLMs and Compilers," 2026
- Cogo et al., "Compiler.next: A Search-Based Compiler for SE 3.0," 2025

### Benchmarks
- **HumanEval, MBPP** — Function-level code generation
- **SWE-Bench** — Project-level software engineering
- **LiveCodeBench** — Temporal evaluation
