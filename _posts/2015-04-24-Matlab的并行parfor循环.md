---
layout: post
title:  "Matlab的并行parfor循环"
date:   2015-04-24 17:04:17
categories: matlab
---

## parfor适用情况
parfor只用于Matlab并行循环。当需要**简单计算的多次循环迭代**时，例如蒙特卡洛(Monte Carlo)模拟。
parfor将循环迭代分组，那么每个worker执行迭代的一部分。当**迭代耗时很长**的时候parfor循环也是有用的，因为workers可以同时执行迭代。

注意当循环中有迭代**依赖其他迭代的结果**时不应该使用parfor循环。每个迭代都必须不依赖其他迭代。
由于parfor循环内有通信消耗，当只有**小数量的简单计算**时使用parfor可能得不到什么好处。


## Matlab的parfor并行编程
通常消耗最多计算资源的程序往往是循环。将循环并行化，或者优化循环体中的代码是最常用的加快程序运行速度的思路。  
Matlab提供了parfor关键字，可以很方便的在多核机器或集群上实现并行计算。  

## parfor关键字的使用
由for关键字引导的循环通常为串行运行，如果改为parfor则可以由多个worker以并行方式执行。  
parfor可以将n次循环分解为独立不相关的m部分，然后将各部分分别交给一个worker执行。  
循环执行的结果应该与n次循环执行的顺序无关。   

## parfor中的变量类型

### 简约变量
一般parfor中各次循环对应的运算应该相互独立，但简约操作可以在多次循环内同时对一个变量操作。这种变量称为简约变量。  
例如下方代码中a就是简约变量：
{% highlight matlab %}  
a = 0;
for i = 1:1000
  a = a+i;
end
{% endhighlight %}

简约操作包括 `+` 、 `-` 、 `*` 、`.` 、`*` 、 `&` 、 `|` 、 `[,]` 、 `[;]` 、 `{,}` 、 `{;}` 、 `min` 、 `max` 、 `union` 、 `intersect` 。  
同一个parfor循环对简约变量的操作必须一致，即必须是同一种简约操作符。而且与操作符的相对位置也必须一致。  
简约变量赋值表达式应该满足结合律和交换律。 `*` 、 `[]` 、 `{}` 底层有特殊处理保证结果的正确性。  

### 切片变量（sliced变量）
parfor中可能需要读取或写入parfor之外的矩阵，读取写入位置与循环变量相关。这样就需要向worker传输大量的数据。  
矩阵如果被Matlab识别为切片变量，则数据可以分段传输到各worker，提高传输效率。  
切片变量矩阵的大小是不可在parfor中改变的！

在一个循环体内只能出现一个切片变量，简单说就是一个循环体内只能出现切片数组的一个元素。  
`a(i)=temp1;a(i+1)=temp2;` 是不正确的！  
另外，切片变量的下标一定要连续，例 `a(2*i)=temp;`是不行的！  

### 循环变量
如上例中的i，表示当前循环的id。

### 广播变量
在parfor之前赋值，在parfor内只进行读取操作。

### 临时变量
作用域局限于parfor内，parfor结束后不存在。不影响parfor之前声明的同名变量。  
{% highlight matlab %}  
tmp = 5;
broadcast = 1;							%广播变量，每次循环中的值不变。
reduced = 0;
sliced = ones(1, 10);
parfor i = 1:10							%i为循环变量
	tmp = i;							%parfor中的tmp是临时变量，parfor结束后tmp的值依然是5，不受临时变量的影响
	reduced = reduced + i + broadcast;  %简约变量，Matlab对其的值将分段由各worker计算后送回主进程处理。
	sliced(i) = sliced(i) * i;			%sliced为切片变量，数据传输有优化提升。
end
{% endhighlight %}

## worker配置

### 并行数n的选择
该方法仅针对多核机器做并行计算！  
n可以不等于核心数量，但如果n小于核心数量则核心利用率没有最大化，如果n大于核心数量则效率反而可能下降！  
因此单核机器就不要折腾并行计算了，否则速度还更慢！  
执行命令 `feature('numCores')` 即可显示CPU的物理核数。  
如果有c个CPU核心，通常可以设置为c。如果是远程服务器，为**防止服务器响应卡顿**，可以设置为 `c-1`。  
如果电脑是双核四线程的，那么只能申两个（而非4个）matlab local pool！对于计算密集型程序，超线程带来的性能提升几乎为0，可以设置为核心数，而不是线程数！

