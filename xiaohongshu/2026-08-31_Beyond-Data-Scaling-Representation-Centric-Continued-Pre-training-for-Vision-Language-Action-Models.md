表示优先！VLA预训练新思路

今天给大家带来一篇有点“反直觉”的VLA工作——VLAct。它不讲数据扩展，而是盯着表示质量，把有限机器人轨迹变成可迁移的视觉-动作知识。

🔑 关键方法
1️⃣ 浅层保护+Caption混合：冻结视觉编码器和下半LLM层，只更新上层；同时混入caption数据，防止窄分布轨迹把VLM的通用视觉语言表示冲掉。
2️⃣ 多头连续动作共监督：同一个backbone同时挂OFT、PI、GR00T三个连续动作头，共享latent预测同一动作块，逼backbone不要偏向某个头的解码几何。
3️⃣ 部分统一跨实施例动作空间：gripper维度跨机器人共享，arm维度按实施例保留并mask非活跃维度，周期关节用wrap-aware loss回归。

💡 核心创新
1️⃣ 把VLA继续预训练从“大规模动作拟合”重新定义为“表示学习”问题，backbone本身成为一等设计变量。
2️⃣ 实验发现离散动作监督会丢细粒度时序/幅度信息，单头连续监督会导致头特异表示坍缩，多头共监督才真正可迁移。
3️⃣ 不引入额外adapter、路由或条件解码器，仅靠部分共享布局和mask实现跨实施例语义对齐。

📊 实验效果
✅ LIBERO-Plus总成功率达82.6%，超过ABot-M0的80.5%。
✅ RoboTwin 2.0 Clean 92.5%、Random 90.8%，接近HoloBrain-0等前沿。
✅ RoboDojo成功率7.60%，排第6，超过所有WAM条目。
✅ RoboCasa-GR1只用20%下游数据达49.5%，反超全量GR00T-N1.6的47.6%。
✅ 真机双臂协调平均72.0%，大幅领先基线44.0%。

只用开源数据和16张GPU，就能跟工业级数据规模掰手腕。说明VLA的下一步，可能不只是堆数据，更得会做表示。你觉得VLA该继续卷data scaling，还是转向representation？评论区聊聊。

论文：Beyond Data Scaling: Representation-Centric Continued Pre-training for Vision-Language-Action Models

欢迎投稿！欢迎合作！