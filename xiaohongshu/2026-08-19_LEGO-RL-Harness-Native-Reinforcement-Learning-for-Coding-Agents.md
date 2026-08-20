**一篇搞定！编码Agent强化学习也能"乐高式"拼接了🧩**

今天给大家带来一个让编码Agent强化学习训练不再"水土不服"的新框架——Lego-RL，它像乐高一样把你现成的Agent工具链直接插进强化学习流水线，不用改内部流程。

🔑关键方法：
1️⃣ 进程内LLM代理：在不改动Agent harness内部逻辑的前提下，通过代理精确捕获生成时的原始token流、对数概率和响应掩码，搞定训练与推理不一致的痛点
2️⃣ 可扩展沙箱编排：自带镜像缓存和分阶段防御机制，专门对抗reward hacking，同时用终止感知过滤机制把基础设施故障的轨迹排除在优化之外
3️⃣ 闭环可观测训练：集成自动化验证和监控插件+Live UI可视化面板，能细粒度诊断每个轨迹、每个终止原因

💡核心创新：
1️⃣ 忠实优化（Faithful Optimization）：针对稀疏MoE模型，还能在训练时重放rollout时的专家路由决策，保证rollout与训练概率相关性稳定在0.99以上
2️⃣ 无需修改Agent内部流程：只需一个轻量适配器连接harness和推理服务，OpenHands SDK、Claude Code、OpenCode全都能直接上手
3️⃣ 异步调度+策略过期控制：解决长尾轨迹导致GPU空转问题，实测相同时间同步训练跑3步，异步能跑7步

📊实验效果：
✅ 在SWE-bench Verified上，OpenHands SDK从64.0%提升到70.4%
✅ Claude Code从62.4%提升到68.2%，OpenCode从57.2%飙到66.6%
✅ 训练过程中策略熵保持稳定不崩塌，但平均响应长度显著增长，说明模型学会了更长的推理链

你们觉得编码Agent的RL训练，最大的拦路虎是环境可靠性还是训练一致性问题？评论区聊聊👇

论文：LEGO-RL: Harness-Native Reinforcement Learning for Coding Agents

欢迎投稿！欢迎合作！