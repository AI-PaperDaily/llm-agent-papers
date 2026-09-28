## 阿里巴巴发布Qwen-Planner-Agent：MobilePA-Bench登顶，成本仅$2.41，Agent能自我进化了？

当大模型从「聊天框」走向「操作台」，一个核心问题浮出水面：AI能否既是被开发的对象，又是参与构建下一代AI系统的工程师？阿里巴巴MAI团队近期发布的Qwen-Planner-Agent，用一套闭环的AI-for-AI框架给出了实践性回答——在真实移动设备规划基准MobilePA-Bench上以77.05%的总分登顶，超越GPT-6 Astra、Claude Opus 5等闭源旗舰，且每千任务输出成本仅$2.41。

## 一、移动Agent的困局：既要长程规划，又要低成本试错

移动端任务天然具备「长时程、跨应用、状态多变」的特征。完成一个高层级用户目标，Agent需要协调多个App之间的动作依赖、在状态变化中维持上下文、从执行失败中恢复，并最终验证结果是否达成。然而，真实设备交互成本高昂、难以并行扩展，严重制约了开发规模。

论文提出的核心思路是：**用执行证据驱动AI自己改进自己**。具体而言，AI负责解读执行反馈、识别能力缺口，并协调数据生产、模型训练与运行时支撑的更新。整个系统由三个闭环阶段构成：

- **AI for Data**：智能体驱动的数据飞轮，用专门的Agent构造任务、收集交互轨迹、筛选训练数据，并根据训练反馈指导下一轮数据生成。
- **AI for Training**：从规划导向的冷启动出发，进入混合环境在线强化学习，并通过CARE机制自适应地调控奖励与优势信号。
- **AI for Harness**：模型与执行框架协同进化，记忆、技能、工具在运行时被统一编排，同时将结构化反馈与失败轨迹反向输入到模型和框架的迭代中。

