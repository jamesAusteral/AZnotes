# LLM Wiki 深度调研报告：从 Karpathy 的"知识编译器"到可落地的开发者路线图

## 执行摘要

2026 年 4 月 2 日，Andrej Karpathy（OpenAI 联合创始人、前特斯拉 AI 负责人）在 X 上发了一条关于"LLM Knowledge Bases"的推文，4 月 4 日（Gist 显示创建于 April 4, 2026 16:25）配套发布了一份名为 `llm-wiki.md` 的 GitHub Gist。这条推文获得约 1600 万次浏览，Gist 在数日内突破 5000+ stars。它不是产品也不是代码，而是 Karpathy 称之为"idea file"（想法文件）的东西——一份可以直接粘贴给 LLM agent（Claude Code、Codex 等）的高层模式描述。

LLM Wiki 的核心主张是：不要在每次查询时对原始文档做 RAG（即时检索），而是让 LLM **增量式地构建并维护一个持久化的 markdown wiki**——这个 wiki 坐落在你和原始资料之间，知识"编译一次、持续保鲜"，而非每次查询重新推导。Karpathy 自己的一个研究 wiki 已增长到约 100 篇文章、40 万词，全部由 LLM 撰写维护。其架构极简：三层目录（raw 原始源 / wiki LLM 生成层 / schema 即 CLAUDE.md 配置文件），三个操作（ingest 摄取 / query 查询 / lint 健康检查）。

本报告基于对 Gist 正文与全部评论、5 个 Hacker News 讨论帖、十余个开源实现、Stanford CS146S 课程、经典知识编译理论（Darwiche & Marquis 2002）以及浙大 SciAtlas 论文（arXiv:2605.22878）的一手抓取与分析。

**核心结论有三**：

(1) "知识整理一次 vs 无记忆 API"的矛盾，本质上由"外部持久化文件系统 + 每次会话重新加载上下文"解决——LLM 本身无状态，但 wiki 文件是状态载体，CLAUDE.md/AGENTS.md 在每次会话注入约定；

(2) "编译"类比在工程直觉上成立（源码→编译产物，摊销式预计算），但在严格的编译器理论意义上**不成立**——传统编译保证语义等价（IR 与源码信息等价），而 LLM 的"知识编译"是有损、可能产生幻觉的合成，这是它与 SSA/IR 优化的根本差异；

(3) 技术层级上，搜索/检索策略与 Agent 编排都位于**应用/编排层**，构建 LLM Wiki 不需要训练模型，主要是 context engineering + 文件系统 + agent 框架的工程活。

对于想动手的开发者，我们的明确建议是：**先用纯 markdown + 一个 agent（Claude Code / Codex）+ index.md 跑通最小闭环，不要一上来就上向量库**。在约 5 万–10 万 token（约 150–200 页）以下，纯上下文/索引方案比 RAG 更简单也更可靠；只有当语料超出上下文窗口（数百万 token 级别）时才引入向量检索。报告末尾给出分阶段技术路线图与选型建议。

---

## 目录

1. Karpathy 的原始想法
2. 现有 Demo 与实现调研
3. 知识编译的类比是否成立？
4. 疑虑分析：无记忆问题与其他工程顾虑
5. 技术层级定位
6. Stanford CS146S 课程（themodernsoftware.dev）
7. 社区反馈（HN / Reddit / X）
8. SciAtlas 对比分析（独立章节）
9. 技术路线图：如果我现在要开始实现 LLM Wiki
10. 参考文献汇总

---

## 1. Karpathy 的原始想法

### 1.1 核心机制（直接引用 Gist）

Gist 标题为 "LLM Wiki"，副标题 "A pattern for building personal knowledge bases using LLMs"。Karpathy 首先界定它与 RAG 的区别：

> "Most people's experience with LLMs and documents looks like RAG: you upload a collection of files, the LLM retrieves relevant chunks at query time, and generates an answer. This works, but the LLM is rediscovering knowledge from scratch on every question. There's no accumulation."

然后给出核心想法：

> "Instead of just retrieving from raw documents at query time, the LLM **incrementally builds and maintains a persistent wiki** — a structured, interlinked collection of markdown files that sits between you and the raw sources. When you add a new source, the LLM doesn't just index it for later retrieval. It reads it, extracts the key information, and integrates it into the existing wiki — updating entity pages, revising topic summaries, noting where new data contradicts old claims... The knowledge is compiled once and then *kept current*, not re-derived on every query."

关键的一句对 IDE 的类比："Obsidian is the IDE; the LLM is the programmer; the wiki is the codebase."

**三层架构**（直接引用）：

- **Raw sources**：策划好的源文档集合（文章、论文、图片、数据）。"These are immutable — the LLM reads from them but never modifies them. This is your source of truth."

- **The wiki**：LLM 生成的 markdown 文件目录（摘要、实体页、概念页、对比、综述）。"The LLM owns this layer entirely... You read it; the LLM writes it."

- **The schema**：即 CLAUDE.md（Claude Code）或 AGENTS.md（Codex），告诉 LLM wiki 如何组织、约定是什么、摄取/查询/维护时遵循什么工作流。"This is the key configuration file — it's what makes the LLM a disciplined wiki maintainer rather than a generic chatbot."

**三个操作**：
- **Ingest**：放入新源，LLM 读取、与你讨论要点、写摘要页、更新 index、更新相关实体/概念页、追加 log。"A single source might touch 10-15 wiki pages."

- **Query**：针对 wiki 提问，LLM 检索相关页、合成带引用的答案。关键洞察："good answers can be filed back into the wiki as new pages." 这样探索也会复利。

- **Lint**：周期性健康检查——查找页面间矛盾、被新源取代的过时声明、无入链的孤儿页、缺页的重要概念、缺失交叉引用、可用网搜补充的数据缺口。

**索引与日志**：`index.md` 面向内容（目录式，每页一行摘要+链接，按类别组织）；`log.md` 面向时间（append-only 追加记录，建议每条以 `## [2026-04-02] ingest | Article Title` 前缀开头，可用 `grep "^## \[" log.md | tail -5` 解析）。Karpathy 明确说："This works surprisingly well at moderate scale (~100 sources, ~hundreds of pages) and avoids the need for embedding-based RAG infrastructure."

**可选 CLI 工具**：随着 wiki 增长，推荐 `qmd`（Shopify CEO Tobi Lütke 开发的本地 markdown 搜索引擎，混合 BM25/向量检索 + LLM 重排，本地运行，同时提供 CLI 和 MCP server）。

**为什么有效**（直接引用）："The tedious part of maintaining a knowledge base is not the reading or the thinking — it's the bookkeeping... Humans abandon wikis because the maintenance burden grows faster than the value. LLMs don't get bored, don't forget to update a cross-reference, and can touch 15 files in one pass."

Karpathy 把这个想法上溯到 Vannevar Bush 1945 年的 Memex 愿景——一个私人的、主动策划的知识库，文档之间的关联与文档本身同样有价值。Bush 没解决的问题是"谁来做维护"，而"The LLM handles that."

### 1.2 提示词建议与工具

