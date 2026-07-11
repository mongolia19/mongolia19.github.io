# Vercel eve：把 Agent 当成一个目录——2026 年最值得关注的开源 Agent 框架

> 在 2026 年 6 月的伦敦，Vercel 做了一次安静的战略转身：他们不再只是「构建 Web 的平台」，而是宣布成为「构建 Agent 的平台」。这次转身的技术载体，是一个叫 **eve** 的开源框架。

这篇文章不打算做发布会通稿式的复述，而是从开发者角度拆解 eve 到底做了什么、它和 LangChain / CrewAI / LangGraph 有什么区别、以及它是否值得你纳入技术选型。

---

## 一、先说结论

eve 不是一个让人眼前一亮的框架。它的技术组件——持久会话、沙箱、模型网关、渠道接入——在 2026 年单独拿出来都不算罕见，很多框架已经做得很好了。

eve 真正值得关注的，是它的**打包方式**：它用一套 filesystem-first（以文件系统为核心）的约定，把 Agent 的搭建、部署、运维全部收束到一个目录里。一个 Agent 就是一组文件，部署就是 `vercel deploy`——和你平时发 Next.js 项目没有任何区别。

这不是功能上的创新，而是**分发上的创新**。Vercel 说他们平台上超过 50% 的部署现在是由 Agent 触发的。不管这个数字有多少水分，它指向的趋势是真实的：当平台上的工作从「写网站」变成「写 Agent」，那么让 Agent 像写网站一样简单的框架，就会获得巨大的分发优势。

---

## 二、eve 是什么

