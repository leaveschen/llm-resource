## Lora

### Original paper

https://arxiv.org/pdf/2106.09685.pdf

这篇文章的核心目标是提出一种名为LoRA（Low-Rank Adaptation）的低秩适应方法，用于在大型预训练语言模型（如GPT-3）上进行高效适应。LoRA方法的主要目标包括：

- 减少可训练参数的数量：通过使用低秩分解，LoRA方法可以显著减少可训练参数的数量，从而降低计算和内存需求。

- 提高适应效率：LoRA方法可以提高模型适应的效率，因为它不需要重新训练整个模型，而只需要优化较小的矩阵A和B。

- 保持模型质量：尽管LoRA方法减少了可训练参数的数量，但它仍然可以在各种自然语言处理任务上实现与全微调相当的性能。

- 无额外推理延迟：LoRA方法在部署时不引入额外的推理延迟，因为矩阵A和B可以直接与冻结的权重矩阵相加。

具体做法是，对于模型中的参数矩阵$W_0\in R^{d\times k}$，考虑其训练过程为参数的更新，表示为$W_0+\Delta W$，对增量进行低秩分解为$W_0+\Delta W=W_0+BA$，其中$B\in R^{d\times r}, A\in R^{r\times k}, r\ll min(d,k)$

A 使用0均值高斯分布初始化，B初始化为0。训练时，用$\frac{\alpha}{r}$ 去 scale $\Delta Wx$，相当于控制学习率

具体的，考虑 transformer 架构，self-attention 模块中有四个权重矩阵（Wq、Wk、Wv、Wo），MLP 模块中有两个权重矩阵，该文中实验方法是对 attention weights 使用 lora，MLP 参数固定

采取不合并权重（同时保留$W_0$和$\Delta W$）可以实现多 lora，代价是 latency 高一些

作者还做了进一步分析，认为 lora 不仅是降低计算成本，还提供了更好的解释性，可以解释更新权重与预训练权重的相关性。作者在 GPT3（175B）上做了进一步实验

1、在 transformer 哪些 weight 上应用 lora

将所有微调参数都放到attention的某一个参数矩阵的效果并不好

即使是秩仅取4也能在$\Delta W$中获得足够的信息

**因此在实际操作中，应当将可微调参数分配到多种类型权重矩阵中，而不应该用更大的秩单独微调某种类型的权重矩阵。**

2、lora 的最优秩是多少

通过一堆实验，证明 r=1，2时仍然有不错的表现。因此作者假设:**更新参数矩阵可能拥有极小的‘内在秩’**。为求证此假设，作者需要计算不同秩对应的子空间之间的重叠程度。

子空间相似度通过如下方式计算，对于和两个秩8和64，首先进行奇异值分解得到两个右奇异矩阵$U_{A_{r=8}}$和$U_{A_{r=64}}$。作者希望得到：$U_{A_{r=8}}$的top-i奇异向量有多少被包含在$U_{A_{r=64}}$的top-j个向量中。可用曼哈顿距离来表示这种子空间之间的相似关系：

$$
\phi(A_{r=8},A_{r=64},i,j)=\frac{||U_{A_{r=8}}^{iT}U_{A_{r=64}}^j||_F^2}{min(i,j)}\in[0,1]
$$

3、adaptation matrix $\Delta W$ 和 $W$ 的关系

这里想度量 adaptation matrix 和原来的 weight matrix 的关系（数学上看 $\Delta W$ 包含多少 $W$ 的 top singular directions）这里的做法是，对于 r-dim 的子空间，对比$||U^TWV^T||_F$和$||W||_F$，其中 U，V 是$\Delta W$奇异值分解的左右矩阵，测试的结论：**在训练过程中，低秩的适应矩阵仅仅放大了对下游任务有用的特征，而不是预训练模型中的主要特征。**

参考：

https://zhuanlan.zhihu.com/p/646791309

### QLoRA

