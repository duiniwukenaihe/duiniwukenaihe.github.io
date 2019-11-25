---
layout: post
title: "kuberntes1.16  question"
date: "2019-10-21 10:00:00"
category: kubernetes
tags:  kubernetes  qustion
author: duiniwukenaihe
---
* content
{:toc}

 

# 描述背景：
注：记录各种常见问题

集群配置：
初始集群环境kubeadm 1.16.1

|  ip           | 自定义域名         |    主机名 |
|  :----:       |     :----:        |   :----:  |
|192.168.3.8      |  master.k8s.io    |  k8s-vip  |
|192.168.3.10    |  master01.k8s.io  |  k8s-master-01|
|192.168.3.5   |  master02.k8s.io  |  k8s-master-02| 
|192.168.3.12   |  master03.k8s.io  |  k8s-master-03|
|192.168.3.6    |  node01.k8s.io    |  k8s-node-01|
|192.168.3.2    |  node02.k8s.io    |  k8s-node-02|
|192.168.3.4    |  node03.k8s.io    |  k8s-node-03|

# 1.网桥配置问题：sd_journal_get_cursor() failed: 'Cannot assign requested address'  [v8.24.0-34.el7]
![bridge.png](/assets/images/qustion/bridge.png)
  ```bash
kubernetes 集群中有内部同一namespace下通信用的直接连service的方式有一service下pod访问另外一service无法访问。但是访问其他同一命名空间下都是没有问题的，kubectl get pod -o wide  看到此pod位于node-07.node-07节点执行journalctl  看日志中有sd_journal_get_cursor() failed: 'Cannot assign requested address'  [v8.24.0-34.el7]，百度了下https://blog.csdn.net/wkb342814892/article/details/79543984应该是没有开启网桥转发：
modprobe br_netfilter
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl -p /etc/sysctl.d/k8s.conf
ls /proc/sys/net/bridge
```
OK问题解决