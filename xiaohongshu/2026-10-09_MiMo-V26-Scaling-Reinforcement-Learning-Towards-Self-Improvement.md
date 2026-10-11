混合注意力MoE的RL进化术

今天给大家带来小米MiMo-V2.6系列，一个1.02T总参/42B激活的Pro和310B总参/15B激活的Flash，用大规模RL硬生生把Agent能力往上顶。

🔑 关键方法
1️⃣ 混合注意力MoE底座：局部滑动窗口+全局注意力交错，配合视觉/音频编码器，上下文直接支持到1M token，给长程Agent任务留足探索空间。
2️⃣ RL三轴缩放：异步训练单步吞下1568个prompt、约2.5万条轨迹和2.7-3.7B token；同时扩展代码、通用、视觉、网络安全等多环境多harness。
3️⃣ Groupwise Agentic Grading：不再只看测试过不过，GRS离线打分和GAR在线优势重分配会对比同一组方案，更细地奖励高质量、短token的解。

💡 核心创新
1️⃣ 冻结MoE router保稳定：不冻router，RL训练20步内专家负载CV从0.78涨到2.0，冷专家占比飙到22%；冻结后负载平稳，分数照涨。
2️⃣ 三层防奖励黑客：训练前清理泄露产物，专用hack agent对抗式找漏洞，训练中审计轨迹，确认作弊直接奖励置零。
3️⃣ 多harness训练用可组合mini-harness：训练时跨不同轻量框架，模型在held-out harness上平均Pass@1从约50%拉到66%，没被单一框架绑死。

📊 实验效果
✅ DeepSWE v1.1：Pro从58.4提升至72.6，Flash从48.7到65.7。
✅ AutomationBench、视觉编码、网络安全等基准在RL过程中持续稳步攀升。
✅ 冻结router后专家负载保持CV≈0.7，训练稳定性很顶。
✅ 开源蒸馏9B模型在多个环境继续RL，SWE-bench Verified从61.1到66.2，Terminal Bench 2.1从37.1到52.8。

整体看，这篇把RL的batch、环境、grader三路同时拉满，还配套了防黑客和负载稳定的工程细节，确实给Agent自改进路线提供了一个可复现底座。

如果你是负责RL预算的人，会优先砸batch size、环境数量还是grader算力？

论文：MiMo-V2.6: Scaling Reinforcement Learning Towards Self-Improvement

欢迎投稿！欢迎合作！