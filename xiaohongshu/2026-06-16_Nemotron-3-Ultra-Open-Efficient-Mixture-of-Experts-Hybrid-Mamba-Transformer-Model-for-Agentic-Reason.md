【MoE混合Mamba-Transformer 推理加速6倍】

今天给大家带来NVIDIA最新开源的Nemotron 3 Ultra模型，550B总参、55B活跃参数的MoE混合Mamba-Attention架构，专为长程Agent推理设计，推理吞吐最高提升5.9倍，准确率持平SOTA。

🔑 关键方法  
1️⃣ MoE混合Mamba-Attention：用Mamba状态空间层替代部分注意力层，大幅降低KV缓存和注意力计算开销，配合LatentMoE提升每参数效率。  
2️⃣ 多Token预测（MTP）与NVFP4预训练：MTP头支持投机解码，NVFP4混合精度预训练20T token，保持稳定且节省显存。  
3️⃣ 多教师同策略蒸馏MOPD+多环境RLVR：训练十余个专用教师模型（代码、搜索、终端等），对学生生成的轨迹进行密集token级指导，迭代两轮提升Agent能力。

💡 核心创新  
1️⃣ 推理预算控制与中等努力模式：支持推理时动态调整思考长度，在准确率和效率之间灵活权衡，2.5倍token节省仅损失7%准确率。  
2️⃣ 从预训练到后训练全流程开源：基座、后训练、NVFP4量化检查点，以及训练数据、配方、RL环境全部公开，支持1M上下文。  
3️⃣ MOPD两轮迭代+自教师机制：第二轮MOPD从学生初始化新教师，融合自我教师信号，在Agent任务上恢复教师性能86%-173%，甚至超越部分教师。

📊 实验效果  
✅ 推理吞吐：对比GLM-5.1、Kimi-K2.6、Qwen-3.5，在8K输入/64K输出场景下分别提速5.9倍、4.8倍、1.6倍，准确率互有胜负。  
✅ Agent基准：SWE-Bench Verified 70.7%，Terminal Bench 56.4%，GDPVal 46.7%，BrowseComp 44.4%，PinchBench 90.0%，均处于公开模型第一梯队。  
✅ 推理与知识：IOI 2025编程竞赛570分（人类前三），IMOAnswerBench 92.3%，MMLU-Pro 86.8%，OmniScience非幻觉率78.7%最高。

你觉得混合Mamba架构会成为下一代大模型的主流吗？评论区聊聊～

论文：Nemotron 3 Ultra: Open, Efficient Mixture-of-Experts Hybrid Mamba-Transformer Model for Agentic Reasoning

欢迎投稿！欢迎合作！