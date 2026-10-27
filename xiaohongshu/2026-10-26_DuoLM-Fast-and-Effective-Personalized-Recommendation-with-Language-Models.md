DuoLM：离线向量+共享KV

今天给大家带来DuoLM：它用编码器LLM把物品文本压成可复用向量，再用个性化ranker LLM结合用户历史和反馈打分，配合高效推理运行时，把推荐做得又快又准。

🔑 关键方法
1️⃣ 物品侧先离线压缩：编码器LLM把每个item文本读成一个向量，线上直接复用，不用每次把全文塞进模型。
2️⃣ 用户侧再个性化打分：ranker LLM拿用户历史里的item向量和反馈文本，对候选item逐个打分；候选之间不互相看，只共享同一段历史。
3️⃣ 训练用推荐标签+教师蒸馏：端到端联合编码器、连续接口和ranker，同时让冻结的全文本教师模型教它候选偏好分布。

💡 核心创新
1️⃣ 连续输入接口把两个LLM接起来，item表示跟用户无关，能跨请求复用，缓解长尾和冷启动对纯ID交互的依赖。
2️⃣ 运行时用VID虚拟token ID直接查GPU里的item向量表，请求只带ID和反馈，不传大向量。
3️⃣ 多item scoring一次prefill共享历史KV，候选分支保持独立；再叠CUDA Graph replay，减少调度和kernel launch开销。

📊 实验效果
✅ Books下一物品推荐NDCG@10到0.5586，超过全文本教师0.5440；Video Games 0.5298，和教师0.5313基本打平。
✅ 单张H100上约26,400候选分/秒，约为vLLM教师吞吐的60倍，最高到SGLang教师的109.9倍。
✅ C=64时全请求p99延迟536.90ms，教师vLLM为17,093.82ms，吞吐上去延迟也压住了。
✅ 消融里MIS带来4.81–29.3倍吞吐提升，CUDA Graph再提1.06–3.54倍；VID让请求体积降99%，C=8吞吐提升66.5%。

你们更想看DuoLM落地电商推荐，还是内容信息流？评论区聊聊～

论文：DuoLM: Fast and Effective Personalized Recommendation with Language Models

欢迎投稿！欢迎合作！