eve 是 Vercel 在 **Ship London 2026**（2026 年 6 月 17 日，伦敦）上正式发布的开源 Agent 框架。Apache-2.0 许可，代码托管在 [github.com/vercel/eve](https://github.com/vercel/eve)，纯 TypeScript，以 pnpm workspace 的 monorepo 形式组织。

几个关键事实：

- **开源但处于 beta**。Vercel 明确说 eve 在 GA 之前 API 和命令可能还会变化，不要把它当作冻结版本。
- **Vercel 原生设计**。持久运行、沙箱、模型路由都依赖 Vercel 专有原语，**无法自托管**。这不是 bug，是 trade-off。
- **Vercel 自己已经在用**。内部运行了 100+ 个 Agent，一个数据分析 Agent 每月处理 30,000+ 个查询。

---

## 三、核心理念：Agent 就是一个目录

这是理解 eve 最关键的一句话。

传统的 Agent 框架需要你画一张图、写一段配置、或者在代码里注册一堆组件。eve 反其道而行：**你在 `agent/` 目录下放什么文件，eve 就读什么能力。**

具体来说：

| 目录 | 作用 | 怎么加能力 |
|------|------|-----------|
| `agent/instructions.md` | Agent 的常驻 system prompt | 改文件内容 |
| `agent/agent.ts` | 运行配置，定义模型 | 改文件内容 |
| `agent/tools/*.ts` | 一个文件 = 一个可调用的工具 | 新增文件 |
| `agent/skills/*` | 按需加载的执行手册 | 新增文件 |
| `agent/channels/*` | 接入 Slack / Discord / HTTP 等渠道 | 新增文件 |
| `agent/subagents/*` | 子 Agent，层次化编排 | 新增文件 |
| `agent/schedules/*.ts` | 定时任务 | 新增文件 |

这个设计其实非常熟悉——**它就是 Next.js 用文件系统定义路由那一招的翻版**。当年「新建一个页面 = 新建一个文件」让 React 应用的结构变得可预测，工具链、文档、部署都能基于这种可预测性做优化。eve 对 Agent 做了完全一样的事。

最小可用的 eve Agent 只有两个文件：

```
my-agent/
├── instructions.md   # system prompt，用纯 Markdown 写
└── agent.ts          # 运行配置，用 defineAgent() 设置模型
```

---

## 四、动手：10 分钟搭建一个 Agent

全程不需要安装额外的基础设施。eve 项目就是一个普通的 Vercel 项目。

### 1. 搭建项目

```bash
npx eve@latest init my-agent
```

一条命令，半分钟内生成一个完整的项目目录结构。

### 2. 编写 Agent 定义

`agent/agent.ts`：

```typescript
import { defineAgent } from 'eve';

export default defineAgent({
  model: 'openai/gpt-5.4-mini',
  // 或者用 Anthropic
  // model: 'anthropic/claude-opus-4-8',
});
```

注意模型名是 `"provider/model"` 格式，通过 Vercel AI Gateway 路由，不需要自己配 API key——在 Vercel 平台上用 OIDC 自动认证。

### 3. 写 system prompt

`agent/instructions.md`：

```markdown
# 你是一个内容运营助手

你的职责是帮助用户分析和润色文章。回答时使用中文，语气专业但不严肃。

# 约束
- 不要编造数据，引用时需要标注来源
- 超过 500 字的内容需要分段
```

这就是 Agent 的常驻人格，每次对话都会带进去。

### 4. 加一个工具

`agent/tools/summarize.ts`：

```typescript
import { defineTool } from 'eve';
import { z } from 'zod';

export default defineTool({
  description: '将一篇文章压缩成 200 字以内的摘要',
  inputSchema: z.object({
    article: z.string().describe('文章内容'),
  }),
  async execute({ article }) {
    // 你的逻辑，比如调用 LLM 或做文本处理
    return { summary: '这里是生成的摘要...' };
  },
});
```

文件名 `summarize.ts` 就是工具名——模型在对话里会看到 `summarize` 这个工具，调用时自动传参进 `execute`。

### 5. 本地运行

```bash
cd my-agent
npm run dev
```

eve 会启动一个本地服务器，默认跑在 Docker 或 microsandbox 沙箱里。

### 6. 部署到生产

```bash
vercel deploy
```

就这样。部署时：
- 沙箱自动切到 Vercel Sandbox（Firecracker 微虚拟机）
- Agent 通过公开 URL 可访问
- 正在执行的会话不中断（在旧版本上继续跑完）
- 每个 commit 自动获得 Preview 部署

### 7. 接入 Slack

```bash
eve channels add slack
```

一行命令，Slack 上就能 @mention 你的 Agent 了。

---

## 五、持久会话：eve 和 Vercel AI SDK 的根本区别

这是最容易搞混的地方。

**Vercel AI SDK** 是一个 streaming 和 UI 层，用来跟模型对话、处理流式输出。它**没有持久会话、没有 Agent 编排能力**。

**eve** 是一个持久 Agent 框架。它的会话跑在 **Vercel Workflow** 之上，核心机制是：

1. 会话的每一步（消息、工具调用、结果）都被记录成事件日志
2. 当发生冷启动、重新部署、或者长时间等待用户回复时，系统用事件日志**确定性重放**来重建状态
3. 会话不会丢，不会断，等用户回来接着跑

这意味着你可以把一个需要跑半小时的 Agent 会话丢到后台，半夜重新部署了项目，它早上醒来继续跑，完全不丢失进度。

会话是通过一个轻量 HTTP API 驱动的：

```
POST /eve/v1/session           # 启动持久会话，返回 x-eve-session-id
GET  /eve/v1/session/:id/stream # 流式读取 NDJSON 生命周期事件
```

因为每次对话都是长运行的、增量流式的，eve 默认启用 Vercel 的 **Fluid Compute**（按活跃 CPU 计费），避免为闲置会话付费。

---

## 六、沙箱与安全防护

eve 给每个 Agent 配了一个隔离的 bash 风格计算环境，有自己独立的文件系统。

- **在 Vercel 上**：跑在 Firecracker 微虚拟机里，适合处理不可信的、模型生成的命令
- **本地**：用 Docker 或 microsandbox

对于 Agent 可能做的风险操作（比如改文件、调外部 API），eve 提供了 **human-in-the-loop 审批机制**：模型调某个工具时可以被暂停——暂停期间**不消耗计算资源**——等人在 Slack 上点确认再继续。这个能力让 Agent 从「敢做不敢用」变成「用了也不慌」。

---

## 七、可观测性与评测

eve 项目在 Vercel 仪表盘上**自动获得 Agent Runs 面板**，不需要写一行 instrumentation 代码。面板展示：

- 按触发源分类的运行趋势
- Token 用量（输入 / 输出 / 缓存）
- 每轮对话的耗时、推理过程、工具调用明细

如果你想把 trace 导出到自己的后端，加一个 `agent/instrumentation.ts` 文件即可。eve 用 **OpenTelemetry 标准**，可以发到 Braintrust、Honeycomb、Datadog、Jaeger 等任何 OTel 兼容的后端。

Trace 层级：

```
ai.eve.turn                    # 每轮对话
├── ai.streamText              # 文本生成
├── ai.toolCall:<tool-name>    # 每次工具调用
└── ...
```

每个 span 上自动注入 session id、环境、轮次 id、渠道类型等上下文。

---

## 八、Vercel Ship 2026 的完整产品矩阵

eve 不是孤立发布的，它是一整套 Agent 基础设施中的一个组件：

| 产品 | 定位 | 状态 |
|------|------|------|
| **eve** | 开源 Agent 框架 | Public Beta |
| **Vercel Connect** | Agent 凭证管理，用短时效范围令牌替代长期密钥 | Public Beta |
| **Vercel Passport** | 企业级 Agent 治理与合规 | Public Beta |
| **Vercel Services** | 微服务部署，7 月 1 日 Beta | Announced |
| **Vercel Agent** | 自主调查告警、开 PR 修复的运维 Agent | Private Beta |
| **Agent Stack** | AI SDK + AI Gateway + Workflow SDK + Sandbox + Connect 的统一原语集 | 已发布 |

其中 **Vercel Connect** 解决了一个最被低估的安全问题：Agent 需要访问 Slack、GitHub、Snowflake 等外部服务，传统做法是存长期 API Key——泄露了就是灾难。Connect 用范围限定的短期令牌替代，自带审计追踪。

**Vercel Services**（7 月 1 日 Beta）让 Agent 架构能跑完整的微服务，支持内部私有网络，意味着 Agent 不只是前端一个 Bot，而是可以在后端有完整的服务拓扑。

---

## 九、框架对比：eve vs 其他

2026 年的 Agent 框架赛道已经相当拥挤了。下表从开发者选型的角度做对比：

| 维度 | Vercel eve | Mastra | LangGraph | CrewAI | Vercel AI SDK |
|------|-----------|--------|-----------|--------|--------------|
| 许可 | Apache-2.0 | Apache-2.0 | Apache-2.0 | MIT | Apache-2.0 |
| 语言 | TypeScript | TypeScript | Python 为主 | Python | TypeScript |
| 平台绑定 | Vercel 专用 | 任意平台 | 任意平台 | 任意平台 | Vercel 原生 |
| 持久会话 | ✅ Vercel Workflow | 部分支持 | ✅ Checkpoint | 部分 | ❌ |
| Serverless | ✅ 原生 | 部分 | ❌ 不兼容 | ❌ | ✅ |
| 沙箱 | ✅ Firecracker | 可选 | ❌ | ❌ | ❌ |
| 自托管 | ❌ | ✅ | ✅ | ✅ | ❌ |
| 启动门槛 | 极低 | 低 | 高 | 中 | 低 |

**选型建议**：

- **你要在 Vercel 上快速上线一个生产级 Agent** → eve，部署体验无与伦比
- **你需要平台无关、能自托管** → Mastra（TypeScript）或 LangGraph（Python）
- **你要做深度状态图、checkpoint 和 time-travel 调试** → LangGraph 仍然是最深的选项
- **你只需要流式输出 + 简单 tool call** → Vercel AI SDK 就够了，用 eve 是杀鸡用牛刀
- **你要做多 Agent 协作、Python 生态** → CrewAI 仍然有优势

---

## 十、我的判断：eve 的胜负手不在功能，在分发

坦率地说，eve 的单项技术能力在 2026 年都算不上惊艳。持久执行、沙箱、模型网关——这些别的框架都有类似方案。

eve 真正的赌注是：**让 Vercel 平台上现有的几百万开发者，写 Agent 的方式和写网站的方式完全一致**。

这个赌注成立的前提是——Vercel 报告 Agent 触发的部署从不到 3% 涨到了超过 50%，而且还在涨。如果这个数字属实，那么 eve 不靠功能优势也能赢，因为它站在最大的 TypeScript 开发者分发的入口上。

但也要看到硬币的另一面：

- **平台锁定是真实的**。你不能把 eve 项目搬到 AWS 或 GCP 上跑，持久运行和沙箱都依赖 Vercel 专有原语
- **它还处在 beta**。API 可能变，命令可能变，文档还不完整
- **Vercel 自己报的数据，需要打折扣看**。100+ 内部 Agent、50% 部署比例都是自述，没有独立验证

---

## 十一、该不该试？

我的建议是：**试，但不要现在就押注**。

1. 用 `npx eve@latest init` 搭一个最小 Agent，感受一下 filesystem-first 的体验
2. 在本地跑几个 eval，测一下真实成本和延迟
3. 如果你们团队已经在用 Vercel，可以拿一个低风险场景（比如内部 Slack Bot）试点
4. 但如果这是你唯一在选型的 Agent 框架，同时调研 Mastra 和 LangGraph——**平台绑定是一个需要在架构层面决策的风险**

---

## 总结

eve 不是最强大的 Agent 框架，但它可能是**最容易让一个团队在一天之内上线一个生产级 Agent 的框架**。它的文件系统优先范式、Vercel 原生部署、持久会话和沙箱，组合在一起是一个「低摩擦」的体验。

2026 年的 Agent 框架之争，最终可能不是技术能力的对决，而是**分发表决的胜负**。Vercel 手握最大的 TypeScript 开发者生态，eve 是它打出的那张牌。

值得玩味的是，eve 的名字本身就是个隐喻——在《黑客帝国》里，eVE 是 Neo 和 Trinity 的朋友，一个连接者和信使。也许 Vercel 想说的是：eve 不是 Agent 本身，而是让 Agent 和平台连接的那个接口。

---

*文章基于 2026 年 6 月 Vercel Ship London 发布材料、[github.com/vercel/eve](https://github.com/vercel/eve) 开源代码、以及 Digital Applied、The Register、The New Stack、博客园等来源的独立技术分析撰写。*
