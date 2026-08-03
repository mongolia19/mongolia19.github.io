# Graph Engineering:为 Agent 画一张"工作路线图"

> **作者**：卡兹克  
> **标签**：`Graph Engineering` | `LLM` | `Agent` | `LangGraph` | `Multi-Agent`

---

> 当你的 Agent 不再是一个循环,而是一组彼此交接工作的节点,**Graph Engineering** 就该上场了。本文基于 [@sairahul1](https://twitter.com/sairahul1) 的范式分层( Prompt → Context → Harness → Loop → Graph)和 [aibuilderclub 2026 年 7 月的实战指南](https://www.aibuilderclub.com/blog/graph-engineering-guide-2026),系统讲清楚这个最近开始流行的新概念到底是什么、不是什么、什么时候该用、怎么动手。

---

## 一、一句话定义

> **Graph engineering is the practice of designing the graph your agents run in: which specialized nodes exist, which edges route work between them, and what shared state travels along those edges.**
>
> (Graph Engineering 是一种"为 Agent 设计运行图"的实践 —— 你设计哪些专门节点存在、哪些边在节点之间路由工作、什么共享状态沿边流动。)

它的根本目标,是把"多个 agent / 多步骤协作"这件事,**像画组织架构图一样**画清楚。

---

## 二、和一个常见误会的区分:Graph Engineering ≠ GraphRAG

读到"Graph Engineering"四个字,很多人(包括半年前的我)第一反应是把它和 Microsoft GraphRAG、Neo4j、知识图谱、LLMGraphTransformer 那一套画等号。这是错的。

原文说得非常直白:

> **Those are about modeling _data_ as entities and relations for retrieval. Graph engineering is about modeling _execution_ — which agent runs next and what state it gets.**
>
> (GraphRAG/知识图谱那一套把"数据"建模成实体和关系用于检索;Graph Engineering 把"执行"建模成节点和边 —— 下一个跑哪个 Agent、传什么状态。)

| 范式 | 关心什么 | 节点是啥 | 边是啥 |
| --- | --- | --- | --- |
| GraphRAG / 知识图谱 | 数据建模 | 实体(人、公司、概念) | 实体间关系(WORKS_AT、ACQUIRED) |
| **Graph Engineering** | **执行建模** | **专门的 Agent / 步骤** | **Agent 间的工作路由 + 状态传递** |

记住:**前者是给 LLM 看的知识,后者是给 LLM 排的班。**

---

## 三、五层堆叠:Prompt → Context → Harness → Loop → Graph

理解 Graph Engineering 最好的方式,是把它放进过去三年 LLM 工程范式的演进轴。注意这五层**是堆叠关系,不是替代关系**:

| # | 层级 | 你在工程化什么 | 核心问题 |
| --- | --- | --- | --- |
| 1 | **Prompt Engineering** | 单条请求 | 我问得清楚吗? |
| 2 | **Context Engineering** | 模型"看到"什么 | 它有正确的信息吗? |
| 3 | **Harness Engineering** | 工具、记忆、脚手架 | 它能对世界动手并记住吗? |
| 4 | **Loop Engineering** | 一个 Agent 内部的重复循环 | 它什么时候检查自己的活、什么时候停? |
| 5 | **Graph Engineering** | **多个 Agent / 步骤之间的协作** | **谁干什么、按什么顺序、共享什么状态?** |

原文点出一个关键事实:

> "**The useful thing about the stack is that it's cumulative, not a ladder you climb _away_ from.** A graph is full of nodes; a good node is a well-designed loop; a good loop needs a real harness — the six components that make an agent able to act at all. Skip a lower layer and the graph on top just fails in a more elaborate way."
>
> (这套栈的关键是它是累积的,不是越爬越远离模型的梯子。Graph 里满是 Node;一个好的 Node 是个好的 Loop;一个好的 Loop 需要一个真的 Harness —— 让 Agent 能动起来的六个组件。跳掉下面任何一层,上面的 Graph 就只是换了一种更复杂的方式失败。)

这也是我之前一版文章犯的错:把"Prompt → Context → Graph"当成范式轴。**实际是五层,不是三层**;而且 Harness、Loop 都比 Graph 更早该被掌握。

---

## 四、什么时候该用 Graph?(一个很反直觉的答案)

原指南第一句话是:

> "**The default answer is: you probably don't.** A single well-scoped task with a clear verifier is a loop, and reaching for a graph there is pure overhead."
>
> (默认答案:你大概率不需要 Graph。范围清晰、有明确验证器的单任务就是个 Loop,这时候用 Graph 是纯粹的浪费。)

判据表 —— 这些是"触发条件",不是"打勾清单":

| 信号 | Loop 够用 | 该用 Graph |
| --- | --- | --- |
| **任务形状** | 一个有清晰终点的任务 | 拆成若干独立专长,需要交接 |
| **并行度** | 步骤是顺序的 | 需要 fan-out(并发)再 join(汇合) |
| **每步的工具/模型** | 全程同一套工具/模型 | 不同步骤用不同模型或工具集 |
| **控制流** | 一个 Agent 可以安全放养 | 需要显式、可审计的"角色路由" |
| **失败隔离** | 一步错了重试就行 | 希望"一个坏节点"不污染其他节点 |
| **谁来验证** | Agent 自检自己的 Loop 输出 | **专门的审稿节点**(通常是另一个模型)检查别人产出 |

判断标准用一句话总结:

> "**The difference is who decides the path, the agent or you.**"
>
> (区别在于:路径由谁决定 —— Agent 自己,还是你。)

---

## 五、两个正反案例:什么样的 Graph 是浪费,什么样的值得建

### 🚫 过度工程的 Graph(你其实不需要)

> "Summarize this PDF." 你建了一个 5 节点 Graph:fetcher、chunker、summarizer、reviewer、formatter,带条件边和共享 state。能跑 —— 但比"一个 Agent 读文件、写摘要"**慢、难调试、贵**。**你用一个组织架构图来回复了一封邮件。**

### ✅ 恰到好处的 Graph(它真的有用)

> "Produce a researched, fact-checked market brief every morning."
> - researcher 节点并行 fan-out 五个数据源
> - synthesizer 节点汇总
> - writer 节点起草
> - skeptical reviewer 节点(不同模型、只读)打分,不过就打回
>
> 每个节点都做着"一个 Loop 干不来"的活,交接本身就是价值。

判据原文:

> "**The tell is whether the graph is _doing work the loop couldn't._** If you can collapse your five nodes back into one agent's loop and lose nothing, you should."
>
> (判断信号在于:这张图是否在做"Loop 做不了"的活。如果你能把五个节点合并回一个 Agent 的 Loop 而毫无损失,那就别建图。)

---

## 六、工具栈(2026 年 7 月)

这些框架在"Graph Engineering"这个词流行之前,就已经把"节点 + 边 + 共享状态"这套做出来了:

| 框架 | 出品方 | 定位 |
| --- | --- | --- |
| **[LangGraph](https://langchain-ai.github.io/langgraph/)** | LangChain | "a low-level orchestration framework and runtime for building, managing, and deploying long-running, stateful agents" —— 你定义 `StateGraph`,加节点,加边 |
| **[AutoGen GraphFlow](https://microsoft.github.io/autogen/)** | Microsoft | AutoGen 的图式多 Agent 编排:描述一组 Agent 怎么连接和交接 |
| **[Google ADK](https://google.github.io/adk-docs/)** | Google | 把"图式架构"作为头条特性,内置 sequential、parallel、loop 三类 workflow agent 和 agent routing |
| **[A2A](https://github.com/a2a-protocol/A2A)** | 开源协议 | Agent2Agent,跨系统、跨团队的 Agent 之间"图与图的边" |

关于"恭喜你重新发明了 LangGraph"这条梗,原文给的回答是:

> "**The technology, largely yes — LangGraph, GraphFlow, and ADK got there first. What's actually new in mid-2026 is narrower and softer: a _shared name_ for the design decisions those frameworks always asked of you.**"
>
> (技术上,是的 —— LangGraph、GraphFlow、ADK 早就做出来了。2026 年中真正新的东西更窄更软:一个共同的名称,去称呼那些框架一直在让你做的设计决策。)

翻译一下:工具早就有了;Graph Engineering 这个词的真正贡献,是**给了一整套命名法和判据**,让你知道什么时候该用、什么时候不该用。

---

## 七、动手指南:把 Loop 升级到 Graph 的 8 步清单

> 原指南: "**Graph engineering is the layer _above_ loop engineering, and the fastest way to build a bad graph is to skip the loop. So the honest first move isn't 'learn graphs.' It's: nail the loop your first node will run.**"

翻译:Graph Engineering 是 Loop Engineering 之上的那一层;建出烂 Graph 最快的方式就是跳过 Loop。所以诚实的起步动作不是"学 Graph",而是**先把第一个节点要跑的 Loop 写好**。

写好 Loop 之后,真要升级到 Graph 时,过这 8 步:

1. **先问一句:能不能还是 Loop?** 单个有验证器的 Agent 能干,就停在这一步。
2. **节点命名要有"专长"**:每个节点要有一个"单 Loop 真的干不了"的活。
3. **先画边再写代码**:画清楚哪里顺序、哪里 fan-out、哪里 fan-in、那条条件/打回边在哪里。
4. **共享状态对象要显式设计**:沿边传什么、谁有权写,要先想清楚。
5. **审稿节点要有牙**:通常最有价值的节点是"另一个模型做的、只读的 verifier"。
6. **失败要隔离**:一个节点失败重试,不能污染共享状态或下游节点。
7. **选框架别手撸**:LangGraph / AutoGen GraphFlow / Google ADK 已经给了你节点、边、状态、fan-out、fan-in、loop。
8. **设花费上限和硬边界**:Graph = 很多 Loop;弱的 verifier 现在是并行烧 token。设 cap。

收尾标准:

> "If you build a graph this week, the win condition isn't 'it has the most nodes.' It's 'every node is doing work a loop couldn't, and I could still explain the whole thing in one breath.'"
>
> (如果你本周要建一张图,胜出的条件不是"它节点最多",而是"每个节点都在做 Loop 做不了的活,而且我能一口气把整张图讲清楚"。)

---

## 八、什么时候**不**该用 Graph Engineering?

来自原指南的警告:

- **任务形状单一、终点清晰** —— 单 Loop 干就行。
- **需要并行但没有"角色区分"** —— 一个 Loop 里 `asyncio.gather` 即可,不要硬上 Graph。
- **验证器不够"独立"** —— reviewer 节点用同一个 Agent 的 prompt 换皮,等于没有 verifier。
- **状态很薄或无状态** —— 没有共享 state,Graph 的最大价值就没了。

更简单的判据:**如果你能用一句话向同事讲清楚 Graph 的拓扑,那它可能合理;如果你需要画完整 ASCII 图才能讲清楚,那它大概率是过度工程。**

---

## 九、和其他概念的关系

| 概念 | 关系 |
| --- | --- |
| **Knowledge Graph / GraphRAG** | **数据建模** vs **执行建模**;前者给 LLM 看,后者给 LLM 排班。可叠加用 |
| **Agent Orchestration** | 同义近义词,但 Graph Engineering 强调"图 + 状态"的设计决策,而 orchestration 是更宽泛的词 |
| **Workflow Engine(Temporal / Airflow)** | 是 Graph Engineering 的"工业级远亲",但 Workflow 引擎通常无状态、确定性;Agent Graph 是有状态、非确定性的 |
| **Microservices / SOA** | 概念同构(节点 + 边 + 状态),但目标不同:微服务是"对外提供 API";Agent Graph 是"协作完成一个 LLM 任务" |

---

## 十、结语:五层都要会,先别跳级

Graph Engineering 在 2026 年中真正新加的东西,其实**不是技术**。LangGraph、AutoGen、ADK 早就有了。新的是:

1. 一个**共同的命名**去称呼"多 Agent 协作设计"这件事
2. 一组**判据**去判断什么时候该用、什么时候不该用
3. 一套**清单**让你在升级到 Graph 时不至于跳层

如果你现在 Agent 工程还在 Prompt 层,就**先别学 Graph**。先把 Context 做好(喂它对的信息),把 Harness 立起来(它能动手、能记住),把 Loop 调对(它知道什么时候停、什么时候打回),然后**任务本身会逼你升级到 Graph**。这时候你再回头读这篇文章,会发现原指南里那 8 步清单,每一步都对应着"前面那层没做对"的症状。

> 工程化 Agent 这件事,和建组织一样:**先把单人产出打好,再谈分工**。

---

## 参考资料

1. [@sairahul1](https://twitter.com/sairahul1) 在 X 上的"Prompt, context, harness, loop & graph engineering, clearly explained!" 推文 —— 五层堆叠范式的原始出处。
2. [AI Builder Club: Graph Engineering Guide 2026](https://www.aibuilderclub.com/blog/graph-engineering-guide-2026) —— 本文核心来源,包含定义、判据、反面案例、8 步清单。
3. [LangGraph](https://langchain-ai.github.io/langgraph/) —— LangChain 的低层 Agent 编排运行时。
4. [Microsoft AutoGen - GraphFlow](https://microsoft.github.io/autogen/) —— Microsoft 的图式多 Agent 编排。
5. [Google ADK](https://google.github.io/adk-docs/) —— Google Agent Development Kit,内置 sequential/parallel/loop workflow agents。
6. [A2A Protocol](https://github.com/a2a-protocol/A2A) —— Agent2Agent,跨系统的 Agent 协作协议。
7. [Mitchell Hashimoto / HumanLayer 关于 Harness Engineering 的论述](https://www.humanlayer.dev/blog/harness-engineering) —— 五层中"Harness"层的来源参考。