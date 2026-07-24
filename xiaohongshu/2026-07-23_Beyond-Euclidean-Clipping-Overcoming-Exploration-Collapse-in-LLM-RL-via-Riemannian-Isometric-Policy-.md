RIPO：打破PPO-Clip的几何魔咒！✨

今天给大家带来一篇ICML 2025最新研究，它从黎曼几何角度揭示了LLM强化学习中PPO-Clip导致探索崩溃的根本原因，并提出了RIPO（Riemannian Isometric Policy Optimization）算法，在多个竞赛级推理基准上碾压GRPO、DAPO等主流方法。

🔑关键方法  
1️⃣ 首次从几何视角诊断PPO-Clip的病根：它用欧几里得度量（importance ratio的绝对值）来衡量策略差异，但策略空间本质是黎曼流形，KL散度诱导的几何度量与概率值相关——高概率区域更新激进，低概率区域更新过于保守，导致探索崩溃。  
2️⃣ 提出黎曼等距裁剪（Riemannian Isometric Clip, RIC）：根据当前token概率动态调整clip边界，公式为 epsilon = sqrt(delta / pi_old)，让每个token的更新消耗等量的KL预算，低概率token允许更大更新，高概率token受到限制。  
3️⃣ 完整实现RIPO算法：在GRPO框架基础上替换clip方式，并保留token-level损失聚合，无需价值模型，计算效率与GRPO持平。

💡核心创新  
1️⃣ 理论揭示“几何不匹配”是PPO-Clip失败的本质原因，而非简单的clip范围问题——DAPO提高clip上限只会让高概率token更激进，治标不治本。  
2️⃣ 证明等距更新自动诱导统计同方差性：低概率token的方差贡献被控制在常数级，实现更优的偏差-方差权衡，梯度更新更平滑。  
3️⃣ 在7个竞赛数学基准（AIME24/25、AMC23等）上，RIPO在Qwen3-8B上平均提升35.1%，在AIME24上相对GRPO提升60%，训练前40步就达到GRPO 200步的效果，收敛速度提升5倍。

📊实验效果  
✅ 一致超越GRPO、DAPO、GSPO、GMPO、DCPO等6种基线，在4种不同规模和架构的LLM上均取得最佳平均分。  
✅ 训练过程中熵值稳定下降后保持适度水平，既不像GRPO那样快速坍缩到零，也不像DAPO过度膨胀——探索-利用完美平衡。  
✅ 梯度范数几乎无波动，其他方法频繁出现尖峰，RIPO的优化稳定性突出。  
✅ 在Pass@128测试中，RIPO在AIME-25上达到60%，HMMT-25上达45.3%，突破了基座模型的容量边界。

所以，RL调LLM还在用手调clip？试试从几何视角重新理解策略更新！你觉得“等距更新”这个思路能推广到多模态或机器人领域吗？评论区聊聊你的想法～

论文：Beyond Euclidean Clipping: Overcoming Exploration Collapse in LLM RL via Riemannian Isometric Policy Optimization

欢迎投稿！欢迎合作！