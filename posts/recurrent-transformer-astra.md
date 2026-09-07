# 从 GPT-6 Astra 的“循环架构”传闻说起：Recurrent Transformer 是如何重塑大模型推理范式的？

> **作者**：卡兹克  
> **标签**：`人工智能` | `大模型` | `Transformer` | `深度学习` | `开源项目`

---

> **导读**：随着 2026 年 9 月初 OpenAI 正式揭开 **GPT-6 (代号 Astra)** 的神秘面纱，AI 圈再次引发技术地震。然而，这次行业讨论的重心不再是“又堆了多少万亿参数”或“烧了多少张 GPU”，而是其传闻中颠覆性的底层架构演进——**循环 Transformer（Recurrent / Looped Transformer）与隐空间递归深度（Recurrent Depth）**。
> 
> 甚至在发布前夕，面对媒体关于“不可观测的隐空间思考（Neuralese）”的争议，OpenAI 首席科学家 Jakub Pachocki 曾罕见下场回应：“计算图深度仅比 GPT-4 增加约一倍，且为了安全可解释性，团队严格限制了循环架构的使用并保留了显式思维链（CoT）监视机制”。
>
> 到底什么是循环 Transformer？学术界为此做了哪些奠基性工作？它对 Astra 乃至整个下一代大模型架构意味着什么？普通开发者又如何动手验证？本文将从两篇重磅学术论文（Geiping 等人的 **Huginn** 与 Saunshi 等人的 **Looped Transformers**）出发，深入剖析其创新点与实验成果，并结合我们开源的轻量复现项目 **Tiny Loop Transformer (100M)**，带你拆解这一前沿架构的工程内幕。

---

## 目录

1. **热点风暴：GPT-6 Astra 为什么引发“循环架构”大讨论？**
2. **底层逻辑：传统 Transformer 的瓶颈与“循环”的必然性**
3. **奠基论文深度拆解：从 Huginn 到 Looped Transformers**
   - 论文一：Jonas Geiping et al. (NeurIPS 2025 Spotlight) —— 隐空间测试时算力
   - 论文二：Nikunj Saunshi et al. (ICLR 2025) —— 循环模型的算法推理威力
4. **实验结果：3.5B 逆袭 50B 的“越级打击”与关键结论**
5. **深入解密：循环架构对于 Astra 的战略意义与对齐困境**
6. **动手实战：在单台 M1 Mac 上跑通 100M 循环 Transformer**
7. **总结与展望：大模型从“参数堆叠”走向“计算重构”**

---

## 1. 热点风暴：GPT-6 Astra 为什么引发“循环架构”大讨论？

在大模型发展历程中，算力扩展（Scaling Law）经历了两个关键阶段：
1. **预训练阶段扩展（Pre-training Compute Scaling）**：以 GPT-3、GPT-4 为代表，通过无休止地增加模型物理参数量（Parameters）和训练 Token 量（Data）来涌现能力。
2. **测试时算力显式扩展（Test-Time Compute Scaling）**：以 OpenAI o1/o3 为代表，通过强化学习让模型在生成最终答案前，生成成千上万个自然语言 Token 的“思维链（Chain of Thought, CoT）”，实现“思考更久，回答更准”。

但进入 2026 年，显式思维链逐渐撞上了物理天花板：生成几千个中间 Token 不仅导致延迟（TTFT/TPOT）飙升，显存 KV-Cache 二次方爆炸，更造成了高昂的算力成本。

于是，**“在隐空间直接进行测试时计算（Test-Time Compute in Latent Space）”** 成为了顶级实验室心照不宣的秘密武器。

在 GPT-6 Astra 发布之际，外界大量报道称其底层采用了类似 **Recurrent Depth（循环深度）** 的设计：模型在预测下一个 Token 之前，让激活状态（Activations）在共享权重的 Transformer 核心层中循环迭代多次——相当于在“潜意识”里完成了多轮自我纠错与推演，最终才吐出一个确定性的 Token。

这也是为什么外媒一度惊呼“AI 开始使用人类看不见的内部语言（Neuralese）私密思考”，迫使 Jakub Pachocki 亲自澄清：**OpenAI 确实探索并利用了循环深度的计算放大效应，但对其施加了收敛约束，并与显式 CoT 保持平衡**。

循环 Transformer，正式从小众前沿理论迈入了工业级核心叙事。

