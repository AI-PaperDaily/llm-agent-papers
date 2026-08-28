## NUS提出JIT-Agent：实现Agent架构即时进化，DeepSeek-V4-Flash性能超越GPT-5.6

## 一、Agent的瓶颈不在模型，而在“脚手架”

大模型Agent的实际能力由两个紧密耦合的因素决定：产生推理与动作的**基础模型**，以及将其置于闭环执行环境中的**Agent Harness**（代理脚手架）。Harness负责历史留存、局部意图形成、工具/技能暴露、动作执行以及验证与恢复触发。一个强大的模型如果被置于错误的内存、规划器或动作协议之后，也可能失败；反之，一个优秀的Harness只有在模型能理解并遵循时才能解锁能力。因此，Agent智能不是模型权重的固有属性，而是**模型–Harness对**的属性。

当前主流方法采用**Ahead-of-Time (AOT)** 范式：将Harness视为一个持久化工件，在经验流上优化，期望其泛化到未来任务。然而不同任务对Harness的先验需求截然不同：宽搜索任务需要并行证据探索，终端任务偏好精简串行ReAct循环，深度研究任务需要检索证据的工作记忆，编码任务则依赖文件系统状态。优化单一AOT Harness跨异构任务既繁琐又低效。针对这一难题，论文提出了**Just-in-Time (JIT)** 视角：训练一个元代理，在任务到达时即时合成任务特定的Harness，由任意现成的Agentic LLM执行。

## 二、核心方法：Harness智能的可训练化

### 2.1 模块化Harness协议

论文将Agent Harness形式化为一个可组合的、机器可生成的工件，通过固定四模块协议进行治理。每个Harness $\mathbf{h}$ 可分解为：

$$\mathbf{h}=(\mathbf{M},\mathbf{P},\mathbf{A},\mathbf{F})$$

其中 $\mathbf{M}$ 表示记忆模块（如何压缩历史）、$\mathbf{P}$ 表示规划模块（如何形成局部指令）、$\mathbf{A}$ 表示动作模块（如何推进控制循环）、$\mathbf{F}$ 表示能力编排模块（如何协调工具与技能）。所有模块遵循统一协议 $\boldsymbol{\Pi}$，其运行时依赖顺序为 $\mathbf{M}\rightarrow\mathbf{P}\rightarrow\mathbf{F}\rightarrow\mathbf{A}$。这一分解将异构程序转化为可比坐标，使Harness生成成为在类型化、可重组设计空间上的组装问题，而非无约束程序合成。

