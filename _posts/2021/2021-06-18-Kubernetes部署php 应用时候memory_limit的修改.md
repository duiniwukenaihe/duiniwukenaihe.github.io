---
layout: post
title: 2021-06-18-Kubernetes部署php 应用时候memory_limit的修改
date: 2021-06-18 2:00:00
category: kubernetes
tags: kubernetes php
author: duiniwukenaihe
---
* content
{:toc}

# 背景：

基础环境：[centos8+kubeadm1.20.5+cilium+hubble环境搭建](https://www.yuque.com/duiniwukenaihe/ehb02i/dkwh3p)，traefik提供对外服务：[Kubernetes 1.20.5 安装traefik在腾讯云下的实践](https://www.yuque.com/duiniwukenaihe/ehb02i/odflm7)。跑了几个基础的php服务。基础镜像是参考的[https://github.com/richarvey/nginx-php-fpm](https://github.com/richarvey/nginx-php-fpm)搭建。然后php报错：Allowed memory size of 134217728 bytes exhausted (tried to allocate 6291488 bytes)临时需要调整个参数。不想重新打镜像啊。咋整？

# 1. 问题复查与解决：

## 1. 找出引发报错的配置项

首先分析一下报错：Allowed memory size of 134217728 bytes exhausted (tried to allocate 6291488 bytes)

仔细看了一眼上面的数字嗯。限制应该是128M。php运行的脚本需要使用134M的资源超了？

先进入容器瞄一眼，看看这可能是哪个参数：

```
php -i 
```

![image.png](/assets/images/2021/06-18/gbwab2bmdr.png)

初步来看是memory\_limit 这个参数限制了128M

## 2.深入了解配置项参数设置与含义

仔细解读了一下memory\_limit这个参数：

一个PHP工作进程即php-fpm所能够使用的最大内存?

![image.png](/assets/images/2021/06-18/qv5stv3rb7.png)

然后：

![image.png](/assets/images/2021/06-18/bd730d5rvc.png)

两个感觉说的都不是一会事情啊？

![image.png](/assets/images/2021/06-18/dqz1pazl2u.png)

参照：[https://docs.rackspace.com/support/how-to/php-memory-limit/](https://docs.rackspace.com/support/how-to/php-memory-limit/)

先不去纠结它了。反正就先这样理解了：

![image.png](/assets/images/2021/06-18/piudppet93.png)

防止写得不好的脚本吃掉服务器上所有可用的内存

## 3. 如何修改参数并验证其是否生效

开始memory\_limit这个参数设置的是128M既然不够了，那就先扩一下？

查看了下dockerfile这个参数是在start.sh启动脚本中将参数设置为128M的：

![image.png](/assets/images/2021/06-18/l6nfmzgsl2.png)

那我现在要么把start.sh脚本进行修改？or 我可不可以设置一下环境变量？

![image.png](/assets/images/2021/06-18/modcp352q7.png)

尝试了一下修改yaml文件并重新部署服务，验证如下：

![image.png](/assets/images/2021/06-18/yyumsi304d.png)

ok生效了。环境变量的优先级是大于启动脚本中的变量的？ 我是否可以这样理解？

# 复盘：

1. memory\_limit这个参数如何设置合适的范围？我觉得我设置为256M这个参数略大。
2. 这个参数设置大后我的并发线程怎么控制....。我的这些资源会不会不够？引起各种的崩溃？先把 我容器的内存先扩大一下呢。

![](/assets/images/2021/06-18/xbnuroyxo0.png)

1. 其实还是觉得是php脚本写的太烂吃掉了内存......就相对比较简单的服务。能吃掉那么多内存也是神了。只能让他们找下问题，优化一下脚本。
2. 主要是觉得活学活用还的。能用变量的尽量去用变量去做各种配置。避免重复构建基础镜像。