Gist 的 "Tips and tricks" 推荐：Obsidian Web Clipper（网页转 markdown）、本地下载图片（让 LLM 可直接看图）、Obsidian Graph view（看 wiki 形态、找 hub 页和孤儿页）、Marp（markdown 幻灯片）、Dataview（对 frontmatter 跑查询）。最后强调："The wiki is just a git repo of markdown files. You get version history, branching, and collaboration for free."

### 1.3 发布时间与初始讨论

- 4 月 2 日（部分来源记为 4 月 3 日）：原始推文 "LLM Knowledge Bases"，约 1600 万次浏览。

- 4 月 4 日：Karpathy 发布 Gist（"idea file"），并发推解释 idea file 的理念——在 LLM agent 时代，分享具体代码/app 的必要性下降，你只需分享想法，agent 会为你实例化。

- Karpathy 还转发并称赞了 Farza Majeed 的 "Farzapedia"：用 LLM 把 2500 条日记/Apple Notes/iMessage 转成约 400 篇个人 wiki 文章。Karpathy 评论说他喜欢这种"显式记忆制品（explicit memory artifact）"的个性化方式，对比当下"AI 用得越多就越懂你"的隐式方案。

---

## 2. 现有 Demo 与实现调研

### 2.1 Gist 评论区中留下链接的实现项目

下表整理了 Gist 评论区中开发者公开的实现/扩展（按出现顺序）：

| 项目                                              | 链接                                              | 技术栈/思路                                                                                                                                               |
| ----------------------------------------------- | ----------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| Bias-Aware Stateful Lint（brtrx）                 | gist.github.com/brtrx                           | 针对 50+ 源的 lint 扩展，分批（每批 5 个）跑 lint + 跨会话持久 scratchpad + 随机化（非摄取顺序）序列，对抗"摄取顺序偏差"。挂载到现有 CLAUDE.md                                                      |
| Mnemosyne（noirblue）                             | github.com/noirblue/IsaacCLupus_mnemosyn_spec   | 把 wiki 当"基础设施"：Rust 核心（schema、任务队列、图遍历、API）+ Python 卫星（文档抽取、LLM 编排）。含 agent 记忆 inbox→committed、双节点部署（Foundry 批量编译 + Frontline 交互服务）、审计、pack 导出       |
| Life_Daily_OS（iddingszhz）                       | github.com/iddingszhz/Life_Daliy_OS             | 日记库，L0→L4 分层（记录→周回顾→跨时间复利对比），协议（CLAUDE.md）与人格分离                                                                                                      |
| Synthadoc（paulmchen / axoviq-ai）                | github.com/axoviq-ai/synthadoc                  | 五态页面生命周期（draft→active→stale→contradicted→archived），SHA-256 哈希失配自动标 stale，导出 llms.txt/graphml/json（含每段溯源行号、状态转换审计、per-page API 成本），Obsidian 插件，查询结果缓存 |
| ai_dev skills（theafh）                           | github.com/theafh/ai-modules                    | 把 wiki 模式套到任务管理（/task 类比 JIRA），一任务一 markdown，文件系统原生 + git 跟踪                                                                                         |
| personal-memory-vault-starter（hegu-1）           | github.com/hegu-1/personal-memory-vault-starter | "个人连续性容器"，强调框架与决策仍由人撰写                                                                                                                               |
| matryca-plumber（MarcoPorcellato）                | github.com/MarcoPorcellato/matryca-plumber      | 针对"扁平 markdown 突变问题"，转向 Logseq（block 级 outliner AST + 每 block UUID）+ 乐观并发控制（OCC）+ mmap，Python daemon + MCP + CLI                                     |
| secure-llm-wiki（NicoBleh）                       | github.com/NicoBleh/secure-llm-wiki             | 安全方向：把"自动摄取 wiki"视为间接提示注入面；不可信源 nonce 分隔、独立第二模型四眼审查、按 host 分信任层、git 溯源、注入语料库映射到 OWASP LLM Top 10 / MITRE ATLAS                                       |
| interview-doc-agent（Shilren）                    | github.com/Shilren/interview-doc-agent          | 面试/简历库，三层一一对应；提出"上下文 vs RAG 是量级问题"的判断标准（见 §4）                                                                                                        |
| AutoSci（skyllwt）                                | github.com/skyllwt/AutoSci，arXiv:2605.31468     | 北大；把 wiki 当"研究记忆"，自主做科研：摄取论文成交叉链接 wiki（含矛盾边）、ideate→实验→写作、项目间"睡眠期"巩固记忆、多 agent DAG。声称已端到端写了 3 篇论文                                                    |
| Dense-Mem（Z-M-Huang / markhuangai）              | github.com/markhuangai/dense-mem                | MCP 记忆服务器；分离 host LLM 与记忆层，存证据/类型化声明/已接受事实/溯源/冲突/嵌入/图召回                                                                                              |
| DPC Messenger / knowledge-in-weights（mikhashev） | github.com/mikhashev/dpc-messenger              | 探索"知识入权重"：52 个实验证明 <10M 参数下 LoRA 注入/TTT 均无法稳定召回事实（见 §3、§5 旁证）                                                                                        |
| my-llm-wiki（MuhammadSaqlainAslam，富士康 HHRI）      | github.com/MuhammadSaqlainAslam/my-llm-wiki     | Agentic 摄取（一条命令搜 arXiv/GitHub/博客/YouTube 转录）、引用情报（每篇论文显示总引用+top10 引用者）、D3 知识图、Claude API(Vertex)+PyMuPDF+KaTeX+GitHub Pages                          |
| Synto（kytmanov）                                 | github.com/kytmanov/synto                       | per-role providers（小模型本地 Ollama，重模型云端）、concept rename 全局重链、本地优先、无向量库、多语言。前身 obsidian-llm-wiki-local（100% 本地 Ollama）                                  |

评论区还有大量深度架构讨论（非项目链接），值得开发者注意：

- **pursultani**：指出该模式默认"收敛认识论"（把矛盾当缺陷消除），在人文学科等对话性领域，矛盾本身是信息；建议用 typed edges（YAML frontmatter relationship blocks，如 `rel: contradicts/extends/supersedes`）+ 配套 lint 策略（在对话性领域，缺失预期矛盾才是 lint smell）。

- **watsonrm**（多条）：git merge 解决文本冲突，但 wiki 真正的失败模式是语义冲突（两个 agent 用不同措辞写同一事实，git 干净合并却产生重复）。主张让写入幂等、可交换（append-only、一行一条、按文件/段分区），以稳定的"claim identity（引用）"而非文本邻近做矛盾检测，真冲突进 review 队列而非自动解决。

- **Archimondstat**：提出幻觉概率累积问题——若每页幻觉概率非零，页数增长时 wiki 含至少一个幻觉页的概率趋近 1；建议"Socrates–Plato–Bayes"：用户先提假设，AI 做苏格拉底式挑战，用户贝叶斯式更新后才入库。

### 2.2 GitHub 上其他 "llm wiki" 实现

GitHub `llm-wiki-karpathy` topic 下已有大量项目。代表性的有：

