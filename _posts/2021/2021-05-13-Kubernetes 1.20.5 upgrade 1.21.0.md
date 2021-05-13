---
layout: post
title: 2021-0513-Kubernetes 1.20.5 upgrade 1.21.0
date: 2021-05-13 6:00:00
category: kubernetes
tags: kubeadm kubernetes1.20.5 upgrade kubernetes1.21
author: duiniwukenaihe
---
* content
{:toc}

# 背景：
集群环境参照：[centos8+kubeadm1.20.5+cilium+hubble环境搭建](https://cloud.tencent.com/developer/article/1806089 )。kubernets有了新的版本。1.21.0。嗯准备升级一下啊。


# 1. Kubernetes 1.20.5 upgrade 1.21.0过程
## 1. 查看当前可升级版本
 
```
yum list --showduplicates kubeadm --disableexcludes=kubernetes
```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620893541424-9c269a29-0b92-4cad-8377-e390d8fc3e89.png#clientId=u84c3379d-e2bf-4&from=paste&height=508&id=u35f7f230&margin=%5Bobject%20Object%5D&name=image.png&originHeight=508&originWidth=1654&originalType=binary&size=73590&status=done&style=none&taskId=u2cb0a600-4435-49bd-a198-ada7608677a&width=1654)
​

​

##  2. 升级kubeadm（所有节点都要操作的）
```
yum install  kubeadm-1.21.0 kubectl-1.21.0 --disableexcludes=kubernetes
```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620893610091-92d5b069-333a-4045-9a4a-cddbdfad5bf4.png#clientId=u84c3379d-e2bf-4&from=paste&height=672&id=ud2833468&margin=%5Bobject%20Object%5D&name=image.png&originHeight=672&originWidth=1616&originalType=binary&size=79365&status=done&style=none&taskId=u6bf0b777-1c9a-43bc-9c74-3cb3c9a69cf&width=1616)
## 3.验证集群是否可以平滑升级
```
kubeadm upgrade plan
```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620893906897-2764a3b6-c1fb-41b7-b00b-ead26cde5d2c.png#clientId=u84c3379d-e2bf-4&from=paste&height=768&id=u2850fbc8&margin=%5Bobject%20Object%5D&name=image.png&originHeight=768&originWidth=1604&originalType=binary&size=91063&status=done&style=none&taskId=ue5a40b4e-9425-4b1a-a57c-b60612cfef7&width=1604) 
## 4 . master节点（控制平面） 平滑升级
注：集群环境是一个ha的环境。这部操作选择了再sh-master-01节点执行
```
kubeadm upgrade apply v1.21.0 --certificate-renewal=false
```
--certificate-renewal=false  嗯 我不想重新生成证书....搭建了才没有多久......
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620893462312-47f85f14-7bd8-4fb5-81a6-8a8176c737a2.png#clientId=u84c3379d-e2bf-4&from=paste&height=341&id=uab00bb0b&margin=%5Bobject%20Object%5D&name=image.png&originHeight=341&originWidth=1876&originalType=binary&size=73766&status=done&style=none&taskId=u40404a64-263b-4730-aa27-025067f5c2f&width=1876)
嗯 报错了 找不到coredns的镜像：我还有个1.16的kubernetes的集群对比了一下，貌似coredns的目录层级多了一级？
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620893989914-94596b89-d228-4a8d-941a-7d5d6135ed96.png#clientId=u84c3379d-e2bf-4&from=paste&height=161&id=u1f5afd2e&margin=%5Bobject%20Object%5D&name=image.png&originHeight=161&originWidth=967&originalType=binary&size=33707&status=done&style=none&taskId=u6b683b12-145e-44fa-a060-d3aa42207fe&width=967)
查看了一眼默认仓库：
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620894072851-b1bb66a6-31ef-425e-b72b-8bf09795616d.png#clientId=u84c3379d-e2bf-4&from=paste&height=141&id=u78b2c5fc&margin=%5Bobject%20Object%5D&name=image.png&originHeight=141&originWidth=1104&originalType=binary&size=23611&status=done&style=none&taskId=uff2d0ec8-dba0-436b-b24b-2255074c2c3&width=1104)
然后 ctr images pull registry.aliyuncs.com/google_containers/coredns/coredns:v1.8.0
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620894696859-007d7346-ef94-48e7-9350-fccbeeea3402.png#clientId=u84c3379d-e2bf-4&from=paste&height=149&id=ufc2c2018&margin=%5Bobject%20Object%5D&name=image.png&originHeight=149&originWidth=1598&originalType=binary&size=29474&status=done&style=none&taskId=u993bfaa8-2bc2-4ecd-8591-910d25fe934&width=1598)
仓库下也确实没有这个image。也特意看了下阿里云 包括腾讯云的镜像仓库貌似不支持这样的层级的目录？可能在设计的时候都没有想到后面有这样的层级吗？
一般来说是下载镜像 然后修改镜像tag.....可是我试了几次upgrade仍然是失败：
各种找文章的顺路看到了[https://blog.51cto.com/u_3252740/2717642](https://blog.51cto.com/u_3252740/2717642)
就顺路玩了一下：
### 1. 修改kubeadm-config configmap
```
kubectl -n kube-system edit cm kubeadm-config
```
coredns的镜像仓库是可以单独定义的：那我就单独定义一下吧,整成k8s.gcr.io佛系定义。就想单独定义一下dns的仓库。
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620894963957-66177219-a58a-4ff8-8907-10e797614224.png#clientId=u84c3379d-e2bf-4&from=paste&height=521&id=uda175762&margin=%5Bobject%20Object%5D&name=image.png&originHeight=521&originWidth=891&originalType=binary&size=42198&status=done&style=none&taskId=u8ce16d9a-8321-434e-bb8c-b63795bf1ae&width=891)
### 2. 下载镜像并修改镜像标签
```
ctr -n k8s.io images pull uhub.service.ucloud.cn/uxhy/v1.8.0
ctr -n k8s.io images pull uhub.service.ucloud.cn/uxhy/coredns:v1.8.0
ctr -n k8s.io images tag uhub.service.ucloud.cn/uxhy/coredns:v1.8.0 k8s.gcr.io/coredns/coredns:v1.8.0
```
然后
master-01 忘了截图借用了下其他节点的截图
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620895268158-643f192e-140c-460a-8e0d-5932b5e12bf2.png#clientId=u84c3379d-e2bf-4&from=paste&height=202&id=u1668ba2e&margin=%5Bobject%20Object%5D&name=image.png&originHeight=202&originWidth=1464&originalType=binary&size=38230&status=done&style=none&taskId=u83a2304e-4cad-44a2-b6b2-8ac93c58127&width=1464)
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
​

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620895597437-f037e2e9-e3f4-42ab-9d2b-7d3659a0c980.png#clientId=u84c3379d-e2bf-4&from=paste&height=670&id=u7a99e13b&margin=%5Bobject%20Object%5D&name=image.png&originHeight=670&originWidth=1660&originalType=binary&size=155140&status=done&style=none&taskId=u3b304700-b8f5-4825-babf-7c7b77cc2ba&width=1660)


## 5 升级work节点


### 1. 升级kubeadm kubelet kubectl
```
yum install -y kubeadm-1.21.0 kubelet-1.21.0 kubectl-1.21.0 --disableexcludes=kubernetes
```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620896066329-bf817ea3-61bd-4d61-be0c-cd2367edddba.png#clientId=u84c3379d-e2bf-4&from=paste&height=759&id=ua5bf0962&margin=%5Bobject%20Object%5D&name=image.png&originHeight=759&originWidth=1626&originalType=binary&size=98528&status=done&style=none&taskId=uc144722c-c64f-4f0e-86a3-833dc42a4c4&width=1626)
### 2. 升级kubelet配置
```
 kubeadm upgrade node
```


![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620896267189-5f47156e-5f9c-48b5-a05a-2614f10d9be8.png#clientId=u84c3379d-e2bf-4&from=paste&height=302&id=uf4f0d343&margin=%5Bobject%20Object%5D&name=image.png&originHeight=302&originWidth=1629&originalType=binary&size=49582&status=done&style=none&taskId=u8a8719ab-8fec-4d71-885c-2c7daa2f6ab&width=1629)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620896313596-7dd2ec83-8198-4b6e-9635-6d94fa911895.png#clientId=u84c3379d-e2bf-4&from=paste&height=133&id=u776958b8&margin=%5Bobject%20Object%5D&name=image.png&originHeight=133&originWidth=1615&originalType=binary&size=20147&status=done&style=none&taskId=u5a46c8e2-3677-4040-af96-c212891bd4e&width=1615)
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
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620898890227-8d39870c-1678-42a3-be94-344086ae1625.png#clientId=u84c3379d-e2bf-4&from=paste&height=125&id=u6151c9e0&margin=%5Bobject%20Object%5D&name=image.png&originHeight=125&originWidth=1193&originalType=binary&size=16650&status=done&style=none&taskId=u9e07ff95-c486-4df0-9825-c258a07822f&width=1193)
```
kubectl get pods -n kube-system
```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1620898913771-ed2a8d91-61cc-4e25-9ca1-190635190bb5.png#clientId=u84c3379d-e2bf-4&from=paste&height=716&id=u3da2b757&margin=%5Bobject%20Object%5D&name=image.png&originHeight=716&originWidth=1349&originalType=binary&size=103708&status=done&style=none&taskId=ufe82f517-df6a-4c1a-ba7d-74b41f4d615&width=1349)
# 后记：  

1. 不知道为什么镜像仓库的设计是否是只设计了一级目录，没有考虑多级的。国外标准一换，国内的都抓瞎了。
1. 再次熟悉下kubeadm-config 的 configmap
1. ctr 命令的熟练使用
1. kubectl drain  kubectl uncordon的命令使用
1. kubeadm upgrade node与之前版本的不一样

​

​

