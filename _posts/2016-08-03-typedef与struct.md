---
layout: post
title:  "typedef与struct"
date:   2016-08-03 10:35:47
categories: test
---


# 在C中定义一个结构体类型要用typedef  
`typedef struct Student{int a;}Stu;`  
在声明变量的时候就可：`Stu stu1;` (如果没有typedef就必须用`struct Student stu1`;来声明)
这里的Stu实际上就是struct Student的别名。

另外这里也可以不写Student，`typedef struct{int a;}Stu;`

# 但在C++里很简单
直接`struct Student{int a;};`　

# 有问题请反馈
在使用中有任何问题，欢迎反馈给我！