- **atomicstrata/llm-wiki-compiler**（即任务指定的 demo，1.1k stars，TypeScript，MIT）：自称"知识编译器"。`llmwiki ingest/compile/query/lint/export/serve` CLI。**两阶段流水线**（Phase 1 从所有源抽取所有概念，Phase 2 生成页面）消除顺序依赖、合并跨源共享概念。SHA-256 增量编译（只有变更源过 LLM）。`query --save` 把答案存回成新页。支持 Anthropic/OpenAI/Ollama/MiniMax 多 provider，输出语言可配（`--lang zh-CN`）。typed page kinds（concept/entity/comparison/overview）、claim-level provenance（`^[file.md:42-58]` 行级溯源）、confidence/contradictedBy 元数据、候选审查队列、MCP server。其 README 明确对比 RAG："RAG: query → search chunks → answer → forget；llmwiki: sources → compile → wiki → query → save → richer wiki → better answers"，并自陈"complementary to RAG, not a replacement"。

- **iamsashank09/llm-wiki-kit**（Python，MCP server）：给 agent 提供对 wiki 页的 CRUD，pip 安装，支持 Claude/Codex/Cursor/Windsurf；摄取 PDF/URL/YouTube。

- **lucasastorian/llmwiki**：本地 app + SQLite 索引 + MCP，Claude 经 MCP 读源写页。

- **nashsu/llm_wiki**：跨平台桌面应用，两步 CoT 摄取、多模态图片摄取（PDF 抽图 + 视觉 LLM 生成 caption）、4 信号知识图（直接链接/源重叠/Adamic-Adar/类型亲和）、Louvain 社区发现、可选 LanceDB 向量检索、本地 HTTP API + MCP。

- **green-dalii/obsidian-llm-wiki**：Obsidian 插件（官方评分 95/100），8 语言，多 provider（含 Ollama 本地），lint 健康扫描。

- **MehmetGoekce/llm-wiki**：L1/L2 cache 架构（类比 CPU 缓存层级——总是需要的规则放 L1 每会话加载，上下文相关知识放 L2），Logseq + Obsidian 支持。

- **swarajbachu/cachezero**（首个 Show HN）：`npx cachezero init` 一条命令，Chrome 扩展 + CLI + Hono server + LanceDB 向量索引 + Obsidian 兼容 vault。

- **Ar9av/obsidian-wiki**、**vanillaflava/llm-wiki-skills**（agentskills.io 标准，跨 agent）、**Anton15K/llm_wiki_agent**（Kotlin/Spring + Apache Tika + MCP，"零 LLM 调用"工具层）等。

### 2.3 架构共性与差异

**共性**：

(1) 几乎都保留 raw（不可变）/wiki（LLM 拥有）/schema（CLAUDE.md/AGENTS.md）三层；

(2) 都以 markdown + `[[wikilinks]]` 为存储底座，Obsidian/Logseq 友好；

(3) 都实现 ingest/query/lint 三操作；

(4) 普遍走 MCP 把 wiki 暴露成 agent 工具；

(5) 多数在小规模拒绝向量库、用 index.md 导航。

**差异轴**：
- **存储粒度**：扁平 markdown（Obsidian，页级）vs block-AST（Logseq，UUID 级，解决并发写突变）。

- **是否引入向量检索**：纯 index（Karpathy 原意，小规模）vs 混合 BM25/向量（qmd、LanceDB、nashsu）。

- **溯源/审计强度**：从无到 claim 行级溯源 + 状态机生命周期 + per-page 成本审计（Synthadoc）。

- **本地 vs 云**：100% 本地 Ollama（Synto/kytmanov）vs 云 API（多数）。

- **单写者 vs 多 agent**：单写者纪律 vs append-only 可交换写入 + worktree 隔离（watsonrm）。

- **基础设施化程度**：纯 markdown 脚本 vs Rust 核心 + 队列 + 部署拓扑（Mnemosyne）。

---

## 3. 知识编译的类比是否成立？

### 3.1 传统编译器与知识编译理论

传统编译器：源码 → 词法/语法分析 → AST/IR → 优化（如 SSA）→ 目标代码。关键性质是**语义等价**：IR 与源码信息等价，但 IR 上可做源码上难做的操作（如基于 SSA 的数据流分析），从而加速。

学界确有"知识编译（knowledge compilation）"这一成熟子领域。其奠基性文献是 Adnan Darwiche（UCLA）与 Pierre Marquis（Université d'Artois）的 **"A Knowledge Compilation Map"（Journal of Artificial Intelligence Research, vol. 17, 2002, pp. 229–264，DOI 10.1613/jair.989，arXiv:1106.1819）**。该工作提出从两个维度分析知识表示（摘要原文）："analyzing different compilation approaches according to two key dimensions: the succinctness of the target compilation language, and the class of queries and transformations that the language supports in polytime."——即 **succinctness（简洁性）**与 **tractability（可处理性，哪些 query/transformation 支持多项式时间）**。它把命题逻辑编译进 NNF（基于 DAG 的否定范式）的各种子语言（如 d-DNNF），核心思想正是：**把知识库预编译成一种目标语言，使得在线查询（如模型计数、蕴含判断）可在多项式时间内回答**——这与"编译一次、查询多次"的摊销直觉高度一致。

### 3.2 类比在哪里成立、哪里不成立

**成立的部分（工程直觉）**：
- "摊销式预计算"成立。多个二手解读（particula.tech、starmorph、levelup）都精准地把 RAG 比作"解释执行"（每次查询重新读原始源、重新解析、运行时合成），把 LLM Wiki 比作"编译执行"（原始源预编译成优化产物，查询跑在预编译知识上，更快、更一致，且受益于跨源分析）。Karpathy 本人也用了"compiled once"的措辞，atomicstrata 直接命名为"compiler"。

- 学界的知识编译同样追求"预编译换查询期可处理性"，方向一致。

**不成立的部分（严格意义）**：

1. **语义等价 vs 有损合成**。编译器/知识编译都保证 IR 与源语义**等价**（d-DNNF 与原 CNF 逻辑等价）。而 LLM 把原始文档"编译"成 wiki 页是**有损、可能引入幻觉**的自然语言合成——这正是 HN 与多篇博客的核心批评（见 §4、§7）。Anand Lahoti 的 "The Hidden Flaw" 一文指出："write-time synthesis is a loan against future correctness"：写入期合成把不可验证的 AI 文本当作真理存入检索语料，幻觉会复利累积。这是与 SSA/IR 的根本区别——SSA 变换是可证明保语义的，LLM 合成不是。

2. **确定性 vs 随机性**。编译是确定的；LLM 编译是随机的（同一源多次摄取可能产生不同页），这也是 watsonrm 强调"幂等、可交换写入"的原因。

3. **"知识 IR 化能否加速 LLM 推理/检索"**：有部分意义上的"是"——预合成的 wiki 页 + index 让 agent 少读文件、减少 token、避免 chunk 边界破坏语义（Shilren、starmorph 都论证了这点）；但它**不是**编译器那种算法复杂度意义上的加速，而是"减少重复 LLM 调用"的工程加速。

### 3.3 学界/工程界用编译器框架讨论知识与 LLM