---

## 2. 底层逻辑：传统 Transformer 的瓶颈与“循环”的必然性

要理解循环架构，首先要看清传统 Transformer 的阿喀琉斯之踵。

### 传统 Transformer：单向流水线
标准的前馈 Decoder-Only Transformer 是严格的单向计算流：
输入序列经过 Embedding，依次流经 Layer 1, 2, ..., N，最后由 LM Head 输出概率分布。

这种拓扑结构意味着：
- **思考深度与参数量死死绑定**：想让模型多计算一倍步数，物理参数量就必须翻倍（从 32 层加到 64 层），显存占用翻倍，跨卡通信开销激增。
- **浅层计算浪费，深层计算不足**：无论简单的标点符号还是复杂的几何证明，每一个 Token 都必须经过完全相同的 N 层。

### 循环 Transformer：参数固定的“深度伸缩器”
循环架构的核心哲学是：**参数空间（Parameters）与时间步/有效深度（Effective Depth）解耦**。

正如人脑在推演棋局时，调用的是同一块大脑皮层回路进行多次反复思考，循环 Transformer 将网络划分为三部分：
1. **Prelude（前奏层）**：负责将输入 Token 投影并适配到适合循环迭代的表征空间；
2. **Recurrent Core（循环核心层）**：一组固定的权重块（例如 8 层），输入向量在其中穿梭 $r$ 次，每一次循环都在隐空间对表征进行提纯与修正；
3. **Coda（尾声层）**：将循环多次后的隐状态做最终解码，投影回输出词表。

如果核心层有 $K$ 层，循环 $r$ 次，模型的**有效计算深度**就是：
$$\text{Effective Depth} = L_{\text{prelude}} + K \times r + L_{\text{coda}}$$

**物理参数量完全保持不变，但计算图深度可以在推理时任意拉伸！**

---

## 3. 奠基论文深度拆解：从 Huginn 到 Looped Transformers

循环架构并非空中楼阁，它在学术界有两座极其坚实的理论与实验里程碑。

### 论文一：Jonas Geiping et al. (NeurIPS 2025 Spotlight)
**《Scaling up Test-Time Compute with Latent Reasoning: A Recurrent Depth Approach》**

该论文提出了基于循环深度的全尺寸语言模型 **Huginn**，其核心创新点包括：

#### 1. 将 Test-Time Compute 引入隐空间（Latent Reasoning）
以往的测试时算力扩展完全建立在“多吐 Token”的假设上。Huginn 证明：**模型在隐向量空间内反复迭代循环，同样能够涌现出与 CoT 相当的测试时扩展定律（Test-Time Scaling Law）**。
- 区别于 CoT 的“自言自语（Verbalized Reasoning）”，隐空间推理属于“非语言直觉推演（Non-verbalized Internal Thinking）”，能够处理那些难以用文字精确描述的高维拓扑或连续逻辑关系。

#### 2. 动态采样训练策略（Random Recurrence Training）
如果模型在推理时需要支持不同的循环次数，训练时该怎么做？
论文设计了非常简洁而有效的策略：**在预训练阶段，对每个 batch 或 step 随机均匀采样循环步数 $r \sim U(r_{\min}, r_{\max})$**。
- 这样迫使共享的 Recurrent Core 无法对某一固定步数过拟合，学会了“自稳定（Self-stabilizing）”的迭代收敛特性；
- 无论让它只循环 1 次快速响应，还是循环 4 次深度思考，模型的表征都不会崩溃发散。

#### 3. 极致的系统与推理红利
- **KV-Cache 零膨胀**：在标准单步生成中，隐空间循环完全在残差流内部进行，序列长度不增加，因此根本不会像长思维链那样耗尽显存 KV-Cache。
- **原生投机解码（Speculative Decoding）与自适应停机（Adaptive Compute）**：同一个模型，循环 1 次就是天然的 Draft Model（草稿模型），循环 4 次就是 Target Model（目标模型），二者参数完全一致，无需额外训练专门的小模型即可实现零成本投机验证。

---

### 论文二：Nikunj Saunshi et al. (ICLR 2025)
**《Reasoning with Latent Thoughts: On the Power of Looped Transformers》**

来自 Google Research 的 Saunshi 等人则从理论和严谨受控实验的角度，解答了一个根本性疑问：**循环模型究竟只是单纯省显存，还是在逻辑推理能力上天生就更强？**

