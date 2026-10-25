# 腾讯微信发布视觉大模型WeVisOne-VL：Dense200 F1达71.3！

## 一、引言

视觉语言模型正在从“看图说话”走向“看图定位、看图分割、看图抠像”，但细粒度视觉感知仍是短板。现有VLM在面对密集小目标、长尾类别、复杂指代和缺失目标时，常出现幻觉式检测、意图误读和漏检，导致真实场景部署可靠性不足。论文提出的**WeVisOne-VL**试图用一套自然语言接口统一对象级与像素级感知：既输出检测框、指代定位、计数、OCR，也输出分割掩码和连续alpha matte。模型基于Qwen3-VL构建2B与4B两个版本，使用约1亿图像和36亿框标注训练，在30多个基准上取得领先结果。其中4B模型在Dense200上平均F1达到**71.3**，在RefCOCOg test上达到**81.8**，在PixMo-Count test上计数准确率达到**89.6%**，并保持较强的缺失目标拒答能力。

![图1：WeVisOne-VL统一对象级与像素级感知能力概览](https://arxiv.org/html/2610.31329v1/teaser.png)

## 二、核心方法

WeVisOne-VL的核心思路是把视觉感知任务统一成**结构化next-token prediction**。模型不依赖独立的大型分割模型，而是让VLM自回归生成类别、坐标和特殊标记，再通过轻量像素解码器把离散token转成密集输出。

### 1. 统一任务形式：坐标token与<OBJ>桥接

论文引入一组专用坐标token `<0>` 到 `<1000>`，用归一化坐标编码点、框等空间输出。检测、指代定位、计数、OCR等任务被组织为“对象引用+空间坐标”的序列生成；目标不存在时，坐标字段输出`None`。对于分割和matting，模型在每个框坐标四元组后生成一个`<OBJ>` token，其隐藏状态作为对应实例的潜在表示，交给像素解码器生成全图mask或alpha matte。这样，同一套输出接口同时覆盖离散定位与密集预测。

![图2：WeVisOne-VL数据引擎概览](https://arxiv.org/html/2610.31329v1/data_engine.png)

### 2. 数据引擎：从语义发现到像素监督

论文构建了**WeVisOne-VL Data Engine**，用共享知识图谱组织类别概念和组合式指代表达，并指导定向图像采集。检测标注先用SAM3和自研DINO式开放词汇检测器生成候选框，再由Qwen3-VL验证查询与实例的对应关系；指代标注基于SAM3 Agent框架并加入Rex-Omni作为定位专家；掩码标注用SAM2生成二值mask。Matting数据则结合BiRefNet、MatAnyone2和专用annotator，将实例标注扩展为连续alpha监督。数据引擎还利用模型错误构建hard negative和hard positive，并经过人工抽查与训练反馈迭代 refinement。

### 3. 轻量像素解码器：9.6M参数连接token与像素

像素解码器由视觉特征适配器、box-guided query decoder和instance-conditioned dense prediction三部分组成。它接收Qwen3-VL多级ViT特征、每个`<OBJ>` token的隐藏状态及其预测框，先构建共享多级视觉memory，再用可变形查询解码器融合实例表示与框先验，最后通过动态mask head和由粗到细的refinement生成全分辨率mask。分割与matting共享这些组件，matting额外使用轻量refinement模块处理细边界和半透明。整个解码器约**9.6M参数**。

![图3：轻量像素解码器结构](https://arxiv.org/html/2610.31329v1/model.png)

### 4. 三阶段SFT与混合任务RL

训练采用三阶段课程SFT。Stage 1进行任务对齐，冻结视觉编码器，仅训练语言模型；Stage 2提升分辨率和密度，最大输入视觉token提升至1960，单响应最大框/点数量提升至500；Stage 3先以分割损失预热像素解码器：

$$\mathcal{L}_{\mathrm{seg}}=20\mathcal{L}_{\mathrm{focal}}+\mathcal{L}_{\mathrm{dice}}$$

预热完成后，VLM与解码器以1:1交替进行NTP和像素任务，联合目标为：

$$\mathcal{L}_{\mathrm{joint}}=\mathcal{L}_{\mathrm{NTP}}+10\mathcal{L}_{\mathrm{seg}}$$

Matting阶段冻结VLM，仅微调解码器，采用连续alpha监督：

$$\mathcal{L}_{\mathrm{mat}}=200\mathcal{L}_{l1}+5\mathcal{L}_{\mathrm{grad}}+50\mathcal{L}_{\mathrm{lap}}+\mathcal{L}_{\mathrm{hard}}$$

SFT之后，论文在不确定性筛选的语料上进行混合任务GRPO。RL直接优化完整结构化响应的grounding质量，奖励函数为：

$$r_{i}=\frac{2\sum_{(\hat{b},b^{*})\in\mathcal{M}_{i}}\operatorname{IoU}(\hat{b},b^{*})^{3}}{N_{\mathrm{gt}}+N_{\mathrm{pred}}}$$

该奖励同时惩罚漏检和重复预测，并对高IoU区间更敏感，从而提升定位精度并抑制重复或幻觉输出。

## 三、实验

论文在对象级和像素级共七类任务上评估，包括目标检测、指代目标检测、计数、指向、OCR、指代分割和promptable matting。对比对象覆盖Qwen3-VL、Qwen3.5、Rex-Omni、LocateAnything，以及专用检测、分割和matting系统。

![图4：指代分割定性对比](https://arxiv.org/html/2610.31329v1/refseg.png)

| 任务/基准 | WeVisOne-VL-4B | 对比模型最佳VLM | 关键指标 |
|---|---:|---:|---|
| COCO 检测 | 57.1 | 54.7（LocateAnything-3B） | F1 Mean |
| Dense200 检测 | 71.3 | 58.7（LocateAnything-3B） | F1 Mean |
| VisDrone 检测 | 44.7 | 39.9（LocateAnything-3B） | F1 Mean |
| RefCOCOg test | 81.8 | 77.6（LocateAnything-3B） | F1 Mean |
| PixMo-Count test | 89.6 | 82.8（LocateAnything-3B） | 精确匹配 |
| SROIE OCR | 68.1 | 44.8（Rex-Omni-3B） | F1 Mean |
| RefCOCO testA 分割 | 84.5 | 83.0（UniPixel-7B） | cIoU |
| Ref-ZOM 拒答率 | 86.0 | 41.9（GSVA-7B） | Rejection Rate |
| P3M-500-NP Matting | 7.3 / 6.6（4B/2B） | 15.4（SAM2-Matting SAM3） | MAD↓ |

结果显示，WeVisOne-VL在通用、长尾和密集检测上均优于同规模VLM，在Dense200和VisDrone上优势尤其明显。指代定位在RefCOCOg、WeRef-Bench、PR-Bench等基准上领先，并显著提升缺失目标拒答率。计数任务在PixMo-Count、CLOC、MixCount等基准上取得最高精确匹配准确率。OCR在HierText、ICDAR2015、TotalText和SROIE上全面超过专用OCR系统和通用VLM。指代分割在RefCOCO系列、ReasonSeg和Ref-ZOM上表现强劲，尤其Ref-ZOM拒答率达到86.0，说明模型能在目标缺失时拒绝输出mask。Matting在box prompt和文本prompt下均优于SAM2-Matting基线，能保留胡须、毛发、半透明边界等细节。

## 四、展望

WeVisOne-VL把对象级定位与像素级预测压进同一个语言接口，为内容创作、工业检测、自动驾驶和具身智能提供了更可靠的视觉感知底座。论文也指出当前方法仍受限于单轮预测、无CoT推理和通用数据规模，未来可结合Agent框架进行多轮工具交互，并增强模糊用户意图推理。统一感知是否会成为下一代VLM的标配能力，值得持续观察。

**论文标题**：WeVisOne-VL: Unified Visual Perception from Objects to Pixels

欢迎投稿！欢迎合作！