- **经典知识编译**：Darwiche & Marquis 2002 及后续扩展——Fargier & Marquis 2008 在该地图上加入"three influential propositional fragments, the Krom CNF one (also known as the bijunctive fragment), the Horn CNF fragment and the affine fragment"。这是"knowledge compilation"的本义，与 LLM 无关，但为"编译换可处理性"提供了严格理论锚点。

- **LLM × 编译器 IR 的真实交叉**：(a) arXiv:2502.06854 "Can Large Language Models Understand Intermediate Representations?"（ICML 2025）——实证发现 LLM 能解析 IR 语法但在控制流/执行语义/循环上挣扎；(b) arXiv:2605.08247 "LLM Translation of Compiler Intermediate Representation"（IRIS-14B，GIMPLE→LLVM IR 翻译）；(c) **Meta LLM Compiler（arXiv:2407.02524）**：基于 Code Llama，"trained on a vast corpus of 546 billion tokens of LLVM-IR and assembly code"（覆盖 x86_64/ARM/CUDA），发布 7B 与 13B 两个规模、16,000 token 上下文窗口，其 FTD 变体达到"77% of the optimising potential of an autotuning search"。这些说明"编译器 IR"与"LLM"是活跃交叉领域，但讨论的是**代码编译**，不是"知识 wiki"——把 LLM Wiki 称作"编译"更多是隐喻而非这条研究线。

- **TIC（arXiv:2402.06608）**用"中间表示"思路把任务描述翻译成逻辑可解释的 IR 再编译成 PDDL，是"知识 IR 化"在规划领域的实例。

**结论**：编译类比作为**沟通隐喻和工程直觉是恰当且有用的**（预计算、产物复用、源码不可变如 raw/），但**不应被当作严格的等价性保证**。LLM Wiki 缺少编译器最核心的"保语义"性质，因此必须靠 lint、provenance、人审来补偿——这恰是社区大量扩展（Synthadoc 生命周期、secure-llm-wiki 四眼审查、cognitive governance 对抗式 plaintiff/judge）的动机。

---

## 4. 疑虑分析：无记忆问题与其他工程顾虑

### 4.1 "知识整理一次" vs "无状态 API" 的矛盾

这是最常被问的问题。HN 上有用户直接发问："a normal LLM is stateless... Is this solution effectively adding Markdown files as part of the prompt?"

**答案：是的，本质就是把外部文件系统当作状态载体，每次会话把相关内容重新加载进上下文。** 拆解：

- LLM 的单次 API 调用确实无状态、无记忆。矛盾的化解不在模型层，而在**模型外的持久化层**：wiki 是磁盘上的 markdown 文件（通常是 git repo），它跨会话、跨机器持久存在。

- 每次新会话，agent（Claude Code/Codex）读取 CLAUDE.md/AGENTS.md（schema，注入约定）+ index.md（导航）+ 按需读取相关 wiki 页进上下文。这就是"记忆"的来源——不是模型记得，而是 agent 每次把状态从文件重新喂进去。

- "整理一次"指的是**合成/bookkeeping 这件昂贵的事只做一次并固化为文件**（摄取时），而非每次查询重做。查询时只是读取已固化的页面，不再重新合成。

这与传统 RAG 的本质区别（综合 Gist、atomicstrata README、HN darkhanakh 评论、多篇博客）：

- **RAG**：corpus 静态，每次查询 retrieve top-k chunks → 生成 → 遗忘。LLM 在 query 时才"发现"关系，且每次从零发现。

- **LLM Wiki**：存在一个**写循环（write loop）**——LLM 主动撰写、维护 wiki，建反向链接、把自己的输出归档。HN 用户 darkhanakh 精准总结："the interesting bit here is the write loop... that's not retrieval, that's knowledge synthesis. In vanilla RAG your corpus is static, here it isn't."

但也有强力反驳：HN 用户 kenforthewin 坚持"This is just RAG"——因为为了让 LLM 写好 wiki，它仍需"检索最相关的知识进上下文"，无论用向量还是 index/文件系统，"那个根本问题——为 LLM 上下文检索最佳数据——就是 RAG"。devmor 称之为"persistent memory RAG"，认为这个模式每隔几天就被人"重新发现"一次。**中肯的判断是：LLM Wiki = 写时合成（write-time synthesis）的有状态知识管理 + 读时仍需某种检索；它没有取消检索，而是把"昂贵的合成"从读时移到了写时，并把合成结果持久化。**

### 4.2 与向量数据库的关系

- 在 Karpathy 原意的规模（~100 源，数百页），**不需要向量库**——index.md 能放进上下文窗口，导航本质是"目录列表"，快速、透明、可审计（Sathish Raju 博客详述）。

- Shilren（Gist 评论）提出被广泛认同的**量级判断标准**：
	- (a) < 约 5 万–10 万 token（约 150–200 页）：纯上下文/index 完胜，检索可靠性 100%（不漏匹配、不因 chunking 切断语义）、近零基建、可对全局推理；

	- (b) 数百万 token 以上：只能 RAG；

	- (c) 之间/生产：混合——核心稳定知识进上下文，海量动态数据交给 RAG。他强调 index.md 不是 RAG（不做向量匹配、不分块）。

- 工程界共识（atomicstrata、starmorph、particula）：LLM Wiki 与 RAG **互补**而非替代。LLM Wiki 本质上接近"手工、可溯源的 Graph RAG"——每个声明链回源、关系显式、结构可读。

### 4.3 内容更新 / 版本控制

- **版本控制天然解决**："The wiki is just a git repo of markdown files. You get version history, branching, and collaboration for free."（Gist）

- **更新机制**：摄取新源时 lint 检测矛盾与过时声明。Synthadoc 做到五态生命周期 + SHA-256 哈希失配自动标 stale + 状态转换审计日志。Electro-resonance 的 LLM-WIKI-MCP 做 provenance-aware 摄取（记录源元数据/时间戳/哈希/位置，重跑时跳过未变、更新已变）。

### 4.4 多语言支持

- 多个实现原生支持：atomicstrata `--lang zh-CN`/`LLMWIKI_OUTPUT_LANG`；green-dalii 原生 8 语言；Synto 多语言；nashsu 语言感知生成。Gist 本身语言无关（输出语言取决于模型与源材料）。

### 4.5 成本与延迟

- 摄取是主要成本（一次源触发 10-15 页更新，多次 LLM 调用）；查询成本低（读已编译页）。Synthadoc 把 per-page API 成本（`ingest_cost_usd`）作为一等审计字段，并加查询结果缓存（按问题文本+wiki epoch+模型为键，wiki 一变即失效）。

- 本地模型（Ollama/LM Studio）可消除 API 成本但牺牲质量；Synto 的 per-role provider 让小快模型本地、重写作模型云端，平衡成本。

### 4.6 内容质量与幻觉

这是最严肃的顾虑（HN 与多篇博客的核心批评）：

