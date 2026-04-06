## 简介

**原文：**[https://arxiv.org/abs/2402.1562](https://arxiv.org/abs/2402.15627)[7](https://arxiv.org/abs/2402.15627)

**作者：**字节和北大

**模型：**175B LLM model

**GPU数：**12,288

**效果：**Even in the largest scale with 12,288 GPUs, MegaScale still outperforms Megatron-LM by 14% MFU. For the smaller scale training, the speedup of MegaScale over the baseline ranges from 1.23× to 1.32×.

## 主要工作

**一套支持了字节的万卡集群的方案，核心是两点：一是系统、算法的联合设计，二是深入的可观测性能力建设。**

. ![](技术开发/LLM/Paper学习/MegaScale%20Scaling%20Large%20Language%20Model%20Training%20to%20More%20Than%2010,000%20GPUs/1.png)

### 系统、算法联合设计相关工作

1. 算法优化：包括 parallel transformer block, sliding window attention and LAMB optimizer.
2. 混合并行训练：leverage mixed parallelism strategies that combine data parallelism, pipeline parallelism, tensor parallelism, and sequence parallelism.
3. 计算与通信并行最大化： 两者不要互相等

- 数据并行：主要是all-gather和reduce-scatter操作。all-gather在foraward前做，比如在数据load时，这样forward计算不需要等。reduce-scatter在backward时做，backward计算也不用等。
- 流水线并行：主要是解耦send和receive操作。把send和receive变成异步，当一个阶段正在计算时，另一个阶段可以同步地进行send操作。
- 张量并行：将通信操作（如参数同步）和计算操作（如矩阵乘和激活函数）错开执行，这样可以在等待数据同步的同时完成一些计算任务

4. 集合通信组初始化：

- 采用非阻塞的异步操作，比如用redis 替换TCPStore
- 降低全局barrier的使用

5. 网络性能优化：

- 网络拓扑设计，降低ECMP哈希冲突
- 拥塞控制算法设计
- 调整重传超时参数

### 可观测性相关工作

1. **采集**

- heartbeat信息，包括IP、Pod信息、硬件信息、进程信息、stdout/stderr log等
- RDMA traffic metrics
- cuda event
- timeout log with errorcode

2. **节点异常检测与告警**

- heartbeat丢失
- 进程状态异常
- 错误日志
- RDMA traffic metric丢失周期性

3. **节点故障定位**

挂起当前训练任务，执行故障测试

- Intra-host network tests，单机网络测试，帮助检测连通性、带宽、配置是否有异常

- RNIC <-> GPU
- RNIC <-> Memory
- RNIC <-> RNIC(same host)

- NCCL tests，帮助 identify potential faults in GPU communication

- host: GPU <-> all GPU
- same TOR: GPU <-> neighbor GPU，all-reduce test

4. **节点快速恢复**

- checkpoint：两阶段阶段，GPU-> host Memory, async host Memory -> HDFS
- recovery：两步，HDFS -> one GPU, GPU -> other GPU(共享相同数据的)

5. **复杂场景故障定位、性能优化和可视化**

典型场景：配置理论相同但实际不同，MFU劣化

关键代码段执行时间：不同于torch profiler，基于cuda event做的

- 慢节点识别并剔除

- heatmap可视化耗时
- 识别慢节点、并剔除

![](技术开发/LLM/Paper学习/MegaScale%20Scaling%20Large%20Language%20Model%20Training%20to%20More%20Than%2010,000%20GPUs/2.png)

- 流水线并行依赖情况识别

- 分布式trace可视化
- 识别流水线并行的依赖情况

![](技术开发/LLM/Paper学习/MegaScale%20Scaling%20Large%20Language%20Model%20Training%20to%20More%20Than%2010,000%20GPUs/3.png)