KV缓存砍4倍！DeepSeek新作发布

今天给大家带来DeepSeek-V4.1-Flash，一款原生多模态MoE，骨干552B，prefill只激活8B、decode激活16B，把百万上下文Agent的KV缓存和部署成本直接打下来。

🔑关键方法
1️⃣ Causal Encoder-Decoder（CED）：前20层做encoder，一次性生成全局KV；后20层decoder的全局KV直接由encoder末层隐状态投影出来，prefill不用跑满全模型，计算量接近砍半。
2️⃣ CSA2 + FP4 KV Cache：Compressed Sparse Attention 2用Full/Reindex/Reuse三种模式跨层复用全局KV、indexer K和Top-K索引，主KV再上FP4，单token全局KV降到890字节，约V4-Flash的1/4。
3️⃣ SWA Bounded Replay：滑动窗口KV不再进持久缓存，miss时只重放最近窗口token做近似重建，持久KV进一步压缩到约1/8，SSD和HBM压力都小很多。

💡核心创新
1️⃣ 非对称激活：CED把prefill和decode设计成8B/16B参数激活，特别适合工具调用频繁、输入超长的Agent工作负载。
2️⃣ CSA2把层维复用和索引选择解耦，不是简单地共享路由；decoder再配合层级稀疏索引器，让后续indexer评分从随上下文线性变成常数级。
3️⃣ FP4只压缩存储，attention前反量化，不要求硬件原生支持FP4矩阵乘；SWA Bounded Replay用近似换容量，实测质量损失可忽略。

📊实验效果
✅ 全局KV缓存每token 890字节，相比DeepSeek-V4-Flash降低约4倍。
✅ 持久KV缓存约V4-Flash的1/8，长上下文部署更省SSD和HBM。
✅ 1M上下文下Decode FLOPs几乎保持恒定，4K扩到1M只多出约25%计算。
✅ Agent表现在线：DeepSWE v1.1达74

论文：DeepSeek-V4.1-Flash: Pushing the Limits of KV Cache Compression

欢迎投稿！欢迎合作！