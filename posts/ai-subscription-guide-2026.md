# 过去半年我把主流大模型买了个遍，这是踩坑后的续费建议

> **作者**：卡兹克  
> **标签**：`AI` | `技术分享`

---

上个月底我拉了一下自己的账单，发现为了尝鲜和干活，不知不觉同时开着 Claude Pro、ChatGPT Plus、Google AI Pro，一个月扣掉一千多块钱。用着用着我发现，很多钱其实花得很冤——各家限流机制完全不同，适合的场景也大相径庭，自己根本没法把每个并发额度都吃满。

这篇就是把我这半年真金白银掏钱试下来的经验总结出来，每家至少聊两个档位，重点说清楚钱到底花在哪、什么场景值得买、什么档位没必要硬上。

---

## 一、各家套餐底牌（含两档对比）

### 1. OpenAI ChatGPT：Plus $20 vs Pro $100~$200

**Plus（$20/月）**

当前主力是 GPT-5.6 系列（代号 Sol / Terra / Luna），GPT-6 Astra 正在向 Plus 用户灰度开放。普通对话、写稿、改代码够用，Canvas 功能做脚本文案修改很顺手——可以划定某段让它单独改语气，不影响全文结构。

推理模型（o 系列）有每周次数上限，如果主要用它做高难度数学推导或算法攻坚，很容易一两天就触发限制。语音模式在这个价位依然是行业内拟真度最高的，做视频旁白试音很好用。

**Pro（$100~$200/月）**

本质是购买最高集群抢占优先级和更大的推理模型配额。GPT-6 Astra 的深度任务、长代码审查对 Pro 用户更稳定开放。社区的普遍结论是：除非每天要跑复杂推理任务且频繁撞墙，普通人订 Pro 是性能过剩，Plus 对 95% 的日常使用者已经足够。

---

### 2. Anthropic Claude：Pro $20 vs Max $100/$200

**Pro（$20/月）**

跑的是 Claude 5 系列，日常主力是 Claude Sonnet 5，复杂 Agent 任务用 Opus 5，Fable 5.1 处理极端推理场景。Claude 4 系列已于今年 6 月全面退役，不再可用。

代码重构和多文件逻辑推演上，Claude 5 Sonnet 在开发者社区里口碑最稳。Artifacts 功能可以在浏览器沙箱里直接预览 React 组件和 SVG 图表，不用来回切编辑器。

值得注意的是，Claude Pro 采用 5 小时滑动窗口限流，计费看的不是发了多少条消息，而是累计塞进去了多少 Token。对话框里贴了 3 万 Token 的项目代码，你问一句"改"，系统也按 3 万多 Token 计。连着改 5 次就突破 15 万，这就是为什么改复杂代码时账号容易锁。另外，之前给 Pro 用户延续了近两个月的 50% 使用量加赠于 9 月 14 日正式到期，基础配额上调 25% 作为补偿，建议续费前先跑一周看实际消耗量。

**Max 5x（$100/月）/ Max 20x（$200/月）**

专为重度 Claude Code 用户设计。5x 提供 5 倍于 Pro 的配额，20x 是 20 倍。如果每天要跑大型代码仓库的自动化重构或 CI 流程里嵌了 Agent 任务，Max 能避免频繁断流。建议先用 Pro 跑一个月，真的每周都在等 5 小时窗口刷新，再升 Max 5x。Max 20x 基本是团队级需求。

---

### 3. Google AI：Pro $19.99 vs Ultra $99.99~$199.99

**Google AI Pro（$19.99/月）**

今年 Google I/O 之后把之前的 Google One AI Premium 改了名字，统一叫 Google AI Pro。底层跑 Gemini 3.x 系列，上下文窗口 100 万到 200 万 Token，附送 2TB 云存储。

2TB 云盘单独买要 $9.99/月，所以这个套餐相当于花 $10 买了旗舰大模型。对于每天重度使用 Gmail 和 Google Docs 的人，AI 直接内嵌在侧边栏，整理会议纪要不用切标签。在长文档批量处理这件事上，Gemini 是目前唯一能把十几篇 PDF 一起塞进去处理的模型，召回质量很稳。代码生成偶尔偷懒省略细节，这是老问题。

**Google AI Ultra（$99.99~$199.99/月）**

