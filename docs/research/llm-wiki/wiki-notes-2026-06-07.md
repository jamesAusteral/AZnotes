---
date: 2026-06-07
categories:
  - wiki日志
---

# 开发第-1/0/1天。。。无所谓了

我尝试理顺wiki的架构，还有其基础原理。

<!-- more -->

## llm wiki的结构，发展架构

https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f

显然这是karpathy的初始想法，非常的抽象，没有给任何细节。

https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/blob/main/README_CN.md

这是一个自动科研agent，可能会有帮助。

https://themodernsoftware.dev/

Stanford cs146s对于开发也或许有帮助。

我想搞清楚，搜索策略和agent构建这两者的技术位于LLM产业的哪里层级。

It reads it, extracts the key information, and integrates it into the existing wiki — updating entity pages, revising topic summaries, noting where new data contradicts old claims, strengthening or challenging the evolving synthesis. The knowledge is compiled once and then _kept current_, not re-derived on every query.
这段话中提到知识只需要被整理一次，但是每次调用api和LLM对话的时候不也是无记忆的和LLM对话吗？

就我的理解，“编译”的核心思想是将一种语言等价转化，解析为一个数据结构，然后根据这个数据结构生成中间表达形式（IR），IR的信息和源代码是相等的，但是在IR上能进行更多的操作，由于SSA等原因，从而加速计算。
IR的信息和源代码是相等的，总有一种方法能加速IR的计算，这是我对编译的大体理解。
如果类比到LLM wiki中，知识编译会不会也遵循这样的机制。

