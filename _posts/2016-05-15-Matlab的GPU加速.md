---
layout: post
title:  "Matlab的GPU加速"
date:   2016-05-15 19:48:00
categories: matlab
---


## GPU加速原理

随着显卡的发展，GPU越来越强大，第一代统一渲染架构的GTX 280核心中就已经拥有240个单独的ALU，因此非常适合并行计算，而且浮点处理能力也远远优于目前的多核CPU，加上GPU为显示图像做了优化。
在众多计算领域上已经超越了通用的CPU。NVIDIA于2006年推出[CUDA](https://developer.nvidia.com/cuda-downloads)（Compute Unified Devices Architecture），可以利用其推出的GPU进行通用计算，将并行计算从大型集群扩展到了普通显卡，使得用户只需要一台带有Geforce显卡的笔记本就能跑较大规模的并行处理程序。
CUDA(Compute Unified Device Architecture)工具包是一种针对支持CUDA功能的GPU（图形处理器）的C语言开发环境，未来还将发布Fortran语言版本。

使用Matlab的时候，很多人都用过里面的并行工具箱，用的最多的应该就是parfor。
实际上，Matlab里面已经有不少工具箱里面都有了支持GPU加速的函数。使用Matlab+GPU加速的前提是，机器必须安装了支持**CUDA**的显卡，而且CUDA驱动的版本在1.3以上。

Matlab目前只支持Nvidia的显卡。如果你的显卡是AMD的或者是Intel的，就得考虑另寻它路了。
并不是所有的电脑都可以用Matlab进行GPU加速计算。想知道自己的电脑有没有这个能力，在Matlab中运行 `gpuDevice` 试试吧！

## GPU加速示例

fft，ifft，三角函数，相关函数xcorr以及常用的运算符等都可以进行加速。
方法也很简单，主要使用到`gpuArray`和`gather`这两个函数。

### 把数据从CPU拷贝到GPU
以xcorr为例，假设求向量A和B的互相关，一般是使用命令 `M = xcorr(A,B)` 。

以下是使用gpu加速的版本：
Ag = gpuArray(A);
Bg = gpuArray(B);
Mg = xcorr(Ag,Bg);
M = gather(Mg);

gpuArray是把数据转换为GPU处理的类型，存储到GPU的显存里。gather是将数据转移回来。除了将现有的矩阵转移到GPU中，gpuArray自带的方法还可以直接在GPU中创建数据。
把数据从CPU拷贝到GPU上非常简单，只要Ag = gpuArray(A);就可以了，实际上Matlab并没有规定一个矩阵定义之后不能改类型。
有时候GPU受限于硬件架构，单精度的计算远快于双精度。这时候可以考虑在拷贝的时候顺便转换一下精度：A = gpuArray(single(B));

一般的小矩阵可能感觉不出来，不过如果矩阵规模很大，而且在多次循环内部，这个区别就很明显了。
mathwork的网站上有对xcorr的gpu加速效率的详细分析报告，基本上是随着矩阵规模扩大，gpu加速的倍数是直线上升。

### 直接在GPU上设置数据
A = zeros(10, 'gpuArray');
这时候运行size函数：size (A)
得到 100 100。所以A其实是一个二维数组。

也可以生成一个一维的随机数组：
r = gpuArray.rand(1, 100) % 一行，一百列
运行class函数：class(r)得到
ans = gpuArray。可见这是一个在gpu上的数组。


下面就是对GPU数据进行操作。首先，MATLAB定义了GPU上丰富的库函数，比如快速傅立叶变换：
 result = fft(r)
这样result就是另一个GPU上的数组，存储了对r做fft的结果。
加减乘除更不在话下：
r2 = (real(result) + r ) / 2
作用是对result取实部之后加r再除以2。这里r2, r, result都是GPU上的数组。


除了相关函数，还有很多支持gpuArray数据类型的函数，具体可以用 `methods('gpuArray')` 查看，其中某个函数的说明可以用 `help gpuArray/functionname` 查看：

gpuArray.ones	gpuArray.colon
gpuArray.zeros	gpuArray.rand
gpuArray.inf	gpuArray.randi
gpuArray.nan	gpuArray.randn
gpuArray.true	gpuArray.linspace
gpuArray.false	gpuArray.logspace
gpuArray.eye

其实，这些函数的用法和对应的普通函数的用法都是类似的。
II = gpuArray.eye(1024,'int32');
size(II)

1024 1024

也可以用下面的命令生成随机数：
parallel.gpu.rng
parallel.gpu.RandStream

值得注意的是，GPU的数据是要存到显存里面的，**显存**可没有内存那么大，虽然Maltab和CUDA为我们做了很多显存管理的工作，但是我们还是要保证处理的矩阵不会把显存撑爆！

对于一般的例子，cpu和gpu版本程序进行对比，改动其实不太大。但对于一些复杂程序，无法用Matlab内部函数进行GPU加速的代码，matlab还提供了一个更强大的工具，就是调用.cu文件。Matlab也对其提供了一套方法来调用，最终编译成.ptx文件。


### 有问题反馈
在使用中有任何问题，欢迎反馈给我！
