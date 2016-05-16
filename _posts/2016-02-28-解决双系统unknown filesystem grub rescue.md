---
layout: post
title:  "解决双系统unknown filesystem grub rescue"
date:   2016-02-28 13:28:43
categories: knack
---

### 症状
双系统电脑出现了故障方法，显示  
`error：unknown filesystem grub rescue`


### 解决方案

1.找出你的Linux盘在那个分区以及grub目录在什么位置  
如果你还记得最好，忘了也无所谓，使用下面命令逐个试探即可。  
`grub rescue>ls`  //列出本机所有磁盘及分区
结果显示类似于`hd0,(hd0,1) ,(hd0,4),(hd0,7),(hd0,8),(hd0,9)`等  
  
循环使用类似于下面的命令，直至显示该分区所包含内容而不是`unknown filesystem`   
`grub rescue>ls (hd0,0)/grub`   
假设我们试到(hd0,8)时，成功显示了内容，则表示grub目录在(hd0,8)。

2.设置路径  
`grub rescue>set  root=(hd0,8) `  //括号里为上一步尝试成功的分区  
`grub rescue>set  prefix=(hd0,8)/grub`

3.退出grub rescue模式  
`grub rescue>insmod  /grub/normal.mod`  
退出了grub rescue模式，进入了熟悉的grub模式。  

4.确定设置  
`grub>normal`

5.修复grub  
进入Linux系统后，执行  
`sudo  update-grub`

6.安装grub  
`sudo grub-install /dev/sda`              //sda是你的启动磁盘


可参考[百度经验](http://jingyan.baidu.com/article/c85b7a640cd7d6003bac95f8.html)

### 有问题反馈
在使用中有任何问题，欢迎反馈给我！
