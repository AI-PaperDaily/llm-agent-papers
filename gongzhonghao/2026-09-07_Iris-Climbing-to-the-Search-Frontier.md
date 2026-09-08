## AllSpark开源Iris搜索Agent：SFT-RL交替训练，BrowseComp达88.6分

搜索Agent需要在动态网页环境中自主规划查询、筛选证据、决定何时停止，这与固定上下文的传统语言模型评测有本质区别。现有系统性能往往被推理时上下文管理（CM）等外围机制干扰，难以看清策略本身的真实能力。论文《Iris: Climbing to the Search Frontier》提出了两种搜索Agent——**Iris-mini（35B-A3B）** 与 **Iris-pro（397B-A17B）**，并完整公开数据构造、训练与评测配方。该方案通过从网页超链接结构逆向构造多跳问题，以SFT-RL交替训练，在BrowseComp、BrowseComp-ZH、DeepSearchQA和HLE四个基准上达到开源最佳水平。尤其关键的是，论文在有无CM两种配置下严格对比，发现CM单项最高可带来21.2分的提升，揭示了多数已有系统可能被高估的搜索智能。

## 一、逆向构造“难且可解”的多跳问题

论文的数据管道分为网页图构建、任务合成与双重验证三个阶段。首先根据种子页面及其外链建立局部子图，蒸馏为紧凑的实体图 $G_e=(V_e,R_e)$。然后生成回答路径至少包含 $N$ 对关系的多跳问题，并将所有非答案实体通过**锚点抽象**改写为描述性指代，防止字符串匹配绕过推理。该约束形式化为：

$$
\mathcal{A}(e):\quad \mathrm{name}(e),\ \mathrm{alias}(e)\notin\mathcal{A}(e)\ \wedge\ \mathcal{A}(e)\ \text{uniquely identifies}\ e,\qquad\forall\,e\in V_{e}\setminus\{y\}.
$$

随后，每个问题必须同时通过难度与可解性双重验证：参考模型在闭卷条件下失败，但在获得实体图作为证据后能够正确回答，只有交集被保留。

## 二、轨迹级粗过滤与轮次级细过滤

强教师模型在ReAct范式下针对每个问题生成轨迹 $\tau$，包含推理、工具调用、观察与最终答案。粗过滤要求轨迹正确、非退化且搜索深度足够。其中非退化检测依靠滑动窗口的压缩比：

$$
\rho_{\text{cr}}(w)=\frac{|w|}{\bigl|\mathrm{zlib}(w)\bigr|},\qquad c_{\text{degen}}(\tau)=\mathbb{I}\!\left[\max_{w}\rho_{\text{cr}}(w)\geq\tau_{\text{cr}}\right].
$$

细过滤则让LLM法官以数据诱导出的失败模式为准则，对每个轮次输出keep或mask，被遮罩轮次仍保留在上下文中但不计入损失。SFT目标为：

$$
\mathcal{L}_{\text{SFT}}(\theta)=-\,\mathbb{E}_{(q,\tau)\sim\mathcal{D}_{\text{sft}}}\sum_{t=1}^{T+1}m_{t}\,\log\pi_{\theta}\!\bigl(u_{t}\mid C_{<t}\bigr),
$$

## 三、RL与SFT-RL攀登

RL阶段使用组相对策略梯度，在实时搜索环境中优化。为解决长程rollout延迟，采用请求级部分展开，已完成前缀在跨策略权重下通过截断重要性采样复用。奖励由内部生成式奖励模型直接给出：

$$
R(q,\tau)=\mathbb{I}\!\left[\operatorname{GenRM}\!\bigl(q,\hat{y}_{\tau},y^{*}\bigr)=\textsc{A}\right],
$$

内部引擎同时承担观察摘要，去除外部API依赖。论文将SFT与RL交替进行，每轮RL后筛选成功且工具调用不少于 $K_{\text{rft}}$ 的最短轨迹，返回下一轮SFT，形成自步课程。

## 四、实验：四个基准上的开源最佳

论文在BrowseComp、BrowseComp-ZH、DeepSearchQA、HLE四个基准上评估，结果如图1所示。

![图1a：BrowseComp性能对比](https://arxiv.org/html/2609.04304v1/BrowseComp.png)
![图1b：BrowseComp-ZH性能对比](https://arxiv.org/html/2609.04304v1/BrowseComp-ZH.png)
![图1c：DeepSearchQA性能对比](https://arxiv.org/html/2609.04304v1/DeepSearchQA.png)
![图1d：HLE性能对比](https://arxiv.org/html/2609.04304v1/HLE.png)

从图1可见，在35B参数级，**Iris-mini** 在BrowseComp（82.2）、BrowseComp-ZH（84.8）、HLE（52.3）三项领先，DeepSearchQA（86.9）略低于XYZ-Aquila-mini的89.5。在~400B参数级，**Iris-pro** 在BrowseComp（88.6）、DeepSearchQA（92.9）、HLE（56.4）三项最高，BrowseComp-ZH与XYZ-Aquila-pro并列85.1。相比XYZ-Aquila-pro，Iris-pro在BrowseComp高出3.8分，HLE高出3.1分。

更值得注意的是无CM与有CM的对比。无CM条件下，Iris-mini在BrowseComp得64.7，BrowseComp-ZH得72.3，已显著超过OpenSeeker-v2（46.0/58.1）和FORT-Searcher（55.9/62.1）。启用discard-all后，Iris-mini在BrowseComp跃升17.5分至82.2，Iris-pro从72.6升至88.6。这表明基础策略本身已具备较强搜索能力，CM只是在长程探索时解除了上下文瓶颈。论文还发现CM增益与任务类型强相关：BrowseComp因长程信息寻求频繁耗尽上下文，增益最大；HLE依赖专家推理，检索并非主要瓶颈，因此增益有限。

## 五、展望：搜索作为原子能力

论文认为搜索更像一种原子能力，而非垂直技能。初步实验显示，搜索数据和搜索特化教师对通用工具使用（BFCL）与协作任务（OfficeQA、APEX）产生正向迁移。未来将探索搜索训练的迁移边界，衡量其在更广泛Agentic场景中的价值。

**论文标题**：Iris: Climbing to the Search Frontier

欢迎投稿！欢迎合作！