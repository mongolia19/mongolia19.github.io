# DSpark 深度解读：DeepSeek 如何用半自回归投机解码把 LLM 推理速度推到新极限

在公司里经常有同事问我：LLM 推理这块，还有没有提效空间？毕竟从模型结构到推理框架，能折腾的似乎都被折腾遍了。量化、剪枝、蒸馏、KV cache 优化、speculative decoding……各个方向都有大量工作，看起来已经卷到头了。

直到我看到 DeepSeek 最近放出的 DSpark 论文，我才意识到：**这个方向远没到头，只是之前的打开方式不太对。**

DSpark 发在 DeepSpec 仓库里（MIT 协议开源），作者来自 DeepSeek-AI 和北京大学。它在 speculative decoding 这个方向上，同时改进了两个东西：一是 draft model 怎么生成候选 token，二是验证阶段怎么剪裁验证长度。单独看每项改进似乎都是增量，但合在一起产生的效果——在 DeepSeek-V4 生产环境中，**单用户生成速度提升 60-85%，聚合吞吐量提升 50%+**——让我觉得值得专门写一篇来拆解。

这篇文章不打算做大而全的技术综述，而是聚焦 DSpark 到底做了什么、为什么能 work、以及它给我们做推理优化的思路带来了什么启发。

## 起点：Speculative Decoding 的困境

先说背景。Speculative decoding 的基本思路其实很朴素：用一个轻量的 draft model 先快速生成一批候选 token，然后用原始目标模型（target model）做并行验证，接受匹配的部分。理想情况下，一次 forward pass 就能确认多个 token，从而突破 autoregressive decoding "一次一个 token" 的瓶颈。

但这里有三个核心矛盾一直没有被很好地解决：

1. **Draft 的效率与质量之间的权衡**。Autoregressive drafters（如 Eagle3、Medusa）质量高但串行瓶颈没有彻底消除；parallel drafters（如 DFlash、SPHINX）速度快但 draft 的质量——尤其是后半段的 token——急剧下降。

2. **Suffix decay 问题**。这是 parallel speculative decoding 最头疼的问题。当你一次 draft 出一整块 token 时，后面的 token 是基于前面 draft 出来的——而不是实际采样出来的——上下文生成的。这导致越往后的 token 和 target model 的分布差异越大，被 reject 的概率接近 1。实践中经常是 draft 了 10 个 token，后面 6-7 个全部白费。

3. **验证长度的浪费**。现有框架大多用固定的验证长度（比如一次 draft 7 个，就验证 7 个），完全不考虑当前请求的"难度"。对于简单问题，draft 质量高，验证全过没什么问题；对于困难问题，draft 质量差，后面的验证几乎是必然 reject——但计算量已经花出去了。

DSpark 同时对这三个问题下手，而且下手方式很有 DeepSeek 的风格：**不走极端，而是在两个极端之间找最优的折中点。**

## 半自回归架构：把"块生成"和"令牌级依赖"结合起来

DSpark 的核心架构创新叫 **semi-autoregressive (半自回归) generation**。名字起得很准确——它既不是纯并行，也不是纯串行，而是两者的融合。

整个 draft model 分成两部分：

**第一部分是一个 parallel backbone**。它的作用是快速生成块级别的初始 token 候选。这里用了一个 tree attention 的变体，每个 token 只依赖块内前面的 token 和自己行（类似 blockwise parallel decoding 的思路）。这部分的计算是高度并行的，效率很高。

**第二部分是一个 lightweight sequential head**。它负责对 parallel backbone 产出的 token 做"精修"——用顺序建模的方式捕捉块内的 token 间依赖关系。论文尝试了两种实现：Markov head 和 RNN head。

Markov head 的建模方式是：

$P(t_{i+1} | t_1, ..., t_i) \approx P(t_{i+1} | t_i)$

即假设下一个 token 只依赖当前 token（一阶马尔可夫假设）。虽然听起来粗糙，但实验结果出奇地好——RNN head 带来的提升微乎其微，论文直接推荐用更简单的 Markov 变体。

