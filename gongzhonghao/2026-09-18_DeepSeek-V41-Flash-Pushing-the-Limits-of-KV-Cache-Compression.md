## DeepSeek发布V4.1-Flash：KV缓存压缩437倍，百万上下文Agent部署成本骤降

![图1：DeepSeek-V4.1-Flash在Agent基准上的性能表现及历代模型每token全局KV缓存大小对比](https://arxiv.org/html/2609.19969v1/figures/teaser_a.png)

![图2：DeepSeek-V4.1-Flash单token解码FLOPs随上下文长度的变化曲线](https://arxiv.org/html/2609.19969v1/figures/decode_flops_curves.png)

当长周期智能体（Agent）逐渐成为大模型的核心工作负载，一个被长期忽视的瓶颈浮出水面：**KV缓存**。它既占用HBM显存，又拖累SSD存储与数据传输带宽，成为制约推理成本下探的最后一道闸门。DeepSeek-AI最新发布的**DeepSeek-V4.1-Flash**，以552B总参数、最高100万token上下文窗口的姿态，将全局KV缓存压缩至每token仅890字节——较上一代V4-Flash再降4倍，较初代V1累计压缩437倍，同时实现了性能的全面反超。

## 一、核心方法：三重维度协同压缩

论文的核心命题清晰：**KV缓存必须在架构、精度、部署三个维度上同时作战**。DeepSeek-V4.1-Flash的答案是一个组合拳：Causal Encoder-Decoder（CED）架构重塑计算分配，Compressed Sparse Attention 2（CSA2）打通层间共享，FP4量化与SWA Bounded Replay则分别从缓存精度与持久化策略上极限挤压。

### CED架构：让prefill计算减半的“不对称设计”

传统的Decoder-only模型在prefill阶段需要全层参与计算。DeepSeek-V4.1-Flash将40层Transformer一分为二：底部20层为**causal encoder**，顶部20层为**decoder**。decoder所有层的全局KV不再从各自的隐藏状态中生成，而是直接从encoder最后一层的输出经投影获得：

$$C_{l}=H_{L/2}W_{l}^{KV},\quad Z_{l}=H_{L/2}W_{l}^{Z},\quad l>\frac{L}{2}$$

这意味着prefill阶段仅需计算encoder的20层，decoder的全局KV“免费”获得。对于序列长度$N \gg n_{\mathrm{win}}$的情形，prefill计算复杂度从$\mathcal{O}(NL)$降至约$\mathcal{O}(NL/2)$，几乎减半。在Agent场景中，频繁的工具调用反复触发prefill，这一优化直接转化为可感知的延迟下降。参数量方面，prefill阶段每token仅激活8B参数，decode阶段为16B，非对称设计精准匹配了输入密集型Agent负载的特征。

### CSA2：从“每层独立缓存”到“层间共享复用”

如果说CED解决了“谁来计算KV”，CSA2则回答了“谁来存储KV”。DeepSeek-V4中的CSA将压缩维度限定在序列长度，而CSA2将压缩扩展至**层间维度**。每个CSA2层被静态分配为三种模式之一：

- **Full Mode**：完整计算main KV、indexer K和Top-K索引，充当共享源头
- **Reindex Mode**：复用前序层的main KV和indexer K，但用自身的indexer Q重新评分并生成新的Top-K索引
- **Reuse Mode**：全盘复用前序层的main KV与Top-K索引，仅保留自身的query与SWA KV

![图4：CSA2的三种运行模式——Full、Reindex、Reuse在main KV、indexer K与Top-K索引的获取方式上的差异](https://arxiv.org/html/2609.19969v1/figures/modes_a.png)

这一设计的精妙之处在于**解耦了缓存共享与索引新鲜度**：Reindex模式允许每层独立调整注意力选择，而不必各自维护一份完整的KV缓存。训练层面，论文通过shadow indexer机制跨越pipeline stage的物理隔离，使得层间共享在分布式训练中透明运作。

### FP4 KV缓存与SWA Bounded Replay：逼近物理极限

在精度维度，论文将**FP4量化**从indexer扩展至main KV缓存。采用E2M1格式、每16通道共享一个E4M3 scale，省略了NVFP4的第二级全局scale。论证逻辑简洁有力：经RMSNorm后的KV latent范数有界于$\sqrt{512} \approx 22.6$，FP4的动态范围上限高达2688，几乎没有溢出风险。相比V4的FP8缓存，存储需求再降近一半。

在部署维度，**SWA Bounded Replay**是一个看似激进实则精妙的取舍。与其在持久化缓存中为每层保存SWA KV（约占V4持久化缓存的一半），不如在缓存未命中时仅重放最近的$n_{\mathrm{win}}$个token，接受一个有界近似。精确重建需要重放$L \times n_{\mathrm{win}}$个token，而实验表明，近似重建对响应质量的影响“可以忽略不计”。这一发现彻底改变了成本结构：SWA KV从SSD持久化中移除，代之以宿主机DRAM中的一个短TTL分布式内存池，全局持久化缓存因此缩减至V4的1/8。

## 二、实验：压缩与性能的同步跃升

![图3：DeepSeek-V4.1-Flash整体架构图，展示了40层网络在causal encoder与decoder之间的划分以及各层的注意力模式配置](https://arxiv.org/html/2609.19969v1/figures/architecture.png)

DeepSeek-V4.1-Flash在45T多模态token上完成预训练，从第34T token起将序列长度扩展至1M。Base模型评测显示，552B参数（prefill激活8B/decode激活16B）的V4.1-Flash在MMLU-Pro上达到74.1%，超过此前1.6T参数的V4-Pro-Base（73.5%），在BigCodeBench上以60.6% Pass@1同样领先。

Post-training阶段的数据形成了更完整的叙事。在核心Agent基准上，V4.1-Flash的DeepSWE v1.1得分达到**74.2%**，一举从V4-Flash的54.4%跃升至与Opus-5（74.0%）和GPT-5.6 Sol（73.0%）的同一水平线。Terminal-Bench 2.1上90.6%的Pass@1同样超过所有对比模型。更具说服力的是**Codeforces评级3471**，超过了V4-Pro的3348和V4-Flash的3289，证明KV缓存的大幅压缩并未以牺牲推理深度为代价。

图2的FLOPs曲线则从计算角度印证了架构设计的效果：上下文从4K扩展到1M（256倍），单token decode FLOPs仅增加约1/4，曲线几乎保持水平。对于需要持久维护超长上下文的Agent部署场景，这一特性将显存成本、存储成本与计算成本三者同时压制在低位。

## 三、展望：当KV缓存不再是瓶颈

DeepSeek-V4.1-Flash展示了一条清晰的路径：在架构层打通共享、在精度层逼近极限、在部署层以有界近似换取系统性成本下降。这个组合策略使得百万token上下文的Agent不再是数据中心里的奢侈品。值得思考的是，当KV缓存不再是瓶颈后，Agent部署的下一个主要矛盾将转向何处——是调度系统的编排效率，还是Agent本身的任务规划上限？对于即将到来的智能体规模爆炸，这个问题的答案可能决定着下一个架构创新的方向。

**论文标题**：DeepSeek-V4.1-Flash: Pushing the Limits of KV Cache Compression

欢迎投稿！欢迎合作！