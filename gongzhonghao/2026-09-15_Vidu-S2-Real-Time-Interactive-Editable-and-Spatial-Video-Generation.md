## 清华与生数科技联合发布Vidu S2：720p实时生成与可编辑空间视频，交互性能全面跃升

实时视频生成正在经历一场从"耐心等待"到"即时交互"的范式转移。当Sora、Veo、Wan等模型将离线生成质量推向新高度的同时，用户依然需要等待数分钟甚至更久才能看到完整结果。面对直播、游戏、虚拟陪伴等强交互场景，毫秒级的响应延迟意味着模型必须具备流式推理能力，而非传统扩散模型的全序列同步去噪。清华大学与生数科技联合推出的Vidu S2，正是对这一技术瓶颈的系统性回应。论文将其拆解为**Vidu S2-Avatar**（实时交互数字人）与**Vidu S2-Editing**（实时视频编辑）两大核心模块，并首次探索了面向VR头显的实时空间视频生成路径。

![图1：Vidu S2整体架构与能力概览](https://arxiv.org/html/2609.11638v1/Vidu_S2_cp.png)

## 一、核心方法：从因果流式生成到自回放训练

如果说Vidu S1证明了实时数字人视频生成的可行性，那么Vidu S2的目标则是在四个维度上完成质变：分辨率从540p跨入720p、参考图像可随时替换、大幅提升指令跟随能力（包括跳舞等大幅身体运动）、以及将能力边界从"数字人"扩展到"视频流编辑"。支撑这些能力跃迁的技术底座，是一套名为**Self-Replay Forcing（SRF）**的训练框架。

在流式生成场景下，模型逐段输出视频，每个片段的条件历史都是模型自身在前序步骤中生成的结果。Teacher Forcing假设历史是干净的、完美的，这导致训练与推理之间存在严重的分布漂移。Diffusion Forcing通过向干净历史注入噪声来缓解这一问题，但噪声历史的构造过程被**切断了梯度流**，无法将后续片段的损失反向传播到前文表示中。论文提出的SRF机制，在自回归rollout完成后，将整个学生生成轨迹独立地重新加噪，并在同一次前向计算中完成梯度回传。其训练损失函数为：

$$ \mathcal{L}_{\mathrm{SRF}}=\mathcal{L}_{\mathrm{DMD}}\left(\boldsymbol{f}_{\boldsymbol{\theta}}^{\mathrm{causal}}\left(\boldsymbol{x}_{\boldsymbol{t}}^{1:N},t,\boldsymbol{r},\boldsymbol{c}^{1:N}\right)\right)+\mathcal{L}_{\mathrm{perc}}\left(\boldsymbol{f}_{\boldsymbol{\theta}}^{\mathrm{causal}}\left(\boldsymbol{x}_{\boldsymbol{t}}^{1:N},t,\boldsymbol{r},\boldsymbol{c}^{1:N}\right)\right) $$

其中 $\boldsymbol{x}_{\boldsymbol{t}}^{1:N}$ 表示按Diffusion Forcing重新加噪后的学生生成轨迹，原始rollout的KV缓存被**梯度截断**，而重放阶段的所有片段保持连接在同一计算图中，允许跨块梯度传播。这一设计的关键价值在于：后续片段的生成质量信号可以反过来优化前序片段的表征，从而抑制长序列流式生成中常见的漂移和崩溃问题。

在模型架构层面，Vidu S2-Avatar采用**音视频联合Diffusion Transformer**，以参考图像 $\boldsymbol{r}$ 和分段条件序列 $\boldsymbol{c}^{1:N}$ 共同驱动首N个片段的去噪预测：

$$ \hat{\boldsymbol{x}}_{0}^{1:N}=\boldsymbol{f}_{\boldsymbol{\theta}}^{\mathrm{avatar}}\left(\boldsymbol{x}_{t}^{1:N},t,\boldsymbol{r},\boldsymbol{c}^{1:N}\right) $$

与生成模型的流式因果化不同，Vidu S2-Editing的核心挑战在于编辑结果必须与源视频的运动和时序严格对齐。论文提出的**帧对齐注意力（Frame-Aligned Attention）**机制规定：每个目标帧只读取同一时间步位置上的源帧信息，同时所有目标帧均可访问参考图像token。这一设计在保持双向训练容量的同时，让模型的注意力模式天然适配后续的因果流式推理。

![图2：Vidu S2-Avatar与Vidu S2-Editing的数据准备流程](https://arxiv.org/html/2609.11638v1/S2_Data_Pipeline_Polished_v11.png)

## 二、数据与推理：720p的硬件现实

将实时生成从540p提升到720p，训练数据的选择策略必须先于模型架构调整。论文发现，名义分辨率是高度不可靠的质量信号：大量1080p视频因重压缩而纹理模糊、伪影严重。为此，研究者构建了一套**多维混合清晰度评估框架**，综合考量分辨率、帧率、编码格式、位深、码率之间的制约关系，并结合专家模型对纹理细节、边缘锐度和压缩伪影进行评分，加权汇总后作为数据筛选依据。

背景漂移是无限长度流式生成的宿敌。舞蹈类视频富含下半身运动和大幅姿态变化，对模型的运动多样性至关重要，但这类视频往往伴随运镜。论文的解决路径是**背景稳定化算子**：对平移有限、旋转和变焦主导的相机运动，通过前背景分割、背景区域特征匹配与几何校正，将动态背景转换为近似固定机位画面。这使得训练数据既能保留丰富的舞蹈动作，又不牺牲背景一致性。

在推理基础设施层面，论文将注意力优化分为两层策略。对于敏感的注意力层，采用SageAttention保精度；对容错率较高的层，则部署SpargeAttention和Sparse-Linear Attention进行更激进的近似。线性层采用per-block W8A8量化GEMM，在细粒度缩放下抑制离群值对量化范围的主导，从而在精度和速度间取得平衡。更进一步，**多GPU上下文并行**采用Ulysses式的序列切分，叠加量化通信来压缩设备间张量交换体积，使VAE编码器、主干网络、Refiner和VAE解码器能够在共享时间线上充分利用GPU资源。Refiner的设计同样值得关注：它从低分辨率潜在表示出发，通过一步去噪完成空间上采样，其历史缓存维持在高噪声水平 $\tau_B$ 以传播粗糙的时序结构，而主干网络的低噪声缓存 $\tau_R$ 则负责保留高频身份细节：

$$ \boldsymbol{z}_{t_r}^{i}=(1-t_r)\mathcal{U}\left(\hat{\boldsymbol{x}}_{0,\mathrm{LR}}^{i}\right)+t_r\boldsymbol{\epsilon}^{i},\qquad\boldsymbol{\epsilon}^{i}\sim\mathcal{N}(\boldsymbol{0},\boldsymbol{I}) $$

## 三、编辑模型与Agentic系统

Vidu S2-Editing覆盖四类编辑任务：风格迁移、虚拟试穿、角色替换和背景替换。训练数据的构造采用条件生成模型作为"数据引擎"：以源视频的表面法线视频为条件，以参考图像为外观引导，训练模型重建原视频，继而替换参考图像为风格化版本，合成风格迁移训练对。对于其余任务，则融合多个开源编辑模型的候选输出，通过**后过滤与对比选择**保留最优的源-编辑视频对。

![图3：Vidu S2-Avatar的Agentic系统管线示意图](https://arxiv.org/html/2609.11638v1/a2a-interface.png)

在用户侧，Vidu S2-Avatar配备了一个**VLM Agentic系统**，负责将用户的口头指令转化为精细的生成提示。系统会从时序维度解析指令："拿起杯子，然后微笑"需要拆分为两个顺序动作。当视觉反馈确认"拿杯"动作完成后，后续提示词会显式维护"持续握持杯子"这一状态约束。Agent对参考图像的分类——手持物、背景还是衣物——决定了提示词的重写策略。

![图4：参考图像控制与配饰操作示例](https://arxiv.org/html/2609.11638v1/reference-control-cases.png)

## 四、实验与性能表现

在Vidu S2-Editing的模型结构上，论文以DiT为骨干，将源视频和参考图像编码为条件token并与噪声目标token拼接，使用条件特定的RoPE编码区分不同token流的时空位置。流式推理场景下，源帧在被对应的目标帧消费后即从缓存中释放，不产生历史累积。

从工程落地角度看，Vidu S2最值得关注的不在于单一指标的刷榜，而在于它把一个**完整的实时视频生成-编辑-空间化栈**压缩到了低成本GPU上。720p实时生成（25~42 FPS）、四类编辑任务的流式处理、以及面向VR头显的双目视图转换，这三者叠加构成了一个此前尚不存在的产品形态：用户上传一张照片，即可与一个跳舞、换装、换场景的数字人在空间视频中持续互动。这种从"生成一段视频"到"维持一段持续对话"的转变，或许才是实时视频生成真正的分水岭。当流式生成的延迟被压缩到人类感知阈值以下，视频模型将不再是工具，而是基础设施。

**论文标题**：Vidu S2: Real-Time Interactive, Editable, and Spatial Video Generation

欢迎投稿！欢迎合作！