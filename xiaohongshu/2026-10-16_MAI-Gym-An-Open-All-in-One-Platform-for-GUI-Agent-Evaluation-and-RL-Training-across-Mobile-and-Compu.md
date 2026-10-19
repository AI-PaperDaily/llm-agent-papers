MAI-Gym：三端统一GUI Agent训练平台

今天给大家带来阿里开源的MAI-Gym，一个打通移动端、电脑端和网页的GUI Agent评估+在线RL训练全栈平台，顺手训出10B以下最强开源GUI智能体。

🔑 关键方法
1️⃣ 共享rollout路径：评估、数据收集、在线RL共用同一套agent harness和观察/动作处理，减少训练和评测不一致的坑。
2️⃣ 环境管理器统一三端接口，支持异步扩展和warm pool，最多跑到20,000个并行环境，重置和故障隔离更稳。
3️⃣ 自动化合成mocked apps与跨应用世界任务，先造连贯的世界状态，再用执行验证的code verifier筛任务。

💡 核心创新
1️⃣ 跨应用世界构建：先把人设、日程、文件、App记录做成一致状态，再生成需要跨应用检索推理的任务，比单文件任务真实很多。
2️⃣ Window-wise history设计：把长轨迹按窗口缓存前缀，prefix cache命中率从42.5%拉到85.9%，训练token直接降84.7%，很适合截图重的GUI RL。
3️⃣ Sim-to-real转移：在12306克隆App上练完，真机任务成功率从28.8%提到48.1%，说明mock环境不白练。

📊 实验效果
✅ MAI-Gym-9B在MobileWorld拿到78.4%，OSWorld-Verified拿到76.2%，登顶10B以下开源GUI Agent。
✅ 超过UI-Venus-2-9B、GUI-Owl-1.5-8B，也压过GUI-Owl-1.5-32B等更大模型。
✅ 联合训练优于分域专家合并，两个主基准都比最佳merge高。
✅ 1000个环境移动端拉起只需328.4秒，重置中位165秒，非评分中断率仅1.2%。
✅ 新App仅靠reward学习，held-out成功率从25.8%升到45.2%，步骤也明显变短。

你们觉得这种mock App练出来的GUI能力，能替代真实环境里的大量试错吗？

论文：MAI-Gym: An Open All-in-One Platform for GUI Agent Evaluation and RL Training across Mobile and Computer Environments

欢迎投稿！欢迎合作！