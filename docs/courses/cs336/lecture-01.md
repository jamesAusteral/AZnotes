以下是试用草稿内容。

# L01

第一个作业的workload等于cs224n的所有作业总和。

## 模型架构
transformer依旧是模型的支柱。
自2017年以来还有很多的小改进：
激活函数
位置嵌入
归一化
MLP
注意力
低维注意力
空间状态模型
## 训练
Optimizer
Learning rate schedule
Batch size
regularization
Hyperparameters

## tokenizer
BPE
Tokenizer free approaches，直接输入词，但是目前没有大规模使用
所以现在还是用BPE

## 第一个作业
1. 写一个BPE
2. 写一个transformer，交叉熵损失。。。
3. 训练数据集
4. 还有一个记分板

## 系统方面
怎么更进一步优化，榨干硬件性能
kernels, parallelism, inference

### kernels
数据在memory和计算单元中传输
triton

### 并行
GPU之间的数据传输
data parral
tensor parral
反正就是多种方法，之后会谈到。
### 推理
关于改怎么使用一个模型
可能不会关注很多，因为课程量已经很大了。

decoding部分，memory会很快塞满。
以及很多其他方法。

## 第二个作业
用triton做一个kernel
2026的会与去年的有点不同，但是大体上相同。

How to scale your model
by google

## 第三个作业
scaling law 
大模型的训练成本很高。

限定浮点数运算的成本下，我应该加大模型尺寸，还是加训练token量。
classic compute-optimal scaling laws. 

我们可以预测一个训练效果！
定义一个training API

## 第四个作业
数据
evaluation等等


## 第五个作业
对齐

## tokenization
文本就是unicode字符串，我们要把字符串转换成一串目录。
tokenizer就是做以上步骤的正反向，来回翻译。

压缩率: compression ratio: bites/ tokens

unicode能表示150k个字符，但是很多字符都很少被用到。
