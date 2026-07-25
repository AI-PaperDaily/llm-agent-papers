## NVIDIA发布OO Agents：把Agent当Python对象写，性能飙升82.2%，代码即Agent？

在寻找构建可靠AI Agent的最佳实践时，开发者们往往被割裂在不同技术栈中：一边是手写提示词模板，一边是配置工具Schema，另一边又要处理各种回调函数与复杂的工作流图。这种碎片化的开发体验使得Agent开发的门槛居高不下。针对这一痛点，NVIDIA研究团队提出了一种颠覆性的新范式：将Agent直接定义为Python对象。该框架在SWE-bench Verified和Terminal-Bench 2.0等多个权威基准测试中取得了领先成绩，最高性能提升达82.2%，打破了通用开源Agent框架与专用系统之间的性能壁垒。

### 一、核心方法：Python即Agent

NVIDIA Object-Oriented Agents（NOOA）框架的设计哲学非常直接：“复用Python，放弃DSL”。它提出，一个AI Agent本质上就应该是一个标准的Python对象。这一思想彻底改变了Agent的开发方式。

#### 创新点一：类即Agent，方法即能力

在NOOA中，开发者通过定义一个继承自`Agent`的Python类来创建Agent。这个类的字段就是Agent的状态，方法就是Agent可执行的动作，而文档字符串（docstring）则直接充当提示词。最巧妙的地方在于，方法体如果是`...`（省略号），就表示这是一个由LLM驱动的智能方法；如果是正常的代码体，则作为确定性逻辑执行。

![图1：NOOA中一个简单Agent的实现 | https://arxiv.org/html/2607.20709v1/x3.png](https://arxiv.org/html/2607.20709v1/x3.png)

这种设计让提示词工程回归软件工程。Agent的行为可以像常规代码一样被测试、追踪、重构和优化。例如，一个去判断“订单是否可退款”的方法，可以用普通Python函数实现；而一个“处理客户投诉”的复杂任务，则可以用`...`体声明，让LLM自主决定调用哪些工具、如何分析并产生最终输出。

#### 创新点二：代码即动作，类型即契约

传统Agent框架通常通过LLM生成JSON调用来执行工具，这需要模型理解复杂的结构化格式。NOOA采用了截然不同的方式：**让模型直接写代码**。在CodeAct模式下，Agent的本质是一个Python REPL（读取-求值-打印循环）。LLM可以编写任意Python代码，调用本地库、进行循环、并发处理，甚至调用其他Agent化方法。这个设计利用了LLM在训练数据中对Python语法的深刻理解，极大降低了模型的学习负担。

同时，类型注解在这里不仅是给开发者看的，更是**模型与运行时之间的契约**。任何Agent化方法都必须返回符合类型注解的结果，否则运行时会拒绝输出并让模型重新尝试。这一机制保证了Agent输出的结构化和可靠性。

$$ \text{Agent} = \text{Python Object}, \quad \text{Method} = \text{Action}, \quad \text{Docstring} = \text{Prompt}, \quad \text{Type Annotations} = \text{Contract} $$

#### 创新点三：按引用传递，跨越上下文窗口的鸿沟

Agent开发中的一个经典难题是上下文窗口的限制。当需要处理包含数百万行数据的大型表格时，不可能将所有数据都塞进提示词里。NOOA的解决方法是**按引用传递**（Pass-by-Reference）。当Agent化方法接收一个大型对象（如列表或DataFrame）时，提示词中只包含该对象的**有界预览**（bounded preview）。例如，一个包含一百个整数的列表，模型看到的可能只是：

```python
records = list ( len=100, [:5]=[42, 17, 89, 33, 8], [-5:]=[56, 71, 12, 45, 28])
```

模型知道这是一个长度为100的`list`，并看到头部和尾部的样本。但在实际的执行环境中，`records`是一个完整的、活生生的Python对象。模型可以通过编写Python代码（`for r in records:`）来处理整个列表，而无需将所有内容放入上下文窗口。这突破了Agent处理数据规模的物理限制。

![图2：NOOA中CodeAct策略的Agent循环 | https://arxiv.org/html/2607.20709v1/x4.png](https://arxiv.org/html/2607.20709v1/x4.png)

### 二、实验结果：全栈制霸，效率与性能双赢

NOOA在多个关键基准测试中完成了全面验证，展现了其强大的通用性和卓越性能。

#### 软件工程与终端交互

在SWE-bench Verified（500个真实GitHub Issue）实验中，NOOA取得了与专用AI编码系统相当的优异成绩。实验结果清晰展示了NOOA的领先地位。

![图3：SWE-bench Verified得分与每任务token消耗对比 | https://arxiv.org/html/2607.20709v1/x5.png](https://arxiv.org/html/2607.20709v1/x5.png)

在终端操作基准Terminal-Bench 2.0中，NOOA以73.0%的得分大幅领先于OpenCode（60.7%）和PI（68.5%），特别是在模型未使用高推理成本的模式下（`Off`），NOOA的优势更加明显，展现了对复杂、多步骤环境中Agent能力的高效支持。

#### 先进交互推理（ARC-AGI-3）

在极具挑战性的ARC-AGI-3交互推理基准上，NOOA展示了其设计哲学所蕴含的巨大潜力。一个结合了“世界模型技能”和内存子系统（Memory）的单一NOOA Agent，在GPT-5.5模型上获得了**50.2%**的得分。而作为对比，没有内存子系统或使用简单的markdown文件替代内存的Agent，得分分别为41.7%和38.4%。这表明NOOA的**对象状态持久化**和**内存机制**是提升Agent推理能力的关键组件。

![图4：ARC-AGI-3得分与时间/成本的帕累托前沿 | https://arxiv.org/html/2607.20709v1/x6.png](https://arxiv.org/html/2607.20709v1/x6.png)

从图中可以发现，NOOA不仅性能更强，还在成本控制上表现优异。在实现最高性能的同时，其每场游戏的运行成本远低于其他方法。

### 三、结语

NVIDIA OO Agents的提出，不仅仅是又一个Agent开发库的诞生，更是对Agent开发思维模式的深刻重塑。它将智能体拉回到开发者最熟悉的软件工程轨道上，用Python的简洁与强大，统一了人类开发者与LLM的认知接口。当“写Agent”变成“写Python”，所有优秀的软件工程实践都能无缝迁移。这或许正是走向可靠、通用、高智能Agent的一条优雅路径。未来，Agent优化将不只是调优提示词，而是对整个“Agent对象”进行重构和进化。这种“代码即Agent”的范式，你们觉得会是未来AI开发的主流吗？

**论文标题**：NVIDIA-labs OO Agents: Native Python Object-Oriented Agents

欢迎投稿！欢迎合作！