- **模型坍缩担忧**：HN 用户 devnullbrain 引用的 Nature 论文是 Shumailov、Shumaylov、Zhao、Papernot、Anderson & Gal, "AI models collapse when trained on recursively generated data," *Nature* 631(8022):755–759（2024 年 7 月 24 日，DOI 10.1038/s41586-024-07566-y），其结论原文："indiscriminate use of model-generated content in training causes irreversible defects... tails of the original content distribution disappear." (在训练过程中不加选择地使用模型生成的内容会导致不可逆的缺陷……原始内容分布的尾部会消失。) 担忧是 wiki 的复利会变成"用更冗长的信息重写有效信息"。

- **幻觉复利**：Anand Lahoti "The Hidden Flaw" 指出，LLM 撰写的内容与原始源一同被索引后，就把"不可验证信息引入了 source of truth"，且是"看似正确、结构良好、自信"的微妙插值；在团队规模会无声漂移。innobu 也指出"hallucinations compound"，关键页人审一旦超出研究玩具阶段就成为必须。

- **缓解手段**（社区方案）：lint 健康检查、claim 行级 provenance（让每段链回源行）、confidence/contradicted 元数据、人审候选队列、对抗式治理（jonadas 的 cognitive governance：plaintiff/defendant/judge 多模型；Archimondstat 的 Socrates-Plato-Bayes 用户先提假设）、secure-llm-wiki 的第二模型四眼审查。

- **"把思考外包"的哲学批评**：HN 用户 qaadika 长评——"grunt work"（交叉引用、归档）恰恰是新想法涌现之处，把它外包给 AI 会失去"个人"性，"Karpathy mistakes the words to be the goal, rather than the thinking that caused the words."

### 4.7 安全：间接提示注入

NicoBleh（secure-llm-wiki）指出关键风险：自动摄取的 wiki 本身是**间接提示注入面**——精心构造的源可植入指令，污染后续会话。其 lint 检查正确性与矛盾，但不检查 adversariality。缓解：不可信输入绝不进入后续被当作可信的通道；nonce 分隔、独立第二模型审查操纵、按 host 分信任层、git 溯源可回滚、注入语料库映射 OWASP LLM Top 10 / MITRE ATLAS。

---

## 5. 技术层级定位

### 5.1 搜索/检索策略与 Agent 构建位于哪一层

按"基础模型层 → 推理框架层 → 应用/编排层 → 产品层"划分：

- **基础模型层**（GPT/Claude/Llama 训练与权重）：LLM Wiki **不触碰**这一层。值得注意的是 mikhashev 在 Gist 评论里报告的 52 个实验：试图把知识写进权重（LoRA 注入/test-time training），在 <10M 参数下 0/600 事实可稳定召回——这从反面印证了"知识应放在外部文件而非权重"是当前正确路线。

- **推理框架层**（vLLM、KV cache、推理优化）：LLM Wiki 也不在此层。注意：LLM Wiki 的"知识只整理一次"≠ KV cache（KV cache 是单次推理内的注意力缓存，会话结束即失效）；二者常被混淆但完全不同。

- **应用/编排层**：**搜索/检索策略（RAG、向量检索、index 导航、qmd 混合检索）与 Agent 构建（工具调用、MCP、多 agent 编排、context engineering）都位于此层。** LLM Wiki 整体是这一层的应用模式。

- **产品层**：Obsidian 插件（green-dalii）、桌面应用（nashsu）、托管 SaaS（Hjarni、Dume Cowork）、CLI 工具（cachezero）。

这与 2026 年广泛讨论的"prompt engineering → context engineering → harness engineering"演进一致：LLM Wiki 本质是一种 **context engineering / harness engineering 实践**——它设计"模型在每次会话能看到什么"（wiki 页、index、schema），并用 agent harness（ingest/query/lint 循环、MCP 工具、lint 审计）把不可靠的 LLM 调用包装成可用系统。

### 5.2 实现 LLM Wiki 需要的具体技术与技术栈要求

最小可行栈（Karpathy 原意，无需写代码）：一个有文件读写能力的 agent（Claude Code / OpenAI Codex / Cursor / Windsurf / Gemini CLI）；Obsidian（或 Logseq / VS Code / 纯文件夹）作为查看界面；Obsidian Web Clipper 把网页转 markdown 入 raw/；一份 CLAUDE.md/AGENTS.md（schema）；git 做版本控制。

进阶各环节技术：

- **chunking**：纯 wiki 模式下尽量避免（chunk 边界破坏语义是 RAG 痛点）；只有上向量检索时才需要，建议按语义边界（AST/段落/标题）切。

- **embedding + vector DB**：仅在语料超出上下文时引入。本地选 LanceDB（cachezero、nashsu）；嵌入模型如 bge 系列、nomic-embed-text（本地 Ollama）。

- **混合检索**：qmd（BM25 + 向量 + LLM 重排，CLI + MCP，本地）。

- **agent framework / 协议**：MCP（Model Context Protocol）是事实标准，把 wiki_search/wiki_ingest/wiki_lint/wiki_graph 暴露成工具，wiki 文件作为 MCP resources。

- **API 调用**：Anthropic/OpenAI/Gemini/DeepSeek/Kimi/GLM/OpenRouter/Ollama 等，多 provider 抽象（atomicstrata、green-dalii）。

- **前端展示**：Obsidian（Graph view、Dataview、Canvas、Bases）、D3.js 知识图（nashsu、my-llm-wiki）、本地 web UI。

- **图分析（可选）**：Louvain 社区发现、Adamic-Adar 链接预测（nashsu）。

对开发者的技术栈要求（综合 Proudfrog 等指南）：**会用终端、git、一个 LLM coding agent 即可；不需要从零写代码**（cachezero/llm-wiki-kit/obsidian-wiki 几分钟装好）。真正的持续工作是策划 schema、决定 raw/ 放什么、审查 agent 写的内容。

---

## 6. Stanford CS146S 课程（themodernsoftware.dev）

### 6.1 课程概况与大纲

**CS146S "The Modern Software Developer"** 是 Stanford 2025 年秋季首开的、关于 AI 驱动软件开发的课程，由 **Mihail Eric**（前 Amazon Alexa 技术负责人、YC 创始人、Stanford AI 博士）讲授。课程网站 themodernsoftware.dev 公开全部材料；作业在 github.com/mihail911/modern-software-dev-assignments（公开）。课程中心论点："programming is not dead, but it has fundamentally changed"——现代开发者从"语法书写者"变为"系统架构师、AI 输出验证者、自主 agent 管理者"。

依据公开课程总结，10 周大致结构：

- **Week 1 — Coding LLM 入门**：预训练（Common Crawl→FineWeb；据 Hugging Face FineWeb，Penedo et al., arXiv:2406.17557，该数据集是"a 15-trillion token dataset derived from 96 Common Crawl snapshots"，占约 44TB 磁盘，ODC-By 1.0 许可，课程称之为"对互联网的有损压缩"）、SFT、RL（链式思考涌现，"模型需要 token 来思考"）、Swiss Cheese 能力模型；可靠性策略（few-shot、CoT、self-consistency、**RAG**、reflection、role prompting）。

