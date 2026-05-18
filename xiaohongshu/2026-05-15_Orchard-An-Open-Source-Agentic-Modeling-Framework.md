🛠️ 微软开源Agent新框架Orchard：环境层即服务

今天给大家带来一篇来自微软研究的论文，他们提出了Orchard——一个轻量级、基于Kubernetes的开源Agent训练框架。核心思想是把环境执行层做成独立服务，让数据采集、SFT、RL训练和评估都能共享同一个后端，大幅降低复现和迁移成本。

🔑 关键方法：
1️⃣ 薄环境层Orchard Env：基于K8s的独立分布式沙箱服务，提供容器生命周期管理、命令执行、文件操作等原子接口。通过运行时注入agent的方式支持任意Docker镜像，无需修改镜像即可适配不同任务。
2️⃣ 三域训练方案：Orchard-SWE面向软件工程，用MiniMax-M2.5和Qwen3.5-397B双教师蒸馏107K轨迹，引入信用分配SFT从失败轨迹中提取有效片段进行监督；Orchard-GUI面向浏览器导航，用4B视觉语言模型在仅0.4K蒸馏轨迹上训练；Orchard-Claw面向个人助手，在0.2K合成任务上训练。
3️⃣ 强化学习优化：采用Balanced Adaptive Rollout（BAR）策略，自适应调整每个prompt的正负样本比例，避免零方差组浪费计算资源，提升稀疏奖励场景下的训练效率。

💡 核心创新：
1️⃣ 环境层解耦：将沙箱管理从训练栈中抽离为独立微服务，REST API接口与agent框架、推理后端、任务域均无耦合，实现跨域、跨框架、跨阶段的可复用。
2️⃣ 教师蒸馏+失败轨迹利用：不仅保留成功轨迹进行SFT，还通过回顾性价值估计从失败轨迹中提取“上升段”作为部分监督信号，有效扩展训练数据量。
3️⃣ 多框架训练增强泛化：在SWE场景下故意使用OpenHands和mini-swe-agent两种框架采集数据，训练出的模型对未见过的框架（如Kimi-CLI）仍保持较高性能，避免单框架过拟合。

📊 实验效果：
✅ Orchard-SWE（Qwen3-30B-A3B）在SWE-bench Verified上达到67.5% resolve rate，超越同参数量所有开源模型，逼近10-30倍大的前沿闭源系统。
✅ Orchard-GUI（4B模型）在WebVoyager、Online-Mind2Web、DeepShop三个基准上平均68.4%成功率，超越所有开源方案且持平Gemini/OpenAI闭源系统。
✅ Orchard-Claw在Claw-Eval上Pass@3达59.6%，搭配更强ZeroClaw框架后提升至73.9%，证明端到端训练能更好地利用不同框架特性。

这种将环境层视为基础设施核心的设计思路，能不能成为Agent训练的“Linux内核”？你觉得环境层独立化是未来趋势还是过渡方案？评论区聊聊！

论文：Orchard: An Open-Source Agentic Modeling Framework

欢迎投稿！欢迎合作！