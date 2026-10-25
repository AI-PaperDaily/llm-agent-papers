一个模型打通框到像素

今天给大家带来腾讯微信视觉团队的WeVisOne-VL，它用结构化next-token预测把检测、指代定位、计数、OCR、分割和抠图统一进一个VLM，从对象框一路做到像素级mask和alpha。

🔑关键方法
1️⃣ 统一成结构化next-token：坐标用0到1000的专用token，检测、指代、计数、OCR、分割、抠图都走同一套输入输出，目标不存在就输出None，方便拒答。
1️⃣ 用<OBJ> token搭桥：每个框后生成一个<OBJ>，把它的隐状态和框先验送进轻量像素解码器，再融合ViT多层特征，直接出分割mask或alpha matte，不另挂大分割模型。
1️⃣ 三阶段SFT加混合任务RL：先任务对齐，再拉高分辨率和密集感知，最后token-像素交替优化；RL用GRPO按完整定位结果算奖励，压重复框和幻觉。

💡核心创新
1️⃣ 对象到像素一体化：点、框、mask、alpha都能在同一自然语言接口里指定粒度，用户说到哪就定位到哪。
1️⃣ 像素解码器很轻：约9.6M参数，靠<OBJ>隐状态和框引导query做稠密预测，结构紧凑，推理链路更短。
1️⃣ 数据引擎闭环：知识图谱组织类别和指代表达，多专家出候选加VLM校验，再接mask和alpha标注；模型错误回流成难例，迭代数据。

📊实验效果
✅ 30多个benchmark上目标级和像素级都很有竞争力，4B在COCO mean F1到57.1、LVIS 55.3、Dense200 71.3，密集小目标更稳。
✅ 指代检测RefCOCOg val/test到80.8/81.8，WeRef-Bench mean 78.7，缺目标拒绝率67.4，不再见啥都框。
✅ 计数PixMo-Count val/test 93.7/89.6，MixCount test 40.4，混合物体场景也能数准。
✅ 指代分割RefCOCO val 81.4，Ref-ZOM cIoU 72.2且拒绝率86.0，负表达敢说没有。
✅ 抠图P3M-500-NP MAD 7.3，AM-2K MAD 5.0，语言提示WeRefMat指代MAD 9.8，发丝和透明边界细节更完整。

你更想让这类统一感知模型先落地到内容创作、工业质检，还是手机相册？评论区聊聊。

论文：WeVisOne-VL: Unified Visual Perception from Objects to Pixels

欢迎投稿！欢迎合作！