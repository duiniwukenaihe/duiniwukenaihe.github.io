---
layout: post
title: 2021-05-13-Kubernetes 1.20.5 upgrade 1.21.0
date: 2021-05-13 6:00:00
category: kubernetes
tags: kubeadm kubernetes1.20.5 upgrade kubernetes1.21 containerd
author: duiniwukenaihe
---
* content
{:toc}

# 背景：

集群环境参照：[centos8+kubeadm1.20.5+cilium+hubble环境搭建](https://cloud.tencent.com/developer/article/1806089)。kubernets有了新的版本。1.21.0。嗯准备升级一下啊。

# 1. Kubernetes 1.20.5 upgrade 1.21.0过程

## 1. 查看当前可升级版本

```
yum list --showduplicates kubeadm --disableexcludes=kubernetes
```

![image.png](/assets/images/2021/05-13/kflx20z71q.png)

## 2. 升级kubeadm（所有节点都要操作的）

```
yum install  kubeadm-1.21.0 kubectl-1.21.0 --disableexcludes=kubernetes
```

![image.png](/assets/images/2021/05-13/hyvj2ca7rz.png)

## 3.验证集群是否可以平滑升级

```
kubeadm upgrade plan
```

![image.png](/assets/images/2021/05-13/g7l7ol4sx3.png) 

## 4 . master节点（控制平面） 平滑升级

注：集群环境是一个ha的环境。这部操作选择了再sh-master-01节点执行

```
kubeadm upgrade apply v1.21.0 --certificate-renewal=false
```

--certificate-renewal=false  嗯 我不想重新生成证书....搭建了才没有多久......

![image.png](/assets/images/2021/05-13/wrmni27ds5.png)

嗯 报错了 找不到coredns的镜像：我还有个1.16的kubernetes的集群对比了一下，貌似coredns的目录层级多了一级？

![image.png](/assets/images/2021/05-13/xd03q2v5z6.png)

查看了一眼默认仓库：

![image.png](/assets/images/2021/05-13/ecmh40sy78.png)

然后 ctr images pull registry.aliyuncs.com/google\_containers/coredns/coredns:v1.8.0

![image.png](/assets/images/2021/05-13/eaud9gfm08.png)

仓库下也确实没有这个image。也特意看了下阿里云 包括腾讯云的镜像仓库貌似不支持这样的层级的目录？可能在设计的时候都没有想到后面有这样的层级吗？

一般来说是下载镜像 然后修改镜像tag.....可是我试了几次upgrade仍然是失败：

各种找文章的顺路看到了[https://blog.51cto.com/u\_3252740/2717642](https://blog.51cto.com/u_3252740/2717642)

就顺路玩了一下：

### 1. 修改kubeadm-config configmap

```
kubectl -n kube-system edit cm kubeadm-config
```

coredns的镜像仓库是可以单独定义的：那我就单独定义一下吧,整成k8s.gcr.io佛系定义。就想单独定义一下dns的仓库。

![image.png](/assets/images/2021/05-13/kanyiskrwj.png)

### 2. 下载镜像并修改镜像标签

```
ctr -n k8s.io images pull uhub.service.ucloud.cn/uxhy/v1.8.0
ctr -n k8s.io images pull uhub.service.ucloud.cn/uxhy/coredns:v1.8.0
ctr -n k8s.io images tag uhub.service.ucloud.cn/uxhy/coredns:v1.8.0 k8s.gcr.io/coredns/coredns:v1.8.0
```

然后

master-01 忘了截图借用了下其他节点的截图

![image.png](/assets/images/2021/05-13/27rlk28b0r.png)

### 3. 继续执行 1.3 升级集群第一个主节点（sh-master-01）

```
kubeadm upgrade apply v1.21.0 --certificate-renewal=false
```

### 4 升级其他控制平面节点

基本上就是重复1.2的步骤 然后**kubeadm upgrade node。之前貌似一直是**kubeadm upgrade apply了？

另外一般的安全操作建议是kubectl drain <node-to-drain> --ignore-daemonsets。去腾空节点，更新完成后kubectl uncordon 将节点设置为可调度。这些我就忽略了。集群基本没有跑太多服务。版本还在测试中。故：

sh-master-02 sh-master-03节点执行一下操作

```
ctr -n k8s.io images pull uhub.service.ucloud.cn/uxhy/v1.8.0
ctr -n k8s.io images pull uhub.service.ucloud.cn/uxhy/coredns:v1.8.0
ctr -n k8s.io images tag uhub.service.ucloud.cn/uxhy/coredns:v1.8.0 k8s.gcr.io/coredns/coredns:v1.8.0
yum install  kubeadm-1.21.0 kubectl-1.21.0 --disableexcludes=kubernetes
kubeadm upgrade node
```

### 5. 升级并重启kubelet（3个控制平面节点都执行）

草率了1.2 执行：yum install  kubeadm-1.21.0 kubectl-1.21.0 --disableexcludes=kubernetes 糊里糊涂怎么没有加上kubelet呢？以后还是加在一起去安装了。

```
yum install -y kubelet-1.21.0  --disableexcludes=kubernetes
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

![image.png](/assets/images/2021/05-13/4si8lf7fr0.png)

## 5 升级work节点

### 1. 升级kubeadm kubelet kubectl

```
yum install -y kubeadm-1.21.0 kubelet-1.21.0 kubectl-1.21.0 --disableexcludes=kubernetes
```

![image.png](/assets/images/2021/05-13/cbtd0ujq2g.png)

### 2. 升级kubelet配置

```
 kubeadm upgrade node
```

![image.png](/assets/images/2021/05-13/e7cfvnjt2a.png)

![image.png](/assets/images/2021/05-13/2s0wulfd43.png)

### 3. 看自己需要是否需要腾空节点

```
kubectl drain <node-to-drain> --ignore-daemonsets
kubectl uncordon
```

### 4 . 重启work节点

```
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

## 6. 验证升级结果

```
kubectl get node
```

![image.png](/assets/images/2021/05-13/smadipr0me.png)

```
 kubectl get pods -n kube-system
```

![image.png](/assets/images/2021/05-13/m4vl8lm09k.png)

# 后记：

1. 不知道为什么镜像仓库的设计是否是只设计了一级目录，没有考虑多级的。国外标准一换，国内的都抓瞎了。
2. 再次熟悉下kubeadm-config 的 configmap
3. ctr 命令的熟练使用
4. kubectl drain  kubectl uncordon的命令使用
5. kubeadm upgrade node与之前版本的不一样

​

​

