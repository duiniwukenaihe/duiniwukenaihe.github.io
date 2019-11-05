---
layout: post
title: "kubernetes集群扩容"
date: "2019-09-18 10:00:00"
category: kubernetes
tags: kubernetes扩容
author: duiniwukenaihe
---
* content
{:toc}

 

## 描述背景：
已经搭建完的k8s集群只有两个worker节点，现在增加work节点到集群。原有集群机器：

|  ip           | 自定义域名         |    主机名 |
|  :----:       |     :----:        |   :----:  |
|                |  master.k8s.io    |  k8s-vip  |
|192.168.0.195    |  master01.k8s.io  |  k8s-master-01|
|192.168.0.197   |  master02.k8s.io  |  k8s-master-02| 
|192.168.0.198   |  master03.k8s.io  |  k8s-master-03|
|192.168.0.199    |  node01.k8s.io    |  k8s-node-01|
|192.168.0.202    |  node02.k8s.io    |  k8s-node-02|


  ```bash 
本地搭建的用了一种不要脸的方式，基本搭建方式如https://duiniwukenaihe.github.io/2019/09/02/k8s-install/， master01  master02  节点host  master.k8s.io绑定了192.168.0.195.master03节点master.k8s.io绑定了192.168.0.197跑了下貌似也是没有问题的。现在将一下三台机器加入集群：

|192.168.0.108    |  node03.k8s.io    |  k8s-node-03|
|192.168.0.111    |  node04.k8s.io    |  k8s-node-04|
|192.168.0.115    |  node05.k8s.io    |  k8s-node-05|


三台绑定host
192.168.0.108
cat /etc/host
192.168.0.197  master.k8s.io      k8s-vip
192.168.0.195  master01.k8s.io  k8s-master-01
192.168.0.197  master02.k8s.io  k8s-master-02
192.168.0.198  master03.k8s.io  k8s-master-03

192.168.111
cat /etc/host
192.168.0.198  master.k8s.io      k8s-vip
192.168.0.195  master01.k8s.io  k8s-master-01
192.168.0.197  master02.k8s.io  k8s-master-02
192.168.0.198  master03.k8s.io  k8s-master-03

192.168.115
cat /etc/host
192.168.0.195  master.k8s.io      k8s-vip
192.168.0.195  master01.k8s.io  k8s-master-01
192.168.0.197  master02.k8s.io  k8s-master-02
192.168.0.198  master03.k8s.io  k8s-master-03


参照https://duiniwukenaihe.github.io/2019/09/02/k8s-install/完成系统参数调优，配置yum源  安装docker kubernetes 
  ```

任一master节点执行kubeadm token list，查看是否有有效期内token和SH256加密字符串

![token.list.png](/assets/images/k8s/token.list.png) 
token默认的有效期是24小时，如果没有有效token，则将创建token
  ```bash 
kubeadm token create

openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
  ```
![token.list1.png](/assets/images/k8s/token.list1.png) 

k8s-node-03 k8s-node-04 k8s-node-05节点加入集群

![token.list2.png](/assets/images/k8s/token.list2.png) 

没有关闭swap  加入集群命令加入--ignore-preflight-errors=Swap：

kubeadm join master.k8s.io:8443 --token u7lv9t.bubw4r69tkmorug2     --discovery-token-ca-cert-hash sha256:74e007d58a810fd4e8a233158d0b8e195641f724ad402e4afddc77e0eb003d95   --ignore-preflight-errors=Swap
![token.list3.png](/assets/images/k8s/token.list3.png)
master节点执行kubectl get nodes
![token.list4.png](/assets/images/k8s/token.list4.png)





