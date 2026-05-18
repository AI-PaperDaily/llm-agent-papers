Orchard：开源Agent训练框架新范式

今天给大家带来微软等团队提出的Orchard开源Agent建模框架，它通过轻量可复用的环境服务层，让SWE/GUI/个人助理三类Agent的训练和泛化都大幅提升。

🔑关键方法
1️⃣ 薄环境服务层Orchard Env：基于K8s原生的独立服务，提供沙箱生命周期管理、命令执行、文件操作等原语，与Agent框架、训练器完全解耦，支持任意Docker镜像，单命令延迟仅0.28秒，千个沙箱并行成功率100%
2️⃣ 信用分配SFT：保留训练时未成功轨迹，用回顾式价值估计提取其中“进步片段”作为监督信号，让模型从部分成功后学习，相比仅用成功轨迹提升1.9个点
3️⃣ 平衡自适应推演BAR：RL阶段动态调整每个提示的组大小，优先组装正负奖励比例平衡的轨迹组，避免零方差组浪费计算，提升梯度信息密度

💡核心创新
1️⃣ 环境层单独拆分：将沙箱管理从训练框架、Agent框架中剥离，使同一个环境服务能支撑蒸馏、在线RL、评估等多个阶段，且跨领域复用
2️⃣ 多教师多框架训练：SWE领域用Qwen3.5-397B和MiniMax-M2.5双教师、双Agent框架（OpenHands/mini-swe-agent）收集107K轨迹，训练出的模型跨框架泛化远优于单框架训练
3️⃣ 开源低成本：自托管K8s方案成本仅为E2B/Daytona等商业服务的47%，使用spot实例可低至10%，大大降低学术门槛

📊实验效果
✅ Orchard-SWE（30B MoE）：SFT达64.3%，SFT+RL达67.5% on SWE-bench Verified，超越所有同规模开源模型，逼近10-30倍参数的大模型
✅ Orchard-GUI（4B视觉语言模型）：仅用0.4K蒸馏轨迹+2.2K训练任务，在WebVoyager/Online-Mind2Web/DeepShop平均68.4%，超越此前开源最优，与OpenAI、Gemini闭源系统持平
✅ Orchard-Claw（30B MoE）：仅0.2K合成任务达59.6% pass@3，结合ZeroClaw框架提升至73.9%，接近Claude Opus 4.6

看下来Orchard最打动人的是“一个环境层支撑所有”的设计思路，真正把基础设施变成了可复用的模块。你觉得Agent落地最难的是基础设施还是训练方法？欢迎讨论～

论文：Orchard: An Open-Source Agentic Modeling Framework

欢迎投稿！欢迎合作！