[[2305.14314] QLoRA: Efficient Finetuning of Quantized LLMs](https://arxiv.org/abs/2305.14314)

这篇文章提出一种高效的微调方法，结合量化与 lora，几个核心技巧为：

- 4 位 NormalFloat，一种信息理论上最佳的量化数据类型，适用于正态分布数据，比 4 位整数和 4 位浮点数产生更好的经验结果。

- Double Quantization，将量化常数进行量化的方法，平均值每个参数节省约 0.37 bits（65B 约为 3 GB）。

- 分页优化器，使用 NVIDIA 统一内存来避免处理小 batch 长 sequence 情况下梯度检查点内存峰值。

基础概念

分块量化（block wise quantization）

常见的量化做法，例如从 fp32 到 int8，方法如下：

$$
X^{int8}=round(\frac{127}{absmax(X^{fp32})}X^{fp32})
$$

这样做的问题在于，一旦有一些比如 outlier，量化效果就会很差，一个常用的解决办法是对 tensor 分块做量化

**4-bit NoramlFloat quantization**

建立在 Quantile Quantization 上（理论上最优的数据类型，保证每个 quantization bin 具有相同数量的值，但其分位数估计的过程是昂贵的，使用一些快速的近似算法如SRAM分位数来估计它们，会对异常值有很大的量化误差，而这些异常值通常是最重要的值）

由于预训练过的神经网络 weights 通常是0均值的高斯分布，考虑将其映射到 [-1, 1] 的范围，假设要量化到 k-bits，具体做法是：

1、估计$2^k+1$ 个在 $N(0,1)$ 分布上的分位数（应该是保证每个分片的面积相等）

2、分位数缩放到 [-1, 1] 的区间

3、使用 absolute maximum rescaling 把 input 映射到 [-1, 1] 区间

两个区间做匹配，能找到量化的区间$i$，量化的值按如下计算：

$$
q_i=\frac1 2 (Q_X(\frac{i}{2^k+1})+Q_X(\frac{i+1}{2^k+1}))
$$

对称的 k-bits 量化的一个问题是，这种方法没有零的精确表示（这在做类似 padding 时很重要），解决办法是做成非对称的，$2^{k-1}$个表示负的，$2^{k-1}+1$个表示正的

**这部分可能要看下代码才能更清楚**

**Double Quantization**

这步其实是对量化常数（上面量化计算中的系数）做进一步量化，这块有收益主要是因为采取分块策略，导致一个大矩阵会有大量的量化常数，文中这一步采用 fp8 进行量化，没有观察到显著的性能下降

**Paged Optimizers**

这块主要是用 NVIDIA 的 unified memory（在 gpu memory 不足时与 cpu memory 交换的优化），文章主要用来优化器状态上

参考：

https://zhuanlan.zhihu.com/p/648239462

### AdaLoRA

https://arxiv.org/abs/2303.10512

这篇论文提出了一种名为AdaLoRA的参数高效微调方法，它在LoRA的基础上进行了改进。LoRA是一种低秩适应方法，通过将预训练权重的增量更新表示为两个小矩阵的乘积来减少微调参数的数量。然而，LoRA在所有预训练权重矩阵的增量矩阵$\Delta W$假设了相同的 rank 值，这是本文的出发点

AdaLoRA的主要改进在于：

1. 将增量更新参数表示为奇异值分解的形式，这使得可以更有效地修剪不重要的奇异值，从而减少参数预算。

2. 提出了一种基于敏感性和不确定性的重要性度量，用于量化每个权重矩阵在模型性能上的贡献。这使得AdaLoRA能够根据权重矩阵的重要性动态地调整增量矩阵的秩。

3. 设计了一个全局预算调度器，从较高的初始预算逐渐降低到目标预算，以便在训练初期探索参数空间，然后在后期关注更重要的权重。

通过这些改进，AdaLoRA在自然语言处理、问答和自然语言生成任务上的性能优于LoRA和其他基线方法，特别是在低预算设置下。

**看上去很有道理，但工程实践起来比较麻烦且效果不一定保障**
