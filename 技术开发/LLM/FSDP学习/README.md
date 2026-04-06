预先阅读：

[[paper] PyTorch Distributed: Experiences on Accelerating Data Parallel Training](https://yuque.antfin.com/ctfsce/gycrh4/tzdly0ckg3l0oby5 "[paper] PyTorch Distributed: Experiences on Accelerating Data Parallel Training")

[https://engineering.fb.com/2021/07/15/open-source/fsdp/](https://engineering.fb.com/2021/07/15/open-source/fsdp/)

## DDP训练流程

![](技术开发/LLM/FSDP学习/1.png)

forward：

1. 每个rank持有相同的参数
2. 通过rank0的broadcast使得各个rank获取相同的model buffer

[pytorch/torch/nn/parallel/distributed.py at main · pytorch/pytorch](https://github.com/pytorch/pytorch/blob/main/torch/nn/parallel/distributed.py)

![](https://cdn.nlark.com/yuque/0/2024/png/12409702/1721899419862-da55b7cb-7cf3-4aad-a8f1-62e92cd5c057.png)

3. 每个rank自身，前向传播，一直到获取loss

backward：

1. 每个rank自身，反向传播，计算出各层各参数的gradient
2. 多个参数为一组进行all reduce，组顺序为反向传播顺序
3. 优化器更新rank的参数

Note：

- model buffer：不需要backward更新的参数，但是模型必不可少的一部分，比如batchnorm层的running_mean值或者有状态优化器的状态

## DDP存在的问题

**每个rank需要持有完整的模型，对显存有压力，会限制训练模型大小**

## FSDP的训练流程

![](技术开发/LLM/FSDP学习/2.png)

forward：

1. 每个rank持有一个模型参数的一部分(shard)
2. 通过all gather获取参数/buffer(几层参数属于一个FSDP组，有多个FSDP组)
3. 每个rank自身，前向传播，一直到获取loss
4. 丢掉不用的参数/buffer

backward：

1. 通过all gather获获取参数/buffer，还是按照FSDP组粒度
2. 每个rank自身，反向传播，计算出各层各参数的gradient
3. 通过reduce scatter获取梯度，只获取自己shard参数对应部分的梯度就行，也是按照FSDP组粒度
4. 优化器更新rank shard的参数
5. 丢掉不用的参数/buffer