- **Week 2/3 — 上下文工程 / AI IDE 与自主 agent**：sync vs async agent、"Semi-Async Zone"、**四种长上下文失败模式（context poisoning / distraction / confusion / clash）**、defensive prompting、"prompt 是新源代码"（Shawn Gross 论点）、**CLAUDE.md 模式**（永久版本化的项目上下文文件，agent 长期记忆）。

- **Week 4 — Coding Agent 模式**：为 agent 设计工具（合并函数、语义化输出、可控冗余、镜像团队环境）、CLAUDE.md 最佳实践。

- **Week 5 — 现代 AI 终端**：产品原则、Strategic vs YOLO agent。

- **Week 6 — AI 测试与安全**：SSRF/凭据窃取/YOLO mode exploit（CVE-2025-3773）、Semgrep 对 11 个 Python 应用的扫描（误报率 82–86%，非确定性）。

- **Week 7 — AI 增强代码评审**：评审层级（心智对齐为底）、AI 评审象限。

- **Week 8 — 自动 UI/应用构建**：复杂度税、v0 类系统流水线。

- **Week 9 — 部署后 agent**：SRE→AI-native operations、动态 runbook、知识图。

### 6.2 与构建 LLM Wiki 直接相关的模块

- **CLAUDE.md 模式（Week 3/4）**：与 LLM Wiki 的 schema 层完全同源——"永久、版本化的项目上下文文件 = agent 长期记忆"，正是 LLM Wiki 的 CLAUDE.md/AGENTS.md。

- **上下文工程与四种失败模式（Week 3）**：直接解释了为什么 LLM Wiki 要用 index 少读文件、为什么 lint 要查矛盾（context clash 会使准确率骤降）、为什么不能"把所有东西塞进上下文"。

- **RAG（Week 1）**：作为可靠性策略之一，是理解 LLM Wiki 与 RAG 关系的基础。

- **为 agent 设计工具 / MCP（Week 4）**：对应把 wiki 操作暴露成 MCP 工具。

- **安全（Week 6）**：对应 secure-llm-wiki 关注的间接提示注入。

- **代码评审/审计（Week 7）**：对应 wiki 的 lint 与人审。

### 6.3 参考价值评估

对想从零实现 LLM Wiki 的开发者，**CS146S 的参考价值高但是间接的**：它不教"如何搭 LLM Wiki"，但系统性地教会了构建 LLM Wiki 所需的全部底层能力——LLM 工作原理、context engineering、CLAUDE.md 模式、agent 编排、工具设计、安全、评审。建议重点学 Week 1（LLM 原理）、Week 3（上下文工程，信息密度最高）、Week 4（agent 工具与 CLAUDE.md）、Week 6（安全）。它是"理论与工程并重"读者的理想前置课程，且材料全部免费公开。

---

## 7. 社区反馈（HN / Reddit / X）

### 7.1 Hacker News

至少有 5 个相关 HN 帖：

1. **"LLM Wiki – example of an 'idea file'"**（item 47640875，72 points，19 comments）——主讨论帖。主流反馈分三派：-

   - **"这就是 RAG"派**：kenforthewin（"This is just RAG... it's building an index file of semantic connections"）、devmor（"persistent memory RAG... this pattern gets discovered every other day"）、mememememememo（"compaction for RAG"）。

   - **"不止是 RAG"派**：darkhanakh（写循环是知识合成不是检索；lint 在做 RAG 不做的事）。

   - **历史/哲学派**：Vetch 引 Licklider 1960 "Man-Computer Symbiosis"；qaadika 长篇批评"把思考外包"，强调 grunt work 是洞察涌现之处。

1. **"Show HN: LLM Wiki – Open-Source Implementation"**（item 47656181）：上传任意文档转高质量 markdown + Claude.ai MCP 30 秒接入。

2. **"Show HN: CacheZero – Karpathy's LLM wiki idea as one NPM install"**（item 47667723）。

3. **"Beyond Karpathy's LLM-Wiki: The Necessity of Cognitive Governance"**（item 47750193，jonadas）：作者用 Karpathy 思路编译了 20 年 300 份阅读笔记，得到"准确、格式良好、哲学上无菌（philosophically sterile）"的百科——像"一个能干的陌生人写的"。论点：编译框架解决了基础设施一半（AOT 编译 > JIT RAG），但留下架构一半——没有显式治理层，编译器会默认回归训练分布、把一切抹平成共识。需要治理层强制提取结构性主张、点名对立观点、找摩擦点。评论者 treetalker 提出对抗式 plaintiff/defendant/judge/appellate 多模型 + 证据规则。

4. **"Show HN: A Karpathy-style LLM wiki your agents maintain (Markdown and Git)"**（item 47899844）。

### 7.2 X / Twitter

- Karpathy 原推（约 1600 万浏览）+ idea file 解释推 + 转发 Farzapedia。

- 各类教程作者（WenHao Yu 的 Zettelkasten 对比、Sathish Raju 的 "RAG Isn't Dead"、Anand Lahoti 的 "Hidden Flaw"）在 X/Medium 广泛传播。

### 7.3 总体倾向

**支持但有保留**是主流。支持点：把维护成本压到近零、持久化复利制品、回归 Memex 愿景。质疑点：

(1) 本质是否只是 RAG/persistent memory 的"重新发现"；

(2) 幻觉/模型坍缩在规模与团队场景下复利；

(3) 把思考外包损失个人洞察；

(4) 100 页能用 ≠ 1 万页能用（innobu、Sathish Raju 均强调超出上下文后向量检索/重排/chunking 会回来）；

(5) 企业场景缺 RBAC/ACID/合规审计（innobu 引 YC 2026 Spring RFS"company brain"）。

---

## 8. SciAtlas 对比分析（独立章节）

### 8.1 SciAtlas 是什么、解决什么问题

**SciAtlas: A Large-Scale Knowledge Graph for Automated Scientific Research**（arXiv:2605.22878，2026 年 5 月 20 日提交，标注"Ongoing Work"/技术报告）由浙江大学 + University College London 团队完成。作者顺序：Shuofei Qiao（浙大/UCL，共同一作）、Yunxiang Wei（共同一作）、Jiazheng Fan、Bin Wu（UCL）、Busheng Zhang、Mengru Wang、Yuqi Zhu、Ningyu Zhang（通讯）、Keyan Ding、Qiang Zhang、Huajun Chen（通讯）。**注意：任务称"浙大的实现"，实际为浙大 + UCL 双单位。**

它要解决的问题（摘要原文）：学术产出爆炸式增长，现有检索工具"predominantly rely on superficial keyword matching or vector-space semantic retrieval, which lack the topological reasoning capabilities required to navigate complex logical connections" (主要依赖于表面的关键词匹配或向量空间语义检索，缺乏处理复杂逻辑关系所需的拓扑推理能力。)；而 agentic deep-research 框架"often prone to logical hallucinations and consuming high inference costs" (经常出现逻辑幻觉，并消耗大量的推理成本。)。SciAtlas 的方案是提供一个**确定性的"拓扑认知基底（topological cognitive substrate）"**。

### 8.2 规模与 schema

