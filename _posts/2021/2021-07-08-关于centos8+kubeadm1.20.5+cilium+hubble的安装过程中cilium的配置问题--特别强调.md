---
layout: post
title: 2021-07-08-关于centos8+kubeadm1.20.5+cilium+hubble的安装过程中cilium的配置问题--特别强调
date: 2021-07-08 2:00:00
category: kubernetes
tags: kubernetes kubeadm cilium
author: duiniwukenaihe
---
* content
{:toc}
# 背景：

参见前文：[centos8+kubeadm1.20.5+cilium+hubble环境搭建](https://www.yuque.com/duiniwukenaihe/ehb02i/dkwh3p)，并升级到了1.21版本（[Kubernetes 1.20.5 upgrade 1.21.0](https://www.yuque.com/duiniwukenaihe/ehb02i/palbae)）。今天无聊查看一下集群呢突然发现一个问题：

```
[root@sh-master-01 ~]# kubectl get pods -n default -o wide
NAME                          READY   STATUS             RESTARTS   AGE    IP           NODE         
csi-app                       1/1     Running            11         106d   10.0.4.204   sh-work-01  
nginx                         1/1     Running            0          13d    10.0.4.60    sh-work-01  
nginx-1-kkfvd                 1/1     Running            0          13d    10.0.5.223   sh-work-02  
nginx-1-klgpx                 1/1     Running            0          13d    10.0.4.163   sh-work-01  
nginx-1-s5mzp                 1/1     Running            0          13d    10.0.3.208   sh-work-03   
nginx-2-8cb2j                 1/1     Running            0          13d    10.0.3.218   sh-work-03  
nginx-2-l527j                 1/1     Running            0          13d    10.0.5.245   sh-work-02   
nginx-2-qnsrq                 1/1     Running            0          13d    10.0.4.77    sh-work-01  
php-apache-5b95f8f674-clzn5   1/1     Running            2          99d    10.0.3.64    sh-work-03
pod-flag                      1/1     Running            316        13d    10.0.5.252   sh-work-02 
pod-nodeaffinity              1/1     Running            0          13d    10.0.4.118   sh-work-01
pod-prefer                    1/1     Running            0          13d    10.0.5.181   sh-work-02
pod-prefer1                   1/1     Running            0          13d    10.0.3.54    sh-work-03
with-node-affinity            0/1     ImagePullBackOff   0          13d    10.0.4.126   sh-work-01
with-pod-affinity             0/1     ImagePullBackOff   0          13d    10.0.5.30    sh-work-02
with-pod-antiaffinity         1/1     Running            0          13d    10.0.4.159   sh-work-01
```

主要是看ip一栏，这.，我记得我的 serviceSubnet: 172.254.0.0/16 , podSubnet: 172.3.0.0/16啊 是不是哪里搞错了了呢？

![image.png](/assets/images/2021/07-08/tjseoz2575.png)

再看一眼service的网络状况：

```
[root@sh-master-01 ~]# kubectl get svc -n default -o wide
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE    SELECTOR
kubernetes   ClusterIP   172.254.0.1      <none>        443/TCP   106d   <none>
php-apache   ClusterIP   172.254.13.205   <none>        80/TCP    99d    run=php-apache
```

这个ip段是对的呢！怎么会事情呢？

# 复盘解决问题

## 1. 确认问题

cilium是oprator的方式安装的，找到oprator的pod:

```
kubectl get pods -n kube-system -o wide
```

![image.png](/assets/images/2021/07-08/gbm60vy15x.png)

查看日志输出：

```
kubectl logs -f cilium-operator-f595b8f7d-7rvzz  -n kube-system
```

貌似找到了 下面箭头指向的这一段ipv4CIDRs是 10.0.0.0/8!

![image.png](/assets/images/2021/07-08/twrbjv2mco.png)

## 2. 阅读官方文档

搭建集群我是参考的肥宅爽文的博客：[02-02.kubeadm + Cilium 搭建kubernetes集群](https://www.yuque.com/xiaowei-trt7k/tw/ah0ls0#SMT45)。博客上面写的比较完整的cilium搭建集群的貌似我俩写的了：[centos8+kubeadm1.20.5+cilium+hubble环境搭建](https://segmentfault.com/a/1190000040299919)。初步怀疑就是helm安装cilium的时候配置网络组件并没有走config.yaml中的配置。仔细看了一眼官方github仓库文件配置项：[https://github.com/cilium/cilium/tree/v1.10.2/install/kubernetes/cilium](https://github.com/cilium/cilium/tree/v1.10.2/install/kubernetes/cilium)。

![image.png](/assets/images/2021/07-08/n8noq98oh6.png)

注：我的版本是1.9.7，正常的找自己对应版本的配置文件看呢。差距不大我就没有去切换分支看了...

对照官方文档：在helm安装的时候没有指定ipam.operator.clusterPoolIPv4PodCIDR参数！网上的很多文章也都是草草来的，并没有详细的对ipv4的cidr进行指定那pod网络都是默认的10.0.0.0/8的cidr!我的[cvm](https://console.cloud.tencent.com/cvm/instance/index?rid=4) [vpc](https://console.cloud.tencent.com/vpc/vpc?rid=4)网络默认是10.0.0.0/8的大网络，这样下来ip地址应该是会有冲突的！

## 3.找一个正确的参考

谷歌搜索文档搜到亚马逊的一篇blog:[https://aws.amazon.com/cn/blogs/containers/a-multi-cluster-shared-services-architecture-with-amazon-eks-using-cilium-clustermesh/](https://aws.amazon.com/cn/blogs/containers/a-multi-cluster-shared-services-architecture-with-amazon-eks-using-cilium-clustermesh/)

亚马逊的blog还是很好的。安装的过程很是详细可以参考一下：

![image.png](/assets/images/2021/07-08/unq2iqayvc.png)

至于我的集群只能upgrade了...

```
 helm upgrade cilium cilium/cilium --version 1.9.7   --namespace=kube-system --set ipam.operator.clusterPoolIPv4PodCIDR="172.3.0.0/16"
```

目测应该是这样的特别强调一下，抄别人安装的时候还是要去看一下官方文档的详细参数定义！国内能搜到的这些cilium的 都大部分没有写podCIDR的配置的吧？参考别人博客的同时一定记得再仔细去看下官方文档参数！