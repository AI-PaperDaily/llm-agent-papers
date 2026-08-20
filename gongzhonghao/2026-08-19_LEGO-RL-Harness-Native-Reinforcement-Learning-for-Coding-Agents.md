## 华为发布Lego-RL：原生Harness训练框架，编码Agent性能暴涨9.4%？

训练编码智能体（Coding Agent）进行强化学习（RL）时，开发者长期面临一个尴尬的矛盾：主流的Agent框架（如OpenHands SDK、Claude Code）虽然功能强大，但它们内部的提示词构造、上下文压缩、历史重写机制，与策略梯度优化所需的精确token级对齐存在天然冲突。为了适配RL框架，开发者往往被迫改造Agent框架，甚至重写其控制流，不仅费时费力，还容易引入训练与推理不一致的问题。

华为联合香港中文大学提出了一种全新框架**Lego-RL**，在不修改原生Agent框架内部逻辑的前提下，实现了高效、忠实、可观测的策略梯度训练。在SWE-bench Verified基准上，该方法将三种主流编码Agent的解决率最高提升了**9.4个百分点**，并保证了训练过程与rollout过程的高度一致性（相关性高于0.99）。

### 一、核心方法

Lego-RL的核心思想是**“Harness-Native”**：将Agent框架视为环境的一部分，只优化其中模型策略部分。框架通过三大支柱解决了原生Harness接入RL训练的关键障碍。

#### 1. 忠实优化：In-Process代理与路由重放

这是Lego-RL最核心的技术创新。当Agent Harness发起模型调用时，Lego-RL通过一个**进程内代理（In-Process Proxy）**在模型服务边界直接捕获精确的生成token、概率和响应掩码，而非事后从轨迹中重建（重建往往因上下文压缩、历史重写而失真）。

论文还特别指出，针对稀疏混合专家模型（MoE），仅仅捕获token还不够。训练时必须**重放（Replay）rollout时的专家路由决策**，才能保证训练侧对数概率与推理侧完全一致。Lego-RL的训练目标采用GSPO（Group Sequence Policy Optimization）：

$$\mathcal{J}_{\mathrm{GSPO}}(\theta) = \mathbb{E}\left[\frac{1}{G}\sum_{i=1}^{G}w(\tau_i)\,\min\!\left(\sigma_i(\theta)\hat{A}_i,\;\mathrm{clip}\left(\sigma_i(\theta),1-\epsilon_{\mathrm{low}},1+\epsilon_{\mathrm{high}}\right)\hat{A}_i\right)\right]$$

![图1：Lego-RL训练基础设施总览](https://arxiv.org/html/2608.17393v1/SWE-Lego-RL-Trainer-framework-v2.png)

其中$\sigma_i(\theta)$表示序列级概率比，$w(\tau_i)$用于过滤因执行故障导致的无效轨迹。该机制允许对于同一任务的多个轨迹计算组内相对优势（Group-Relative Advantage），从而获得稳定的学习信号。

#### 2. 可靠执行：奖励完整性与异步调度

编码Agent的RL训练中，高达90%以上的时间花在Agent执行上，且耗时分布存在显著的长尾效应。Lego-RL设计了**全异步rollout调度**，避免同步批处理中“最慢者决定整体速度”的问题。同时，框架内置了多层奖励完整性防御：

- **网络隔离**：通过特权sidecar控制网络访问，阻止Agent访问测试依赖或篡改验证逻辑；
- **仓库历史隐藏**：在Agent执行期间隐藏Git历史，防止通过diff等方式作弊；
- **镜像Cache与懒加载**：基于Nydus快照技术，将冷启动镜像拉取延迟降低1.7倍，峰值延迟降低23倍。

此外，对于执行失败或被截断的轨迹，Lego-RL会通过**终止感知的轨迹准入机制**进行过滤，确保进入优化器的奖励信号真实反映策略行为，而非环境或基础设施故障。

#### 3. 可观训练：闭环工作流与Live UI

Lego-RL构建了一个五阶段闭环工作流：数据准备、运行验证、训练运行、实时监控与人工审查。其**Live UI**将训练指标与任务级、轨迹级证据关联起来，能回答“为什么奖励下降了”“哪个任务导致策略退化”等问题。

![图2：Lego-RL闭环操作工作流](https://arxiv.org/html/2608.17393v1/swe_lego_live_rl_framework.png)

### 二、实验结果

Lego-RL使用稀疏MoE模型Qwen3.5-35B-A3B，在三个主流Agent Harness上进行了验证。训练数据集来自OpenSWE候选池，经过静态过滤、构建验证和rollout难度筛选后，最终得到2,699个训练任务（与SWE-bench Verified完全不相交）。

#### 主要性能提升

在SWE-bench Verified基准上，三个Harness均获得显著性能提升：

| 编码Agent | 初始解决率 | 训练后解决率 | 提升幅度 |
|:---|:---:|:---:|:---:|
| OpenHands SDK | 64.0% | **70.4%** | +6.4% |
| Claude Code | 62.4% | **68.2%** | +5.8% |
| OpenCode | 57.2% | **66.6%** | +9.4% |

![图3：三种编码Agent的训练行为对比](https://arxiv.org/html/2608.17393v1/swe_lego_live_rl_framework.png)

#### 训练一致性验证

论文提出，忠实优化的核心指标是rollout侧与训练侧的对数概率一致性。Lego-RL在三个Harness上中位数Pearson相关系数均不低于0.998，p99层面的token对数概率平均差异低于$3\times10^{-3}$。对于MoE模型，路由重放机制将一致性从0.9946提升至0.9993，对数概率差异从0.0062降低到0.0025。

#### 任务难度筛选

研究者还通过消融实验证明了**策略相对难度筛选**的重要性。使用未筛选的任务池进行训练，72.7%的任务从未被模型解决，13.4%的任务始终被解决——这两类任务在组内相对优势估计中无法提供任何学习信号。

![图4：不同难度筛选策略下的验证集性能](https://arxiv.org/html/2608.17393v1/swe_lego_live_rl_framework.png)

在完全相同配置下，使用完整难度区间的951任务池，后激活验证平均解决率达0.671；而使用未筛选任务池的对照组，验证解决率始终停留在初始水平。

### 三、展望

Lego-RL的价值在于它解决了一个长期被忽视的问题：**Agent的固有控制流设计本身，就是RL优化的一部分**。该方法让我家更清晰地认识到，编码Agent的RL系统设计需要同时兼顾可靠性、忠信性与可观测性。随着智能体在软件工程等复杂场景中的应用越来越深入，如何让训练基础设施适配Agent生态，而非让Agent适配训练框架，将成为未来RL系统设计的关键方向。Lego-RL这一技术路线为强化学习在复杂Agent场景落地提供了新的重要思路。

**论文标题**：LEGO-RL: Harness-Native Reinforcement Learning for Coding Agents

欢迎投稿！欢迎合作！