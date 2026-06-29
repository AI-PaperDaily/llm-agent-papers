今天给大家带来一篇关于RL后训练隐藏的“免费午餐”的论文，证明策略与参考策略的log概率比可以直接用作步骤级优势信号，省去昂贵的奖励模型训练。

🔑关键方法
1️⃣ 理论推导：在通用随机MDP下，RL训练后策略与参考策略的log概率比恰好等于最优优势函数，称为Progress Advantage。
2️⃣ 免标注计算：该信号直接来自标准RL后训练检查点对（如Qwen3.5-9B和Qwen3.5-9B-Base），无需人工标注、蒙特卡洛估计或额外训练。
3️⃣ 算法普适性：适用于含显式KL惩罚（PPO、GRPO）和仅用裁剪替代（DAPO）的主流RL算法，理论证明裁剪目标隐含等效KL正则化。

💡核心创新
1️⃣ 首次将隐式奖励公式从确定性推理环境扩展到随机代理交互环境，解决工具调用、用户反馈等带来的非确定性挑战。
2️⃣ 提出Progress Advantage作为通用、免任务训练的步骤级评估信号，在测试时扩展、不确定性量化、故障归因三个场景中直接可用。
3️⃣ 证明该信号超越了昂贵的预训练奖励模型（如WildReward、ThinkPRM）和置信度基线（Self-Certainty、DeepConf），且能跨模型评估其他策略的轨迹。

📊实验效果
✅ 测试时扩展（best-of-8采样）：在BFCLv4-MT、WebShop、AgentDojo、τ²-bench四个基准上，Progress Advantage平均提升15.5%（Gemma4-4B）和11.3%（Qwen3.5-9B），大幅优于所有基线，甚至超过专门训练的AgentPRM-7B。
✅ 不确定性量化：在τ²-bench Airline上AUROC达0.865（Gemma4-4B），超过Sonnet-4.6（0.615）和WildReward（0.312）；且能用Gemma4-4B准确预测Qwen3.5-9B轨迹的成功率（AUROC 0.754）。
✅ 故障归因：在Who & When多代理系统中，步骤级错误定位精度接近专门训练的AgenTracer，在手写合成场景中与其持平。

你平时训练LLM代理时，步骤级奖励是不是最难搞的部分？这个免费的Progress Advantage信号你打算试试吗？

论文：Neglected Free Lunch from Post-training: Progress Advantage for LLM Agents

欢迎投稿！欢迎合作！