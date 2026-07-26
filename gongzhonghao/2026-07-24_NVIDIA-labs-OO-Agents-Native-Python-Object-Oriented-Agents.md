## NVIDIA发布OO Agents：Python原生框架突破，Agent编写如普通代码，SWE-bench达82.2%

传统Agent开发正陷入一场“语言分裂”的困境：**提示模板**、**工具Schema**、**回调函数**与**工作流图**各成一体，开发者需要学习多种领域专用语言，而大模型面对这些割裂的接口同样水土不服。针对这一痛点，NVIDIA实验室近期提出一种颠覆性方案——**NOOA（NVIDIA Object-Oriented Agents）**，将Agent彻底定义为**Python对象**。其方法即模型可调用的能力，字段即状态，文档字符串即提示，类型注解即契约。这一设计在SWE-bench Verified上达到82.2%的通过率，在Terminal-Bench 2.0上达到73.0%，并以单Agent一页技能的形式刷新ARC-AGI-3互动推理基准的得分-成本帕累托前沿。

## 一、核心方法

### 1. 回归本质：Agent即Python对象

NOOA的核心理念极具颠覆性：**在Python已拥有成熟抽象的地方，直接采纳，而非发明新语言**。在NOOA框架中，Agent是一个Python类，其能力表现为普通方法，类型注解承担边界契约，异步操作遵循asyncio标准，而工具与编排逻辑即为原生Python代码。

![图1：NOOA中一个简单Agent的实现](https://arxiv.org/html/2607.20709v1/x1.png?format=PNG)

如图所示，一个完整的支持Agent示例包含四个核心部分：`Ticket`类型定义了带有验证规则的输出结构；`SupportAgent`类以文档字符串承载系统提示；普通方法`is_refund_eligible()`为确定性逻辑；而主体为`...`的方法则由LLM驱动循环运行。这意味着**源码、提示、工具接口、类型契约与状态边界被统一在一个对象之中**，彻底消弭了开发视角与模型视角之间的鸿沟。

### 2. 代码即行动：用Python编程替代工具调用

传统Agent框架中，模型通过生成JSON格式的工具调用来驱动外部函数，这一过程不仅导致大量序列化开销，更迫使模型学习一套新的“工具语言”。NOOA提出了**Code as Action（代码即行动）**范式：模型直接编写Python代码进行思考与行动。

```python
@ strategy(CodeActStrategy())
async def triage(self, message: str, photo: Image | None, order: Order | None) -> Ticket:
    ...
```

当一个CodeAct方法被调用时，框架启动一个Python REPL（读取-求值-打印循环）。模型可以在REPL中调用`execute_python(...)`来执行任意Python代码，包括调用其他方法、遍历数据结构、执行异步操作，甚至使用`asyncio.gather`并行派生子Agent。与通过JSON“远程过程调用”不同，**模型操作的是同一个Python进程中的实时对象**。

![图2：CodeAct策略的Agent循环](https://arxiv.org/html/2607.20709v1/x2.png?format=PNG)

上图展示了CodeAct策略的执行流程：从渲染上下文到调用LLM，从执行Python代码到更新事件与状态，最后到验证返回值。这一循环让Agent拥有了**与开发者完全相同的编程能力**，包括循环、分支、库调用与异常处理。

### 3. 引用传递：突破上下文窗口的物理极限

传统Agent框架中，工具调用的输入输出均为序列化文本，这意味着**所有数据都必须嵌入提示词窗口**。NOOA的第三个关键创新是**Pass by Reference（引用传递）**：模型接收的是指向内存中对象的引用而非全文序列化。

例如，当方法接受一个包含100个整数的列表时，提示词中出现的仅为紧凑预览：

```
records = list(len=100, [:5]=[42, 17, 89, 33, 8], [-5:]=[56, 71, 12, 45, 28])
```

然而，变量`records`本身是完整的100个元素的列表对象，模型可以通过`for r in records:`对其进行全部遍历，但提示词窗口中仅出现10个样本值。这种设计意味着**Agent能处理的数据量由执行环境边界而非提示词长度决定**。一个方法可以接收百万行数据表，Agent通过编写代码整体操作，而提示词仅携带固定大小的预览。

### 4. 长时记忆与场景工程

NOOA还建立了模型可见的上下文工程API。静态上下文块（如系统提示）在整个调用周期内保持稳定；动态上下文块（如待办事项状态）每一轮重新计算；事件历史则形成结构化的执行轨迹。这些接口对开发者和模型同样开放，Agent可以主动查询、压缩或编辑自身的事件历史与记忆。

![图3：NOOA中的上下文渲染机制](https://arxiv.org/html/2607.20709v1/x3.png?format=PNG)

论文还提出了一种规范化的记忆系统，Agent通过七种工具（`remember`、`recall`、`search`、`update_memory`、`forget`、`associate`、`deref`）自主管理长期记忆，而一种BeforeTurn钩子则基于最近事件自发检索关联记忆，注入动态上下文。这种**主动式记忆管理**在ARC-AGI-3上带来了+11.8个百分点的收益。

## 二、实验

NOOA在四个互补的Agent基准测试中接受了全面评估，覆盖软件工程、命令行交互、网络安全与互动推理。

### 1. SWE-bench Verified与Terminal-Bench 2.0

在包含500个真实GitHub仓库问题的SWE-bench Verified上，NOOA搭配GPT-5.5以xhigh推理强度达到**82.2%**，显著超越OpenCode的78.6%和PI的78.2%。在包含89个终端任务（软件安装、配置、调试）的Terminal-Bench 2.0上，NOOA以GPT-5.5+high推理达到**73.0%**，领先OpenCode 12.3个百分点。

![图4：SWE-bench Verified的得分-成本帕累托前沿](https://arxiv.org/html/2607.20709v1/x4.png?format=PNG)

**更关键的是，更高分数并非源于更长轨迹**：NOOA在SWE-bench上平均使用约28次模型调用、110万token即可达到82.2%，而PI使用66次调用、220万token才达到78.2%。NOOA因此定义了该基准的**准确率-成本帕累托前沿**。

### 2. CyberGym与ARC-AGI-3

在针对网络安全漏洞发现的CyberGym L1上，NOOA取得了**86.8%**的识别率，在所有开源Agent中排名第一。在更具挑战性的交互推理基准ARC-AGI-3上，NOOA以单Agent+50行技能+记忆系统的极简配置，取得了**50.2%**的RHAE，相比无记忆的消融版本（38.4%）提升了11.8个百分点，相比基线技能（41.7%）提升了8.5个百分点。

### 3. 能力测试

论文构建了包含88个测试实例、覆盖36个功能系列的能力测试套件，在10个模型上每个测试运行5次（共4,400个记录）。结果显示总通过率为**97.9%**，表明**当前模型无需任何针对NOOA接口的训练即可流畅使用**。即使测试中最具压力的6个系列（批量处理、错误恢复、迭代探索等），大/前沿模型仍保持了93.9%的通过率。

## 三、展望

NOOA的核心洞察在于：**在Python已提供成熟抽象的地方，不再发明新语言**。这一设计将Agent开发从“学习新框架”带回“编写普通软件”的舒适区，同时让大模型能够利用其训练数据中丰沛的Python知识。当Agent既是代码也是模型的操作对象时，提示工程回归软件工程，Agent行为可以像普通代码一样被测试、跟踪、重构与优化。NOOA不仅是一个框架，更是一种拥抱软件原生力量的Agent哲学。

**论文标题**：NVIDIA-labs OO Agents: Native Python Object-Oriented Agents

欢迎投稿！欢迎合作！