并行计算时，在一台机器上运行4个worker是没有必要的！  
因为还是1个Core在工作，这样并不会提高你的运算速度，相反还有可能使运算速度变慢。  
只有你的电脑有与worker数量相同的Core时，才能最高效率地加快运算速度！    

那么是不是只能最多运行4个worker呢？
不是的！你可以组成一个机群，也就是说使几台机器连起来一起工作。
为了使几台机器上的资源像是在一台机器上一样，Matlab提供了MATLAB  Distributed Computing Server，通过它你就可以很方便地将几台机器连起来一同工作了。

在进行并行计算时，Matlab中对worker的数量有这样一个限制：“在同一台机器上最多运行有4个worker”，因为目前还没有超过4个core的CPU。
那么是否可以在一台单核机器上运行4个worker呢？完全可以！
因为matlab本身提供了一种“虚拟机“，这样即使你的机器是单核的也同样可以运行最多达4个worker(matlab中在并行计算时将worker称为lab)。
最明显的例子就是pmode，当你打开pmode时，就弹出有4个lab，每个lab对应一台虚拟的电脑。

### 启动worker
在运行程序之前，一定需要配置worker！否则parfor循环将以普通for循环的形式运行，无法并行！  
使用`matlabpool`命令可以开启关闭本机的并行计算池。 
`matlabpool('size')` 或 `matlabpool size` 命令可检查并行计算池状态，若不大于0则为关闭状态。
`matlabpool n` 命令可以打开n个worker。`matlabpool open configname` 按照指定配置打开，默认配置为 `local` 。 
以上两句也可通过 `matlabpool('open','local',n);` 来配置。  
程序运行结束后，应该使用 `matlabpool close` 关闭worker。Matlab关闭后，matlabpool也会自动关闭，一般没必要手动关闭。  
配置项的修改可以通过 `Parallel -> Manage Cluster Profile` 完成。  



## parfor的使用
将传统的for循环改为parfor循环，就会将循环体作为整体分到到一个个处理器中，从而一次性进行多组运算。
在使用parfor时，代码的编写有一些注意事项，最主要的是其中变量的处理！

在parfor中，变量不再是随心所欲的使用，有着其自己的分类。
运行时，经常遇到的错误 `Error: The variable xxx in a parfor cannot be classified.` 就是变量xxx不能被正常划分到正确的类别中。





## 注意
### parfor、spmd不可以相互或者自身嵌套。


### 要注意程序的细节
很多地方都会报错。比如下标必须为连续的整数！

### 循环次数n的选择
循环次数m最好能整除以worker个数n，否则部分worker会分配较多的循环，造成一部分worker闲置一段时间，降低了并行性。  
并行运行时各个worker之间会进行通信，要注意大量数据传输带来的性能下降。尤其对于广播变量，如果较大可尝试变为切片变量。

### 慎用（最好勿用）eval赋值
一个程序并行时要共享内存，而eval语句可能使程序进入错误的workspace，因此不要用eval，改用不同index赋值。

### Matlab版本引起的问题
从R2013b开始，parpool命令取代了matlabpool命令。

### 输出顺序问题
for 语句是按照i的序列顺序执行的，而parfor是由多个worker同时执行i为不同值的结果：  
`for i = 1:12 fprintf(' %d',i); end` 的结果为 `1 2 3 4 5 6 7 8 9 10 11 12`；  
而 `parfor i = 1:12 fprintf(' %d',i); end` 每次执行的顺序不一样，在n=2时，本人尝试了几次，结果依次如下：
{% highlight matlab %}  
 4 3 2 1 11 12
 8 7 6 5 10 9
 
 4 3 2 1 10 9 12
 8 7 6 5 11
 
 4 3 2 1 10 9 11
 8 7 6 5 12
 
 4 3 2 1 11
 8 7 6 5 10 9 12
{% endhighlight %}
在n=3时，结果依次如下：

## Matlab 并行计算工具箱及MDCE介绍

### 有问题反馈
在使用中有任何问题，欢迎反馈给我！
