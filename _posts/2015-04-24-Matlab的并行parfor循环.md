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
parfor循环的语法和普通的for语法没有什么区别，但是有以下几点需要注意的！  

Matlab并行是基于**client-worker模式**的。首先，Matlab有个总体负责的client，它将任务合理分配给每个worker，worker的个数即为 `matlabpool open size` 的size。每个worker运行完之后将结果回传给client。这样，当client接收到所有的结果后，程序运行完成。   
打开了Matlab并行开关之后，可在任务管理器中看到Matlab进程的数目是**size+1**，即为client和worker的数目之和。在运行过程中，worker会满载CPU；但是client不会满载CPU，它只负责分配任务、传递数据和最后的数据采集。

## parfor关键字的使用
由for关键字引导的循环通常为串行运行，如果改为parfor则可以由多个worker以并行方式执行。  
parfor可以将n次循环分解为独立不相关的m部分，然后将各部分分别交给一个worker执行。  
循环执行的结果应该与n次循环执行的顺序**无关**。   

## worker配置

### 并行数n的选择
该方法仅针对多核机器做并行计算！  
n可以不等于核心数量（物理核数），但如果n小于核心数量则核心利用率没有最大化，如果n大于核心数量则效率反而可能下降！  
因此单核机器就不要折腾并行计算了，否则速度还更慢！  
执行命令 `feature('numCores')` 即可显示CPU的**物理核数**。  
如果有c个CPU核心，通常可以设置为c。如果是远程服务器，为**防止服务器响应卡顿**，可以设置为 `c-1`。  
如果电脑是双核四线程的，那么只能申两个（而非4个）matlab local pool！对于计算密集型程序，超线程带来的性能提升几乎为0，可以设置为核心数，而不是线程数！

并行计算时，在一台机器上运行4个worker是没有必要的！  
因为还是1个Core在工作，这样并不会提高你的运算速度，相反还有可能使运算速度变慢。  
只有你的电脑有与worker数量相同的Core时，才能最高效率地加快运算速度！    

那么是不是只能最多运行4个worker呢？
不是的！你可以组成一个机群，也就是说使几台机器连起来一起工作。
为了使几台机器上的资源像是在一台机器上一样，Matlab提供了MDCS（MATLAB  Distributed Computing Server），通过它你就可以很方便地将几台机器连起来一同工作了。

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
程序运行结束后，应该使用 `matlabpool close` 关闭worker。Matlab关闭后，matlabpool也会自动关闭，一般没必要手动关闭（除非想要修改n的值）。   
配置项的修改可以通过 `Parallel -> Manage Cluster Profile` 完成。  



## parfor中的变量类型

Matlab的parfor循环内的变量可分为**五大类**，parfor对这五类变量有不同的处理方式。如果Matlab无法对parfor内的某变量进行归类，或者该变量不满足该类别变量的要求，就会导致出错，此时便不能使用parfor。
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

### 简约变量（Reduction 变量）
一般parfor中各次循环对应的运算应该相互独立，但简约操作可以在多次循环内同时对一个变量操作。
例如下方代码中x2就是简约变量：
{% highlight matlab %}  
x2 = [];
n = 10;
parfor i = 1:n
    x2 = [x2, i];
    i,
end
%有意思的是，i并不一定按照1到10的顺序输出，但x2的结果却必然是1到10
{% endhighlight %}

简约操作包括 `+` 、 `-` 、 `*` 、`.` 、`*` 、 `&` 、 `|` 、 `[,]` 、 `[;]` 、 `{,}` 、 `{;}` 、 `min` 、 `max` 、 `union` 、 `intersect` 。  
同一个parfor循环对简约变量的操作必须一致，即必须是同一种简约操作符。而且与操作符的相对位置也必须一致。  
简约变量赋值表达式应该满足结合律和交换律。 `*` 、 `[]` 、 `{}` 底层有特殊处理保证结果的正确性。  

### 切片变量（Sliced变量）
每次循环只访问该变量的特定位置，与Loop 变量有关。如果该变量是输出变量，访问必须是**连续**的。    
矩阵如果被Matlab识别为切片变量，则数据可以分段传输到各worker，提高传输效率。  
切片变量矩阵的大小是不可在循环中改变的！
{% highlight matlab %} 
parfor i = 1:n
	x(i) = a(2*i*i);  % allowed;
    y(i+2) = a(i) + b(i+1); % allowed;
    c(i+1) = c(i) + 1; % not allowed;
    z(2*i) = i; % not allowed;
    a(i) = [];  % not allowed;
    a(end+1)=i; % not allowed
end
{% endhighlight %} 

