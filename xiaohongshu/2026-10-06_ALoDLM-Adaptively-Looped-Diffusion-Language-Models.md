扩散LM终于不平均用力了

今天给大家带来一篇Amazon AGI的新工作ALoDLM，直接用token自适应循环深度，把扩散语言模型的质量拉到能打同量级AR模型。

🔑 关键方法
1️⃣ 把Qwen3拆成Prelude、Recurrent Core、Coda三段，循环只放在中间层，同一套参数反复滚，不额外堆参数。
2️⃣ 每个去噪步内，用预测熵挑出已经稳的mask token先commit成离散上下文；还没解决的token保留隐状态

论文：ALoDLM: Adaptively Looped Diffusion Language Models

欢迎投稿！欢迎合作！