小米MoE把RL三线拉满

今天给大家带来小米的MiMo-V2.6系列，直接把强化学习算力、环境、评分三线拉满，让1T参数的MoE模型在agent任务上稳定自我进化。

🔑关键方法
1️⃣ 超大异步RL：单步1568个prompt，每组16条rollout，一次吞吐2.7~3.7B token，上下文最高1M，用Muown优化器加冻结MoE router保持稳定。
2️⃣ 多域多harness环境：同时覆盖代码、通用工具、视觉设计、网络安全四类任务，用模块化mini-harness做多样交互，提升跨harness泛化。
3️⃣ 群体智能评分：GRS离线把pass方案再打质量分，GAR在线对比同组轨迹并重新分配优势，从“过没过”升级为“过得好不好”。

💡核心创新
1️⃣ 混合SWA+GA MoE架构：滑动窗口注意力和全局注意力交错，搭配轻量ViT/音频编码器，让1M长上下文训练成本可控。
2️⃣ Groupwise Agentic Grading：用组内排名细粒度区分通过方案，把奖励从二元测试释放出来，引导模型输出更短更精准的解法。
3️⃣ 多层防奖励黑客：环境清理加黑客agent对抗筛查加离线审计，并冻结router防止专家负载崩塌，实测黑客比例压到2%以下。

📊实验效果
✅ DeepSWE v1.1：Pro从58.4涨到72.6，Flash从48.7涨到65.7，RL算力堆出持续提升。
✅ AutomationBench v1.0.6：Pro 53.1、Flash 52.3，比上代MiMo-V2.5的16.0翻了几倍。
✅ CyberGym：Pro 94.0，Flash 95.1，而MiMo-V2.5只有40.0。
✅ 多harness泛化：held-out harness平均Pass@1从约50%增至66%，和训练harness差距明显缩小。
✅ 开源9B蒸馏模型RL后SWE-bench Verified 66.2，多harness RL在21个数据集-harness对全部提升。

你们觉得这种“群体评分+防黑客”的RL思路，能直接抄进自己的agent训练吗？

论文：MiMo-V2.6: Scaling Reinforcement Learning Towards Self-Improvement

欢迎投稿！欢迎合作！