在一个循环体内只能出现一个切片变量，简单说就是一个循环体内只能出现切片数组的一个元素。  
`a(i)=temp1;a(i+1)=temp2;` 是不正确的！  
另外，切片变量的下标一定要连续，例 `a(2*i)=temp;`是不行的！  

### 循环变量（Loop 变量）
如上例中的i，表示当前循环的id。  
限制是循环内不能对循环变量再次赋值！  
循环变量一定是**连续增加**的整数！`parfor i=[1 3]`将会出错：  
`The range of a parfor statement must be increasing consecutive integers`

### 广播变量（Broadcast 变量，外部变量）
在parfor之前赋值，在循环内未被重新赋值，只进行读取操作。

### 临时变量（Temporary 变量）
作用域局限于循环内，循环结束后不存在。不影响parfor之前声明的同名变量。  
该变量不会通过Loop 变量引用，否则归类为Sliced 变量！





## parfor的使用
将传统的for循环改为parfor循环，就会将循环体作为整体分到到一个个处理器中，从而一次性进行多组运算。
在使用parfor时，代码的编写有一些注意事项，最主要的是其中变量的处理！

在parfor中，变量不再是随心所欲的使用，有着其自己的分类。
运行时，经常遇到的错误 `Error: The variable xxx in a parfor cannot be classified.` 就是变量xxx不能被正常划分到正确的类别中。





## 注意

### parfor与spmd不可以相互或者自身嵌套。
spmd（Single Program Multiple Data，单程序多任务进行任务并行）是Matlab实现并行计算的另外一种方法，详见文末。  

### 要注意程序的细节
很多地方都会报错。比如下标必须为**连续**的整数！

### 循环次数n的选择
循环次数m最好能**整除**以worker个数n，否则部分worker会分配较多的循环，造成一部分worker闲置一段时间，降低了并行性。  
并行运行时各个worker之间会进行通信，要注意大量数据传输带来的性能下降。尤其对于广播变量，如果较大可尝试变为切片变量。

### 慎用（最好勿用）eval赋值
一个程序并行时要共享内存，而eval语句可能使程序进入错误的workspace，因此不要用eval，改用不同index赋值。

### Matlab版本引起的问题
从**Matlab R2013b**开始，**parpool**命令取代了matlabpool命令。

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
（本人电脑是双核的，而其他四核电脑配置的是R2013b，暂均无法正常实验）

### 错误示例
如果在嵌套循环中引用了矩阵，那么在parfor-loop中就不可以再其他地方再使用：
{% highlight matlab %} 
A = zeros(4, 10);
parfor i = 1:4
    for j = 1:10
        A(i, j) = i + j;
    end
    disp(A(i, 1))
end
{% endhighlight %}

是错误的，Matlab将提示 `Error: The variable A in a parfor cannot be classified.  
See Parallel for Loops in MATLAB, "Overview".` 。  
 在循环中disp又引用了矩阵A，导致矩阵A变量类型无法归类。  
可以改成：
{% highlight matlab %} 
A = zeros(4, 10);
parfor i = 1:4
    v = zeros(1, 10);
    for j = 1:10
        v(j) = i + j;
    end
    disp(v(1))
    A(i, :) = v;
end
{% endhighlight %}

## 程序的调试
一般推荐先用for循环写好，程序运行正常之后把for改成parfor，这时如果运气好的话可以直接并行运行，但往往会报一些warning或者error，这就要考验变量分类了。
而且就算没报warning或者error，程序在运行过程中也可能出错，往往这种错误**不像**串行程序报出是在哪一行的错误，并行的错误一般都会报出“No Remote Error Stack”，按照字面意思就知道，没有栈信息。
因为按照普通的代码，调用函数前会先将变量入栈以及保护现场，而到了并行编程，每个worker出错的具体情况是**不会**回传给client的，很难通过返回到command window的错误信息判断程序到底出了什么错。  

在parfor循环内部也是**不能加断点**的，本来不同的循环变量针对的就是不同的worker，在parfor内部加断点的话，软件根本不知道编程人员想要看哪个worker的数据。

不过，Matlab还提供了另一种并行调试模式——**pmode**，主要命令下面3种：  
{% highlight matlab %} 
pmode start % 开启并行调试（默认核数）  
pmode start size % 开启size个核的并行调试  
pmode exit % 退出并行调试模式  
{% endhighlight %}
pmode并不好用。用该调试模式的话需要手动一行一行输m语言。而且在每个worker中的变量还需要手动传递到client，就是各种不方便。

## Matlab并行计算工具箱及MDCE介绍
（内容待添加）

## Matlab基于cluster的并行
（内容待添加）

## SPMD（Single Program Multiple Data，即单指令多数据）
（内容待添加）

# 有问题请反馈
在使用中有任何问题，欢迎反馈给我！
