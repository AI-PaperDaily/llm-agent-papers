## DeepSeek发布V4.1-Flash：KV缓存压缩至1/4，百万上下文成本近乎恒定

大模型Agent的规模化部署正遭遇一场沉默的瓶颈转移。当推理框架通过**稀疏注意力**与**滑动窗口机制**将长序列计算成本大幅降低后，**KV缓存的存储、迁移与带宽压力**却悄然成为新的成本天花板。HBM容量限制着单请求的运行时状态，SSD与主机内存约束着可持久化的前缀缓存尺寸，而I/O与互联带宽则掌握着缓存加载与迁移的效率命脉。

DeepSeek-AI在最新技术报告中给出了一个系统性的回应：**DeepSeek-V4.1-Flash**，一款支持百万token上下文的552B参数多模态MoE模型。其核心指标极具冲击力——**全局KV缓存足迹压缩至每token 890字节**，约为前代DeepSeek-V4-Flash的四分之一；**持久化KV缓存足迹进一步降至八分之一**。同时，从4K扩展到1M上下文，单token解码FLOPs仅增长25%，近乎恒定。

![图1：KV缓存每token大小对比](https://arxiv.org/html/2609.19969v1/figures/teaser_b.png)

## 一、核心方法：从架构到部署的三层联合压缩

DeepSeek-V4.1-Flash的KV压缩并非单一技术突破，而是**模型架构、缓存精度、部署策略**三个维度协同作用的结果。

**CED架构：prefill计算量减半的因果编码器-解码器**

传统Transformer在prefill阶段需要对全部层执行前向计算，为每个token生成各层的KV缓存。DeepSeek-V4.1-Flash引入**Causal Encoder-Decoder（CED）**架构，将40层网络均分为20层编码器与20层解码器。解码器层的全局KV不再从自身隐藏态计算，而是直接由编码器最后一层隐藏态经层相关投影权重生成：

$$C_{l}=H_{L/2}W_{l}^{KV},\quad Z_{l}=H_{L/2}W_{l}^{Z},\quad l>\frac{L}{2}$$

其中$C_l$为第$l$层的KV条目，$Z_l$为对应的压缩权重，$H_{L/2}$为编码器末层隐藏态。这一设计使prefill阶段仅需计算前20层即可获得全部全局KV缓存，将prefill复杂度从$\mathcal{O}(NL)$降至约$\mathcal{O}(NL/2)$。在输入密集的Agent工作负载中，这意味着**prefill激活参数从16B降至8B**，几乎将新请求的处理成本腰斩。

![图2：DeepSeek-V4.1-Flash整体架构](https://arxiv.org/html/2609.19969v1/figures/teaser_a.png)

**CSA2：三层维度上的跨层KV复用**

如果说CED解决的是“算多少”的问题，那么**Compressed Sparse Attention 2（CSA2）**解决的是“存多少”的问题。它从三个乘法维度同时压缩：**条目尺寸**（GQA/MLA）、**序列维度**（每m个token压缩为一个条目）、**层维度**（跨层共享缓存）。CSA2为每层静态分配三种模式之一：

- **Full Mode**：计算完整的main KV、索引器K，并运行索引器生成全新的Top-K索引
- **Reindex Mode**：复用前序层的main KV与索引器K，但用自己的索引器Q重新评分并产生新的Top-K索引
- **Reuse Mode**：直接复用前序层的main KV和Top-K索引，跳过索引计算直接执行稀疏注意力

编码器18层分为3组，每组首层为Full Mode、其余5层为Reuse Mode，压缩率$m=2$；解码器20层分为5组，首组首层为Full Mode，其余各组首层为Reindex Mode、后续3层为Reuse Mode，压缩率$m=1$。这种分层策略在保持注意力选择灵活性的同时，大幅削减了冗余缓存存储与索引器计算开销。

![图3：CSA2三种运行模式](https://arxiv.org/html/2609.19969v1/figures/teaser_a.png)

**FP4量化与SWA Bounded Replay：精度与部署的边界突破**

在缓存精度层面，模型将**main KV缓存量化至FP4**（E2M1格式，每16通道共享一个E4M3 scale），并在post-training阶段引入量化感知训练（QAT）。论文指出，FP4格式的动态范围上限为448×6=2688，而实际缓存值的模长上界约为$\sqrt{512}\approx 22.6$，训练中观测到的最大值仅约10，因此省略NVFP4的二级全局scale不会造成精度损失。

部署层面，**SWA Bounded Replay**引入了一个关键取舍：不再将滑动窗口注意力（SWA）的KV缓存持久化到SSD，而是仅保留最近的$n_{\mathrm{win}}$个token用于近似重建。这一策略将原本需要$L \times n_{\mathrm{win}}$个token的精确重建代价替换为常数级的轻量重放，经实验验证对响应质量的影响可忽略不计。SWA KV由此从持久化缓存中移除，转而进入主机DRAM中的短TTL分布式内存池，持久化KV足迹因此再降一半。

![图4：解码FLOPs随上下文长度变化](https://arxiv.org/html/2609.19969v1/figures/decode_flops_curves.png)

## 二、配套优化：从训练到推理的系统级协同

压缩架构的真正落地离不开训练与推理栈的深度适配。训练侧，**Shadow Indexer**机制允许共享注意力组件的层跨流水线部署；微批次级共享状态管理协调跨stage的中间表示生命周期。推理侧，单个Reuse Mode层的执行仅需**15个kernel（prefill）/ 11个kernel（decode）**，通过将RoPE、注意力、KV转换融合进FlashMLA风格的统一核函数来实现。

新增的**Engram**条件记忆模块提供196B稀疏访问参数，以FP8精度存储N-gram嵌入，通过Sinkhorn平衡优化器替代Adam以削减优化器状态内存。**DSpark**推测解码模块在backbone冻结的前提下独立训练，以半自回归方式并行草稿5个位置，结合置信度调度动态调整验证长度。

## 三、实验评估：以更小的缓存换取更强的性能

在基础模型对比中，DeepSeek-V4.1-Flash-Base以**8B/16B激活参数**击败了49B激活的DeepSeek-V4-Pro-Base：MMLU-Pro达到**74.1**（vs 73.5），BigCodeBench达到**60.6**（vs 59.2），HumanEval达到**79.4**（vs 76.8）。

Post-training后的Chat版本在Agentic场景中表现更为突出。DeepSWE v1.1以**74.2%的解决率**超越Opus-5（74.0%）与GPT-5.6 Sol（73.0%）；Terminal-Bench 2.1达到**90.6%**，领先所有对比模型；AutomationBench达到**54.8%**，同样位于榜首。推理能力上，Codeforces rating达到**3471**，超越DeepSeek-V4-Pro（3348）。模型还提供了**可控推理努力机制**，通过在系统提示中注入一个1-100的标量$b$来控制推理深度，对应的长度惩罚系数随努力值指数衰减：

$$k(b)=k_0\exp\left(-\frac{b-b_{\min}}{\tau}\right)$$

在努力值从25提升至100的过程中，八项推理基准平均Pass@1从67.1%升至76.3%，DeepSWE v1.1从66.0%升至74.2%，代价是输出token量增加约2.5倍。

## 四、展望

DeepSeek-V4.1-Flash的深层意义在于，它以工程系统化的方式验证了一个方向：**KV缓存压缩可以成为比单纯扩大模型规模更有效的成本杠杆**。百万级上下文不再是旗舰模型的专属，而开始向轻量级、可负担的方向迁移。当全局KV缓存从V1时代的390KB/token压缩到890字节/token，437倍的降幅背后，是长期Agent部署从“技术可行”走向“经济可行”的关键一步。这一范式是否会成为后续开源模型底座的标准配置，值得持续观察。

**论文标题**：DeepSeek-V4.1-Flash: Pushing the Limits of KV Cache Compression

欢迎投稿！欢迎合作！