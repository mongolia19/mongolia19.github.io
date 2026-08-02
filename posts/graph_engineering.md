# Graph Engineering:当 AI 工程师开始像数据工程师一样构建知识图谱

> **作者**：卡兹克  
> **标签**：`Graph Engineering` | `LLM` | `知识图谱` | `RAG` | `GraphRAG`

---

> 你可能已经熟悉了 Prompt Engineering、Context Engineering，那么 **Graph Engineering** 是什么？为什么大厂和前沿团队都在悄悄把它列为 2025-2026 年最值得投入的工程范式之一？本文结合一份 YouTube 上的英文科普视频，并补充 GraphRAG、LangGraph、Microsoft GraphRAG 等一线实战资料，系统梳理这个新概念的定义、动机、技术栈、典型流水线以及落地挑战。

---

## 一、从 Prompt 到 Graph：LLM 工程的演进轴

要理解 Graph Engineering，最好先把它放在过去三年 LLM 工程范式演进的轴线上看：

| 范式 | 关注对象 | 核心动作 | 典型产物 |
| --- | --- | --- | --- |
| Prompt Engineering | 单条提示词 | 调措辞、加示例、加 CoT | 一组能跑出好答案的 prompt |
| Context Engineering | 喂给模型的全部上下文 | 检索、压缩、注入工具、记忆管理 | 一个能稳定产出可信答案的 RAG/Agent 系统 |
| **Graph Engineering** | **结构化的知识图谱本身** | **设计本体、抽取实体关系、对齐融合、推理查询** | **一张可演化的领域知识图谱 + 查询接口** |

一句话区分：

- **Prompt Engineering** 在调"问题怎么问"。
- **Context Engineering** 在调"上下文怎么喂"。
- **Graph Engineering** 在调"知识本身怎么长出来、长成什么形状、怎么被检索"。

GraphRAG、Neo4j + LangChain、Microsoft 的 GraphRAG 库、ESWC 2024 上的"自配置 KG 构造流水线"，本质上都是 Graph Engineering 的具体实践。

---

## 二、为什么 2024-2026 年 Graph Engineering 突然重要？

三个推动力同时到位：

1. **LLM 的"幻觉天花板"**：再强的模型也会瞎编。RAG 用检索缓解了，但传统向量 RAG 在跨文档、跨实体、跨时间的关系推理上仍然弱——"2020 年 A 公司收购 B 公司之后，B 的 CEO 是谁？" 这个问题纯向量召回通常答不对。
2. **企业知识资产化诉求**：内部 FAQ、合同、Wiki、工单散落在几十个系统里，老板想要的不是"一个聊天机器人"，而是"一张能查、能分析、能演化的知识地图"。知识图谱（KG）是最自然的形态。
3. **LLM Agent 把 KG 的构造门槛打下来了**：以前建一张企业 KG 需要 NLP 团队写规则、写模板、跑 ETL；现在 LLM 抽实体关系、社区检测算法自动聚类，几小时就能出一张百万节点的 KG。Microsoft 的 GraphRAG 论文 *From Local to Global: A Graph RAG Approach to Query-Focused Summarization* 就是这一波的标志性工作。

也就是说：**LLM 让"建图"变得便宜，"用图"的需求又真实存在**——Graph Engineering 应运而生。

---

## 三、Graph Engineering 的标准流水线

不管你用 Neo4j + LangChain、LlamaIndex + 微软 GraphRAG，还是自研流水线，**Graph Engineering 的核心阶段基本一致**。下面给一个工业界常见的 7 段式：

