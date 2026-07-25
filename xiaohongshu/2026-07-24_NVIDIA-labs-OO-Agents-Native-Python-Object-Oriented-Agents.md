对象即智能体！NOOA把Agent写成了Python类

今天给大家带来NVIDIA最新开源的NOOA框架——把智能体直接定义成一个Python类，方法就是LLM的动作，字段就是状态，开发者和大模型用同一套接口，彻底告别提示模板、工具Schema和回调代码的割裂。

🔑关键方法：
1️⃣ Agent即Python对象：用class定义，方法体写“...”表示LLM驱动循环，普通方法保持确定性，类型注解当契约
2️⃣ 双策略驱动：PredictStrategy单次LLM调用+类型验证；CodeActStrategy迭代REPL，模型可写Python代码来调用工具、访问状态、返回结果
3️⃣ 上下文分离与按引用传递：静态/动态上下文块+事件历史，大对象只传预览不序列化全量，模型操作真实Python对象而非文本

💡核心创新：
1️⃣ 统一编程模型：不再分散在prompt模板、tool schemas、回调代码；Agent就是代码，开发者和大模型都用Python语法
2️⃣ 代码即动作：模型不写JSON工具调用，而是直接写Python代码，利用预训练的编程知识，可循环、条件、异步
3️⃣ 按引用传递+实时对象状态：输入输出是live Python对象，持久状态存在self字段，不在对话历史里丢失

📊实验效果：
✅ SWE-bench Verified达82.2%（GPT-5.5），超越OpenCode和PI，且每任务仅1.1M tokens（PI需2.2M）
✅ Terminal-Bench 2.0达73.0%，成本更低，大模型下领先12个百分点
✅ ARC-AGI-3交互推理：单Agent+记忆系统RHAE达50.2%（GPT-5.5），相比无记忆提升11.8点，每局成本<20美元
✅ CyberGym L1漏洞发现86.8%，开源SOTA，接近顶级闭源方案

这种“Agent就是对象”的设计，你觉得会成为未来Agent开发的主流吗？欢迎评论区聊聊～

论文：NVIDIA-labs OO Agents: Native Python Object-Oriented Agents

欢迎投稿！欢迎合作！