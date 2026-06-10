---
date: 2026-06-08
categories:
  - wiki日志
---

# 06-08

我有很多疑惑。

<!-- more -->

https://github.com/atomicstrata/llm-wiki-compiler

这个项目实现了一种知识编译的想法。

---

## 一些背景材料

- Karpathy 原始提案：https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f 
- 已知 demo 之一（llm-wiki-compiler）：https://github.com/atomicstrata/llm-wiki-compiler 
- 相关课程参考：https://themodernsoftware.dev/ 
- zju的一个demo：https://github.com/zjunlp/SciAtlas 

---

## 喂给Claude的调查方案。

### 1. Karpathy 的原始想法
- Karpathy 在 Gist 中具体描述了什么？核心机制是什么？
- 他在 Gist 评论区、X/Twitter、HN 等公开场合对此有何补充表态？
- 这个想法最初发布于何时，引发了什么讨论？

### 2. 现有 Demo 与实现调研
- 搜索并整理 Karpathy Gist 评论区中所有留下链接的实现项目，列出名称、链接、技术栈、
  实现思路简述。
- 除评论区外，GitHub 上是否有其他以 "llm wiki" 为关键词的相关实现？
- 各实现之间架构上有何共性与差异？
- https://github.com/zjunlp/SciAtlas 这个zju组的demo实现了什么，怎么实现的，用了什么技术，解决了什么问题，和llm wiki有什么相关性。

### 3. 知识编译的类比是否成立？
- 传统编译器的核心思想：源码 → AST/IR → 优化 → 目标代码。
  IR 与源码信息等价，但 IR 上可以做更多操作（如 SSA）从而加速计算。
- 将这一机制类比到 LLM Wiki：知识是否也存在"中间表达形式"？
  知识的 IR 化是否能加速 LLM 的推理/检索？
- 学界或工程界是否有人用编译器理论框架来讨论知识表示与 LLM？
  请搜索相关论文或博客。

### 4. 疑虑之一：无记忆问题
- Karpathy 说"知识只需整理一次"，但每次 API 调用都是无状态的。
  这个矛盾如何解决？现有实现是怎么处理的？
  （RAG？KV cache？System prompt 注入？还是其他机制？）
- LLM Wiki 与传统 RAG pipeline 的本质区别是什么？
- 我也有其他疑虑，其实我对于llm应用开发这一块不是很懂，我不能描述出来所有我的问题，所以你其实可以推测一下我可能的问题，这个没关系。

### 5. 技术层级定位
- 搜索策略（Search/Retrieval strategy）和 Agent 构建，
  分别位于当前 LLM 技术栈的哪个层级？
  （基础模型层 → 推理框架层 → 应用/编排层 → 产品层）
- 实现 LLM Wiki 需要用到哪些具体技术？
  对开发者的技术栈要求是什么？（embedding、vector DB、chunking、
  agent framework、API 调用、前端展示等各环节）

### 6. Stanford CS 146S 课程的相关性
- https://themodernsoftware.dev/ 这门课涵盖了哪些内容？
- 其中哪些模块与构建 LLM Wiki 直接相关？
  （如 agent 构建、RAG、工具调用、软件架构等）
- 这门课对于一个想从零实现 LLM Wiki 的开发者有多大参考价值？

### 7. 社区反馈
- Hacker News 上对 LLM Wiki 这一想法的讨论帖是什么？
  主流反馈是支持、质疑还是有更深入的批评？
  请列出具体的 HN 帖子链接和关键评论观点。
- Reddit、Twitter/X 上是否有相关技术讨论？

---

## 输出要求

1. 输出为完整 Markdown 调查报告，包含目录
2. 每个结论须附原始来源链接，不得虚构
3. 对我的疑虑（知识整理一次 vs 无记忆 API等等；编译类比是否成立；技术层级定位）
   须有明确的、有据可查的回答，而非泛泛而谈
4. 报告末尾给出"如果我现在要开始实现 LLM Wiki，推荐的技术路线图"，
   包括技术选型建议和学习资源
5. 如果某些问题调研结果不足，请明确标注"待补充"，不要编造


Claude根据以上prompt得出了一个报告。
但是我还有其他的一点疑问：

1. auto research的架构对wiki的启发？
   https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep#-research-wiki--persistent-research-memory 

2. 如此庞大的项目，我必定使用vibe coding，那怎么保证我学到了东西呢？

3. 持久化？开发这样的一个系统也能用得到os的知识吗？有没有必要回头去学jyy的课？

4. benchmark这一块，似乎没有直接可用在上面的。

