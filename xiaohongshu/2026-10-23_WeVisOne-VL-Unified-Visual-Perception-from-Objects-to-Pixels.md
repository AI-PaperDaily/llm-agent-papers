对象到像素，一个模型打通

今天给大家带来腾讯微信视觉的WeVisOne-VL，它把检测、指代定位、计数、OCR、分割和抠图塞进一个VLM里，从框到掩码再到alpha一次搞定。

🔑关键方法
1️⃣结构化next-token预测：坐标用0到1000的专用token编码，目标不存在就输出None，输出更紧凑。
2️⃣用OBJ token当桥：每个框后生成OBJ，把隐藏态送进轻量像素解码器，结合共享ViT多层特征出mask或alpha matte。
3️⃣三阶段SFT加混合任务RL：先任务对齐，再拉高分辨率和密集感知，最后token-像素交替训练；RL用不确定性筛样本，再用GRPO优化完整grounding输出。

💡核心创新
1️⃣对象级和像素级真统一：一个模型支持点、框、掩码、alpha，不用再外挂一个大分割模型。
2️⃣数据引擎会“找难点”：知识图谱组织类别和指代表达，检测器提候选、VLM做语义校验，掩码和抠图监督贯通，难例反馈继续迭代。
3️⃣解码器很轻，约9.6M参数：用OBJ latent和box先验做实例条件密集预测，分割与抠图共享组件，抠图再加细边界refinement。

📊实验效果
✅COCO mean F1 57.1，Dense200 mean F1 71.3，密集小目标下比同规模Qwen3-VL更稳。
✅指代检测RefCOCOg test mean F1 81.8，Ref-ZOM拒识率86.0，目标不存在时更少瞎框。
✅PixMo-Count val 93.7；OCR在ICDAR2015、SROIE等领先；P3M-500-NP抠图MAD 7.3，WeRefMat指代抠图MAD 9.8。
✅30+ benchmarks覆盖对象级和像素级任务，拥挤场景召回和拒识都挺能打。

你更想让它先落地哪个场景：电商抠图、内容审核，还是机器人视觉？评论区聊聊。

论文：WeVisOne-VL: Unified Visual Perception from Objects to Pixels

欢迎投稿！欢迎合作！