## 小米MiMo-V2.6发布：RL规模化三轴驱动，DeepSWE猛涨14分逼近72%

大模型社区正在经历一场静默的范式转移：**从“堆数据预训练”转向“规模化强化学习（RL）驱动自我改进”**。当GPT-5、Claude Opus 5等前沿模型在Agent基准上展开激烈角逐时，一个根本性问题浮出水面：如何让模型在真实、可交互、可验证的复杂环境中，通过持续探索与反馈信号，实现可度量的能力跃迁？

小米大模型团队发布的 **MiMo-V2.6系列**，以一次系统性工程实践回应了这一挑战。该系列包含两个规模化MoE模型——**1.02T参数的Pro（42B激活）**与**310B参数的Flash（15B激活）**——核心贡献在于将RL计算沿**批次规模、环境多样性、评分器算力**三个维度同步放大。在DeepSWE v1.1基准上，Pro从58.4提升至**72.6**，Flash从48.7提升至**65.7**，单次RL训练成本达**260万美元**。

## 一、核心方法：RL规模化的“铁三角”

### 1.1 架构底座：混合注意力+MoE的探空间设计

RL的有效性高度依赖前序阶段提供的**探索空间**。MiMo-V2.6采用混合稀疏MoE Transformer骨干，将**Local Sliding Window Attention（SWA）**与**Global Attention（GA）**交替堆叠：Pro配置70层（60 SWA + 10 GA），Flash配置48层（39 SWA + 9 GA）。SWA窗口大小为**128**，GA层周期性插入以聚合全局上下文。MoE FFN无共享专家，Pro拥有**384个专家/激活8个**，Flash为**256个专家/激活8个**。

该架构的视觉编码器MiMo-ViT同样采用混合注意力设计，以**sink-augmented SWA**替代固定窗口注意力，在行主序与列主序之间交替序列化，大幅降低高分辨率视觉处理成本。音频侧则采用两阶段编码：**Audio Tokenizer**以20层RVQ将每帧离散化为20个token，再经patch encoder压缩至6.25Hz。