更值得注意的是一个工程上的细节：**DSpark 只用 2 层 sequential head 就超过了 DFlash 的 5 层配置**。这意味着在同样的计算预算下，DSpark 能够把更多的容量分配到 backbone 上，从而获得更好的整体效果。

另一个让我觉得有意思的发现是：**block size 从 4 增大到 16 时，latency 开销只增加了 0.2% 到 1.3%**。这在推理优化中是非常罕见的特性——通常生成长度翻倍意味着延迟至少翻倍。这里的开销之所以这么低，根本原因在于 sequential head 只有 2 层，而且 Markov 假设下计算图极其简单。

从学术角度看，semi-autoregressive 架构最深刻的 insight 是：**parallel drafter 的后半段 token 质量差，本质原因不是"并行生成这个想法不对"，而是"并行的假设不满足时没有补偿机制"。** DSpark 用轻量 sequential head 做补偿，而且补偿的计算量远小于 full autoregressive 的代价。这是一个典型的"与其根除问题，不如用廉价方案管理问题"的工程思维。

## 从 suffix decay 到 suffix revival

DSpark 在 benchmark 上确实压制了 baselines。先看离线评测结果：

- vs Eagle3（autoregressive drafter）：accepted token length 提升 26.7%-30.9%
- vs DFlash（parallel drafter）：提升 16.3%-18.4%
- vs MTP-1（DeepSeek 之前的生产 baseline）：提升 42.8%-49.8%

有意思的是 domain 之间的差异。数学推理的 accepted length 最高（5.57），代码次之（5.12），对话最低（3.49）。这个规律其实很好理解：数学和代码生成的模式更确定，token 分布的熵更低，draft model 更容易猜对。对话生成则更发散，draft 的命中率自然更低。

但 DSpark 在 block size 增大时的表现才是真正的看点。当 block size W=7 时，数学领域提升 16%、代码 15%、对话 18%。当 W=15 时，数学提升飙到 30%、代码 26%、对话 22%。**大块 draft 时 suffix decay 本来更严重，而 DSpark 恰恰在此时受益最大**——这证明 sequential head 确实有效缓解了 suffix decay。

论文里有一个实验数据特别说明问题（来自 Figure 3）：在 MMLU 评测中，传统的 parallel drafter 在 positional 3-4 之后 token acceptance rate 就急剧跳水到 20% 以下；而 DSpark 在 positional 4-5 依然能保持 60%+ 的 acceptance rate。这就是 suffix decay 被抑制的直接证据。**与其叫 suffix decay，不如说 DSpark 实现了 suffix revival。**

## Confidence-Scheduled Verification：不要让验证变成浪费

如果说 semi-autoregressive architecture 解决的是"怎么生成更好的 draft"，那 confidence-scheduled verification 解决的就是"怎么更聪明地验证"。

这个思路的直觉其实很简单：不同的请求、不同的生成阶段，draft 的置信度是不一样的。简单问题（比如"你好"的回复）draft 几乎全对，复杂推理（比如数学证明）draft 很容易在后半段就开始跑偏。

现有的 speculative decoding 框架处理这个问题的方式全部是：**一刀切**。固定验证 N 个 token，不管内容难度。结果就是简单请求验证长度不够（draft 还有潜力），困难请求验证长度太长（后半段必然 reject，浪费算力）。

DSpark 把验证长度当成一个优化问题来解：

$L^* = \arg\max_L \text{Throughput}(L)$

其中 Throughput(L) 是在给定验证长度 L 下的系统吞吐量。这是一个典型的**非凸优化问题**，它没有解析解，但有工程解。

DSpark 的做法是：

1. **训练一个生存概率预测器**。用一个 calibration module 来估计 draft 中每个位置的 token 被 target model 接受的置信概率。

2. **建立吞吐量模型**。根据验证长度 L、单次验证的耗时、draft model 的速度、target model 的并行吞吐量特性，拟合一个 throughput 曲线。

3. **在每个请求到达时**，根据生存概率曲线和当前的 engine load（队列深度、剩余算力），动态选择验证长度 L*。