- **规模**（Table 1，已核实）：43.30M 论文（26 个学科，医学占比最大 18.56%）、109.70M 作者、3.76M 关键词、0.12M 机构、4.52K topic、252 subfield、26 field、4 domain、0.28M source——**实体共 157M，关系共 3B**（其中 COAUTHOR 边 2.06B 是主导）。数据源为 OpenAlex（4.8 亿+ 出版物）。

- **9 类实体 + 12 类关系**（论文正文有"11 vs 12"的内部不一致）。四级分类法 Domain→Field→Subfield→Topic。论文跨四个组织层次互联：语义层（CITES/RELATED_TO）、概念层（HAS_KEYWORD/COOCCUR）、方向层（学科层级）、社会层（COAUTHOR/AUTHORED/AFFILIATED_WITH）。

- 部署在 **Neo4j**，Cypher 查询；嵌入维度 1024，模型 **bge-large-en-v1.5**，COSINE 相似度。关键词由 **Qwen3-30B-A3B-Instruct-2507** 从摘要抽取（每篇 3–8 个，带重要性分）。

### 8.3 核心算法：neuro-symbolic 检索（tri-path collaborative recall + graph reranking）

这是 SciAtlas 最具技术含量的部分，"tri-path"即融合三种信号：语义相关性、图拓扑支持、全局引用重要性。

**三路节点匹配（把 query 映射成 KG 种子节点）**：

1. **Keyword Matching**：LLM 抽关键词（带分数）→ 精确文本匹配 + 向量匹配（阈值 θ_kw 默认 0.7，每词留 top-3）。

2. **Semantic Matching**：嵌入 query → 标题/摘要嵌入索引各召回 top-60 → bge-reranker-large 重排各留 top-15 → 标题 0.4 + 摘要 0.6 加权。

3. **Title Matching**：用 GROBID 抽标题（含参考文献标题）→ LLM 给置信度 → 精确(1.0)/模糊(0.65·LCS+0.35·Jaccard) 匹配，阈值 θ_title 默认 0.88。

**图重排（graph reranking）**：种子节点做 2-hop 子图传播（每跳每类型上限 500 节点）→ 按 log 缩放引用算重要性 → 边权按类型设定（HAS_KEYWORD 1.20、CITES 1.00、RELATED_TO 0.90、AUTHORED 0.80、COAUTHOR/COOCCUR 0.60）→ **Random Walk with Restart（RWR，引 Tong/Faloutsos/Pan 2006，ε=1e-6，最多 50 轮）** → 最终分 = λ_pre·语义相关 0.35 + λ_graph·拓扑支持 0.45 + λ_imp·引用重要性 0.20，返回 **top-20 论文**带"path-based explanations"。**关键效率声明**："The entire retrieval process can be completed within 2 minutes, significantly shorter than LLM-based deep research frameworks."

### 8.4 应用任务与评估

支持六类任务（§4，均为定性 running example）：文献综述、idea grounding 与 evaluation、idea generation（放松远端约束做发散）、研究趋势预测、相关作者检索、研究者背景综述。**重要：论文无任何定量基准/baseline 对比**——作者明言"merely present running examples... remaining at the qualitative analysis level"，唯一定量声明是"2 分钟内完成检索"的效率。最接近的竞品 OmniScientist（Shao et al. 2025）仅定性讨论，被指其基于 Elasticsearch 的检索"merely relies on simple propagation through citation and reference relationships"，缺乏深度拓扑推理。

仓库（github.com/zjunlp/SciAtlas）以 pip 客户端 + CLI 形式封装，提供 `search-papers` 等命令、可编辑 JSON skills、以及可移植的 **Agent Skill pack**（迁移到 Codex/Claude Code），后端为托管 API（scinet.openkg.cn），用户无需自建 Neo4j。

### 8.5 与 LLM Wiki 的关联性与差异（对比分析）

| 维度    | Karpathy LLM Wiki                                                  | SciAtlas                                              |
| ----- | ------------------------------------------------------------------ | ----------------------------------------------------- |
| 知识表示  | markdown 页 + `[[wikilinks]]`（人类可读、非结构化文本 + 弱类型链接）                  | 严格的属性图（9 实体/12 关系，Neo4j，强类型）                          |
| 规模定位  | 个人/团队（~100 源、数百页，避免向量库）                                            | 全学科（43M 论文、3B 边），必须图数据库                               |
| 检索机制  | index.md 导航 + 可选混合检索；LLM 读页合成                                      | neuro-symbolic：三路召回 + RWR 图重排（确定性拓扑推理）                |
| "确定性" | 写时 LLM 合成，有损、可能幻觉                                                  | 强调"deterministic association discovery"，用图拓扑对抗 LLM 幻觉 |
| 谁拥有内容 | LLM 撰写并拥有 wiki 层                                                   | KG 由数据管线构建，LLM 仅做关键词抽取/下游合成                           |
| 矛盾处理  | lint 标记矛盾（默认当缺陷）                                                   | 图本身不消解矛盾，靠拓扑结构呈现关联                                    |
| 共同点   | 都反对"每次查询从零 RAG"、都追求结构化持久知识、都提供 Agent Skill / MCP 让 coding agent 调用 | 同左                                                    |

**核心差异**：LLM Wiki 是"**LLM 写时合成的非结构化 wiki**"，赌注押在 LLM 的合成能力上，用 lint/人审补偿幻觉；SciAtlas 是"**数据管线构建的结构化大规模 KG**"，赌注押在图拓扑的确定性上，用 RWR 等符号算法对抗 LLM 幻觉与高成本。二者恰好代表了"知识 IR 化"光谱的两端：LLM Wiki 的 IR 是人类可读的散文（有损但灵活），SciAtlas 的 IR 是机器可读的三元组（精确但需重型基建）。值得注意的是，二者都在 §3 讨论的"编译类比"框架内——SciAtlas 更接近经典知识编译（预构建结构化表示换查询期可处理性），而 LLM Wiki 更接近隐喻意义的"编译"。对开发者的启示：个人/中小规模选 LLM Wiki 路线；当语料达百万级、需多跳确定性推理、可承受 Neo4j/管线成本时，SciAtlas 式 GraphRAG 才是答案。Gist 评论区的 AutoSci（北大）正是介于两者之间——用 LLM Wiki 模式构建研究记忆 + 多 agent DAG 做科研。

---

## 9. 技术路线图：如果我现在要开始实现 LLM Wiki

### 阶段 0：最小可行闭环（半天，零代码）

- 装 Obsidian，新建 vault；装 Obsidian Web Clipper（网页→markdown 入 `raw/`）。

- 在 vault 根放 `CLAUDE.md`（schema），用 Claude Code / Codex 打开该文件夹。

- 把 Karpathy 的 Gist 粘给 agent，让它实例化：建 `raw/`（不可变）、`wiki/`（concepts/entities/synthesis）、`index.md`、`log.md`。

- 跑通三操作：`/ingest`（摄取一篇文章，触发多页更新）、`/query`（提问，答案可 `--save` 回 wiki）、`/lint`（健康检查）。

- **从 10 个源起步**，不要一次导入全部。先让 ingest/query/lint 手感自然，再扩展。

