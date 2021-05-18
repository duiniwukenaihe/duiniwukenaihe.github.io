---
layout: post
title: 2021-05-14-Kubernetes 1.20.5 upgrade1.21.0后遗症
date: 2021-05-14 2:00:00
category: kubernetes
tags: kubernetes  Prometheus controller-manager  kube-scheduler
author: duiniwukenaihe
---
* content
{:toc}


查看文章
* 编辑文章发布保存草稿草稿箱 (2)
切换到富文本编辑器
Kubernetes 1.20.5 upgrade1.21.0后遗症
BISC|h1h2h3|QCOL|LΣ未选择任何文件T|E
# 前因后果： ## 1. 升级后的报警 [Kubernetes 1.20.5 upgrade 1.21.0](https://blog.csdn.net/saynaihe/article/details/116761740?spm=1001.2014.3001.5501)，升级完成突然发现Prometheus discover中两个服务down了,收到微信报警 ![image.png](/assets/images/2021/05-17/49l9jxvrnq.png) 登陆Prometheus控制台一看controller-manager  kube-scheduler服务确实是down： ![image.png](/assets/images/2021/05-17/67lub4v5im.png) ## 2. 查看服务状态确认相关服务是正常状态 登陆集群查看kubectl get pods -n kube-system服务都是正常的。当然了也可以kubectl logs -f $podname -n kube-system去查看一下相关pod的log日志进行确认一下。 ![image.png](/assets/images/2021/05-17/et7wt3068z.png) ## 3. 定位原因 仔细一想是不是升级的时候controller-manager  kube-scheduler服务的配置文件给升级了呢...记得在搭建Prometheus-oprator的时候手动修改过两个服务的配置文件what仔细一想也对...upgrade的时候 它难道把两个配置文件改了？ ## 4. 修改kube-controller-manage kube-scheduler配置文件 继续参照：[Kubernetes 1.20.5 安装Prometheus-Oprator](https://cloud.tencent.com/developer/article/1807805)中1.5 查看controller-manager  kube-scheduler服务的配置： 
查找
无结果
# 前因后果：

## 1. 升级后的报警

[Kubernetes 1.20.5 upgrade 1.21.0](https://blog.csdn.net/saynaihe/article/details/116761740?spm=1001.2014.3001.5501)，升级完成突然发现Prometheus discover中两个服务down了,收到微信报警

![image.png](/assets/images/2021/05-17/49l9jxvrnq.png)