这里面最妙的工程细节是 **post-hoc STS calibration**。简单说，就是将预测的置信度分数与实际的接受率对齐。论文报告说，这个 calibration 把 ECE（Expected Calibration Error）从 3-8% 降到了 1%。这个精度的提升直接把 acceptance rate 从 45.7% 推到了 95.7%（对话场景）、76.9%→92.5%（数学）、67.6%→92.0%（代码）。

你可能想说：**这些数字好得不太真实吧？** 确实，95.7% 的 acceptance rate 听起来像是验证几乎不 reject——这在技术上是可能的，前提是验证长度被裁剪得足够保守。但问题在于：验证长度太保守 = draft 效率低，一个块只能确认 2-3 个 token，那整体速度反而下降。

关键在于 confidence-scheduled verification 的优化目标不是"最大化 acceptance rate"，而是**最大化端到端吞吐量**。在某些场景下它宁愿接受 60% 的 acceptance rate 也要用更长的验证长度，因为算力花在验证上比花在重新 draft 上更划算。

这背后的工程直觉是：**在 GPU 推理中，验证的计算开销是一个"固定启动成本+边际成本"的结构。** 验证 3 个 token 和验证 7 个 token 的前向 pass 时间几乎一样（受 batch size 影响），区别只在于被 reject 的部分浪费了。所以只要 acceptance rate 不低于某个阈值，长验证比短验证更划算。

## 生产环境验证：DeepSeek-V4 的现实结果

理论说得再好，不上生产跑一跑谁都不信。DSpark 论文大方地给出了在 DeepSeek-V4 上的生产评估结果，而且是两个版本的模型都测了：

**V4-Flash（轻量版）：**
- 单用户生成速度：提升 60-85%
- 聚合吞吐量：提升 51%
- 严格 SLA（120 tok/s/user）下：吞吐量提升 661%

**V4-Pro（完全版）：**
- 单用户生成速度：提升 57-78%
- 聚合吞吐量：提升 52%
- 严格 SLA 下：吞吐量提升 406%

单用户 60-85% 的提升已经很大了，但 661% 的 SLA 吞吐量提升才是真正让我停下来说"卧槽"的地方。它在说什么呢？在生产环境中，很多请求的延迟敏感度很高（比如 streaming chat），你不能让用户等太久。当每个用户的最低速度要求是 120 tok/s 时，DSpark 能让系统在单位时间内服务的用户数提升接近 7 倍。

这个提升的来源是：**confidence-scheduled verification 在高负载下的效果被放大了。** 当系统过载时，draft model 和 target model 的时序关系变得紧张，你不能让 target model 做太多不必要的验证。动态裁剪验证长度在高负载下的边际收益远大于低负载。

论文还特别强调了一个点：DSpark **把 serving 系统的 Pareto frontier 向外推了**。这意味着在 latency 和 throughput 的 tradeoff 曲线上，DSpark 让系统在任意 latency 预算下的吞吐量都比之前高。这是一个稀缺的能力——很多优化只是沿着 Pareto frontier 移动，而不是真正地将其外推。

## 和其他方案的关系与对比

在 speculative decoding 的谱系上，DSpark 的定位非常有意思。

- Eagle3（autoregressive）追求最高质量的 draft，代价是串行瓶颈没有完全消除  
- DFlash（parallel）追求最高的 draft 速度，代价是 suffix decay 严重  
- **DSpark 正好站在两者中间**：用 parallel backbone 吃掉大部分串行瓶颈，用 lightweight sequential head 抑制 suffix decay

从算力分配的角度来看，DSpark 的策略是：**把预算留给 backbone，head 只做最轻量的修边工作。** 2 层 head 胜过 5 层 parallel，说明 parallel 模型在解决 suffix decay 时的效率是边际递减的——它把算力浪费在了"试图从上下文补全中推断已丢失的信息"这种高难度任务上，而 sequential head 只需要做一步修正。

另外值得一提的是 DSpark 和 speculative decoding 的分布式关系的处理。论文提到 DSpark 的 draft model 和 target model 可以运行在不同的 GPU 上（通过 **async engine** 机制），draft model 提前生成下一块的候选，和 target model 的验证形成流水线。这在生产环境中非常关键——如果 draft 和 verify 必须串行运行在同一个 GPU 上，那收益会被大幅稀释。

## DeepSpec：把赛道从论文铺成代码

