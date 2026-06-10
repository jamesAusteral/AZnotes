---
date: 2026-06-10
categories:
  - wiki日志
---

# 06-10

今天搭建最小demo

<!-- more -->


今天我让DeepSeek接入Claude code先根据karpathy的最初思路llm-wiki.md，建立了一个最简单版的llm-wiki。
并且上传到GitHub中，push了第一版的wiki系统。

wiki如今能支持DeepSeek的api。

## 第一次尝试：面对pku compiler course documentation 建立wiki

我把之前学的北大编译实践的实验文档下载了下来放到源文件中。

让我来尝尝咸淡。

真长啊，怎么会有这么多词条啊。



## 代码阅读（还没写reflection）

### 代码阅读顺序

**第一步：看产出（Obsidian 里看，10 分钟）**

打开 Obsidian，看点有代表性的页面：

- **wiki/index.md** — wiki 的目录，了解生成的内容结构
- **随便打开几个** **[[page]]** — 看页面长什么样，感受交叉引用的密度
- **Graph view**（左边工具栏点图谱图标）— 看知识网络的形状，哪些页面是枢纽
- **看一个页面的 Backlinks**（右侧面板）— 感受「哪些其他页面引用了这个页面」

**第二步：理解 Prompt（最重要，20 分钟）**

wiki_engine/llm_client.py

这是整个系统的核心。重点看这三个东西：

| 名称 | 作用 | 说明 |
|------|------|------|
| BASE_SYSTEM_PROMPT | 告诉 LLM「你是 wiki 维护者」 | 定义了 LLM 的角色和行为准则 |
| INGEST_PROMPT_TEMPLATE | 告诉 LLM 怎么处理新资料 | 定义了输出格式——LLM 就按这个格式返回结构化结果 |
| QUERY_PROMPT_TEMPLATE / LINT_PROMPT_TEMPLATE | 查询和检查的指令 | 同理 |

**关键认知**：在这个系统里，Prompt 就是「代码」。你通过自然语言指令编程，LLM 执行。这和传统编程的区别只是——编译器变成了 LLM，C++ 语法变成了 markdown格式指令。

下面的 DeepSeekClient._call_api() 和 AnthropicClient._call_api() 是纯粹的网络调用——怎么把 prompt 发给 API、怎么取回结果。跟调用任何 HTTP API没本质区别


**第三步：理解编排（15 分钟）**

wiki_engine/engine.py

重点看三个方法的流程：

- ingest() — 读文件 → 调 LLM → 解析输出 → 写页面 → 更新 index → 记 log
- query() — 搜 index → 读页面 → 调 LLM → 展示答案（可选保存为页面）
- lint() — 读全部页面 → 调 LLM → 展示报告

**关键认知**：Engine 做的是「体力活」（文件 I/O、解析、index 维护），LLM 做的是「脑力活」（理解内容、写文章、找联系）。这种分工是这个架构的核心。

方法 _parse_ingest_result() 值得细看——它展示了如何把 LLM 的自然语言输出解析成结构化的数据（页面列表、更新列表、交叉引用等）。这是 AI应用中常见的模式：**LLM 输出半结构化文本 → 正则解析 → 变成程序可操作的数据**。

**第四步：基础设施（10 分钟）**

wiki_engine/indexer.py   — index.md 的维护逻辑（怎么加条目、怎么更新摘要）

wiki_engine/logger.py    — log.md 的追加逻辑（怎么记录操作、怎么 grep）

**第五步：入口（5 分钟）**

cli.py                   — argparse 子命令，调用 engine

config.yaml              — 所有可配置项

**总结：这个系统的「骨架」**
```

你 → cli.py → engine.py → llm_client.py → DeepSeek API
                      ↓              ↑
                indexer.py    system prompt（核心逻辑）
                logger.py
                      ↓
                wiki/*.md（持久化产物）
```

从外到内读一遍，你应该能理解「用 LLM 构建应用」的基本模式：**Prompt 定义行为，Engine 做编排，文件系统做持久化，LLM API做推理**。这和你写过的任何传统程序本质上是一样的——只是有一部分逻辑不是用代码写的，而是用自然语言写的。

### reflection: 



