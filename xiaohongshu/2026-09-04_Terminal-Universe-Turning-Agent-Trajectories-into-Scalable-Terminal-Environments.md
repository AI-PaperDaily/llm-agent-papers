轨迹反推环境，智能体训练新思路

今天给大家带来阿里Qwen团队Terminal-Universe，核心是把终端Agent轨迹逆向重建为可复用可验证环境，再合成单工作区、跨工作区和多轮任务做SFT。

🔑关键方法
1️⃣ 确定性回放+智能补全：按时间回放轨迹里的读写和编辑操作，把文件恢复到智能体改动前；补全Agent再填缺失文件与依赖，留任务充分的环境。
2️⃣ 四种再查询：意图恢复、单工作区合成、跨工作区依赖出题、多轮用户反馈，把一个环境榨成多条训练轨迹。
3️⃣ 验证器过滤：每个任务配pytest验证器，只保留测试全过的轨迹用于SFT，多轮则做round级验证。

💡核心创新
1️⃣ 轨迹变环境：把静态演示反向当成环境压缩观测，不再从零造repo。
2️⃣ 广度+深度扩展：跨工作区挖方向依赖，多轮用user agent

论文：Terminal-Universe: Turning Agent Trajectories into Scalable Terminal Environments

欢迎投稿！欢迎合作！