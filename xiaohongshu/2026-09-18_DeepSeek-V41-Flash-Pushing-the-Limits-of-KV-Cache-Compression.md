KV Cache压缩新王炸！

今天给大家带来DeepSeek-V4.1-Flash，一个把KV Cache压缩做到极致的多模态MoE模型，552B参数却只激活8B/16B，专为长程Agent场景设计。

🔑关键方法：
1️⃣ CED因果编码器-解码器架构：前20层编码器生成全局KV，后20层解码器直接投影复用，prefill只跑一半层数，计算量近乎减半，特别适合工具调用频繁的输入密集型任务。

2️⃣ CSA2压缩稀疏注意力二代：跨层共享main KV和indexer K，三种模式（Full/Reindex/Reuse）解耦缓存共享与索引复用，配合层级稀疏索引器，decode阶段索引成本从线性降为常数。

3️⃣ FP4 KV Cache量化：主KV缓存直接用FP4精度存储，采用E2M1格式每16通道共享一个scale，配合QAT训练，精度损失可忽略但存储直接砍半。

💡核心创新：
1️⃣ 把KV压缩从“序列维度”扩展到“层维度”，全局KV每token只需890 bytes，是V4-Flash的1/4，比V1降了437倍。

2️⃣ SWA有界重放：不持久化滑动窗口KV，命中缓存时只重放最近n_win个token近似重建状态，持久化KV直接降到1/8，把灾难性miss变成优雅降级。

3️⃣ 可控制推理强度：引入1-100的effort标量作为训练条件，一个checkpoint就能在成本-质量曲线上滑移，API只暴露max/high/low三档，低档省2.5倍token但只掉几个点。

📊实验效果：
✅ Codeforces 3471分，超过V4-Pro的3348，开源顶尖水平
✅ DeepSWE v1.1达到74.2%，超过Opus-5的74.0%和GPT-5.6 Sol的73.0%
✅ Terminal-Bench 2.1拿90.6%，登顶开源且超过多数闭源
✅ 1M context解码FLOPs相比4K只增加25%，近常数复杂度
✅ AutomationBench 54.8%，Agents’ Last Exam 31.8%，均为领先

家人们，KV Cache从V1到V4.1压缩了400多倍，这波操作你们觉得对长程Agent部署意味着什么？欢迎评论区聊聊～

论文：DeepSeek-V4.1-Flash: Pushing the Limits of KV Cache Compression

欢迎投稿！欢迎合作！