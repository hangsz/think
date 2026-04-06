paper: [https://arxiv.org/pdf/1910.02054](https://arxiv.org/pdf/1910.02054)

## 摘要

大模型的数据和参数需求与有限的设备存储是一对矛盾，当前的方案解决的都不好，本文提出了ZeRO的全新方案来解决， 核心思路是降低内存冗余（在不怎么增加通信开销的情况下），以达成更多的设备可以训练更大模型的目标。

![](技术开发/LLM/Paper学习/ZeRO%20Memory%20Optimizations%20Toward%20Training%20Trillion%20Parameter%20Models/1.png)

## 效果

在与普通DP通信量保持不变的情况下，最高实现Nd(DP degree)倍多的缓存降低，也即模型大小的提升

![](技术开发/LLM/Paper学习/ZeRO%20Memory%20Optimizations%20Toward%20Training%20Trillion%20Parameter%20Models/2.png)

## 主要工作

ZeRO-DP：关注优化器状态/梯度/参数

ZeRO-R：关注激活值、临时buffer等临时使用的值

**重点看下ZeRO-DP的工作。**

假设参数量为：Ψ，fp16精度存储，参数和梯度显存占用4Ψ，优化器显存占用与优化器有关

**第一步：优化器状态分片**

forward：每个rank 对优化器状态进行all gather获得全量的优化器状态

通信量：没有增加，因为标准DP也要优化器状态保持一致(rank0 broadcast)，all gather与broadcast通信量相同

显示占用：降低为1/4

![](技术开发/LLM/Paper学习/ZeRO%20Memory%20Optimizations%20Toward%20Training%20Trillion%20Parameter%20Models/3.png)

**第二步：梯度分片**

backward: 每个rank对梯度进行reduce-scatter，不需要拿到全部最新梯度，只需要自己的分片部分。最后对参数进行all gather，获得完整的参数。

通信量：没有增加，因为标准DP需要进行allreduce，而本方案需要reuduce scatter（梯度)，all gather（参数）

显示占用：与第一步共同降低为1/8

![](技术开发/LLM/Paper学习/ZeRO%20Memory%20Optimizations%20Toward%20Training%20Trillion%20Parameter%20Models/4.png)

**第三步：参数分片**

forward: 每个rank对参数进行all gather，用完即丢

backward：每个rank还需要进行all gather，用完即丢。但这一步和梯度分片中的all gather重合

通信量：增加与参数量大小相同的通信量，总计增加为之前的1.5倍

显存占用：与前两步共同降低为1/分片数