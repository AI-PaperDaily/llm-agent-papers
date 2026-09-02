门控残差+QSA新架构太省了

今天给大家带来Qwen3.8-Flash-Next，一个125B总参、6B激活的稀疏MoE，靠混合线性注意力、门控残差和外置n-gram记忆，用约1/9训练FLOPs追平甚至反超前代397B旗舰。

🔑 关键方法
1️⃣ 混合Token Mixing：每4层里3层用Gated DeltaNet做线性递归压缩前缀，保留1层全局注意力；继续预训练时换成QSA稀疏注意力，用micro-block压缩索引降低长序列成本。
2️⃣ Gated Residual门控残差：残差流加宽到4条分支，通过elementwise gate读取，既增加容量又替代pre-norm，稳定性更好。
3️⃣ Muon优化器+重拟缩放律：矩阵型权重交给Muon做正交化，重新拟合scaling law，把最优学习率和批大小往上抬，省掉batch size warmup。

💡 核心创新
1️⃣ 容量外置：51B n-gram embedding表放在加速卡外，按需从host memory预取，几乎不增加每token FLOPs和延迟。
2️⃣ QSA压缩索引：把key按块平均池化后打分选top块，索引成本从O(n²)降到O(n²/r)，1M上下文prefill快7.6倍、decode快4.9倍。
3️⃣ 三维联合设计：把loss/benchmark、效率、稳定性当一个问题做，多次发现只看预训练loss会误判。

📊 实验效果
✅ 14个预训练基准里8个超过前代397B-A17B，其余最多只差2.6分，激活参数仅1/3、训练token仅1/3、训练FLOPs约1/9。
✅ 长文本RULER 512K-1M从90.08升到93.00，MRCR 512K从30.66升到40.53。
✅ 4倍最优学习率压力测试下，GR+Muon实现0 loss spike，旧结构频繁spike；生产级训练全程无loss spike和异常梯度。
✅ QSA在短上下文不降点，平均分还从75.9到76.8，MTP投机解码接受长度几乎不降。

你会更愿意为9倍训练成本省下来买单，还是继续堆稠密大模型？评论区聊聊。

论文：On the Design of Qwen3.8-Next Architecture: Evaluation, Efficiency, and Training Stability

欢迎投稿！欢迎合作！