AI造AI！闭环移动规划登顶

今天给大家带来阿里MAI团队的Qwen-Planner-Agent——一个用AI训练AI的闭环框架，通过数据飞轮、能力感知强化学习和模型-Harness协同演化，在移动规划基准MobilePA-Bench上拿下77.05%的SOTA成绩，成本还超低。

🔑 关键方法
1️⃣ AI for Data：智能体驱动数据飞轮，用专用agent构建任务、收集交互轨迹，并根据训练反馈调整数据采样与生成。
2️⃣ AI for Training：混合环境在线强化学习，用CARE（能力感知奖励与优势工程）根据当前成功率动态切换奖励策略。
3️⃣ AI for Harness：统一模型-Harness运行时，整合Skills、持久记忆与执行反馈，支持离线诊断与模型-Harness协同演化。

💡 核心创新
1️⃣ 闭环AI-for-AI框架：AI不仅是被开发对象，也参与数据生产、训练配置和部署优化，形成反馈闭环。
2️⃣ CARE机制：能力感知的奖励调度与质量保持优势校准，成功率高时自动转向效率优化，同时防止小效率差异被放大。
3️⃣ 模型-Harness协同演化：训练时交替更新模型参数和Harness指令，修改Harness会改变模型后续学习的数据分布，实现双向适应。

📊 实验效果
✅ MobilePA-Bench总体77.05%，超越GPT-6 Astra(76.84%)、Claude Opus 5(75.71%)等，排名第一。
✅ 相比Qwen基线，27B版本提升9.83个点（67.22%→77.05%），35B-A3B提升15.01个点。
✅ 效率突出：每千任务输出成本约$2.41，远低于其他模型（$3.06-$67.76），且输出token减少32.5%。
✅ 长历史记忆：在BEAM-10M上，加Harness后分数从21.99%提升至67.24%，大幅改善超长上下文检索。
✅ 泛化能力：在多数非移动agentic基准（如BFCL-v4、Toolathlon等）上提升，通用能力（MMLU-Redux、C-Eval等）基本保持。

你觉得“AI自己训练AI”这条路能走多远？移动Agent会是第一个被彻底重塑的领域吗？评论区聊聊你的看法~

论文：Qwen-Planner-Agent: A Closed-Loop AI-for-AI Framework for Real-World Mobile Planner Agents

欢迎投稿！欢迎合作！