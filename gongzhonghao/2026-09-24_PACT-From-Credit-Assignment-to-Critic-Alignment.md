# AllSpark推出PACT：重构Token级信用分配，数学推理72.87%超GRPO 8.8点

## 一、引言

大模型强化学习后训练中，奖励通常是整条轨迹结束后给出的标量结果，但优化更新发生在每个生成 token 上。这种粒度错配带来一个核心问题：最终奖励应如何分配到轨迹中的每个 token？如果分配不当，critic 估计误差可能淹没微小的局部信用信号，导致 PPO 类方法在长程推理与智能体任务中训练不稳定甚至策略崩溃。

AllSpark 团队在 PACT 论文中从数学上回答了这一难题：研究者提出三条正则条件，唯一确定 token 级信用表征，并据此设计 **Actor-then-Critic** 训练流程，让 critic 对齐更新后的 actor。最终该方法在四项数学推理基准上取得 **72.87%** 平均精度，超出 GRPO **8.8** 个百分点；在 SWE-bench Verified 上达到 **67.4%** 的 pass@1。

## 二、唯一信用表征：条件奖励预测之差

论文首先为 token 级信用定义三条正则条件：**完备性**、**前缀一致性**与**中性**。完备性要求所有 token 信用之和等于最终奖励相对初始预测的偏离；前缀一致性要求已生成前缀的累计信用不随未来信息改变；中性要求每个 token 在已知前序信息下的期望信用为零。

在这三条条件下，论文证明 token 级信用存在且唯一，并可表示为条件奖励预测之差：

$$C_i = V_i - V_{i-1} = \mathbb{E}[R|\mathcal{F}_i] - \mathbb{E}[R|\mathcal{F}_{i-1}]$$

其中 $V_i$ 表示第 $i$ 个 token 及环境观察揭示后的条件奖励期望，$\mathcal{F}_i$ 表示可用信息。该表征说明，信用不是额外的可调量，而是奖励预测随信息增加的增量。该结果同时为响应级信用与 turn 级信用提供统一说明：它们只是 token 级信用在连续片段上的聚合。

## 三、已有算法的统一解释

基于唯一信用表征，论文重新审视了 OPD、RLOO 与 GAE。在理想教师设定下，OPD 的 token 级信号在期望上等价于信用表征产生的策略梯度，因此 OPD 教师实际上是一个隐式 critic。RLOO 尽管使用响应级 baseline，但其 token 策略梯度估计在期望意义下同样等同于信用表征。

更关键的是 GAE 的误差结构。若 critic 估计为 $\widehat{V}_i = V_i + \varepsilon_i$，则 GAE 优势估计可分解为：

$$\widehat{A}_t^\lambda = \sum_{i=t}^\tau \lambda^{i-t} C_i - \varepsilon_{t-1} + (1-\lambda)\sum_{i=t}^{\tau-1}\lambda^{i-t}\varepsilon_i$$

当 $\lambda<1$ 时，中间 critic 误差项可能接近甚至超过真实信用；当 $\lambda=1$ 时，该误差项消失，只留下前缀估计误差。这解释了为什么 DeepSeek-R1 等工作在长程任务中选择 $\lambda=1$ 更稳定，也说明 critic 与当前策略错位会严重污染精细信用估计。

## 四、PACT：先 Actor 后 Critic 的同步训练

传统 PPO 中，critic 在 rollout 时使用的是上一轮策略对应的值函数，而 actor 更新后 critic 又滞后一个策略版本。对于依赖连续条件值之差来恢复细小信用的 LLM 训练，这种滞后会显著伤害价值估计。

PACT 改变更新顺序：先完成 actor 更新，再对同一批 rollout 数据做一次前向计算，用更新前后策略的重要性比修正 critic 训练目标：

$$I_t = \prod_{k=t}^\tau \frac{\pi(T_k|\mathcal{F}_{k-1})}{\mu(T_k|\mathcal{F}_{k-1})}$$

critic 训练目标由原始奖励 $R$ 替换为 $I_tR$，从而将值函数从旧策略 $\mu$ 的期望调整为更新后策略 $\pi$ 的期望。同时该方法使用 sigmoid 参数化 critic，并采用 BCE 损失：

$$\mathcal{L}_{BCE}(\phi) = \mathbb{E}[-R\log \widehat{V}_{\phi,i} - (1-R)\log(1-\widehat{V}_{\phi,i})]$$

相比 MSE，BCE 在值预测上具有更快的收敛特性。PACT 在理论上缓解了 critic 与 actor 的错位，在实现上仅需一次额外前向计算，无需新增 rollout 数据。

![图2：PPO与PACT训练流程对比](https://arxiv.org/html/2509.21357/figures/2.png)

## 五、实验结果

在数学推理任务中，研究者基于 Qwen3.5-4B 与 OpenCode，在 DAPO-Math-17k 子集上训练。PACT 在 AIME 2025、AIME 2026、BeyondAIME 与 HMMT Nov. 2025 四项基准上全部取得最高精度，平均准确率 72.87%。

| 方法 | AIME 2025 | AIME 2026 | BeyondAIME | HMMT Nov. 2025 | 平均 |
| --- | --- | --- | --- | --- | --- |
| Base Model | 46.67 | 51.25 | 28.63 | 37.50 | 41.01 |
| GRPO | 76.50 | 74.78 | 41.81 | 63.19 | 64.07 |
| PPO $\lambda=0.95$ | 32.33 | 29.38 | 17.86 | 25.67 | 26.31 |
| PPO $\lambda=1.0$ | 66.04 | 73.96 | 40.31 | 58.54 | 59.71 |
| SAO | 51.25 | 63.33 | 36.63 | 53.33 | 51.14 |
| PACT w/o IS | 76.04 | 82.50 | 51.38 | 61.04 | 67.74 |
| PACT | **83.12** | **85.21** | **51.69** | **71.46** | **72.87** |

![图1：PACT与GRPO、PPO、SAO在数学推理和编程基准上的性能对比](https://arxiv.org/html/2509.21357/figures/1.png)

在 SWE-bench Verified 上，PACT 训练 Qwen3.6-35B-A3B 的 pass@1 达到 67.4%，超过 PPO($\lambda=1.0$) 的 65.0%、GRPO 的 65.4% 与 SAO 的 63.6%。

消融实验进一步显示，在相同 rollout 数据下，BCE critic 相比 MSE critic 获得更低的损失与更大的正负样本值分离；移除重要性采样修正后，PACT w/o IS 的平均准确率降至 67.74%，说明 critic 与更新后 actor 的同步对稳定训练贡献显著。

![图3：固定策略下BCE与MSE critic训练对比](https://arxiv.org/html/2509.21357/figures/3.png)

## 六、展望

PACT 展示了由信用分配理论直接驱动 critic 训练的方法论，为长程推理、Agent 工具调用等场景提供了一条新的 actor-critic 工程路径。未来可以进一步探索低方差重要性比估计、层级信用聚合，以及将唯一信用表征推广到多智能体交互中的信用归因。

**论文标题**：PACT: From Credit Assignment to Critic Alignment

欢迎投稿！欢迎合作！