```
[原始数据]
   ├─ 结构化（DB / API / 表格）
   └─ 非结构化（文档 / 工单 / 邮件 / IM）
            │
            ▼
   ① 本体设计 Ontology Design
            │   ← 业务专家 + LLM 协作
            ▼
   ② 实体-关系抽取 Triple Extraction
            │   ← LLMGraphTransformer / LLMPathExtractor
            ▼
   ③ 实体对齐 / 消歧 Entity Resolution
            │   ← 指代消解 + 向量相似度合并
            ▼
   ④ 嵌入与索引 Embedding & Indexing
            │   ← BGE-M3 / text-embedding-3-small 等
            ▼
   ⑤ 社区检测与摘要 Community Detection & Summarization
            │   ← Leiden / Louvain 算法 + LLM 摘要
            ▼
   ⑥ 图存储 Graph Storage
            │   ← Neo4j / Memgraph / NebulaGraph / TigerGraph
            ▼
   ⑦ 查询与推理 Query & Reasoning
            ← 自然语言 → Cypher → 子图 → LLM 生成答案
```

每一段都对应可工程化的组件，下面展开几个最关键的。

### 3.1 本体设计（Ontology Design）

KG 不是把所有节点都扔进去——**先定形状，再灌数据**。

- 业务侧：定义实体类型（Person、Company、Product、Ticket）、关系类型（WORKS_AT、ACQUIRED、REPORTED_BY）、属性约束。
- 技术侧：可以由 LLM 辅助生成初版本体，再由领域专家审核、合并、剪枝。
- 经验法则：**宁少勿多、可演化**。本体一旦定下，后续抽取和查询都依赖它；过度设计会拖垮工程进度。

### 3.2 实体-关系抽取（Triple Extraction）

这是 Graph Engineering 最 LLM-heavy 的一段。两种主流路径：

- **LangChain `LLMGraphTransformer`**：把文档分块，让 LLM 按本体 schema 输出 `(head, relation, tail)` 三元组。
- **LlamaIndex `LLMPathExtractor` / `SimpleLLMPathExtractor`**：在节点遍历过程中提取路径型三元组，适合更长文档。

提示词设计是关键：

- 必须强制 LLM 输出**结构化 JSON**（最好带 schema 校验）。
- 必须显式告知允许的实体类型和关系类型，避免幻觉出新概念。
- 批量化时要注意 token 预算——一篇 10k 字文档，LLM 抽完可能要几千 token。

### 3.3 实体对齐 / 消歧（Entity Resolution）

抽取出来的实体经常出现：

- 同名异指：两个 "Apple" 一个是水果一个是公司。
- 异名同指：同一个人在不同文档里被叫 "张博士"、"张老师"、"老张"。

解法：

- **基于向量**：用 BGE-M3、text-embedding-3-small 等多语种 embedding 算相似度，把相似度高于阈值的合并。
- **基于规则**：人名加正则匹配、公司名加工商注册号匹配。
- **基于 LLM**：把候选对喂给 LLM 判定是否合并，准确率高但成本也高。
- 这一步直接决定 KG 的"噪音水平"——垃圾进，垃圾出。

### 3.4 嵌入与索引（Embedding & Indexing）

实体和关系都需要向量化，方便后续做：

- 相似实体查找（消歧用）。
- 自然语言到实体的语义检索（用户问"我们公司的 CEO 是谁" → 找到 Person 节点）。
- 混合检索中的向量召回。

中文场景下 **BAAI/bge-m3** 几乎是默认选择（多语言、多粒度、支持 8k 输入），英文场景 text-embedding-3-small 性价比最高。

### 3.5 社区检测与摘要（Community Detection & Summarization）

这是 Microsoft GraphRAG 的招牌环节：

- 用 **Leiden / Louvain** 等算法把整张图切成层级化的社区（hierarchy of communities）。
- 每个社区用 LLM 单独生成一段摘要（community summary）。
- 高层查询（"我们公司过去一年的战略主线是什么"）→ 走社区摘要；低层查询（"张博士去年写过哪些文档"）→ 走实体邻居遍历。

这一步让 KG 从"一张图"变成"一个分层索引"，支持从宏观到微观的多粒度问答。

### 3.6 图存储（Graph Storage）

主流选型：