- **门槛/选型基准**：此阶段坚决不上向量库——只要语料 < 约 5 万–10 万 token（约 150–200 页），index.md 足够。

### 阶段 1：固化纪律与溯源（1–2 周）

- 在 CLAUDE.md 中写死页面模板、命名约定、`## [date] ingest | Title` 日志前缀。

- 引入 **claim 行级 provenance**（`^[source.md:42-58]`，参考 atomicstrata），让每段链回源行——这是对抗幻觉的第一道防线。

- 加 confidence / contradictedBy 元数据；lint 增加 low-confidence、contradicted-page、broken-link、orphan 规则。

- git 初始化，每次 ingest 后 commit（免费版本史 + 可回滚）。

- **若想要现成工具**：`npx cachezero init`（最低摩擦）或 `pip install llm-wiki-kit`（MCP，多 agent）或 atomicstrata/llm-wiki-compiler（`npm i -g llm-wiki-compiler`，两阶段编译 + 增量 + 多 provider）。

### 阶段 2：检索升级（当 wiki 增长到数百页、index 放不进上下文时）

- **触发基准**：index.md 本身超出上下文窗口、或查询开始漏页时。

- 引入 **qmd**（本地，BM25 + 向量 + LLM 重排，CLI + MCP）作为搜索层；或 LanceDB（本地向量）。

- 嵌入模型：云端 bge 系列；本地用 Ollama + nomic-embed-text。

- 把 wiki 操作暴露成 **MCP server**（wiki_search/ingest/lint/graph），让任意 MCP agent（Claude Desktop/Cursor/Codex）调用。

- 仍坚持"核心稳定知识进上下文 + 海量动态数据走检索"的混合策略。

### 阶段 3：质量、安全与并发（团队/生产场景）

- **幻觉治理**：人审候选队列（compile --review）；对关键页引入对抗式审查（第二模型四眼，参考 secure-llm-wiki）或 cognitive governance（plaintiff/defendant/judge）。

- **安全**：把源视为不可信，nonce 分隔、按 host 分信任层、git 溯源可回滚，对照 OWASP LLM Top 10 / MITRE ATLAS。

- **并发**：多 agent 时采用 append-only 可交换写入 + 按文件/段分区 + worktree 隔离（watsonrm），以 claim identity（引用）而非文本邻近做矛盾检测与去重。

- **生命周期**：五态（draft→active→stale→contradicted→archived）+ SHA-256 哈希失配自动标 stale（参考 Synthadoc）。

- **成本**：per-page 成本审计 + 查询结果缓存（按问题+wiki epoch+模型为键）。

### 阶段 4：规模化（语料达百万级、需确定性多跳推理）

- 此时纯 markdown 不再够用，转向 **结构化 KG / GraphRAG**：Neo4j + 属性图 + RWR/图传播（SciAtlas 路线），或 Microsoft GraphRAG（向量做入口 + 图做多跳）。

- 2026 年社区共识：不再争"向量 vs 图"，而是**混合**——向量做语义入口检索，图做多跳关系深度。

- 企业场景补齐 RBAC / ACID / 合规审计（YC 2026 Spring RFS 称之为"company brain"的缺失原语）。

### 选型速查

- **agent**：Claude Code（CLAUDE.md 原生）/ Codex（AGENTS.md）/ Cursor / Gemini CLI。

- **界面**：Obsidian（手动编辑多、插件生态强）；Logseq（agent 写得多、block 级并发安全）。

- **本地优先**：Ollama / LM Studio + Synto（per-role provider）。

- **搜索**：小规模 index.md；中等 qmd；大规模 LanceDB / Neo4j。

- **协议**：MCP（事实标准）。

### 学习资源

- **一手**：Karpathy Gist 及全部评论。

- **课程**：Stanford CS146S（themodernsoftware.dev），重点 Week 1/3/4/6；作业 github.com/mihail911/modern-software-dev-assignments。

- **理论**：Darwiche & Marquis "A Knowledge Compilation Map"（JAIR 17, 2002, arXiv:1106.1819）。

- **参考实现**：atomicstrata/llm-wiki-compiler、iamsashank09/llm-wiki-kit、nashsu/llm_wiki、SciAtlas。

- **批判性阅读**：Anand Lahoti "The Hidden Flaw"、jonadas "Cognitive Governance"、HN item 47640875 全部评论。

---

## 10. 参考文献汇总

**一手来源**：Karpathy "LLM Wiki" Gist（含全部评论，gist.github.com/karpathy/442a6bf555914893e9891c11519de94f）；Karpathy 原推/idea file 解释推/Farzapedia 转发（x.com/karpathy）；atomicstrata/llm-wiki-compiler；zjunlp/SciAtlas（论文 arXiv:2605.22878）；Stanford CS146S（themodernsoftware.dev，作业 github.com/mihail911/modern-software-dev-assignments）。

**社区实现**：iamsashank09/llm-wiki-kit、lucasastorian/llmwiki、nashsu/llm_wiki、green-dalii/obsidian-llm-wiki、MehmetGoekce/llm-wiki、swarajbachu/cachezero、Ar9av/obsidian-wiki、vanillaflava/llm-wiki-skills、Anton15K/llm_wiki_agent、axoviq-ai/synthadoc、noirblue/IsaacCLupus_mnemosyn_spec、NicoBleh/secure-llm-wiki、skyllwt/AutoSci(arXiv:2605.31468)、markhuangai/dense-mem、MarcoPorcellato/matryca-plumber、kytmanov/synto。

**Hacker News**：item 47640875（主帖）、47656181、47667723、47750193、47899844。

**知识编译与 LLM×IR 理论**：Darwiche & Marquis "A Knowledge Compilation Map", JAIR 17 (2002), pp.229–264, DOI 10.1613/jair.989, arXiv:1106.1819；Fargier & Marquis "Extending the Knowledge Compilation Map"(AAAI 2008)；"Can LLMs Understand Intermediate Representations?" arXiv:2502.06854(ICML 2025)；"LLM Translation of Compiler IR"(IRIS-14B) arXiv:2605.08247；Meta LLM Compiler arXiv:2407.02524；TIC arXiv:2402.06608。

**模型坍缩与数据集**：Shumailov et al. "AI models collapse when trained on recursively generated data," Nature 631(8022):755–759 (2024), DOI 10.1038/s41586-024-07566-y；Penedo et al. "FineWeb" arXiv:2406.17557。

**分析/评论**：Anand Lahoti "The Hidden Flaw"(Medium)；jonadas "Cognitive Governance"；Sathish Raju "RAG Isn't Dead. But Something Is."(Medium)；WenHao Yu Zettelkasten 对比；particula.tech / starmorph / proudfrog / innobu 指南；vector DB vs KG / GraphRAG 2026 综述（Atlan、Neo4j、Glean）。

> ⚠️ 待补充：(1) Karpathy X 后续回复全文（依据二手转述与可见推文）；(2) CS146S 官方逐周 slide 一手细节（网站为 JS 渲染，依据多份公开总结交叉印证，周次编号或略有出入）；(3) SciAtlas 定量基准（论文明确尚未提供，仅有"2 分钟内完成检索"的效率声明）。