#### 1. 架构形式化与表达能力
作者将循环模型记为 $(k \otimes L)$，表示一个 $k$ 层的 Transformer 块迭代应用 $L$ 次；对比基线为相同参数和深度的前馈模型 $(kL \otimes 1)$。
论文通过构造性理论证明：Looped Transformer 天生具备模拟**循环算法（如图遍历、动态规划、多步归纳推理）**的归纳偏置（Inductive Bias）。

#### 2. 核心创新发现：困惑度（Perplexity）与推理能力的解耦
学术界过去迷信“测试集困惑度越低，模型越强”。Saunshi 等人打破了这一神话：
- 在常规语言建模上，Looped Transformer 的 Perplexity 可能略逊于全参数非共享模型（因为其总物理参数少，用于死记硬背事实知识的容量较小）；
- **但是在推理基准（Multi-hop Induction、大数算术、GSM8K）上，Looped Transformer 完胜相同计算量/参数量的标准 Transformer！**
- 论文得出了极其优美的经验公式：**推理任务的准确率随有效计算深度的对数呈线性扩展（Accuracy scales with $\log(\text{Effective Depth})$）**，这与显式 CoT 增加思考步数的 Scaling 特征惊人一致。

---

## 4. 实验结果：3.5B 逆袭 50B 的“越级打击”

两大团队的实证数据给出了令人信服的证据：

| 指标 / 特性 | 传统 Transformer (Feedforward) | 显式思维链 (Explicit CoT) | 循环 Transformer (Huginn / Looped) |
| :--- | :--- | :--- | :--- |
| **物理参数量** | 随设计层数线性增加 | 固定 | **固定且小（共享核心）** |
| **测试时计算量** | 恒定不可调 | 随输出 Token 数量伸缩 | **随隐空间循环次数 $r$ 自由伸缩** |
| **推理显存开销** | 随层数常数级 | 随生成长度平方级增长 | **恒定小尺寸，KV-Cache 完全复用** |
| **首字输出延迟** | 快 | 极慢（需等待长思维链生成） | **可控（根据算力预算调节 $r$）** |
| **可解释性** | 无思考过程 | 高（可阅读自然语言 Token） | **中低（隐空间迭代，需特殊探针解析）** |

### 震撼的实验实证：
1. **Huginn 的越级表现**：
   Geiping 团队训练的 **3.5B Huginn 模型**（800B Token 训练），在将测试时隐空间循环次数展开到更高深度时，其在 GSM8K 与代码推理评测中的表现，**跨越了多个数量级，直接媲美 30B–50B 规模的标准稠密模型**。
2. **算术与归纳逻辑任务的反杀**：
   Saunshi 等人在受控推理任务中发现，单向前馈 Transformer 即使参数量大几倍，在超出训练深度的复杂多跳推理上也极易产生注意力分散与幻觉；而 Looped Transformer 依靠反复刷新的隐状态，完美执行了算法收敛，展现出极强的长度外推泛化能力。

---

## 5. 深入解密：循环架构对于 Astra 的战略意义与对齐困境

把学术成果映射回 OpenAI 发布 GPT-6 Astra 的大背景，很多谜团迎刃而解。

### 1. 破解推理商业化的“内存墙（Memory Wall）”
在大规模商用落地中，最贵的不是 GPU 算力（FLOPS），而是**高带宽显存（HBM）容量与访存带宽**。
传统超大模型每次推理必须将数百 GB 的权重从 HBM 搬到片上 SRAM。而循环 Transformer 的物理核心极小（例如物理层仅十几层），权重可以长时间**驻留在高速片上缓存（SRAM/L2 Cache）中**，反复执行循环计算。这直接将推理从“访存受限（Memory-Bound）”推向了“算力受限（Compute-Bound）”，能效比与吞吐量产生质的飞跃。

### 2. 人类思维的“双系统模型”落地
认知科学中的 Kahneman 双系统理论：
- **系统 1（快思考）**：直觉反应，耗时极短；
- **系统 2（慢思考）**：逻辑推演，反复校验。

循环 Transformer 是在架构层面原生支持双系统的典范：
遇到“你好”这类日常寒暄，模型将循环次数设为 $r=1$，毫秒级吐字；
遇到复杂算法题，模型内部自动将 $r$ 提升至 4 或更高，在隐空间进行充分推演。