| 数据库 | 适合场景 |
| --- | --- |
| **Neo4j** | 工业界事实标准，Cypher 生态成熟，社区版免费 |
| **Memgraph** | 内存图，适合实时 / 流式场景 |
| **NebulaGraph** | 国产开源，超大规模分布式 |
| **TigerGraph** | 企业级，分析型查询极快 |
| **Kùzu** | 嵌入式，轻量级，适合单机原型 |

选型要素：数据量级、查询模式（OLTP 还是 OLAP）、是否需要分布式、生态成熟度。

### 3.7 查询与推理（Query & Reasoning）

最后一公里：

1. **NL2Cypher**：用户问自然语言 → LLM 生成 Cypher → 执行 → 结果 → LLM 写成自然语言答案。
2. **Hybrid Retrieval**：向量召回 + 图遍历 + 关键词搜索三路融合。
3. **Agentic Traversal**：让 LLM Agent 自己决定走哪条路径（DRIFT 搜索就是这种思路）。

---

## 四、和相关领域的关系

Graph Engineering 不是凭空冒出来的，它站在多个老领域的肩膀上：

| 相关领域 | 关系 |
| --- | --- |
| **传统知识图谱（KG）** | 基础学科，提供本体论、Cypher、推理规则；Graph Engineering 是 LLM 时代的"复活版" |
| **图神经网络（GNN）** | 算法侧邻居，给 KG 提供节点分类、链接预测等能力 |
| **RAG（检索增强生成）** | GraphRAG 是 RAG 的"图增强版"，用图遍历代替 / 补充向量检索 |
| **LangGraph / Agent** | 工程侧编排，用状态图来驱动多步检索与工具调用 |
| **Data Engineering** | 流程相似（采集 → 清洗 → 建模 → 服务化），但建模对象从表换成了图 |
| **Context Engineering** | 上游：Context Engineering 喂给 LLM 的"上下文"经常就来自一张 KG |

**最容易混淆的两个概念**：

- **Graph Engineering（本文主题）**：构建并运营知识图谱的工程范式。
- **GraphRAG**：用 KG 改进 RAG 的一种具体技术方案，可以理解为 Graph Engineering 的一种应用形态。

---

## 五、典型工具栈（2026 年实战版）

按"最小可用"到"工业级"排序：

**入门版（单机原型）**

- LlamaIndex + Neo4j + LangChain `LLMGraphTransformer`
- 一个 Python Notebook 跑通端到端

**进阶版（小团队生产）**

- Neo4j + LangGraph + BGE-M3 + Leiden
- 加社区检测 + 实体消歧
- 用 LangGraph 编排 NL2Cypher + 多轮对话

**工业级（企业级生产）**

- 微软 GraphRAG 库 + NebulaGraph + 自动化本体演化
- 增量更新 pipeline（Kafka → 抽取 → 对齐 → 入图）
- 监控：节点/边数量、抽取成功率、社区稳定性、查询 P99 延迟

**关键开源项目**

