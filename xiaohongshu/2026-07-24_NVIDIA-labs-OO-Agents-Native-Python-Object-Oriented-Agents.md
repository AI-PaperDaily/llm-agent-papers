🚀 Agent就是Python对象！NVIDIA开源的OO Agents你见过吗？

今天给大家带来NVIDIA最新开源的NOOA框架，一句话总结：把Agent定义成一个Python类，方法就是动作、字段就是状态、文档字符串就是提示词、类型注解就是契约，让开发者和LLM用同一套面向对象的编程语言搞定一切。

🔑 关键方法：
1️⃣ 对象即Agent：一个完整的Agent就是一个Python类，普通方法直接执行（确定性代码），带"..."的代理方法由LLM循环驱动。方法签名定义输入输出契约，文档字符串成为prompt，self上的方法和导入库自动变成工具。
2️⃣ 按引用传递（pass by reference）：模型看到的是Python对象的引用而非序列化文本。例如列表参数只展示类型、长度、头尾预览（len=100, [:5]=...），模型可以直接用代码操作整个对象，上下文窗口不爆炸。
3️⃣ 代码即动作（Code as action）：默认CodeAct策略让模型在REPL中写Python代码，可以循环、条件、异步、调用子Agent，而不是用JSON定义工具调用。工具调用变成普通函数调用，结果保留为Python变量。

💡 核心创新：
1️⃣ 复用Python现有抽象：类定义Agent，方法定义能力，字段定义状态，asyncio处理并发，类型注解做契约。没有新DSL，学习成本为零，且LLM训练数据里都是Python，天然适配。
2️⃣ 六种模型可见接口统一集成：类型化I/O、按引用传递、代码动作、可编程循环工程、显式对象状态、模型可调用的harness API。对比14个主流框架，NOOA是第一个全部支持的。
3️⃣ 长期记忆子系统：Agent自己通过7个工具（remember/recall等）维护记忆，SQLite存储，ACT-R激活排序，支持异步反射合并、遗忘、链接。非侵入式安装/卸载，保留状态。

📊 实验效果：
✅ SWE-bench Verified上GPT-5.5 xhigh达到82.2%，超过OpenCode（78.6%）和PI（78.2%），且token更少（1.1M vs 2.2M）。
✅ Terminal-Bench 2.0上GPT-5.5 high达到73.0%，领先OpenCode 12.3点，PI 4.5点。
✅ ARC-AGI-3交互推理上，单Agent+50行skill +记忆子系统达到50.2%（GPT-5.5），相比弃用记忆提升11.8点，是将之前多Agent系统压缩成单Agent后的效果，成本<$20/局。
✅ CyberGym L1安全漏洞发现达到86.8%，仅落后于微软和Claude专属方案，是所有开源方案第一。

所以，未来的Agent开发可能就是写一个Python类，然后让模型自己写代码、维护状态、迭代优化。你觉得这种“把Agent当普通软件”的哲学，能成为下一代框架的标准吗？评论区聊聊。

论文：NVIDIA-labs OO Agents: Native Python Object-Oriented Agents

欢迎投稿！欢迎合作！