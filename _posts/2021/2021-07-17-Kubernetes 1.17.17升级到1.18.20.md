
---
layout: post
title: 2021-07-17-Kubernetes 1.17.17升级到1.18.20
date: 2021-07-17 2:00:00
category: kubernetes
tags: kubernetes kubeadm
author: duiniwukenaihe
---
* content
{:toc}# 背景：

参照：[https://www.yuque.com/duiniwukenaihe/ehb02i/kdvrku](https://www.yuque.com/duiniwukenaihe/ehb02i/kdvrku) 完成了1.16.15到1.17.17 的升级，现在升级到1.18版本

## 集群配置

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

![image.png](/assets/images/2021/07-17/36edaad43b4f75854a0b54240a8a7207.png)

[https://v1-17.docs.kubernetes.io/zh/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/](https://v1-17.docs.kubernetes.io/zh/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)

kubeadm 创建的 Kubernetes 集群从 1.16.x 版本升级到 1.17.x 版本，以及从版本 1.17.x 升级到 1.17.y ，其中 y > x。现在继续将版本升级到1.18。从1.16升级到1.17有点理解错误：1.16.15我以为只能先升级到1.17.15。仔细看了下文档是没有这说法的。我就准备从1.17.17升级到1.18的最新版本了！

![image.png](/assets/images/2021/07-17/443a964b2f291bf5881b2c0b776d8999.png)

## 2. 确认可升级版本与升级方案

```
yum list --showduplicates kubeadm --disableexcludes=kubernetes
```

通过以上命令查询到1.18当前最新版本是1.18.20-0版本。master有三个节点还是按照个人习惯先升级k8s-master-03节点

![image.png](/assets/images/2021/07-17/7638dcc0087ef1c771680a2476475e0d.png)

## 3. 升级k8s-master-03节点控制平面

依然k8s-master-03执行：

### 1. yum升级kubernetes插件

```
yum install kubeadm-1.18.20-0 kubelet-1.18.20-0 kubectl-1.18.20-0 --disableexcludes=kubernetes
```

![image.png](/assets/images/2021/07-17/2bf8ea0079a28957e118d0baae4c125f.png)

### 2. 腾空节点检查集群是否可以升级

特意整一下腾空（1.16.15升级到1.17.17的时候没有整。就当温习一下drain命令了）

```
kubectl drain k8s-master-03 --ignore-daemonsets
sudo kubeadm upgrade plan
```

![image.png](/assets/images/2021/07-17/e3d6f5a621e726df15d81fd54d534118.png)

### 3. 升级版本到1.18.20

#### 1. 小插曲

嗯操作升级到1.18.20版本

```
kubeadm upgrade apply 1.18.20
```

![image.png](/assets/images/2021/07-17/abba12e22a560f2d0acbe33b41254a54.png)

嗯有一个work节点没有升级版本依然是**1.16.15版本**哈哈哈提示一下。master节点应该是向下兼容一个版本的。先把test-ubuntu-01节点升级到1.17.17

```
[root@k8s-master-01 ~]# kubectl get nodes
NAME             STATUS                     ROLES    AGE    VERSION
k8s-master-01    Ready                      master   317d   v1.17.17
k8s-master-02    Ready                      master   317d   v1.17.17
k8s-master-03    Ready,SchedulingDisabled   master   317d   v1.17.17
k8s-node-01      Ready                      node     569d   v1.17.17
k8s-node-02      Ready                      node     22d    v1.17.17
k8s-node-03      Ready                      node     569d   v1.17.17
k8s-node-04      Ready                      node     567d   v1.17.17
k8s-node-05      Ready                      node     567d   v1.17.17
k8s-node-06      Ready                      node     212d   v1.17.17
sh02-node-01     Ready                      node     16d    v1.17.17
test-ubuntu-01   Ready,SchedulingDisabled   <none>   22d    v1.16.15
tm-node-002      Ready,SchedulingDisabled   node     174d   v1.17.17
tm-node-003      Ready                      <none>   119d   v1.17.17
```

![image.png](/assets/images/2021/07-17/87ed36af0f736b2c91b7a357370e87e4.png)

先升级下test-ubuntu-01节点如下(此操作在test-ubuntu-01节点执行):

```
sudo apt-get install -y kubelet=1.17.17-00 kubectl=1.17.17-00 kubeadm=1.17.17-00
sudo kubeadm upgrade node
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

登陆任一master节点确认版本都为1.17.17版本（正常在k8s-master-03节点看就行了，我xshell开了好几个窗口就那01节点看了）：

![image.png](/assets/images/2021/07-17/b639269326f6cc053a80577c12b1972c.png)

#### 2. 升级到1.18.20

k8s-master-03节点继续执行升级：

```
kubeadm upgrade apply v1.18.20
```

![image.png](/assets/images/2021/07-17/56cc28afd49da765ac5cd2be7097678d.png)

#### 3. 重启kubelet 取消节点保护

```
[root@k8s-master-03 ~]# sudo systemctl daemon-reload
[root@k8s-master-03 ~]# sudo systemctl restart kubelet
[root@k8s-master-03 ~]# kubectl uncordon k8s-master-03
node/k8s-master-03 uncordoned
```

![image.png](/assets/images/2021/07-17/c954491c4bed6759be5ff8ebc77a2bd3.png)

## 4. 升级其他控制平面（k8s-master-01 k8s-master-02）

k8s-master-01 k8s-master-02节点都执行以下操作（这里就没有清空节点了，看个人需求）：

```
yum install kubeadm-1.18.20-0 kubelet-1.18.20-0 kubectl-1.18.20-0 --disableexcludes=kubernetes
kubeadm upgrade node
systemctl daemon-reload
sudo systemctl restart kubelet
```

![image.png](/assets/images/2021/07-17/37cd25a3da4cbe6487cdd6252b64e4f9.png)

## 5. work节点的升级

没有执行清空节点（当然了 例行升级的话还是最后执行以下清空节点），直接升级了，如下：

```
yum install kubeadm-1.18.20-0 kubelet-1.18.20-0 kubectl-1.18.20-0 --disableexcludes=kubernetes
kubeadm upgrade node
systemctl daemon-reload
sudo systemctl restart kubelet
```

## 6. 验证升级

```
kubectl get nodes
```

![image.png](/assets/images/2021/07-17/7c9c75d27e56b50b5a679365e0c95615.png)

注意：test-ubuntu-01忽略。

目测是升级成功的，看一下kube-system下的几个系统组件发现：ontroller-manager的clusterrole system:kube-controller-manager的权限又有问题了？

![image.png](/assets/images/2021/07-17/c595b190871b8ae8ea4903d6fe12bccd.png)

同1.16.15升级1.17.17一样：

```
kubectl get  clusterrole system:kube-controller-manager -o yaml > 1.yaml
kubectl delete  clusterrole system:kube-controller-manager
```

```
cat <<EOF >  kube-controller-manager.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: "2021-03-22T11:29:59Z"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-controller-manager
  resourceVersion: "92"
  uid: 7480dabb-ec0d-4169-bdbd-418d178e2751
rules:
- apiGroups:
  - ""
  - events.k8s.io
  resources:
  - events
  verbs:
  - create
  - patch
  - update
- apiGroups:
  - coordination.k8s.io
  resources:
  - leases
  verbs:
  - create
- apiGroups:
  - coordination.k8s.io
  resourceNames:
  - kube-controller-manager
  resources:
  - leases
  verbs:
  - get
  - update
- apiGroups:
  - ""
  resources:
  - endpoints
  verbs:
  - create
- apiGroups:
  - ""
  resourceNames:
  - kube-controller-manager
  resources:
  - endpoints
  verbs:
  - get
  - update
- apiGroups:
  - ""
  resources:
  - secrets
  - serviceaccounts
  verbs:
  - create
- apiGroups:
  - ""
  resources:
  - secrets
  verbs:
  - delete
- apiGroups:
  - ""
  resources:
  - configmaps
  - namespaces
  - secrets
  - serviceaccounts
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - secrets
  - serviceaccounts
  verbs:
  - update
- apiGroups:
  - authentication.k8s.io
  resources:
  - tokenreviews
  verbs:
  - create
- apiGroups:
  - authorization.k8s.io
  resources:
  - subjectaccessreviews
  verbs:
  - create
- apiGroups:
  - '*'
  resources:
  - '*'
  verbs:
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - serviceaccounts/token
  verbs:
  - create
EOF
kubectl apply -f kube-controller-manager.yaml
```

目测是可以了.......................截图过程都找不到了，昨天晚上写的东西没有保存停电了......

这里的flannel没有什么问题 就不用看了 Prometheus依然是有问题的......我就先忽略了。因为我还准备升级集群版本...升级版本后再搞Prometheus了