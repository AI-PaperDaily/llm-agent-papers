# 微软开源Orchard框架：解构Agent训练基础设施瓶颈，三个领域取得SOTA成绩

## 引言

大模型Agent正成为解决复杂任务的核心范式，从软件工程到网页导航，从个人助手到工具调用，Agent的能力边界在持续拓展。然而，开源研究在这一领域的进展却始终受到基础设施和训练流程的制约。许多高性能Agent系统依赖闭源代码库、模型或服务，而现有开源框架大多聚焦于Agent编排和框架设计，而非通过可扩展的模型训练来提升大模型的Agent能力。

这种现状揭示了一个深层问题：**环境层**作为Agent训练的基础组件，其封闭性或与特定训练栈的强耦合，导致上层的数据、训练配方和评估流程难以复用。微软研究院联合哥伦比亚大学和UIUC的研究者，提出了Orchard——一个面向可扩展Agent建模的开源框架，其核心设计理念是：将环境层作为轻量级、可复用的独立服务，从而释放整个Agent生态的创新能力。

## 一、核心方法：解构与复用

Orchard框架的核心洞察在于，**环境层不应是Agent训练栈中的“附属品”，而应是支撑一切Agent活动的“基础设施”**。研究者设计了一个名为Orchard Env的Kubernetes-native环境服务，它通过最小化API接口实现跨领域、跨框架、跨流程的复用。

### 1. Orchard Env：轻量级环境服务层

Orchard Env是一个三层架构的服务，包含：客户端SDK、编排器（Orchestrator）和Pod内Agent（In-Pod Agent）。

