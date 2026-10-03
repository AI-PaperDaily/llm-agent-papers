证据图锚定真实文件训工作Agent

今天给大家带来GraphForge，一个把真实文件、证据图谱和可验证评分串起来的工作Agent训练数据合成框架。

🔑 关键方法
1️⃣ 职业种子控多样性：从O*NET抓职业、详细工作活动和执行模式，按边际覆盖选种子，避免任务扎堆高频职业和泛化分析。
2️⃣ 真实文件工作区构建：搜索Agent按种子实例化公共案例，下载原生文件并去重，给文件打隐藏角色（core/supporting/confuser/ambient），保留真实干扰与冗余。
3️⃣ 证据图编译任务和rubrics：对文件间依赖建图，任务需求和每个评分标准都锚定到具体文件节点，judge按锚点查源证据打分，不是空泛LLM评分。

💡 核心创新
1️⃣ 任务生成和结果验证双双接地真实文件，解决“模型造文件不真实、真文件没验证器”的老大难。
2️⃣ 执行条件一步修订：初始rollout后，revision agent对着原始文件、证据图和轨迹修任务与rubrics，只改需要改的字段，可执行性拉高。
3️⃣ 证据锚定judge可定位：删掉某评分标准引用的工作表，对应分数掉0.377，非目标标准几乎不动，说明它真在核文件而不是说套话。

📊 实验效果
✅ 2,169条GraphForge轨迹SFT Qwen3.6-27B，GDPVal升到1445.7（+65.7 Elo，OpenHands）。
✅ Workspace-Bench-Lite从56.0拉到63.7（+7.7），Claude Code下。
✅ SpreadsheetBench II从10.3跳到24.0（+13.7），跨OpenHands、Codex、Claude Code都涨。
✅ 同一份数据训Qwen3.6-35B-A3B也大涨，说明轨迹泛化不止一个底座。
✅ 再加rubric-guided RFT，三项benchmark继续提升；随机选轨迹反而掉点，证据锚定选择信号确实有用。

你觉得这种“真实文件+证据图谱”的合成路线，会不会成为工作Agent训练的默认方案？评论区聊聊。

论文：GraphForge: Training Working Agents with Graph-Anchored Workspace Synthesis

欢迎投稿！欢迎合作！