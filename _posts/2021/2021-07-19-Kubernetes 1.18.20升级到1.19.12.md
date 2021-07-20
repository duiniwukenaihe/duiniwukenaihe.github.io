---
layout: post
title: 2021-07-19-Kubernetes 1.18.20升级到1.19.12
date: 2021-07-19 2:00:00
category: kubernetes
tags: kubernetes kubeadm
author: duiniwukenaihe
---
* content
{:toc}
# 背景：

升级是一件持续的事情：[Kubernetes 1.16.15升级到1.17.17](https://www.yuque.com/duiniwukenaihe/ehb02i/kdvrku),[Kubernetes 1.17.17升级到1.18.20](https://www.yuque.com/duiniwukenaihe/ehb02i/ln04dq)

## 集群配置:

| 主机名 | 系统 | ip |
|:----|:----|:----|
| k8s-vip | slb | 10.0.0.37 |
| k8s-master-01 | centos7 | 10.0.0.41 |
| k8s-master-02 | centos7 | 10.0.0.34 |
| k8s-master-03 | centos7 | 10.0.0.26 |
| k8s-node-01 | centos7 | 10.0.0.36 |
| k8s-node-02 | centos7 | 10.0.0.83 |
| k8s-node-03 | centos7 | 10.0.0.40 |
| k8s-node-04 | centos7 | 10.0.0.49 |
| k8s-node-05 | centos7 | 10.0.0.45 |
| k8s-node-06 | centos7 | 10.0.0.18 |


## 1. 参考官方文档

参照：[https://kubernetes.io/zh/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/](https://kubernetes.io/zh/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)

## 2. 确认可升级版本与升级方案

```
yum list --showduplicates kubeadm --disableexcludes=kubernetes
```

通过以上命令查询到1.19当前最新版本是1.19.12-0版本。master有三个节点还是按照个人习惯先升级k8s-master-03节点

![image.png](https://ask.qcloudimg.com/http-save/1006587/21b5fc168b6892cf1ed6a2dc2e15bf50.png)

## 3. 升级k8s-master-03节点控制平面

依然k8s-master-03执行：

### 1. yum升级kubernetes插件

```
yum install kubeadm-1.19.12-0 kubelet-1.19.12-0 kubectl-1.19.12-0 --disableexcludes=kubernetes
```

![image.png](https://ask.qcloudimg.com/http-save/1006587/eee9f9e4dd022c4c3615b23224e423ad.png)

### 2. 腾空节点检查集群是否可以升级

依然算是温习drain命令：

```
kubectl drain k8s-master-03 --ignore-daemonsets
sudo kubeadm upgrade plan
```

![image.png](https://ask.qcloudimg.com/http-save/1006587/239cd20e88e6fb897bc07d587d00c4bf.png)

### 3. 升级版本到1.19.12

```
kubeadm upgrade apply 1.19.12
```

注意：特意强调一下work节点的版本也都是1.18.20了，没有出现夸更多版本的状况了

![image.png](https://ask.qcloudimg.com/http-save/1006587/473d4431b9290d52f0961fb677aad0f4.png)

![image.png](https://ask.qcloudimg.com/http-save/1006587/bab644ca3a7e140c4260a8ffd38414e1.png)

```
[root@k8s-master-03 ~]# sudo systemctl daemon-reload
[root@k8s-master-03 ~]# sudo systemctl restart kubelet
[root@k8s-master-03 ~]# kubectl uncordon k8s-master-03
node/k8s-master-03 uncordoned
```

![image.png](https://ask.qcloudimg.com/http-save/1006587/2d80e998c91c70041137917108385e40.png)

## 4. 升级其他控制平面（k8s-master-01 k8s-master-02）

```
sudo yum install kubeadm-1.19.12-0 kubelet-1.19.12-0 kubectl-1.19.12-0 --disableexcludes=kubernetes
sudo kubeadm upgrade node
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

![image.png](https://ask.qcloudimg.com/http-save/1006587/0d4aba05e16672884ecda621234acea3.png)

![image.png](https://ask.qcloudimg.com/http-save/1006587/6f70fe0222c73f0b66d676997ecd8146.png)

## 5. work节点的升级

```
sudo yum install kubeadm-1.19.12-0 kubelet-1.19.12-0 kubectl-1.19.12-0 --disableexcludes=kubernetes
sudo kubeadm upgrade node
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

![image.png](https://ask.qcloudimg.com/http-save/1006587/b1d96282756bb2ea5a478198e6fc99ab.png)

## 6. 验证升级

```
 kubectl get nodes
```

![image.png](https://ask.qcloudimg.com/http-save/1006587/a4c8eff6d737db3b874d69d1565faec2.png)

## 7. 其他

查看一眼kube-system下插件的日志，确认插件是否正常

```
kubectl logs -f kube-controller-manager-k8s-master-01 -n kube-system
```

![image.png](https://ask.qcloudimg.com/http-save/1006587/631bf3ddbfdca94f94b1334133350048.png)

目测是没有问题的就不管了....嗯Prometheus的问题还是留着。本来也准备安装主线版本了。过去的准备卸载了.如出现cluseterrole问题可参照：[Kubernetes 1.16.15升级到1.17.17](https://www.yuque.com/duiniwukenaihe/ehb02i/kdvrku) 

![image.png](https://ask.qcloudimg.com/http-save/1006587/243124f46e381343845a30da472d6bda.png)

![](https://ask.qcloudimg.com/http-save/1006587/9fbe45fd87a561d557c09b2fbfd3f595.png)