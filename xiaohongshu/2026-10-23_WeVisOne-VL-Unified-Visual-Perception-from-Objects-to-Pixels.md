一个Token搞定对象到像素

今天给大家带来腾讯微信视觉的WeVisOne-VL，它用一个结构化next-token接口，把对象级定位和像素级分割、抠图统一进同一个VLM。

🔑 关键方法
1️⃣ 所有视觉任务都变成结构化next-token预测：检测、指代、计数、OCR、分割、抠图共用一套输出，坐标用<0>到<1000>的专用token编码，框后追加<OBJ> token。
2️⃣ Qwen3-VL骨干接约9.6M参数的轻量像素解码器，用<OBJ>隐状态、预测框和多级ViT特征直接出mask或alpha matte，不再外挂大分割模型。
3️⃣ 三阶段SFT课程先任务对齐，再拉高分辨率和密集目标，最后交替优化token与像素；之后做混合任务GRPO，用grounding奖励压重复框和幻觉。

💡 核心创新
1️⃣ 对象级和像素级感知真正统一：点、框、mask、alpha matte都能按用户粒度输出。
2️⃣ <OBJ> token当实例“暗号”，把VLM的离散定位和像素解码器桥起来，轻量但端到端。
3️⃣ 数据引擎+课程+RL闭环：约100M图像、3.6B框级标注，硬负例和缺失目标拒答一起训。

📊 实验效果
✅ 检测：4B在COCO mean F1 57.1、LVIS 55.3、Dense200 71.3、VisDrone 44.7，密集小目标提升明显。
✅ 指代：RefCOCOg test mean F1 81.8，WeRef-Bench拒答率67.4%，目标不存在时更少乱框。
✅ 计数与OCR：PixMo-Count test 89.6%，MixCount test 40.4%；SROIE mean F1 68.1。
✅ 分割与抠图：RefCOCO val cIoU 81.4，Ref-ZOM拒答率86.0%；P3M-500-NP抠图MAD 7.3，优于SAM2-Matting的15.4。

作者也留了坑：目前偏基础视觉感知，没走CoT，多轮Agent工具交互是后续方向。

你们更想看它做“指代分割+抠图”的落地，还是“密集小目标检测+计数”的实战？

论文：WeVisOne-VL: Unified Visual Perception from Objects to Pixels

欢迎投稿！欢迎合作！