使用量上限是 Pro 的 5 到 20 倍，云存储 20TB~30TB，另外可以优先访问 Gemini Spark 和高级视频生成能力。对个人用户而言，Pro 的配额已经够宽松，Google AI 极少有限流中断。Ultra 更多是给团队或内容生产线用的。

---

### 4. Kimi：Andante ¥49 vs Allegretto ¥199

**Kimi Andante（¥49/月）**

最低档，适合日常长文档阅读和轻度任务。9 月 11 日 Kimi 上线了 K2.8 Preview，**最值得关注的是：1M 上下文权限现在对所有会员档位全部开放，包括 Andante**。原本只有高档用户才能用的百万 Token 窗口，现在 49 块钱就能用上。K2.8 支持 low / high / max 三档思考强度，普通写作和文档整理用 low，复杂任务开 high。

**Kimi Allegretto（¥199/月）**

旗舰级 Kimi K3（2.8 万亿参数 MoE 架构）从 Moderato（¥99）档起就能访问，Allegretto（¥199）的区别在于更高并发优先级、生成队列更快，以及 K3 配合百万上下文的完整权益。K3 支持图片和视频输入，长程代码任务和多 Agent 协作上表现稳，但纯英文复杂架构推演比 Claude 5 稍弱。

中文公文、政策解读、深度研究这几块是 Kimi 的主场。"深度研究"功能会自动并发几十次搜索并交叉验证，最后输出带引注的长文，写竞品分析或行业调研用这个省大量时间。建议先买 Andante 体验一个月，用量紧张再升。

---

### 5. 智谱清言 GLM：Coding Lite ¥118 vs Coding Pro ¥538

**GLM Coding Lite（¥118/月）**

周积分制，每 5 小时刷新一次。**工作日 14:00~18:00 以外的时段积分消耗打 5 折**，晚上和周末写代码变相翻倍额度。通过 MCP 协议直连 Cursor、Cline、Claude Code 等 20 多款工具，国内直付、合规发票，底层跑 GLM-5.x 旗舰系列。公司不让报销海外订阅的团队，这是目前最省事的国内替代路线。

**GLM Coding Pro（¥538/月）**

每周积分是 Lite 的 6 倍，高峰时段有优先资源保障。Lite 用户高峰期可能需要排队，Pro 用户基本不受影响。如果团队白天业务时间要密集跑 Agent 任务，Pro 的高峰保障才有实际意义。个人开发者建议从 Lite 起步，跑一个月看积分消耗节奏再决定要不要升。

---

### 6. OpenCode Go：Go $10 vs BYOK 免费

**OpenCode Go（$10/月）**

聚合当前开源界主力代码模型（DeepSeek、Kimi K3、GLM-5.x、Qwen 等），统一提供一个 API key，直接接进 OpenCode CLI：

```bash
opencode auth login
opencode connect --provider opencode-go
```

$10 月费折算的实际 Token 价值约是订阅费的 6 倍。对于习惯在终端跑自动化 Agent、不想到处充 API 的开发者，这是最省事的统一入口。限制是只有命令行和代码环境，没有图文生成和独立 Web 聊天界面。

**BYOK（Bring Your Own Key）免费模式**

OpenCode 本身开源免费，可以自带各家的 API key 按量付费。已经在某几家充了 API 余额的用户，或者想用 Go 套餐以外特定模型的用户，BYOK 更灵活。两个模式可以随时切换。

---

### 7. Ollama Cloud：Pro $20 vs Max $100

**Ollama Cloud Pro（$20/月，含 $60 使用额度）**

8 月底切换成了新的计费模型：月订阅包含额度池，超出后按 Token 后付费，不强制升档。Pro 档每月给 $60 的额度，跑 Qwen、Llama、DeepSeek 等开源权重模型。

核心卖点是 Zero Data Retention：服务端推理完成后即释放上下文，不记录 Prompt，不参与任何模型训练。需要用开源模型但不想自己管 GPU 硬件的团队，这是目前最省事的合规路线。

**Ollama Cloud Max（$100/月，含 $300 使用额度）**

额度是 Pro 的 5 倍，适合同时跑多个 Agent 流程的团队。没有正式 SLA 保障，不适合作为生产核心链路的唯一依赖，但用作开发和测试环境的云端开源推理平台完全够用。

---

## 二、四类人的直接建议

**程序员与技术团队**

