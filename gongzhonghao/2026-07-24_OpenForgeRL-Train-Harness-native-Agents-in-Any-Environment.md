# 哥伦比亚大学&微软发布OpenForge RL：破解Agent规模化RL训练难题，多项基准超越强基模型

## 引言

当前最先进的AI Agent很少是孤立的语言模型；它们被包裹在日益复杂的**推理脚手架（inference harnesses）**中——如Claude Code、Codex和OpenClaw——这些框架管理着多轮交互、工具调用和外部系统连接。然而，这些强大的“马具”也使得Agent难以在开放基础设施上进行端到端训练：现有的SFT/RL堆栈无法原生表达有状态、多进程的脚手架推理过程。论文提出**OpenForge RL**，一个开源框架，通过在标准RL代码库与容器化部署环境之间建立轻量级代理和Kubernetes编排器，首次实现了在任何环境下对基于harness的Agent进行规模化端到端训练。该框架在Claw和GUI两大领域的六个基准测试中，以仅几百到几千个任务的训练量，全面超越同尺寸开源模型，并在GUI任务上匹配或超越若干倍于自身规模的模型。

## 一、核心方法：解耦训练与推理的轻量级基础设施

### 1.1 问题定义与形式化框架

论文将复杂任务形式化为一个马尔可夫决策过程（MDP）⟨𝒮, 𝒜, 𝒯, ℛ, γ⟩。在每一步，基于LLM的Agent πθ接收任务指令和观测s_t，生成动作a_t，并转移到下一观测。

对于复杂任务，Agent被包裹在推理脚手架ℋ(π)中，该脚手架内部提供工具和控制流。论文将这一过程抽象为：每个提示-响应对记为(ℋ(s_t), a_t)，完整轨迹记为：

$$\tau=\left\langle(\mathcal{H}(s_0),a_0),(\mathcal{H}(s_1),a_1),\ldots,(\mathcal{H}(s_T),a_T)\right\rangle$$

### 1.2 两大核心组件

![图2：OpenForge RL概述](./x2.png)

**Component 1：轻量级代理（Proxy）**
该代理拦截harness对模型服务的所有生成请求，将其路由到RL框架的推理引擎，并记录所有IO对。当一次rollout完成时，代理收集终端奖励r_T和所有提示-响应对，重建训练样本：

$$ \tau=(s^{\mathcal{H}}_{0},a_{0},r_{0}),(s^{\mathcal{H}}_{1},a_{1},r_{1}),\ldots,(s^{\mathcal{H}}_{T},a_{T},r_{T});\quad r_{t}=\gamma^{T-t}\cdot r_{T}$$

**Component 2：Kubernetes编排器**
每个rollout在独立的远程容器中运行，编排器负责在云服务商（如微软Azure）上创建、管理和删除这些容器。这种设计解决了两个关键问题：
- **解耦计算**：harness rollouts所需的容器化环境（专用CPU和内存）无需与训练节点共置。
- **弹性伸缩**：可弹性管理并发rollout数量，不增加训练节点负担。

### 1.3 数据合成管道

![图3：数据/任务合成管道](./x3.png)

针对数据稀缺的非编码领域（如日常工具使用、计算机控制），论文构建了一个SFT和RL任务合成管道。该管道模拟人类策划任务的方式：并行生成候选指令、过滤低质量任务、构建可执行环境和验证脚本、通过另一LLM/VLM测试任务并修补缺陷。每个环境由自定义Dockerfile定义，可预装任意harness（如OpenClaw、Codex）。

![图4：任务分布统计](./x4.png)

## 二、实验结果：全面碾压同尺寸模型

### 2.1 Claw Agent：仅需343个任务，性能飙升

使用Qwen3-30B-A3B-Thinking作为骨干模型，论文在三种流行的harness（ZeroClaw、OpenClaw、Codex）上训练了**OpenForge-Claw**模型。

