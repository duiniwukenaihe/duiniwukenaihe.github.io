---
layout: post
title: "2019-10-28-k8s-helm-install"
date: "2019-10-28 10:00:00"
category: kubernetes
tags:  kubernetes  helm
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

# 安装Helm2

  ```bash
helm 现在大版本有2和3两个版本，个人安装了helm2版本的1.15.1
version=v2.15.1
wget https://get.helm.sh/helm-${version}-linux-amd64.tar.gz
tar -zxvf helm-*-linux-amd64.tar.gz
cp linux-amd64/helm /usr/local/bin/helm
#安装tiller
helm init --tiller-image=gcr.azk8s.cn/kubernetes-helm/tiller:${version}
#查看helm版本
helm version --short
Client: v2.15.1+gcf1de4f
Server: v2.15.1+gcf1de4f
但是kubernetes 1.16.2安装貌似会有权限问题的，执行以下命令：

kubectl -n kube-system create serviceaccount tiller
kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
helm init --tiller-image=gcr.azk8s.cn/kubernetes-helm/tiller:${version} --service-account tiller --upgrade
[root@k8s-master-03 ~]# kubectl -n kube-system get all | grep tiller
pod/tiller-deploy-76858c97cd-sgmf6          1/1     Running   1          2d6h
service/tiller-deploy             ClusterIP   10.31.70.242    <none>        44134/TCP                          2d6h
deployment.apps/tiller-deploy   1/1     1            1           2d6h
replicaset.apps/tiller-deploy-76858c97cd   1         1         1       2d6h

```