![图2：Orchard框架总览，显示Orchard Env作为核心组件](https://arxiv.org/html/2605.15040/x3.png)

这个架构的设计选择非常精妙：
- **控制面与数据面分离**：Pod创建/删除等控制操作通过Kubernetes API Server完成，而命令执行、文件读写等热路径操作则直接路由到Pod IP，避免了kubectl exec或WebSocket的开销。
- **运行时Agent注入**：通过Kubernetes Init Container机制，在Pod启动时将执行Agent注入到用户提供的Docker镜像中，无需针对每个任务镜像进行修改。
- **标准Kubernetes原生**：整个服务运行在标准Kubernetes之上，继承了云原生生态的工具链、多云可移植性和成本优化能力（如集群自动扩缩容、Spot实例）。

### 2. 架构细节与设计权衡

![图3：Orchard Env三层架构](https://arxiv.org/html/2605.15040/figures/orchard-architecture.png)

Orchard Env的三层分离体现了三个关键设计选择：
1. **独立扩缩**：编排器和Pod内Agent可以独立部署和扩缩，生命周期决策（创建、删除、就绪）通过中央编排器流转，而命令执行流量则直接分发到每个沙箱的Pod内Agent。
2. **镜像兼容性**：Pod内Agent在运行时注入，而非构建时固化，使得任意任务镜像无需修改即可集成。
3. **生态兼容性**：整个服务运行在标准Kubernetes原语之上，继承开放生态工具链、多云可移植性和成本优化能力。

### 3. 三套Agent训练配方

在Orchard Env之上，研究者构建了三套针对不同领域的Agent训练配方，每一套都展示了环境层的复用能力。

#### Orchard-SWE：软件工程Agent

在软件工程领域，Orchard-SWE解决了两大瓶颈：有限监督和稀疏奖励。研究者收集了来自MiniMax-M2.5和Qwen3.5-397B的107K条轨迹，并通过**信用分配SFT**（credit-assignment SFT）和**平衡自适应Rollout**（Balanced Adaptive Rollout, BAR）来优化。

信用分配SFT的核心思想是：不仅使用成功的轨迹进行监督，还从失败的轨迹中提取“上升片段”（rise segments）——那些虽然最终失败但任务取得进展的部分。通过回顾式价值估计，研究者可以识别出这些片段：

$$V(s_{t})\;=\;\mathbb{P}\bigl(\text{resolve}\,\big|\,h_{t},\,\text{outcome}\bigr)$$

其中$V(s_t)$表示在历史$h_t$下解决任务的概率。通过计算相邻时间步的价值差：

$$c_{t}\;=\;V(s_{t+1})-V(s_{t})$$

提取出价值上升的连续子序列作为训练信号。这种方法使得32,536条失败的轨迹也能贡献有意义的监督信息。

BAR算法则解决了GRPO在长序列Agent任务中的两大问题：当模型对某个提示已经很擅长时，所有轨迹都倾向于成功，导致零方差和零优势；而当模型完全不擅长时，所有轨迹都失败，同样没有信息量。BAR通过逐步生成轨迹直到能够构建一个正负奖励平衡的训练组，确保了每个梯度批次的平均信息密度。

#### Orchard-GUI：浏览器导航Agent

Orchard-GUI展示了环境层在视觉-语言任务中的复用能力。仅使用0.4K蒸馏轨迹和2.2K训练任务，训练一个4B参数的视觉-语言Agent，取得了出色的结果。

#### Orchard-Claw：个人助手Agent

Orchard-Claw聚焦于电子邮件、日历等日常工具使用任务，仅使用0.2K合成任务就达到了有竞争力的性能。

## 二、实验与结果

Orchard框架在三个领域的实验结果令人印象深刻。

### 1. Orchard-SWE性能

![图1左：Orchard-SWE与其他模型的性能对比](https://arxiv.org/html/2605.15040/x1.png)

在SWE-bench Verified基准上，Orchard-SWE（基于Qwen3-30B-A3B-Thinking，仅约3B活跃参数）取得了67.5%的解决率，创下了同等规模开源模型的新纪录，同时与10-30倍参数量的前沿MoE系统竞争。

| 模型 | SWE-bench Verified (%) |
|------|:----------------------:|
| Qwen3-30B-A3B-Instruct | 22.0 |
| Qwen3-Coder-30B-A3B-Instruct | 51.6 |
| Orchard-SWE (SFT) | 64.3 |
| **Orchard-SWE (SFT+RL)** | **67.5** |

### 2. 跨框架与跨任务泛化

研究者进一步评估了模型在不同Agent框架和任务上的泛化能力。结果显示，单框架训练的模型（如Scale-SWE、OpenSWE-32B）在未见过框架上的性能急剧下降，而Orchard-SWE保持了更高的鲁棒性。

| 系统 | OpenHands | mini-swe-agent | Kimi-CLI |
|:---:|:---------:|:--------------:|:--------:|
| Scale-SWE | 64.0 | ✗ | ✗ |
| OpenSWE-32B | 62.4 | 54.9 | 3.6 |
| **Orchard-SWE** | **62.1** | **64.3** | **45.0** |

### 3. Orchard-GUI性能

![图1右：Orchard-GUI与其他模型的性能对比](https://arxiv.org/html/2605.15040/x2.png)

在GUI导航任务中，Orchard-GUI的4B模型平均成功率达68.4%，成为最强的开源模型，同时与OpenAI和Google的专有系统竞争。

### 4. 环境层系统评估

Orchard Env的最大优势之一是成本效益。在128个并行沙箱运行240小时的场景下，Orchard Env的成本仅为Daytona和E2B的47%（按需实例），如使用Spot实例更是低至10%。

| 系统 | 成本（$） | 归一化对比 |
|:----:|:--------:|:---------:|
| Daytona | 7,078 | 1.0x |
| E2B | 7,078 | 1.0x |
| **Orchard Env (按需)** | **3,362** | **0.47x** |
| **Orchard Env (Spot)** | **673** | **0.10x** |

## 三、展望

Orchard框架的核心理念——将环境层作为独立、可复用的服务——为Agent研究社区提供了一个全新的基础设施范式。它不仅降低了Agent训练和复现的门槛，更重要的是，它让轨迹数据、训练配方和评估流程能够跨越不同领域、框架和管道阶段自由流动。

当环境层的约束被解除，上层创新将迎来真正的爆发。这或许是开源Agent生态走向成熟的转折点。

**论文标题**：Orchard: An Open-Source Agentic Modeling Framework

欢迎投稿！欢迎合作！