## 一、核心方法：三项关键设计组合出击

DeepSeek-V4.1-Flash 的 KV 缓存压缩并非单一技术突破，而是**架构层、精度层、部署层**的联合优化。论文将总压缩目标分解为两个独立指标：**全局 KV 缓存**（始终驻留 HBM，约束运行时吞吐）与**持久 KV 缓存**（驻留 SSD 或主机内存，约束前缀复用能力）。围绕这两个指标，论文给出了三个层面的对应解法。

### 1.1 Causal Encoder-Decoder：把 Prefill 计算砍掉近一半

在长 horizon Agent 场景中，工具调用频繁触发新的 prefill 请求，而传统稠密 Transformer 的 prefill 复杂度为 $\mathcal{O}(NL)$——每一层都要处理每一个 token。CED 架构的出发点很直接：**让上半层网络直接复用下半层网络产生的全局 KV**，从而让大部分 prompt token 只需经过前 20 层编码器。

对于全局注意力，CED 中解码器第 $l$ 层的 KV 不再来自本层隐藏状态，而是从编码器末层隐藏状态 $H_{L/2}$ 经层相关投影得到：

$$C_{l}=H_{L/2}W_{l}^{KV},\quad Z_{l}=H_{L/2}W_{l}^{Z},\quad l>\frac{L}{2}$$

这一定义使得 prefill 阶段只需计算前 $L/2$ 层即可获得解码器所需的全部全局 KV。对于序列长度 $N \gg n_{\mathrm{win}}$ 的情形，整体 prefill 复杂度从 $\mathcal{O}(NL)$ 降至 $\mathcal{O}(NL/2 + n_{\mathrm{win}} \times L/2) \approx \mathcal{O}(NL/2)$。配合 **Decoder SWA Bounded Replay**——仅对 prompt 最后 $n_{\mathrm{win}}$ 个 token 做解码器 SWA 前向计算——解码器 prefill 的额外开销被压至最低。

