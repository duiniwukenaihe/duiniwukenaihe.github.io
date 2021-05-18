---
layout: post
title: 2021-04-26-Centos8-Failed to restart network.service_ Unit network.service not found
date: 2021-04-26 4:00:00
category: Centos
tags: centos network
author: duiniwukenaihe
---
* content
{:toc}

# 背景：

线上都是云商的服务器有小部分centos8但是这个的一般都没有重启过network服务的，依稀还能记得systemctl restart network命名。今天闲来无事内网搭建一台centos8服务器。安装完毕后修改network配置：ONBOOT=yes.设置系统启动时激活网卡：

```
 vi  /etc/sysconfig/network-scripts/ifcfg-ens18
```

当然了 ifcfg后面的应该都是不一样的一般都是eth0

![image.png](/assets/images/2021/04-26/k5wcylsxhy.png)

然后systemctl  restart network.就出现了Failed to restart network.service: Unit network.service not found......

what？网络服务没有了？

![image.png](/assets/images/2021/04-26/fx6t9vx67h.png)

百度搜索下一下相关信息。详见：[https://www.cnblogs.com/djlsunshine/p/9733182.html](https://www.cnblogs.com/djlsunshine/p/9733182.html)。centos8启动了nmcli的命令去做这些配置:

```
nmcli networking off
nmcli networking on
```

我就嘴简单的办法 off下然后启动下先激活网卡了：

![image.png](/assets/images/2021/04-26/b1twpy1hsn.png)

# 一. 详细了解一下nmcli

## 1.nmcli --help

了解一个命令的方式最简单的应该是看他的文档。linux中常用的就是--help，命令的基本格式：nmcli OPTIONS... ARGUMENTS...

![image.png](/assets/images/2021/04-26/6kj5gpczkj.png)

nmcli c h 查看connection帮助信息

![image.png](/assets/images/2021/04-26/8gjmujm30n.png)

不求甚解，还是研究下自己需要的吧......

## 2.个人觉得一些有用的命令：

### 1. nmcli c 查看连接信息

```
[root@bogon ~]# nmcli c
NAME   UUID                                  TYPE      DEVICE 
ens18  354e9f6f-82a2-4df9-9950-e3df760da4b8  ethernet  ens18  
```

### 2. nmcli d show ens18 显示网卡详细信息

```
[root@bogon ~]# nmcli d show ens18
GENERAL.DEVICE:                         ens18
GENERAL.TYPE:                           ethernet
GENERAL.HWADDR:                         56:59:8A:15:59:56
GENERAL.MTU:                            1500
GENERAL.STATE:                          100 (connected)
GENERAL.CONNECTION:                     ens18
GENERAL.CON-PATH:                       /org/freedesktop/NetworkManager/ActiveConnection/1
WIRED-PROPERTIES.CARRIER:               on
IP4.ADDRESS[1]:                         192.168.0.173/24
IP4.GATEWAY:                            192.168.0.1
IP4.ROUTE[1]:                           dst = 0.0.0.0/0, nh = 192.168.0.1, mt = 100
IP4.ROUTE[2]:                           dst = 192.168.0.0/24, nh = 0.0.0.0, mt = 100
IP4.DNS[1]:                             192.168.0.13
IP4.DNS[2]:                             192.168.0.8
IP4.DOMAIN[1]:                          joychina.net
IP6.ADDRESS[1]:                         fe80::172b:1344:148a:3c4a/64
IP6.GATEWAY:                            --
IP6.ROUTE[1]:                           dst = fe80::/64, nh = ::, mt = 100
IP6.ROUTE[2]:                           dst = ff00::/8, nh = ::, mt = 256, table=255
```

### 3. 禁用启用网卡

```
nmcli con down ens18
nmcli con up ens18
```

![image.png](/assets/images/2021/04-26/avnk4gc98f.png)

![image.png](/assets/images/2021/04-26/o9w6scaqet.png)

太复杂的我就不讲了..个人也不太关心......

# 二 . 由nmcli 延伸到centos8与centos7的一些变化

centos8与7的差异具体的可参照：[https://access.redhat.com/documentation/en-us/red\_hat\_enterprise\_linux/8/html-single/considerations\_in\_adopting\_rhel\_8/index](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html-single/considerations_in_adopting_rhel_8/index)还是的看官方文档。也可以看下：[https://blog.csdn.net/seaship/article/details/109292307](https://blog.csdn.net/seaship/article/details/109292307)

对于个人来说。最大的差距是dnf的命令....。当然了yum还是可以用的。个人还是用的yum，其他的差异就在工作中慢慢去看吧......




