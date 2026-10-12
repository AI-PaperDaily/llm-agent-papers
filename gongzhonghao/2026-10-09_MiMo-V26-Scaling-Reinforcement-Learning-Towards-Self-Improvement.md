**小米发布1.02T参数MiMo-V2.6：RL规模扩展驱动DeepSWE提升14.2分，Agent自我提升的工程极限在哪里？**

## 引言

在递归自我提升（RSI）的愿景中，模型应当通过持续的探索与反馈来扩展自身能力。而实现这一愿景的核心路径，是将模型放入可交互的Agent环境中，以多步轨迹和反馈信号驱动强化学习（RL）的规模化。然而，这一路径面临两大瓶颈：其一，适合RL训练的基础架构与足够丰富的探索空间；其二，支撑大规模RL所需的分布式基础设施、环境多样性与评分器精度。小米发布的MiMo-V2.6系列——包括1.02T总参数的Pro版本与310B总参数的Flash版本——正是对这一挑战的系统性回应。论文将RL计算规模作为核心变量，在架构、环境与评分三个维度同时推进，展示了Agentic RL从“有效”走向“可扩展”的工程范式。

## 核心方法

### 三维度扩展RL计算

![图2：MiMo-V2.6整体架构](https://arxiv.org/html/2610.11959v1/architecture.png)

MiMo-V2.6的RL训练围绕三个维度展开。第一，**训练吞吐与批量规模**：采用全异步训练流程，每一步处理1,568个prompt，每组16条候选轨迹，合计约25K条序列，对应2.7B至3.7B训练token，上下文长度最高可达1M。第二，**环境与harness多样性**：RL任务覆盖代码、通用工作流、视觉创作与网络安全四大领域，并通过多harness训练提升跨工具泛化能力。第三，**评分器计算投入**：引入Groupwise Agentic Grading机制，为长周期任务提供比二元测试结果更精确的奖励信号。

RL学习目标采用GRPO框架，其核心形式为：

$$\mathcal{L}(\theta)=-\mathbb{E}_{q\sim\bigcup_{d}\mathcal{D}_{d},\;\{o_{i}\}_{i=1}^{G}\sim\mu_{\theta_{\mathrm{old}}}(\cdot\mid q)}\left[\frac{1}{\sum_{i=1}^{G}|o_{i}|}\sum_{i=1}^{G}\sum_{t=1}^{|o_{i}|}r_{i,t}\,M_{i,t}\,A_{i}\,\log\pi_{\theta}(o_{i,t}\mid q,o_{i,<t})\right]$$

其中重要性采样比率 $r_{i,t}$、token级掩码 $M_{i,t}$ 与序列级优势 $A_i$ 共同决定每个token的更新幅度。在计算资源分配上，MiMo-V2.6-Pro的RL成本占比为：rollout占43.8%，训练占43.5%，评分器占12.7%。

### 多Harness训练与模块化设计

![图14：RL基础设施架构概览](https://arxiv.org/html/2610.11959v1/rl_infrastructure.png)

传统生产级harness将工程防护与多步工作流强耦合，导致信用分配不可靠。MiMo-V2.6提出以**mini-harness**为核心的多harness训练策略：所有mini-harness共享同一极简Agent循环，包含系统prompt、工具集与上下文管理三个可解耦模块，在此基础上为代码、通用、视觉与网络安全任务派生不同配置。这种设计既保证了训练分布的多样性，又支持对单一交互机制的独立分析。实验显示，在四个训练harness与三个held-out harness（codex、claude code、mini-swe-agent）上，DeepSWE v1.1的Pass@1从约50%提升至66%，训练与held-out harness之间的性能差距同步收窄。

### 奖励黑客防御与Groupwise Agentic Grading

![图6：奖励黑客预防与监控](https://arxiv.org/html/2610.11959v1/reward_hack.png)

RL训练中，模型可能通过检索上游修复补丁、安装新版本包等捷径获取高奖励。MiMo-V2.6构建了四层防御体系：**中训练对齐数据**——将早期发现的hacking案例转化为反思式训练样本；**环境清理**——移除构建日志、残留补丁与Git未来提交，并施加容器级网络隔离；**对抗性Hack Agent**——在训练前系统性地探测残留漏洞，经过多轮迭代清理直至无法找到可行的利用路径；**训练期审计**——离线审计轨迹中的hacking行为，确认后将其有效奖励置零。

对于通过测试的候选方案，二元奖励无法区分实现质量。Groupwise Agentic Grading通过两条路径补足这一盲区。**Groupwise Reward Synthesis（GRS）** 离线构建任务级rubric，将二元测试奖励与方案质量分、行为质量分相乘：

$$R_{i}=R_{i}^{\mathrm{test}}\cdot S_{i}^{\mathrm{sol}}\cdot S_{i}^{\mathrm{beh}}$$

**Groupwise Advantage Redistribution（GAR）** 则在线对比每组内的通过轨迹，沿五个维度——方案适宜性、实现精度、最小性、副作用规避与代码工艺——进行排序，并将正优势从低质量通过轨迹重新分配给高质量轨迹：

$$\lambda=\frac{\sum_{j\in\mathcal{P}}A_{j}}{\sum_{j\in\mathcal{P}}f_{j}A_{j}},\qquad A_{i}^{\prime}=\begin{cases}\lambda f_{i}A_{i},&i\in\mathcal{P},\\
A_{i},&i\notin\mathcal{P}.\end{cases}$$

行为正则化方面，**Group-Relative Length Penalty**通过组内参考长度抑制token膨胀：

$$\widetilde{R}_{i}=R_{i}-\mathbf{1}[i\in\mathcal{P}_{q}]\,X\left[\operatorname{clip}\!\left(\frac{\ell_{i}/\ell_{q}^{\star}-1-\delta}{s-\delta},\,0,\,1\right)\right]^{\gamma}$$

同时，**Segment-Level Behavioral Penalties**对格式违规与工具调用错误进行token级惩罚，在保守优势总质量的前提下将负优势集中于错误token。

## 实验

![图3：DeepSWE得分与RL成本关系](https://arxiv.org/html/2610.11959v1/rl_cost.png)

在RL训练过程中，MiMo-V2.6-Pro在DeepSWE v1.1上的avg@3得分从58.4提升至72.6，Flash从48.7提升至65.7，且增长随累计RL成本单调递增。最终评估中，Pro版本在DeepSWE v1.1上达到71.9，Flash版本达到67.9，对比Claude Fable 5为70.0，GPT-5.6 Sol为73.0。在MiMo Code Bench上，Pro以63.2分超越GPT-5.6 Sol的59.3分，但低于Claude Opus 5的68.6分。在AutomationBench v1.0.6上，Pro以53.1分超过Claude Opus 5（50.3）和GPT-5.6 Sol（45.8）。在Cybersecurity领域，Pro在CyberGym上达到94.0分，较前代MiMo-V2.5 Pro的40.0分提升超过一倍；但在ExploitGym上仅获17.8分，显著低于Claude Fable 5的28.4分。

Router冻结实验揭示了MoE架构在RL训练中的脆弱性：当router可训练时，专家负载的变异系数在20步内从0.78升至2.0，冷专家比例从0.5%升至22%。而将step-20 checkpoint的router参数恢复至初始值后，负载平衡恢复但benchmark性能不变，证明崩溃源于router漂移而非专家权重劣化。冻结router后，CV保持约0.7，peak load稳定在5.5倍均值。

开源验证方面，MiMo-V2.6-Distill-Qwen-9B在SFT后，SWE-bench Pro从Qwen3.5-9B的32.0提升至44.6；经单harness GRPO后进一步升至47.6。多harness RL则在7个harness上全面超越SFT基线，MiMo Code Bench（mini）的均值从53.1提升至59.0。

## 展望

MiMo-V2.6的工作表明，RL计算规模的扩展并非简单的GPU堆叠，而是需要在架构稳定性、环境多样性、评分精度与基础设施吞吐之间建立精细的协同。论文开源的Distill模型、RL环境与mini-harness框架，为社区提供了一个可复现的Agentic RL研究起点。当router冻结成为MoE-RL的默认选择，当groupwise grading从外部工具演变为训练流程的内在环节，RL驱动的模型自我提升正在从演示性的可能性走向系统化的工程实践。这一范式的边界在哪里——是评分器的信息上限，还是异步训练的一致性极限——仍然是一个开放的问题。

**论文标题**：MiMo-V2.6: Scaling Reinforcement Learning Towards Self-Improvement

欢迎投稿！欢迎合作！