| 模型 | ClawEval (pass@3) | QwenClawBench | MCPAtlas |
|-----|-----|-----|-----|
| Qwen3-30B-A3B-Thinking | 39.8 | 21.8 | 12.4 |
| **OpenForge-Claw (SFT+RL)** | **55.9** | **33.7** | **28.1** |
| Claude Opus 4.6 | 80.8 | 59.5 | 76.4 |

在ClawEval上，SFT+RL相比SFT提升显著（pass@3从52.1提高到55.9，pass^3从21.7提高到31.7），证明了在线交互训练的独特价值。

### 2.2 GUI Agent：超越超大模型

使用Qwen3-VL-8B作为骨干，论文训练了**OpenForge-GUI**模型，仅用2.5k任务就在三个GUI基准测试上取得惊人成绩：

| 模型 | OSWorld-Verified | Online-Mind2Web | WebVoyager |
|-----|-----|-----|-----|
| Qwen3-VL-8B | 29.4 | 38.7 | 49.2 |
| UI-TARS-1.5-7B | 27.4 | 31.3 | 66.4 |
| MolmoWeb-8B | – | 35.3 | 78.2 |
| **OpenForge-GUI (SFT+RL)** | **37.7** | **63.0** | **72.3** |
| GPT 5.4 | 72.7 | 92.8 | – |

OpenForge-GUI在Online-Mind2Web上超越MolmoWeb近30个百分点，在OSWorld-Verified上超越UI-TARS约10个百分点。

## 三、深层洞察：Harness选择如何影响学习？

### 3.1 不同Harness的学习难度差异

论文评估了模型在四种不同复杂度harness上的表现：

| 模型 | ReACT* | ZeroClaw | OpenClaw | Codex |
|-----|-----|-----|-----|-----|
| Base | 39.8 | 44.7 | 19.3 | 18.6 |
| SFT+RL | **55.9** | **67.1** | **27.8** | **51.5** |

关键发现：支持直接添加自定义工具的ReACT和ZeroClaw达到最高性能；SFT+RL在所有harness上带来大幅提升，但OpenClaw仅获得中等增益，其更长提示和上下文可能加大了学习难度。

### 3.2 跨Harness泛化能力

论文训练了两个模型：一个仅用ZeroClaw训练，另一个用三种harness联合训练：

- ZeroClaw训练模型在未见的OpenClaw和Codex上分别提升3.3和4.6
- 多harness联合训练则在所有harness上达到最佳，Codex上提升达20.3

这表明多样化训练能显著提升模型对不同工具和场景的鲁棒性。

### 3.3 RL教会了Agent什么？

通过对100条SFT和SFT+RL轨迹的对比分析，论文发现RL带来三大关键变化：

1. **工具选择优化**：通用shell调用从22.6%下降到13.9%，转向专用服务工具
2. **自验证能力提升**：模型学会在执行后进行自我确认
3. **工具覆盖更广**：R L模型在每个任务中使用更多必要工具

然而，错误恢复能力仍是最薄弱的环节，即使经过RL训练也未显著改善，这成为未来研究的重点方向。

## 四、总结与展望

OpenForge RL通过创新的解耦架构，首次将复杂推理脚手架与标准RL框架无缝连接，使Agent能够在真实部署环境中进行端到端训练。论文在Claw和GUI两大领域、七个基准测试上验证了该方法的有效性，证明仅需少量数据即可训练出超越同尺寸模型、甚至匹敌大模型的Agent。

这一工作为AI Agent领域提供了可扩展的研究基础设施，其开源框架将显著降低研究人员训练和分析基于harness的Agent的门槛。展望未来，如何针对性地强化Agent的错误恢复能力，以及如何进一步优化多harness联合训练策略，将是该方向的重要突破点。

**论文标题**：OpenForgeRL: Train Harness-native Agents in Any Environment

欢迎投稿！欢迎合作！