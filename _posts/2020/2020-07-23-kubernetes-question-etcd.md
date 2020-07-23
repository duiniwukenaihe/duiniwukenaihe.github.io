---
layout: "post"
title: "2020-07-23-qestion-etcd(error execution phase check-etcd)"
date: "2020-07-23 10:00:00"
category: "kubernetes"
tags:  "kubernetes1.16.8 etcd  check-etcd etcd 监控检查失败"
author: duiniwukenaihe
---
* content
{:toc}



## 背景


> 昨天搭建1.18.6kubeadm ha集群的时候 xshell 没有仔细看 手贱，把老集群的master02节点给kubeadm reset了。然后master01几点重新生成token,将master02节点介入集群出现etcd检查失败的错误日志，然后发现了超级小豆丁的日志也整过类型的问题：http://www.mydlq.club/article/73/详情见大佬博客
 
基本过程：（直接抄写豆丁大佬的了，基本就那么操作的）
### 1.  kubectl describe configmaps kubeadm-config -n kube-system。发现k8s-master-02节点依然存在。
### 2. 万恶的etcd，当剔除一个 master 节点时 etcd 集群未删除剔除的节点的 etcd 成员信息，该信息还存在 etcd 集群列表中。手工删除etcd成员信息。
 ```


kubectl get pods -n kube-system | grep etcd
etcd-k8s-master-01                      1/1     Running   2          212d
etcd-k8s-master-03                      1/1     Running   3          212d

kubectl exec -it etcd-k8s-master-01 sh -n kube-system
## 配置环境
export ETCDCTL_API=3
alias etcdctl='etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=etc/kubernetes/pki/etcd/server.key'

## 查看 etcd 集群成员列表
etcdctl member list

b3e5838df5f510, started, k8s-master-01, https://10.0.0.41:2380, https://10.0.0.41:2379
aab0efa6cc544b57, started, k8s-master-02, https://10.0.0.34:2380, https://10.0.0.34:2379
dd2b426e7bda1609, started, k8s-master-03, https://10.0.0.26:2380, https://10.0.0.26:2379
 ## 删除 etcd 集群成员k8s-master-02
etcdctl member remove  aab0efa6cc544b57

## 再次查看 etcd 集群成员列表
 etcdctl member list

 b3e5838df5f510, started, k8s-master-01, https://10.0.0.41:2380, https://10.0.0.41:2379
dd2b426e7bda1609, started, k8s-master-03, https://10.0.0.26:2380, https://10.0.0.26:2379

 exit
  ```
### 3 . k8s-master-02节点重新加入集群。
  ```
kubeadm reset  

kubeadm join 10.0.0.37:6443 --token xzd67o.xgnzqkwkjem7kcmf     --discovery-token-ca-cert-hash sha256:56ccafb865957c0692f5737cd8778553910c1049ef238a7781b7a39f5fd3a99a     --control-plane 
 mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl get nodes
NAME            STATUS   ROLES    AGE    VERSION
k8s-master-01   Ready    master   212d   v1.16.8
k8s-master-02   Ready    master   3d2h   v1.16.8
k8s-master-03   Ready    master   212d   v1.16.8
k8s-node-01     Ready    <none>   212d   v1.16.8
k8s-node-02     Ready    <none>   212d   v1.16.8
k8s-node-03     Ready    <none>   212d   v1.16.8
k8s-node-04     Ready    <none>   210d   v1.16.8
k8s-node-05     Ready    <none>   210d   v1.16.8
k8s-node-06     Ready    <none>   210d   v1.16.8
  ```

