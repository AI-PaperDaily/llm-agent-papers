RCCA让RL精准改代码

今天给大家带来一篇把RL信用分配做细到代码token的交互式网页生成工作：RCCA，训练出的Ling-RCCA-Flash在MiniAppBench和ArtifactsBench双榜刷分。

🔑 关键方法
1️⃣ Rubric驱动数据构造：每个训练任务都围绕明确的功能rubric构建，分初始状态和动态交互两类，像“点击按钮后打开面板”“输入框内容要同步显示”，每条都能单独执行验证。
2️⃣ 四层门控层次奖励：先过格式、源码、运行时三个硬性门，再评估功能rubric。语法错误、跑不起来、功能缺失不再被混在一起，奖励信号更干净。
3️⃣ 诊断到token的信用分配：评估器会输出文本定位，指出具体出错代码段，比如事件绑定、state更新、DOM片段、CSS选择器，再映射到生成token权重。

💡 核心创新
1️⃣ 把rubric级结果还原成token级优化信号，解决GRPO“一个总分均摊到所有token”的粒度错配。
2️⃣ 层次奖励替代单一大分，让不同失败模式在group内更好区分，而不是都拿一个模糊分数。
3️⃣ 负优势样本集中更新诊断出的错误代码段，正优势样本不额外强化这些区域，避免把正确信号带偏到错误片段。

📊 实验效果
✅ MiniAppBench平均通过率41.25，比Ling-3.0-Flash提升32.20分，略超Claude Opus 4.5的41.14。
✅ 训练轨迹清晰：基础模型9.05，SFT到26.85，RCCA再拉到41.25，净提升14.40。
✅ Hard任务37.60、Humanities 54.29、Visualization 63.46、Lifestyle 70.00，类目表现很能打。
✅ ArtifactsBench达到76.19，超过GPT-5官方榜72.55，登顶官方榜单。

你觉得这种把评估反馈精准打到代码token的RL方式，会不会成为前端Agent训练的新范式？

论文：Rubric-to-Code Credit Assignment for Reinforcement Learning

欢迎投稿！欢迎合作！