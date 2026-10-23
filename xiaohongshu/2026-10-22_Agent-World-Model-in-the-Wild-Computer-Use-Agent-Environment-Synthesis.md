WildAWM开源造三接口世界

今天给大家带来一篇Agent世界模型新作WildAWM，它不靠闭源模型，纯开源就能批量合成可交互的电脑环境与长程任务，让CUA在GUI、MCP、CLI三种接口里来回切换训练。

🔑关键方法
1️⃣ 世界合成：从用户画像和日程出发，用coding agent逐日构建个人电脑状态，复用技能库生成网站、MCP服务和本地文件，得到640个多日世界状态。
2️⃣ XAI任务合成：基于世界状态生成必须横跨GUI、CLI、MCP三接口的任务，并配套可执行verifier、rubric和hint。
3️⃣ 五道质量门：用异构agent rollout、多级judge一致性、初始不满足、可解性、难度和反事实探针过滤任务，确保可解、有难度、验证可靠。

💡核心创新
1️⃣ 首个纯开源pipeline合成PC级多接口环境，不像很多工作只盯单一GUI或浏览器，且不依赖GPT/Claude等闭源模型。
2️⃣ 提出execution-based质量门，尤其counterfactual probe检查judge是否谄媚，大幅提升verifier可信度。
3️⃣ 把合成world直接用于GRPO在线RL，用checklist部分奖励+格式惩罚，长程任务上小模型也能学。

📊实验效果
✅ 合成341个XAI任务，平均含4.2个子任务、3.1次接口切换，部分需超100步，GPT-6-Astra只解决约60%。
✅ 仅在这些合成世界训练Qwen3.6-35B-A3B，8个OOD benchmark平均+10.58分，覆盖Web、Claw、Computer-Use。
✅ 4B模型平均+8.21分，证明小模型也能跨场景泛化。
✅ 消融显示五个质量门都有贡献，去掉judge consistency或initial unsatisfaction掉点最狠。

整体看，WildAWM把“环境供给”从手工或闭源生成中解放出来，给通用CUA训练提供了一条可复现的scaling路径。你觉得纯开源合成世界，未来能不能替代真实环境做RL？评论区聊聊。

论文：Agent World Model in the Wild: Computer-Use Agent Environment Synthesis

欢迎投稿！欢迎合作！