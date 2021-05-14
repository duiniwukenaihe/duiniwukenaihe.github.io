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


# 前因后果：
## 1. 升级后的报警
[Kubernetes 1.20.5 upgrade 1.21.0](https://blog.csdn.net/saynaihe/article/details/116761740?spm=1001.2014.3001.5501)，升级完成突然发现Prometheus discover中两个服务down了,收到微信报警
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620908351728-5f71813a-9d95-4b85-b2b1-3a5d97a71f62.png#clientId=u945295fd-377b-4&from=paste&height=783&id=u247598a4&margin=%5Bobject%20Object%5D&name=image.png&originHeight=783&originWidth=569&originalType=binary&size=65266&status=done&style=none&taskId=u955bc8ff-8e42-4438-a74e-8fd0b9517e3&width=569)
登陆Prometheus控制台一看controller-manager  kube-scheduler服务确实是down：
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620907566880-d618c6e8-6d7e-42d2-8e3e-805dd38ec696.png#clientId=u945295fd-377b-4&from=paste&height=751&id=u960575a3&margin=%5Bobject%20Object%5D&name=image.png&originHeight=751&originWidth=1865&originalType=binary&size=152532&status=done&style=none&taskId=u5fa7ce93-36e1-4fab-9926-2cc0c94b611&width=1865)
## 2. 查看服务状态确认相关服务是正常状态
登陆集群查看kubectl get pods -n kube-system服务都是正常的。当然了也可以kubectl logs -f $podname -n kube-system去查看一下相关pod的log日志进行确认一下。
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620955917551-3ed0905c-076a-4c06-be1d-0701ee096687.png#clientId=u3664346e-0d8c-4&from=paste&height=713&id=u9805b6b9&margin=%5Bobject%20Object%5D&name=image.png&originHeight=713&originWidth=707&originalType=binary&size=84605&status=done&style=none&taskId=u45cf5d32-77c8-4a68-858f-029255e7f64&width=707)
## 3. 定位原因
仔细一想是不是升级的时候controller-manager  kube-scheduler服务的配置文件给升级了呢...记得在搭建Prometheus-oprator的时候手动修改过两个服务的配置文件what仔细一想也对...upgrade的时候 它难道把两个配置文件改了？
## 4. 修改kube-controller-manage kube-scheduler配置文件
继续参照：[Kubernetes 1.20.5 安装Prometheus-Oprator](https://cloud.tencent.com/developer/article/1807805)中1.5 查看controller-manager  kube-scheduler服务的配置：
注：配置文件路径为：/etc/kubernetes/manifests
cat  cat kube-controller-manager.yaml 发现--bind-address=127.0.0.1了 恢复了初始的设置，在安装Prometheus-oprator的时候将其修改为0.0.0.0的同理修改。
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620907774191-eb614a03-0a52-4351-b0d2-43f88b1c6794.png#clientId=u945295fd-377b-4&from=paste&height=534&id=uc4835724&margin=%5Bobject%20Object%5D&name=image.png&originHeight=534&originWidth=814&originalType=binary&size=56795&status=done&style=none&taskId=ub609ab24-fe4a-4895-babe-3328ed0abf3&width=814)
修改scheduler配置文件--bind-address=0.0.0.0
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620907811310-4a957660-66a2-48a6-9ebb-7603f04f3dea.png#clientId=u945295fd-377b-4&from=paste&height=547&id=u5912444a&margin=%5Bobject%20Object%5D&name=image.png&originHeight=547&originWidth=724&originalType=binary&size=39368&status=done&style=none&taskId=u06165d70-ca93-458e-961d-bc08ea135cd&width=724)
注： 修改配置文件是针对所有master节点配置文件的。
## 5. 重启kubelet服务
重启 三个master节点kubelet服务：
注：我记得应该修改了相关的配置服务 控制平面的相关pods是会重启的吧？重启kubelet只是个人确保一下啊正常......
```
 systemctl restart kubelet
```
等待服务跑起来running......
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620907879848-11e76473-c17b-47e2-aa24-aa35ea0909bb.png#clientId=u945295fd-377b-4&from=paste&height=702&id=u30bb9ffd&margin=%5Bobject%20Object%5D&name=image.png&originHeight=702&originWidth=767&originalType=binary&size=84212&status=done&style=none&taskId=uab145a9d-66a1-4e83-a408-8da4a0c68ce&width=767)
## 6. 确认Prometheus控制台status状态up
登陆Prometheus web控制台确认监控恢复正常状态：


![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620907929155-33ac1bf7-73b3-4991-99de-2e0cac6da7ab.png#clientId=u945295fd-377b-4&from=paste&height=781&id=u1715e522&margin=%5Bobject%20Object%5D&name=image.png&originHeight=781&originWidth=1748&originalType=binary&size=131959&status=done&style=none&taskId=uc4371dee-c280-4c13-a02b-61a12e77496&width=1748)
# 问题复盘：

1. 必须认同升级是必然的过程。保证集群的健康安装需要保证集群的不断updata,当然并不一定是最新的。
1. 相关服务的安装修改需要熟练牢记。就像Prometheus-oprator中controller-manager kube-scheduler服务一样，起码能明确记得修改过两个相关配置文件。确认服务正常后，可以去很明确的查看定位配置文件的修改。
1. 监控还是很有必要的，不管你是轻视还是重视。很大程度上面可以避免更多的失误，作为一个衡量指标
