GraphForge证据图训练工作Agent

今天给大家带来一篇工作Agent数据合成的新框架GraphForge，它让任务和验证都锚在真实文件上，训练出来的模型在工作类基准上拉高一大截。

🔑关键方法：
1️⃣ 职业种子控多样性。从O*NET职业库抽取职业、详细工作活动、执行模式，按边际覆盖贪心选种子，避免任务坍缩成常见职业和泛泛的分析题。
2️⃣ 真实文件工作空间+证据图。先让搜索Agent抓真实公开文件，按core/supporting/confuser/ambient分配隐藏角色；再让模型构建跨文件依赖的证据图，节点记录文件+事实+角色，边记录跨文件推理关系。
3️⃣ 证据锚定rubric+轨迹筛选。任务书和评分rubric都从证据图编译，每条评分标准锚到具体文件节点；跑一遍初始rollout后让revision agent修补，最后用证据锚定judge打分，只留高质量轨迹。

💡核心创新：
1️⃣ 把“任务可验证性”做成了文件级证据锚定。每条rubric都能回到原始文件核查，不再是只看环境状态或没有细则。
2️⃣ 职业种子和真实文件分工明确：种子管方向，文件管内容和验证，多样性和真实性不打架。
3️⃣ 证据锚定judge可以当选择器，做rejection fine-tuning时，比无锚定或随机选轨迹更稳。

📊实验效果：
✅ Qwen3.6-27B在GraphForge数据上SFT后，GDPVal拉到1445.7（+65.7），Workspace-Bench-Lite 63.7（+7.7），SpreadsheetBench II 24.0（+13.7）。
✅ Qwen3.6-35B-A3B同样大涨，GDPVal直接+101.7，说明框架跨基座模型可迁移。
✅ 训练用Codex脚手架，评估在OpenHands、Codex、Claude Code上都涨，说明学到的是通用工作能力，不是绑死某个工具习惯。
✅ 污染审计显示训练文件和GDPVal测试文件0重叠，未覆盖职业上反而涨得更明显，基本排除背题。
✅ 证据锚定RFT再进一步，Workspace-Bench-Lite和SpreadsheetBench II继续提升；随机选轨迹则变弱，说明rubric信号确实有用。

整体看，GraphForge走的是“真实文件+证据图”路线，把数据合成里最头疼的真实性和可验证性一起解决了。你们做Agent训练时，任务验证一般怎么搞？会考虑用这类证据锚定rubric吗？

论文：GraphForge: Training Working Agents with Graph-Anchored Workspace Synthesis

欢迎投稿！欢迎合作！