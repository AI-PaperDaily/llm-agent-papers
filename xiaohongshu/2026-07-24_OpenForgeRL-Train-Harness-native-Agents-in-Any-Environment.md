标题：OpenForge RL: Harness原生Agent训练

今天给大家带来OpenForge RL，一个让任意harness和环境的Agent都能端到端训练的开源框架，用轻量代理+K8s编排器彻底解耦了训练与推理流程。

🔑关键方法：
1️⃣ 轻量代理拦截harness的模型调用，将每次交互记录为prompt-response对，自动重建轨迹并转化为标准RL训练样本（如veRL格式）。
2️⃣ Kubernetes编排器在云端容器中运行每个rollout，支持弹性扩缩容，避免与训练节点资源冲突，并设置超时机制防止单点阻塞。
3️⃣ 数据合成管线：先让模型生成任务提议，再剪枝低质量、构建可执行环境与验证脚本，最后通过测试-修补循环确保任务可用。

💡核心创新：
1️⃣ 首次实现“任何harness × 任何环境”的端到端训练，无需为训练重写简化版harness，彻底消除训练-部署不匹配。
2️⃣ 覆盖文本工具使用（Claw）和多模态GUI（浏览器/桌面）两大场景，验证了框架的通用性，仅用数百至数千任务即可超越同规模开放模型。
3️⃣ 深入分析harness选择与RL对Agent行为的影响：简单对齐的harness更易学；单一harness训练可泛化到未见harness；RL主要提升自验证、工具覆盖和多步完成能力，但错误恢复仍是短板。

📊实验效果：
✅ Claw领域：OpenForge-Claw（30B-A3B MoE）在ClawEval上pass^3达31.7，pass@3达55.9，QwenClawBench pass@1 33.7，MCPAtlas 28.1，均优于同尺寸基线（如Qwen3-30B-A3B-Thinking仅14.3/39.8）。
✅ GUI领域：OpenForge-GUI（8B）在OSWorld-Verified达37.7，Online-Mind2Web达63.0，WebVoyager达72.3，超越UI-TARS-7B、OpenCUA-7B等，并匹配数倍大的模型（如GPT-5.4、Claude Opus）。
✅ 效率惊人：MolmoWeb-8B用200k+任务训练，OpenForge-GUI仅用2.5k任务即在其之上，RL进一步带来显著提升（SFT→RL提升3-10个点）。

大家觉得在复杂GUI场景里，RL最难教给Agent的能力是什么？是错误恢复还是多步规划？评论区聊聊~

论文：OpenForgeRL: Train Harness-native Agents in Any Environment

欢迎投稿！欢迎合作！