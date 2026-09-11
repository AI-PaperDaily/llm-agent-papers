122B MoE终端Agent靠RL封神

今天给大家带来一篇终端智能体强化学习的新工作T1，用122B MoE模型在真实Linux沙箱里跑长程任务，靠执行验证器给奖励，硬核拿下Terminal-Bench 2.1的64.0%。

🔑关键方法
1️⃣ TITO（token-in-token-out）：训练时直接用采样器输出的token ID，彻底避开harness重渲染造成的token漂移，实现训练推理token级零误差。
2️⃣ R3（rollout routing replay）：记录推理时MoE每层选中的专家路由，训练时原样重放，消除稀疏模型训练推理路由不一致的坑。
3️⃣ 密集验证奖励：不只看最终成功/失败，而是按通过断言的绝对数量给分，固定全局S=20，部分完成也有梯度信号。

💡核心创新
1️⃣ 稳定MoE强化学习栈：TITO+R3组合把训练-推理log概率差从0.021砍到0.013，loss区域token drift精确为0。
2️⃣ 密集过程奖励：把验证器的每个断言变成可学习的奖励信号，比二元奖励稠密一个量级，训练稳定性肉眼可见提升。
3️⃣ 全OOD训练语料：递归合成任务与评估基准Terminal-Bench 2.1完全隔离，证明是真实能力迁移而非刷榜。

📊实验效果
✅ Terminal-Bench 2.1上：基座43.8% → SFT 49.4% → T1 64.0%，相对提升28.5%，超过GPT-5.4 (54.8%)、DeepSeek-V4-Flash (56.9%)，逼近Claude Opus 4.7 (66.1%)。
✅ Long-Horizon Terminal Bench：T1 27.9%，超过GPT-5.4 (27.2%)和GLM-5.1 (26.7%)，追平Gemini-3.1-Pro。
✅ Terminal-Bench Hard：38.0%，超过DeepSeek-V4-Pro (36.0%)，比SFT提升9.7个点。
✅ 在debugging和系统管理子域分别拿到100%和88.9%，超过GPT-5.6 Sol一大截。

你觉得用真实执行反馈做RL，会不会是下一代Agent的必经之路？评论区聊聊～

论文：T1: Terminal Agent Reinforcement Learning for Long-Horizon Tasks

欢迎投稿！欢迎合作！