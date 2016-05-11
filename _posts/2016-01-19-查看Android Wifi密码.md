---
layout: post
title:  "查看Android Wifi密码"
date:   2016-01-19 22:22:17
categories: knack
---


### 方法
	要想查看Android手机的Wifi密码，必须先获取手机的**root权限**！！！  

1.下载并安装好 **RE管理器**
2.进入data/misc/wifi（RE管理器可以显示**隐藏的文件**，手机自带的文件系统不行）
3.使用文本查看器打开bcm_supp.conf或wpa_supplicant.conf


打开后将是类似于以下的内容：
{% highlight java %}  
network={  
	ssid="TP-LINK_92E3C2"  
	psk="cheng21210882"  
	key_mgmt=WPA-PSK  
	priority=10  
} 
{% endhighlight %}

`ssid=`后面的就是你连接的WIFI网络名。  
`psk=`后面的就是此WIFI网络的**密码**。  
`key_mgmt=`后面内容为加密类型。  
`priority=`越小优先级越高  

	还能同时修改一下自动连接wifi的**优先级**顺序哦！ O(∩_∩)O~


### 有问题反馈
在使用中有任何问题，欢迎反馈给我！