![图3：DeepSeek-V4.1-Flash整体架构。40层网络分为各20层的因果编码器与解码器，前三层中的前两层仅用SWA，其余层使用CSA2；模型同时集成Single-Pass mHC、Engram、DSpark与Hierarchical Sparse Indexer](https://arxiv.org/html/2609.19969v1/figures/teaser_a.png)

参数激活的不对称性是 CED 的直接收益：**prefill 阶段仅激活 8B 参数，decode 阶段激活 16B**。对于输入 token 数远大于输出 token 数的 Agent 工作负载，这一非对称设计恰好命中了成本结构的关键。

### 1.2 CSA2：在“层”维度上压缩 KV 缓存

KV 缓存占用可以沿三个乘法维度压缩：**条目大小**（GQA/MLA 已解决）、**序列维度**（CSA 的 $m$-token 压缩已在 V4 中引入）、以及**层维度**——此前尚未被系统化利用。CSA2 的核心创新在于同时覆盖全部三个维度，并引入三种静态分配的工作模式：

- **Full Mode**：本层计算完整的主 KV、索引器 K 与 Top-K 索引，承担完整 CSA2 计算路径。
- **Reindex Mode**：复用前序 Full 层的主 KV 与索引器 K，但使用本层索引器 Q 重新评分并生成新的 Top-K 索引。
- **Reuse Mode**：同时复用主 KV 与 Top-K 索引，直接执行稀疏注意力，不计算索引器 Q 或评分。

三种模式的差异仅在于**主 KV、索引器 K、Top-K 索引**三个量从何处获取，而每层都计算自己的主 Q 与 SWA KV。这一设计将“缓存共享”与“索引复用”解耦，避免了 IndexCache 只省索引计算不省存储、或全网络路由共享损害性能的缺陷。

![图4：CSA2的三种运行模式。绿色表示当前层计算量，黄色表示从最近Full层复用的主KV与索引器K，红色表示从最近产生索引的层复用的Top-K索引](https://arxiv.org/html/2609.19969v1/figures/teaser_b.png)

进一步的优化来自 **Hierarchical Sparse Indexer**。在解码器中，后续索引层不必扫描全上下文，而是只在首个 Full 层构建的候选池内搜索。对于每条查询，首个 Full 层在全部因果可见位置中执行 Top-K 选择，并按块级最大分数选出最多 2048 个块（每块 8 个位置，合计 16384 个候选位置）。后续 Reindex 层仅在这 16384 个候选位置内评分，将单条查询的索引计算复杂度从**线性于上下文长度**变为**常数级**。

| 模式 | 主 KV | 索引器 K | Top-K 索引 | 索引器 Q |
|------|-------|----------|-----------|----------|
| **Full** | 本层计算 | 由本层主KV投影 | 本层计算 | 本层计算 |
| **Reindex** | 复用前序Full层 | 复用前序Full层 | 本层重新评分 | 本层计算 |
| **Reuse** | 复用前序Full层 | 复用前序Full层 | 复用最新索引 | 不计算 |

### 1.3 FP4 主 KV 缓存：精度换存储的最后一公里

在条目大小维度，DeepSeek-V4.1-Flash 将主 KV 缓存从 V4 的 FP8 进一步压缩至 **FP4**，采用 **E2M1 格式**、每 16 通道共享一个 E4M3 缩放因子，直接对齐 NVFP4 格式但省略了二级全局缩放。论文给出了一组边界分析：缓存中最大 RMSNorm 权重约为 1，经归一化后 512 通道 KV 潜变量的 L2 范数上界为 $\sqrt{512} \approx 22.6$，RoPE 旋转保持该范数，因此 FP4 格式的动态范围（最大量级 2688）远高于实际需求，省略全局缩放不会引入可测量的精度损失。

FP4 主 KV 缓存将 HBM 中的全局 KV 占用**再压缩一半**，与 CSA2 的跨层共享叠加，最终将全局 KV 缓存足迹从 V4-Flash 的水平压缩至约 **1/4**，达到 **890 字节/token**。

## 二、实验：基准表现与缓存压缩的双重验证

论文在 Base 模型和 Post-Training 模型两个层面给出了系统评测。Base 模型阶段的核心结论是：DeepSeek-V4.1-Flash-Base 在激活参数仅为 DeepSeek-V4-Pro-Base 约 **1/6**、总参数量约 **1/3** 的前提下，在 MMLU-Pro（74.1 vs 73.5）、BigCodeBench（60.6 vs 59.2）、HumanEval（79.4 vs 76.8）等基准上达到或超越 Pro 水平，同时在内部 held-out 语料上取得最低 BPB（bits-per-byte），表明预训练数据管线的改进对真实研发场景的泛化能力有明确增益。

![图1：DeepSeek-V4.1-Flash在Agent基准上的表现（a）与跨代模型每token全局KV缓存大小（b），相对V4-Flash减少约4倍，相对V1减少约437倍](https://arxiv.org/html/2609.19969v1/figures/decode_flops_curves.png)

Post-Training 阶段的结果更直接地体现了“压缩不损性能”的主张。在 **Terminal-Bench 2.1** 上，V4.1-Flash 以 **90.6%** 的 Pass@1 超越 Opus-5（89.1%）与 GPT-5.6 Sol（88.8%）；在 **DeepSWE v1.1** 上以 **74.2%** 的 Resolved 率超过 Opus-5（74.0%）与 GPT-5.6 Sol（73.0%）；在 **AutomationBench** 上取得 **54.8%**，领先第二名 GLM-5.3（48.8%）约 6 个百分点。即使在模型自称存在差距的科学 Agent 任务上（Terminal-Bench 4.0），31.2% 的成绩也显著优于 DeepSeek-V4-Pro（12.4%）与 Kimi-K3（12.6%），仅是落后于 Opus-5 的 51.8% 和 GPT-5.6 Sol 的 39.9%。

图 2 展示了跨代模型的 **单 token Decode FLOPs 随上下文长度的变化曲线**。从 4K 扩展到 1M tokens（256 倍），V4.1-Flash 的 Decode FLOPs 仅增加约 **25%**，几乎保持常数，而 V4-Flash 在同一范围上增长显著更陡。这是 CSA2 稀疏注意力与 Hierarchical Sparse Indexer 共同作用的结果：解码阶段每 token 的注意力计算量不再随上下文线性增长。配合 15 个 kernel（prefill）和 11 个 kernel（decode）的融合执行流程，长上下文推理的算力成本被结构性重塑。

![图2：跨代DeepSeek模型的单token Decode FLOPs随上下文长度的变化。V4.1-Flash在4K到1M范围内仅增加约25%的Decode FLOPs，几乎保持常数](https://arxiv.org/html/2609.19969v1/figures/pretrain_inhouse_ppl.png)

在可控制的推理努力（Reasoning Effort）维度上，论文引入了一个标量条件信号 $b \in \{1,\ldots,100\}$，通过长度惩罚系数的指数衰减来诱导不同预算下的行为分化：

$$k(b)=k_{0}\exp\left(-\frac{b-b_{\min}}{\tau}\right),\qquad \tau=\lambda\overline{\Delta b}$$

从 effort 25 到 100，八个推理密集基准的平均 Pass@1 从 67.1% 提升至 76.3%，DeepSWE v1.1 从 66.0% 提升至 74.2%，代价是约 2.5 倍的输出 token。实验进一步揭示收益前倾特征：60–80 区间已恢复最大值的大部分准确度，而 effort 100 的最后一步仅带来边际改善。

## 三、展望：压缩边界的再定义

DeepSeek-V4.1-Flash 的价值不在于某一项技术的极限，而在于将 KV 缓存压缩从**单点优化**升级为**架构—精度—部署的系统性联合设计**。CED 让计算成本向 decode 倾斜，CSA2 在层维度上消灭冗余存储，FP4 在数值表达上逼近物理极限，SWA Bounded Replay 则重新定义了持久缓存的最小保留集。这套组合拳最终将全局 KV 缓存压至 890 字节/token，持久缓存降至 V4-Flash 的 1/8，同时并未以明显性能损失为代价——这在长上下文模型部署中是一个非平凡的工程贡献。

论文在 Limitations 部分坦诚指出，CSA2 的稀疏选择误差和 SWA Bounded Replay 的近似状态重建，在未测试的边界条件下可能造成能力退化。这一风险是否会在真实生产环境中被放大，尤其是在多轮会话和极端长文档场景下的错误累积效应，值得持续观察。

一个更具价值的追问是：当 KV 缓存每 token 占用已经压至 890 字节、decode FLOPs 几乎不随上下文增长时，长 horizon Agent 部署的下一个瓶颈会转移到哪里？模型权重本身的加载与分发、多 Agent 协同的通信开销、还是训练侧的数据合成与 RL 扩展效率？答案或许比这篇论文本身更值得关注。

**论文标题**：DeepSeek-V4.1-Flash: Pushing the Limits of KV Cache Compression

欢迎投稿！欢迎合作！