![图2：Qwen-Planner-Agent的AI-for-AI生命周期，包含数据、训练、部署三个互连阶段](https://arxiv.org/html/2609.29892v1/ai_for_ai_lifecycle_rename.png)

## 二、CARE：当成功饱和后，让模型学会「省着用」

强化学习训练移动Agent面临一个微妙问题：任务成功率的信号是稀疏的，当模型在某个任务组上已经稳定成功时，标准的组内归一化会将微小的效率差异放大到与成败差异相当的尺度。这会导致模型为了追求短输出而牺牲可靠性，或者反过来——在成功率已经饱和后继续浪费思考token。

论文提出的**Competence-Aware Reward-and-Advantage Engineering（CARE）** 正是针对这一症结。其奖励调度依据组内成功率将任务划分为三个机制：

$$
R_i = s_i + \begin{cases}
\lambda_{\text{prog}} R_{\text{prog},i}, & \bar{s} \in [0, p_{\text{low}}), \quad \text{progress shaping} \\[4pt]
0, & \bar{s} \in [p_{\text{low}}, p_{\text{high}}), \quad \text{outcome consolidation} \\[4pt]
-\lambda_{\text{eff}} e_i, & \bar{s} \in [p_{\text{high}}, 1], \quad \text{efficiency refinement}
\end{cases}
$$

当组内成功率低时，辅助奖励聚焦于可验证的进展；当成功率处于中间区间时，只保留二元成功信号以巩固任务完成；当成功率饱和时，则引入标准化的执行成本惩罚。关键在于，CARE同时对优势计算施加了**质量保持的校准**：在效率优化机制中，优势计算的分母被一个由阈值$p_{\text{high}}$推导出的锚定标准差$\sigma_{\text{anchor}}$所约束，从而避免效率差异的过度放大。

$$
\widehat{A}_i = \begin{cases}
\dfrac{R_i - \bar{R}}{\sigma_R + \epsilon}, & \text{progress or outcome regime} \\[8pt]
\dfrac{R_i - \bar{R}}{\max(\sigma_R, \sigma_{\text{anchor}}) + \epsilon}, & \text{efficiency-refinement regime}
\end{cases}
$$

消融实验证实了该设计的必要性：完整CARE在保持与Vanilla RL相当训练精度的同时，输出token减少了32.5%；而去掉优势校准后，输出虽更短但精度持续下降。

![图4：Planner Model的训练框架，包含SFT冷启动与混合环境在线RL](https://arxiv.org/html/2609.29892v1/training_framework.png)

## 三、数据飞轮与模型-框架协同进化

数据层面，论文构建了**人类把关的智能体数据飞轮**。任务构造Agent将能力需求转化为可执行任务规格，自动化流程收集交互轨迹，策划工作流对数据进行清洗与标注。反馈驱动的精炼机制在任务层面减少已掌握任务的冗余、增加不稳定行为的覆盖率，在能力层面针对开发集表现生成缺失能力的定向任务。

更值得关注的是**模型与Harness的协同进化**。Harness在运行时根据当前工具集选择技能指导、检索持久记忆、传递执行反馈；离线阶段，部署轨迹与开发集反馈经过AI诊断后，分别导向模型侧（任务分解、工具路由、参数接地）和Harness侧（技能指导冲突、内存策略、检索机制）的更新。

$$
\theta^{(k+1)} = \mathcal{U}_M(\theta^{(k)}; \mathcal{B}^{(k)}, \eta^{(k)}), \quad \eta^{(k+1)} = \mathcal{U}_H(\eta^{(k)}; \mathcal{F}(\theta^{(k+1)}, \eta^{(k)}))
$$

这一交替更新机制使得Harness的修订不仅影响即时执行效果，还重塑了模型下一轮训练所接触的经验分布。在MobilePA-Internal上，四轮协同进化将总分从模型单体的82.67%推升至88.50%，在通用工具使用基准MCPMark上三轮迭代后从38.00%提升至46.98%。

![图3：AI-for-Data工作流，展示任务构造、轨迹收集与反馈精炼的循环](https://arxiv.org/html/2609.29892v1/ai_for_data_v2.png)

## 四、实验：登顶MobilePA-Bench，长程记忆优势显著

在包含1700+可执行任务、200+移动工具、13个查询任务域的MobilePA-Bench上，Qwen-Planner-Agent 27B以77.05%的总分位列所有评估模型之首。相较其27B基线模型提升了9.83个百分点，在工具使用（77.79%）、记忆（74.76%）、技能（86.25%）和子Agent协调（59.55%）四个维度上均实现显著进步。

| 模型/系统 | 总体 | 工具使用 | 记忆 | 技能 | 子Agent |
|---|---|---|---|---|---|
| GPT 6 Astra | 76.84 | 75.71 | 74.73 | 93.25 | 53.93 |
| Claude Opus 5 | 75.71 | 77.60 | 71.81 | 83.00 | 59.55 |
| GLM 5.3 | 73.88 | 76.44 | 72.07 | 77.00 | 58.43 |
| **Qwen-Planner-Agent 27B** | **77.05** | **77.79** | 74.76 | 86.25 | 59.55 |

长历史记忆评估进一步揭示了Harness的独特价值。在BEAM-10M超长上下文设定中，直接上下文推理的最佳结果仅为25.64%，而配备Harness的Qwen-Planner-Agent 27B达到67.24%。值得注意的是，在BEAM-500K、1M和10M三个长程设定上，所有四组匹配的Qwen模型对在Harness加持下全部获得提升。这表明持久记忆系统的优势在历史长度超出上下文窗口时才真正凸显。

![图6：CARE训练动态，完整CARE在保持精度的同时大幅降低输出长度](https://arxiv.org/html/2609.29892v1/Figure/Experiment/vanilla_vs_care_selected_env_training_dynamics.png)

## 五、AI for AI的边界与可能

Qwen-Planner-Agent证明了闭环AI-for-AI框架在具体场景中的可行性。其价值不仅在单一基准上的登顶，更在于展示了一条可扩展路径：让执行经验驱动数据、训练与部署的协同改进，而非孤立的模型提升。当前框架中的人类审核仍不可或缺，尤其在数据精炼与安全关键决策节点。随着模型能力的进一步增强，这层人工屏障是否可能被逐步压缩，将是AI构建AI的下一个关键问题。

**论文标题**：Qwen-Planner-Agent: A Closed-Loop AI-for-AI Framework for Real-World Mobile Planner Agents

欢迎投稿！欢迎合作！