![图2：JIT-Agent方法概述](https://arxiv.org/html/2608.25593v1/jit_method_overview_final.png)

### 2.2 三阶段训练管线

JIT-Agent是一个**27B参数**的元代理，基于Qwen3.6-27B训练，通过三阶段管线获得三种核心能力：**适应性**（匹配Harness到任务与骨干模型）、**可靠性**（确保可执行行为并在合成失败时恢复）、**可进化性**（将执行反馈转化为更强的未来Harness）。

**Stage I：任务条件定制**。使用一个冻结的更强教师模型 $q_{\phi}$ 在四模块协议下合成任务自适应Harness。每个任务采样三个参考脚手架，教师生成候选Harness，仅保留通过协议验证与执行检查的样本，形成监督微调数据集。同时利用偏好学习，当候选Harness在奖励提升且不牺牲延迟或成本的条件下优于基线时，构建偏好对：

$$\mathbf{h}^{+}\succ_{\tau}\mathbf{h}^{-} \iff r^{+}>r^{-} \land \ell^{+}\leq\ell^{-} \land \kappa^{+}\leq\kappa^{-} \land (\ell^{+}<\ell^{-} \lor \kappa^{+}<\kappa^{-})$$

最终Stage I目标结合生成模仿损失与参考锚定的偏好损失，使模型偏向同时更有效且更高效的Harness。

**Stage II：反馈驱动修复**。Stage I生成的Harness不一定可执行。对于静态或运行时验证失败的样本，教师提出结构化补丁 $\Delta^{(k+1)}$，在最多两轮内修复。保留那些最终变得可执行的修复轨迹，训练JIT-Agent在短视距内根据诊断报告恢复协议有效执行，而非一次性重新合成。这一步将失败样本转化为高杠杆修复监督。

**Stage III：Evo-GDPO在线进化**。该阶段将测试时Harness进化本身作为可训练能力。Evo-GDPO (Evolutionary Group-Decoupled Policy Optimization) 从当前策略采样一组候选Harness，与来自Harness银行的现有前沿设计在相同骨干、预算和评估种子下执行。奖励通道主导，效率通道仅在候选保持奖励不低于现有前沿时激活：

$$R^{\mathrm{rew}}_{i}=r_{i}+\lambda_{\mathrm{evo}}\,[r_{i}-b_{r}]_{+}$$

$$R^{\mathrm{lat}}_{i}=\mathbb{I}[r_{i}\geq b_{r}]\,[b_{\ell}-\bar{\ell}_{i}]_{+}$$

$$R^{\mathrm{cost}}_{i}=\mathbb{I}[r_{i}\geq b_{r}]\,[b_{\kappa}-\bar{\kappa}_{i}]_{+}$$

三个信号分别归一化后合并，最终PPO风格裁剪目标驱动策略更新。训练后，Harness银行保守更新：仅当候选达到或超过当前奖励前沿并在至少一个前沿维度（奖励、延迟或成本）严格改进时才保留。这使测试时Harness改进成为一种训练得到的能力，而非外部搜索启发式。

![图3：JIT-Agent训练管线](https://arxiv.org/html/2608.25593v1/jit_training_pipeline_frontier.png)

## 三、实验：大规模基准验证

论文在九项异构基准上评估JIT-Agent，覆盖深度研究、日常工作、规划和工作区四类任务。主结果表格（Table 3）显示：**JIT-Agent与GLM-5.2结合，在九个基准中的七个排名第一**，包括DeepSearchQA 93.9、AgentIF 69.9、PinchBench 93.3；配备JIT-Agent的DeepSeek-V4-Flash在所有报告基准上超过更强的DeepSeek-V4-Pro，平均优势8.7分。直接匹配的18对骨干–基准中，JIT生成Harness全部优于默认脚手架，GLM-5.2平均提升7.7分，DeepSeek-V4-Flash平均提升8.8分。

![图1：四个代表性Agent基准的排行榜](https://arxiv.org/html/2608.25593v1/four_benchmark_leaderboard.png)

与先进固定Harness（Claude Code、Codex、OpenCode、Hermes、NanoBot）的受控对比中，**JIT-Agent在六个设置中四个取得最高性能，所有六个设置中拥有最低的token消耗和API成本**。以DeepSeek-V4-Flash为骨干，JIT-Agent在DeepSearchQA上得分85.1，成本0.066美元，相比最强固定Harness NanoBot（80.4分，0.131美元）性能提高4.7分，成本降低49.6%。平均成本降低36.0%，证明JIT生成的Harness不是通过更长轨迹购买精度，而是实现了更短、更选择性的执行。

![图4：DeepSearchQA和AgentIF上的成本–性能前沿](https://arxiv.org/html/2608.25593v1/advanced_harness_cost_performance_dsqa_agentif.png)

泛化测试覆盖DeepSeek V4、Qwen3.6、Mimo-V2.5三家族各两个变体，共24对比较中JIT生成Harness全部超过ReAct基线，平均提升7.6分。测试时进化实验显示，Streaming JIT（保留并更新Harness银行）在三项规划/工作区任务流上的累积准确率均高于Static JIT，且成本与工具调用轨迹未明显增加。可视化案例表明，同一生成器能够针对不同任务结构生成截然不同的Harness：为跨应用工件生产生成基于DAG的Palimpsest，为多跳研究生成带递归委派的Trapdoor，证明共享协议约束接口而不限制行为。

## 四、Harness智能：超越模型缩放的新维度

JIT-Agent首次将Harness工程从人工AOT设计转变为可学习、可迁移、可复利的**Harness智能**。其核心价值在于：不改变模型权重、不增加推理计算，仅通过即时优化操作脚手架，即可让高效骨干超越前沿模型，让强模型获得额外增益。这一工作指向一种更长远的**模型–Harness协同设计**范式：未来基础模型将不仅学习在给定Harness内行动，更学习改进它们赖以行动的Harness本身，为自适应接口、可验证运行时修改以及模型与执行系统联合缩放开辟研究议程。
```

**论文标题**：JIT-Agent: Scaling Harness Intelligence via Just-in-Time Harness Evolution

欢迎投稿！欢迎合作！