### 3. 安全性与不可观测性博弈（Alignment vs Neuralese）
为什么 OpenAI 没有在 Astra 中无底线地堆叠循环深度？
**因为“可解释性”是现代 AI 安全的底线。**
如果在隐空间循环几百次，模型内部表征会演化出一种高度紧凑、非线性的向量推演逻辑（即所谓 Neuralese）。人类现有的安全监视技术（如对中间 Token 的关键词过滤、意图审查）将完全沦为瞎子。
OpenAI 的做法是：**适度的循环深度（比 GPT-4 增加约一倍图深度）+ 显式思维链监督**。用隐空间循环做局部算力放大，用语言 Token 保证宏观思考过程的透明审计。

---

## 6. 动手实战：在单台 M1 Mac 上跑通 100M 循环 Transformer

学术界的 3.5B 和工业界的 Astra 固然强大，但作为开发者，我们能否亲手验证这一机制？

为了打破黑盒神秘感，我们开源了：
👉 **[Tiny Loop Transformer for Astra Reproduction](https://github.com/mongolia19/Tiny-Loop-Transformer-for-Astra-Reproduction)** (`recurrent-transformer-100m`)

这是一个专为个人开发者和研究人员设计的极小循环 Transformer 复现项目。**不依赖多卡 GPU 集群，仅在单台 16GB 统一内存的 Apple M1 Mac 上，就能在 5 分钟内完整跑通数据预处理、分词器训练、模型前向反向、检查点重载与变深度生成的全流程！**

### 1. 极简架构拓扑设计

本项目精准映射了 Huginn 的三段式经典结构，配置如下：
- **词表大小**：16,000（本地训练的 SentencePiece BPE）
- **隐藏维度**：`d_model = 768`，12 个注意力头，RoPE 旋转位置编码
- **前馈网络**：SwiGLU 激活，`mlp_hidden_size = 2048`，RMSNorm 归一化
- **层级设计**：
  - `prelude`: **2 层** 独立非共享 Transformer Block
  - `recurrent_core`: **8 层** 共享核心 Transformer Block
  - `coda`: **2 层** 独立非共享 Transformer Block
- **参数规模**：物理参数量严格收敛在 **97,241,856（约 97.2M）**！

当循环步数 $r$ 取不同值时，有效计算深度动态变化：
- $r=1$ 时：$2 + 8\times 1 + 2 = \mathbf{12\text{ 层}}$
- $r=2$ 时：$2 + 8\times 2 + 2 = \mathbf{20\text{ 层}}$
- $r=3$ 时：$2 + 8\times 3 + 2 = \mathbf{28\text{ 层}}$
- $r=4$ 时：$2 + 8\times 4 + 2 = \mathbf{36\text{ 层}}$

**参数始终只有 97M，但你拥有了一台可以在 12 层到 36 层之间随时自由切换的“变形金刚”！**

### 2. 核心代码实现解析

模型在 PyTorch 下的实现极其干净优雅。在 `src/recurrent_transformer/model.py` 中，前向传播仅用几行核心逻辑便完成了循环展开与重计算（Activation Checkpointing）：

```python
class RecurrentTransformer(nn.Module):
    def __init__(self, config: ModelConfig) -> None:
        super().__init__()
        self.config = config
        self.token_embedding = nn.Embedding(config.vocab_size, config.d_model)
        self.prelude = nn.ModuleList(TransformerBlock(config) for _ in range(config.prelude_layers))
        # 核心：定义唯一的 8 层共享核心
        self.recurrent_core = nn.ModuleList(
            TransformerBlock(config) for _ in range(config.recurrent_core_layers)
        )
        self.coda = nn.ModuleList(TransformerBlock(config) for _ in range(config.coda_layers))
        self.final_norm = RMSNorm(config.d_model, config.rms_norm_eps)
        self.lm_head = nn.Linear(config.d_model, config.vocab_size, bias=False)
        self.lm_head.weight = self.token_embedding.weight  # 权重绑定 (Weight Tying)

    def forward(self, input_ids: Tensor, labels: Tensor | None = None, num_recurrences: int | None = None):
        recurrences = num_recurrences or self.config.min_recurrences
        
        # 1. 前奏处理 (Prelude)
        hidden = self._run_blocks(self.token_embedding(input_ids), self.prelude)
        
        # 2. 核心隐空间循环 (Recurrent Loop)
        for _ in range(recurrences):
            hidden = self._run_blocks(hidden, self.recurrent_core, checkpointed=True)
            
        # 3. 尾声处理与词表投影 (Coda & LM Head)
        hidden = self._run_blocks(hidden, self.coda)
        logits = self.lm_head(self.final_norm(hidden))
        
        return ModelOutput(logits=logits, loss=loss)
```

### 3. 训练时的核心魔法：Recurrence Sampling
在训练循环（`train.py`）中，为了让这 8 层共享参数适应不同的推理深度，每个优化步或 Micro-batch 都会随机抽取一个循环步数：

```python
# 训练时随机采样循环深度，增强跨深度泛化能力
recurrences = rng.randint(min_recurrences, max_recurrences)
with torch.autocast(device_type=resolved_device.type, dtype=autocast_dtype):
    result = model(batch, labels=batch, num_recurrences=recurrences)
(result.loss / gradient_accumulation_steps).backward()
```

### 4. 离线 5 分钟上手验证

如果你手头有一台 Mac 或带有 GPU 的笔记本，可以立即检出体验：

```bash
git clone https://github.com/mongolia19/Tiny-Loop-Transformer-for-Astra-Reproduction.git
cd Tiny-Loop-Transformer-for-Astra-Reproduction

python3 -m venv .venv
source .venv/bin/activate
pip install -e '.[dev]'

# 运行自动化测试套件
pytest -q

# 离线运行一个完整的轻量训练流水线（自带双语小语料）
python -m recurrent_transformer.cli tiny-pipeline \
  --english tests/fixtures/en.txt \
  --chinese tests/fixtures/zh.txt \
  --output artifacts/tiny \
  --steps 1
```

训练完成后，你可以用同一个 Checkpoint 测试不同循环深度下的推理生成：

```bash
python -m recurrent_transformer.cli generate \
  --checkpoint artifacts/smoke/checkpoint-000020.pt \
  --prompt '循环 Transformer 的优势在于' \
  --recurrences 1,2,4 \
  --max-new-tokens 32
```
终端将分别输出 `[recurrences=1]`、`[recurrences=2]`、`[recurrences=4]` 的生成结果。你可以直观看到相同的权重如何在不同的计算深度下运作。

---

## 7. 总结与展望：大模型从“参数堆叠”走向“计算重构”

回看大模型演进的脉络：
- **2017 年**：Attention Is All You Need，确立了前馈自注意力机制的标准流水线；
- **2020–2023 年**：暴力出奇迹，行业通过不断拉高物理参数验证了 Scaling Law；
- **2024–2025 年**：o1 与 o3 揭开了测试时算力（Test-Time Compute）的冰山一角；
- **2026 年**：伴随着 GPT-6 Astra 的讨论与 Huginn/Looped Transformer 等学术突破，**循环架构正在重新定义“模型深度”的内涵**。

当深度不再等同于参数，当思考可以在隐空间以极高的缓存命中率迭代发生，大模型的推理成本与智能密度必将迎来新的指数级跨越。

与其在谣言与宣传通稿中雾里看花，不如克隆代码，亲手在自己的电脑上运行一次循环前向传播。**未来的大模型，或许不再是臃肿的千亿巨兽，而是一个个精巧内省、深思熟虑的循环引擎。**

---

### 参考资料与开源项目
1. **本项目开源仓库**：[mongolia19/Tiny-Loop-Transformer-for-Astra-Reproduction](https://github.com/mongolia19/Tiny-Loop-Transformer-for-Astra-Reproduction)
2. **Jonas Geiping et al.**, *Scaling up Test-Time Compute with Latent Reasoning: A Recurrent Depth Approach*, NeurIPS 2025 Spotlight. arXiv:2502.05171.
3. **Nikunj Saunshi et al.**, *Reasoning with Latent Thoughts: On the Power of Looped Transformers*, ICLR 2025. arXiv:2502.17416.
4. **seal-rg/recurrent-pretraining**: [GitHub 仓库](https://github.com/seal-rg/recurrent-pretraining)
5. **B 站参考解析视频**：`BV1gBto6QEqa` 《Astra架构曝光：OpenAI的循环Transformer到底有多炸？》 (杜雨说AI)