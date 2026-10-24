腾讯视觉大一统模型来了

今天给大家带来腾讯微信视觉团队开源的WeVisOne-VL，一个把检测、指代定位、计数、OCR、分割和抠图全部塞进同一个VLM的视觉大一统模型。

🔑关键方法
1️⃣ 所有任务统一成结构化next-token prediction，用<0>~<1000>专用坐标token编码框和点，告别多套任务头。
2️⃣ 特殊<OBJ> token桥接轻量像素解码器，把VLM隐状态、预测框和ViT多级特征喂给仅9.6M参数的解码器，直接出mask和alpha matte。
3️⃣ 三阶段课程SFT+混合任务RL：先对齐任务格式，再拉高分辨率到1960 visual tokens、单图最多500框，最后交替优化token与像素；RL阶段用不确定采样+GRPO直接优化定位质量。

💡核心创新
1️⃣ 物体级和像素级感知统一到一个自然语言接口，不接单独SAM级大模型，部署更轻。
2️⃣ <OBJ> token像“像素嘴”，把离散语言输出和密集像素预测打通，轻量但能输出全图mask/alpha。
3️⃣ WDE数据引擎约100M图像、3.6B框级标注，用知识图谱发现概念、专家模型提proposal、VLM做语义验证，还把拒识负样本和模型错误回流，越训越硬核。

📊实验效果
✅ 30+ benchmark全面领先同尺度VLM，COCO mean F1 4B 57.1，Dense200 mean F1 71.3，密集小目标很能打。
✅ 指代定位RefCOCOg test F1 81.8，PR-Bench拒识率96.4%，对“图中没有”的query不乱画框。
✅ 计数PixMo-Count test 89.6%，MixCount test 40.4%，按exact match算，差一个也算错。
✅ 指代分割RefCOCO val cIoU 81.4，ReasonSeg val gIoU 71.2，Ref-ZOM拒识86%，负样本不硬出mask。
✅ 抠图P3M-500-NP MAD 6.6，AM-2K MAD 5.0，比SAM2-Matting更稳，发丝和透明边缘更细。

你觉得这种视觉大一统模型，最可能先改变内容创作、电商还是工业检测？评论区聊聊。

论文：WeVisOne-VL: Unified Visual Perception from Objects to Pixels

欢迎投稿！欢迎合作！