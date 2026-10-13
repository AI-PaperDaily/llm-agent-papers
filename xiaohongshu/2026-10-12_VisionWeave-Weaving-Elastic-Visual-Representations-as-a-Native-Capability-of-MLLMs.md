视觉token弹性编织省43%

今天给大家带来阿里Qwen团队的VisionWeave，让多模态大模型学会按图像内容“弹性编织”视觉表示：信息密集区域保留细粒度token，冗余背景自动换成粗粒度token，省算力但不牺牲空间覆盖，平均能省43%视觉token还几乎不掉点。

🔑关键方法
1️⃣ 门控空间池化器：把4×4 patch块池化成1个粗粒度token，通过可学习门控和通道权重聚合，与原生细粒度token共享MRoPE坐标，空间位置对得齐。
2️⃣ 粒度路由器：用局部+全局交叉注意力探ViT浅层、中层、深层特征，逐块判断该用粗还是细，兼顾低层细节和高层语义。
3️⃣ 三阶段自蒸馏：先全粗粒度训pooler，再软混合训练路由分配，最后硬路由微调pooler和LLM，全程不用额外人工标注。

💡核心创新
1️⃣ 首次把“表示粒度”变成frontier级MLLM的原生能力，不是事后剪枝或合并。
2️⃣ 内容自适应分配粒度，保留空间覆盖，不像FastV/VisionZip那样直接丢关键token，文档和GUI定位尤其稳。
3️⃣ 只靠自蒸馏就能训练，还专门适配SGLang等现代serving引擎，工程部署不脱节。

📊实验效果
✅ 在Qwen3.8-27B上平均省43.0%视觉token，8个benchmark平均性能只掉1.08%。
✅ 同样省50%token时，FastV†只保留88.2%性能，VisionZip仅87.5%，VisionWeave几乎无损。
✅ 文档、InfoVQA等密集场景自动降低节省比例，视频场景省50%以上仍保持稳定。
✅ 跨分辨率和视频帧数鲁棒，256帧上限比原生128帧在LongVideoBench还高1.20分且token更少。
✅ 部署SGLang后端到端吞吐2.30x，平均TTFT降54.4%，TPOT降60.6%。
✅ 推理时调阈值就能控制节省比例，一个模型多档位复用。

你觉得“按需分配视觉粒度”会不会成为下一代多模态大模型标配？

论文：VisionWeave: Weaving Elastic Visual Representations as a Native Capability of MLLMs

欢迎投稿！欢迎合作！