- 追求逻辑质量：**Claude Pro（$20）**，搭配 Claude Code 终端，重构和架构推演手感最好。注意 5 小时窗口机制，长代码任务多开新窗口，减少单会话积累。
- 需要高强度不断流：升 **Claude Max 5x（$100）**，或接受偶尔切换 GLM 补充。
- 国内合规直连、要开票：**GLM Coding Lite（¥118）**，晚上写代码积分打折，接 Cursor 稳定。
- 纯命令行 Agent 流：**OpenCode Go（$10）**，一个 key 打通主流开源代码模型。
- 用开源模型但不想管硬件：**Ollama Cloud Pro（$20）**，数据不留存。

**白领办公族**

- 国内日常：先买 **Kimi Andante（¥49）**，百万上下文现在已经开放了，不够用再升 Allegretto（¥199）。
- 重度 Google Workspace 用户：**Google AI Pro（$19.99）**，邮件文档直接嵌入，附送 2TB 云存储，综合算下来纯模型成本只要 $10。

**多媒体与内容创作**

**ChatGPT Plus（$20）**。Canvas 改分镜文案不破坏全文格式，DALL-E 在同一对话里调图，高级语音模式做旁白试音。GPT-6 Astra 正在向 Plus 灰度，创意类任务质量持续提升。

**学生与科研人员**

**Google AI Pro + Kimi Andante（¥49）**。十几篇 PDF 塞进 Gemini 列对比表，标注每篇的数据集、指标、优缺点，没提到的写"未提供"。国内中文文献用 Kimi 补。引用文献必须自己核验原文 DOI，不要直接用模型输出的参考文献。

---

## 三、3 条实操建议

1. 不要在同一个对话里堆几个月的上下文。越聊越长，模型越容易答非所问。一个任务单独开新窗口，常用提示词写进 Claude Projects 或 ChatGPT Custom Instructions 固化。

2. 海外服务不要走代充。Claude 和 OpenAI 对黑卡和共享号的风控极严，用了两三天就被封是常见情况，钱基本追不回来。用外区 Apple ID 礼品卡走 iOS 内购是最稳的路子。

3. 公司敏感代码不要直接贴进网页端。不是所有中文界面的 AI 工具都在国内服务器跑，处理私有源码和财务数据，要么用有明确数据合规承诺的服务，要么走 Ollama Cloud 的零留存协议。

---

## 四、30 秒对号入座

| 场景 | 推荐 |
| :--- | :--- |
| 写高质量代码、重构、Claude Code 终端 | Claude Pro（重度用户升 Max 5x） |
| 国内合规开发、Cursor 直连、要开票 | GLM Coding Lite |
| 命令行 Agent、不想管多家 API | OpenCode Go |
| 开源模型不留数据、不想买卡装机 | Ollama Cloud Pro |
| 全能：生图+改稿+语音 | ChatGPT Plus |
| 大批文献吞吐、Google 全家桶 | Google AI Pro |
| 国内中文公文、行业深度调研 | Kimi Andante 起步 |
| 高强度 K3 长程代码或 Agent 任务 | Kimi Allegretto |

---

## 五、各家官网直达

订阅前建议直接去官网看当前价格，各家调整套餐不打招呼，下面的链接都是订阅/定价页面：

| 平台 | 套餐/定价页面 |
| :--- | :--- |
| **OpenAI ChatGPT** | https://openai.com/chatgpt/pricing |
| **Anthropic Claude** | https://claude.ai/upgrade |
| **Google AI（Gemini）** | https://gemini.google/subscriptions |
| **Kimi（月之暗面）** | https://kimi.moonshot.cn（登录后左下角「会员计划」） |
| **智谱清言 GLM** | https://open.bigmodel.cn/pricing（Coding Plan 在天猫官方旗舰店也有入口） |
| **OpenCode Go** | https://opencode.ai/go |
| **Ollama Cloud** | https://ollama.com/pricing |

> 国内用户注意：OpenAI、Claude、Google 的订阅页面需要科学上网，且对 IP 纯净度有要求。Kimi 和智谱清言国内直连，微信/支付宝直接付款，智谱还支持开增值税发票。

---

*各家套餐调整很频繁，文中价格和模型以 2026 年 9 月第二周为准，买之前直接查官网当前页面。有踩坑或好用的经验，评论区聊聊。*