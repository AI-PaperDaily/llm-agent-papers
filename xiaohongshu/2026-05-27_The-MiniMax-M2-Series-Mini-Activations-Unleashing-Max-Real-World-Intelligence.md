标题：9.8B激活决战千亿大模型？  
今天给大家带来一篇MiniMax最新发布的M2系列论文，核心就一句话：用超小激活参数，打出接近世界顶级大模型的Agent和推理性能，关键是全套技术细节全公开了。

🔑关键方法  
1️⃣ 极致MoE架构：229.9B总参数但每Token只激活9.8B，256个细粒度专家，Sigmoid门控替代Softmax，摆脱辅助损失函数，负载均衡更稳。  
2️⃣ 全程注意力不掉队：放弃之前混合注意力方案，全面采用全注意力+GQA，配合192K原生上下文窗口，避免长语境下SWA的检索退化。  
3️⃣ 多Token预测黑科技：预训练时嵌MTP模块，推理时扩展为三步推测解码，Draft模型的权重来自主模型Copy初始化，训练稳定且吞吐大幅提升。

💡核心创新  
1️⃣ Agent数据工厂：从GitHub真实PR构建可执行环境，自动生成Bug修复、功能添加、代码审查等多样任务；AppDev部分由专家注入系统Prompt，再用Agent-as-Verifier做三层执行/交互/审美评估，拒绝采样保证奖励信号真可靠。  
2️⃣ Forge强化学习系统：把Agent当做MDP处理，策略和外围环境解耦；发明窗口FIFO调度解决长尾轨迹堵头问题；前缀树合并让训练加速40倍且零精度损失。支持白盒和黑盒Agent随便接入。  
3️⃣ 自我进化初现：M2.7能自动诊断训练事故、修改自己的Agent模板、跑多轮自我改进，实验中吸收30%到50%的日常迭代工作量，甚至让自身性能再涨30%。

📊实验效果  
✅ SWE-bench Pro 56.2，SWE-bench Multilingual 76.5，Multi-SWE-bench 52.7，Terminal-Bench 2.0 57.0  
✅ BrowseComp 77.8，Wide Search 75.2，RISE 64.3，与Claude/GPT/ Gemini正面互有胜负  
✅ AIME 2026 94.2，GPQA-Diamond 89.8，IFBench 76.0，纯推理几乎追平头部闭源模型  
✅ 从M2到M2.7每版稳定提升，BrowseComp和金融建模涨幅尤其惊人，分别+33.8和+23.2

现在问题来了：大模型卷到最后，到底是“大力出奇迹”还是“巧劲破天花板”更适合落地？你觉得9.8B激活能成为AGI的Super Token吗？评论区聊聊你的看法！

论文：The MiniMax-M2 Series: Mini Activations Unleashing Max Real-World Intelligence

欢迎投稿！欢迎合作！