DSpark 的算法本身只是 DeepSpec 仓库的一部分。DeepSpec 是一个完整的 speculative decoding 训练和评估框架，支持 DSpark、DFlash、Eagle3 等多种算法。

从工程视角来看，DeepSpec 有几个值得关注的设计选择：

1. **训练数据用 Open-PerfectBlend**。这个数据集的 130 万条样本是这样分布的：17.6% chat、39.4% math、38.9% code、4.1% instruction-following。数学和代码占了接近 80%，这直接解释了为什么 DSpark 在 math 和 code 上表现更好——训练分布本身就偏向它们。如果你的应用场景主要是对话生成，可能需要重新调整训练分布。

2. **默认配置是单机 8 卡**。这不算大（参考一下，38TB 的 target cache 是给 Qwen3-4B 默认设置用的），对大部分有 A100/H100 集群的团队来说是可以接受的硬件门槛。

3. **Cache 容量 38TB**。这个数字看起来吓人，但实际上是 target KV cache——它存储的是 target model 的 key-value 对，用于验证阶段的并行推理。DeepSpec 在 cache 管理上做了分片和淘汰策略来保证实际可用性。

仓库 5.5k stars / 440 forks（截至发稿），MIT 协议。这意味着 DSpark 的算法可以直接在你的 infra 上复现。

## 启示与思考

读完整篇论文，我的核心感受是：DSpark 不是一个"拍桌子想出来的新架构"，而是**在深刻理解现有系统瓶颈后做的工程优化**。

几个值得思考的点：

1. **Semi-autoregressive 的思路可以推广到什么场景？** 任何需要"并行生成但依赖局部顺序"的任务都可以受益。比如 code completion 中跨行的依赖、机器翻译中短语级别的顺序约束、甚至图像生成中 spatial token 的局部一致性。轻量 sequential head 加 parallel backbone 的框架可能是一个通用范式。

2. **Confidence-scheduled verification 本质上是一个调度问题**，而调度问题在分布式系统中已经被研究得很透了。DSpark 的贡献是把这些调度思想引入到 speculative decoding 中，并且做了很好的 engineering implementation。未来可能看到更多类似"把 serving 问题转化成优化问题"的工作。

3. **DSpark 在 V4 上的收益和在其他模型上的收益可能需要重新校准**。论文在 Qwen3 系列和 Gemma4-12B 上做了离线评估，结果一致性好，但生产结果是只在 DeepSeek-V4 上跑的。V4 的架构细节我们不完全清楚，DSpark 的收益是否与 V4 的某些架构特性耦合，需要更多实践来验证。

4. **从技术传播的角度**，DeepSpec 以 MIT 协议开源整个训练和评估框架，比单发论文的意义大得多。Speculative decoding 之前的问题之一是研究结果难以复现——不同的 batch size、不同的 draft model 大小、甚至不同的 CUDA 版本都会影响结果。一个标准化的 benchmark 框架本身就推动了整个赛道的前进。

## 写在最后

DSpark 和 DeepSpec 给我的最大启发不是某个具体的技术点，而是一种**方法论的启示**：在 LLM 推理优化这个看似"挤满了"的赛道上，真正有突破性的工作往往不是发明一个全新的范式，而是**在一堆现有的 tradeoff 中找到一个之前没人发现的更优折中点**。

Parallel vs. sequential 的 tradeoff，验证长度多 vs. 少的 tradeoff，draft 质量 vs. 速度的 tradeoff——这些 tradeoff 每一个研究者都知道，但 DSpark 用一套连贯的方案同时优化了两个维度，而且是在生产环境中验证过有效。

从这个意义上说，DSpark 不仅仅是 DeepSeek 的一个技术里程碑，更是给整个推理优化社区的一个信号：**那块看似挖尽的矿，换个角度可能还有大东西。**

---

*如果你对 speculative decoding 的实现细节感兴趣，DeepSpec 的代码和论文都在 [github.com/deepseek-ai/DeepSpec](https://github.com/deepseek-ai/DeepSpec)（MIT 协议）。论文标题是 "DSpark: Confidence-Scheduled Speculative Decoding with Semi-Autoregressive Generation"。*