- [microsoft/graphrag](https://github.com/microsoft/graphrag)
- [neo4j/neo4j-graphrag-python](https://github.com/neo4j/neo4j-graphrag-python)
- [neo4j-graphacademy/llm-knowledge-graph-construction](https://github.com/neo4j-graphacademy/llm-knowledge-graph-construction)
- [langchain-ai/langgraph](https://github.com/langchain-ai/langgraph)

---

## 六、落地挑战：现实没有 PPT 那么顺

Graph Engineering 在 PoC 阶段都很漂亮，上生产会撞到几个常见问题：

1. **本体的"熵增"**：业务一变，节点/关系就要跟着变。**本体版本管理**比想象中难。
2. **抽取的幻觉成本**：LLM 抽错了几个实体，可能污染整张图。必须有**人工审核回路**或 LLM-as-a-judge 自检。
3. **增量更新的死锁**：增量抽取很容易产生"中间态"——一个三元组加进去破坏了上下文，需要回滚。常见解法是引入**事务图（temporal graph）**。
4. **查询体验分裂**：自然语言查询的 hit rate 远低于向量 RAG。LLM 生成的 Cypher 经常语法错、效率差。需要 **prompt 缓存、few-shot 示例库、Cypher 校验器**。
5. **成本**：百万级文档抽三元组，token 费用可以轻松烧掉几千美元。需要**分块策略 + 抽样 + 缓存**。
6. **评估难**：KG 质量不像分类任务有标准指标。常用做法是**问答命中率 + 实体准确率 + 社区稳定性** 三维度评估。

---

## 七、和 Adobe Project Graph 等"可视化工作流"的关系

你可能听过 Adobe Project Graph——它把 AI 创作流程做成可视化节点图，让设计师能编辑每个节点。这是**"图作为用户界面"**。

Graph Engineering 的"图"则是**"图作为数据结构和推理底座"**。

两者形态相似（都是节点+边），但目标不同：

- Project Graph 是给人类看的执行流；
- Graph Engineering 是给 LLM 推理用的知识底座。

未来这两者很可能融合：用可视化图编辑器直接构造企业 KG，下游自动接到 LLM Agent。

---

## 八、入门建议：怎么开始一个 Graph Engineering 项目？

如果你想动手：

1. **选一个小而具体的领域**——比如"公司内部技术博客的知识图谱"，不要一上来就做"全公司知识库"。
2. **先用 GraphRAG 跑通 PoC**——装 Neo4j Desktop + microsoft/graphrag，一份文档跑一遍，看效果。
3. **把本体设计当作产品决策**——找业务方坐下来聊一次，定下实体和关系清单。
4. **引入人工审核环节**——前 1000 个三元组全人工 review，建立信心后再自动化。
5. **接一个真实查询场景**——比如"客户 X 的所有相关工单和文档"，把图谱和真实业务流打通。
6. **把评估写进代码**——节点/边数量、抽取命中率、查询 P99、用户满意度，每一周跑一次。

**Don't**：

- 不要一开始就用最复杂的图数据库——Neo4j 单机版够用 90% 的场景。
- 不要试图让本体一步到位——本体是迭代长出来的。
- 不要忽略数据治理——KG 一旦上线，它就变成了"事实来源"，错的图比没图更糟。

---

## 九、展望：Graph Engineering 接下来的 12-24 个月

几个值得关注的趋势：

1. **本体自动演化**：用 LLM 监控新文档，自动建议新增实体/关系类型，由人审批。
2. **多模态 KG**：图里不仅有文本实体，还有图片、表格、视频片段。CLIP 类模型会成为标配。
3. **图 + Agent 双向耦合**：Agent 既消费 KG（查实体）也生产 KG（执行后写回边）。
4. **Temporal KG 工业化**：几乎所有业务场景都涉及时间，"事件 A 之后，状态 B 如何变" 这类问题会越来越普遍。
5. **Graph Foundation Model**：类似 LLM，但专门做图推理——目前还在论文阶段，但 Neo4j、Anthropic 都在跟踪。

---

## 十、结语

Graph Engineering 不是要取代 Prompt Engineering 或 Context Engineering，而是 LLM 工程栈的**下一层基础**：

> 当你调好 prompt、设计好上下文，仍然发现模型在跨实体、跨时间、跨系统的复杂问题上答不对——你就该考虑 Graph Engineering 了。

工程化知识这件事，十年前因为太贵没人做；现在 LLM 让它变便宜了。剩下的就是把它做对、做稳、做成产品。

---

## 参考资料

1. Microsoft Research, *From Local to Global: A Graph RAG Approach to Query-Focused Summarization*, 2024.
2. ESWC 2024, *Towards self-configuring Knowledge Graph Construction Pipelines using LLMs*.
3. IEEE 综述, *Graph Large Agent (GLA): A Unified Blueprint for Complex Systems*, 2025.
4. Neo4j GraphAcademy, *LLM Knowledge Graph Construction* 课程.
5. LangChain / LlamaIndex / LangGraph 官方文档（2024-2026）.
6. *从 Prompt Engineering 到 Context Engineering*, 腾讯研究院, 2025.
7. [YouTube: Graph Engineering 科普视频](https://youtu.be/tn_I4_1yFSY).