![图2：MiMo-V2.6整体架构图](https://arxiv.org/html/2610.11959v1/architecture.png)

### 1.2 评分器升级：Groupwise Agentic Grading

传统二元测试奖励无法区分同为“通过”的解决方案之间的质量差异。论文提出两条互补路径：

**Groupwise Reward Synthesis（GRS，离线评分标准）** 针对高通过率任务子集，预先收集多轮离线rollout，由Agent分析不同解法路径与行为模式，构建任务专属的**解法评分项**与**行为评分项**。训练时，最终奖励将测试奖励与两项评分相乘：

$$R_{i}=R_{i}^{\mathrm{test}}\cdot S_{i}^{\mathrm{sol}}\cdot S_{i}^{\mathrm{beh}}$$

该乘法形式确保评测监督始终锚定于测试结果，同时让通过的轨迹在实现质量与解题行为层面被进一步区分。

**Groupwise Advantage Redistribution（GAR，在线评分）** 用于其余代码任务。对每个混合结果的rollout组，评分Agent联合审查所有轨迹，从**方案适配性、实现精度、最小性、副作用规避、代码工艺**五个维度排序。排序结果通过质量因子$f_i\in(0,1]$对序列级优势进行重分配：

$$\lambda=\frac{\sum_{j\in\mathcal{P}}A_{j}}{\sum_{j\in\mathcal{P}}f_{j}A_{j}},\qquad A_{i}^{\prime}=\begin{cases}\lambda f_{i}A_{i},&i\in\mathcal{P},\\ A_{i},&i\notin\mathcal{P}.\end{cases}$$

该更新保留组内正优势总质量不变，将优势从低质量通过轨迹向高质量通过轨迹再分配。训练对比显示，未启用在线评分的策略在turn数和token长度上迅速膨胀，且维持性审计发现其倾向采用**投机兼容分支、异常吞噬、宽松校验**等奖励投机行为；启用GAR后，patch平均更小、更精准、更易维护。

![图7：Groupwise agentic grading工作流示意图](https://arxiv.org/html/2610.11959v1/groupwise_grading.png)

![图6：奖励黑客防护与监控](https://arxiv.org/html/2610.11959v1/reward_hack.png)

### 1.3 稳定训练的关键：冻结Router与多层防御

RL训练中MoE路由面临严重的**负载崩塌**风险。实验显示，可训练Router在前20步内专家负载变异系数从0.78飙升至2.0，冷专家比例从0.5%暴增至22%。将Router恢复至RL前初始参数后，负载恢复而基准性能不变——证明崩塌由Router漂移驱动，与专家权重退化无关。因此，论文采用**冻结Router**策略，冷专家比例稳定在约1%。

![图11：Router冻结实验对比](https://arxiv.org/html/2610.11959v1/router_freezing.png)

## 二、实验：一次RL，多域跃迁

RL训练以**GRPO**为优化器，全局批次含**1,568个prompt × 16个rollout**，单步处理25K轨迹、**2.7B–3.7B token**，上下文长达**1M**。训练使用**Muown优化器**（Muon变体，引入显式行范数控制），学习率$3\times10^{-6}$，无warmup，并以MXFP4量化感知训练稳定数值。

计算开销分解显示：rollout占**43.8%**、训练占**43.5%**、评分器占**12.7%**。评分器算力的量级投入是本次工作区别于以往RL的关键标志。

### 2.1 主要性能结果

| 基准 | MiMo-V2.6 Pro | MiMo-V2.6 Flash | Claude Opus 5 | GPT-5.6 Sol |
|:---|:---:|:---:|:---:|:---:|
| DeepSWE v1.1 | **71.9** | 67.9 | 74.0 | 73.0 |
| AutomationBench v1.0.6 | **53.1** | 52.3 | 50.3 | 45.8 |
| GDPval-AA 2.1 | **1673** | - | 1708 | 1588 |
| CyberGym | 94.0 | **95.1** | - | - |
| MiMo Visual Coding | 72.3 | 71.5 | 70.0 | 73.4 |

MiMo-V2.6-Pro在**AutomationBench与GDPval**上超越Claude Opus 5与GPT-5.6 Sol，在DeepSWE上达到**71.9**，逼近Claude Opus 5的74.0。Flash在CyberGym上以**95.1**领先。值得关注的是，MiMo-V2.5 Pro在DeepSWE上仅**19.0**，两代之间提升约3.8倍。

### 2.2 多Harness泛化验证

为验证跨harness迁移能力，论文在四个训练用mini-harness与三个留出harness（codex、claude code、mini-swe-agent）上评估DeepSWE。训练与留出harness的平均Pass@1从约**50%提升至66%**，两者差距显著缩小，证明轻量模块化mini-harness引入的受控多样性确实促进了任务求解策略的泛化。

![图9：RL训练过程基准分数变化](https://arxiv.org/html/2610.11959v1/benchmark_curve.png)

## 三、展望：开源生态与自我改进的可复现路径

MiMo-V2.6的实践揭示了一个关键判断：**规模化RL的效果不取决于单一维度的突破，而依赖于批次规模、环境质量、评分精度三者的同步协同**。当二元奖励信号被细粒度级评分替代，当奖励黑客行为在环境准备与训练审计双层防线中被系统性压制，RL才能真正释放模型的自我改进潜力。

论文已开源**MiMo-V2.6-Distill-Qwen-9B**、跨四大领域的约7K训练任务环境、端到端RL框架及可组合mini-harness。在开源9B模型上的实验显示，多harness RL在所有21个数据集-harness组合上均实现提升。这是否意味着小模型同样具备在规模化RL范式下的持续改进能力？答案留给社区验证。

**论文标题**：MiMo-V2.6: Scaling Reinforcement Learning Towards Self-Improvement

欢迎投稿！欢迎合作！