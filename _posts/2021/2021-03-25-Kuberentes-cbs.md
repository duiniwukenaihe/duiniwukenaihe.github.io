---
layout: post
title: Kuberentes集群添加腾讯云CBS为默认存储
date: 2021-03-26 17:00:00
category: kubernetes1.20
tags:  kubernetes CBS
author: duiniwukenaihe
---
* content
{:toc}

# 前言
接上文[https://blog.csdn.net/saynaihe/article/details/115187298](https://blog.csdn.net/saynaihe/article/details/115187298) kubeadm  搭建高可用ha集群，接下来考虑的度量有存储，对外暴露服务。日志收集，监控报警几项。个人习惯就先将存储优先来讨论了。
关于存储storageclass
如[https://kubernetes.io/zh/docs/concepts/storage/storage-classes](https://kubernetes.io/zh/docs/concepts/storage/storage-classes)所示，常见的有很多类型,如下：
![image.png](https://img-blog.csdnimg.cn/img_convert/dde52dd300170764556021abdf4c2a46.png#align=left&display=inline&height=195&margin=[objectObject]&name=image.png&originHeight=390&originWidth=248&size=11290&status=done&style=none&width=124)
由于我的集群建在公有云上面 腾讯云有开源的cbs 的 csi组件。在kubernetes1.16-1.18 环境使用docker 做runtime的环境中使用过腾讯云的开源组件。这就又拿来用了。方便集成。
# 初始环境
参加：[https://editor.csdn.net/md/?articleId=115187298](https://editor.csdn.net/md/?articleId=115187298)

| 主机名 | ip | 系统 | 内核 |
| --- | --- | --- | --- |
| sh-master-01 | 10.3.2.5  | centos8 | 4.18.0-240.15.1.el8_3.x86_64 |
| sh-master-02 | 10.3.2.13 | centos8 | 4.18.0-240.15.1.el8_3.x86_64 |
| sh-master-03 | 10.3.2.16 | centos8 | 4.18.0-240.15.1.el8_3.x86_64 |
| sh-work-01 | 10.3.2.2 | centos8 | 4.18.0-240.15.1.el8_3.x86_64 |
| sh-work-02 | 10.3.2.2 | centos8 | 4.18.0-240.15.1.el8_3.x86_64 |
| sh-work-03 | 10.3.2.4 | centos8 | 4.18.0-240.15.1.el8_3.x86_64 |

# 集成腾讯云 CBS CSI
## 1. clone 仓库
注： kubernetes-csi-tencentcloud中包括 CBS CSI，  CFS CSI 与 COSFS CSI。这里我就只用CBS块存储了。其他两个也用过，感觉用起来还是不太适合。
```
git clone https://github.com/TencentCloud/kubernetes-csi-tencentcloud.git
```
各种名词可以参照：[https://github.com/TencentCloud/kubernetes-csi-tencentcloud/blob/master/docs/README_CBS_zhCN.md](https://github.com/TencentCloud/kubernetes-csi-tencentcloud/blob/master/docs/README_CBS_zhCN.md)。
## 2.参照文档前置要求完成kubernetes集群的配置修改
### 1. master节点
参照[https://github.com/TencentCloud/kubernetes-csi-tencentcloud/blob/master/docs/README_CBS_zhCN.md](https://github.com/TencentCloud/kubernetes-csi-tencentcloud/blob/master/docs/README_CBS_zhCN.md)。参照一下图片前置要求
![image.png](https://img-blog.csdnimg.cn/img_convert/4c103c1c600f27936fcb33428538261b.png#align=left&display=inline&height=339&margin=[objectObject]&name=image.png&originHeight=678&originWidth=1240&size=96553&status=done&style=none&width=620)对三台master节点修改 kube-apiserver.yaml kube-controller-manager.yaml kube-scheduler.yaml。增加如下配置（查看对应版本对应需求）
```
 - --feature-gates=VolumeSnapshotDataSource=true
```
![image.png](https://img-blog.csdnimg.cn/img_convert/dc9d51d2d0935e29203a5e713bfe42fd.png#align=left&display=inline&height=246&margin=[objectObject]&name=image.png&originHeight=491&originWidth=959&size=49878&status=done&style=none&width=479.5)
![image.png](https://img-blog.csdnimg.cn/img_convert/e78eb4b1ccc2d77a212eb399cc871c6a.png#align=left&display=inline&height=228&margin=[objectObject]&name=image.png&originHeight=456&originWidth=963&size=45375&status=done&style=none&width=481.5)
![image.png](https://img-blog.csdnimg.cn/img_convert/6c1c842b7db9cc48fb049cf7e5a5fcef.png#align=left&display=inline&height=224&margin=[objectObject]&name=image.png&originHeight=448&originWidth=973&size=38729&status=done&style=none&width=486.5)
### 2. 修改所有节点的kubelet配置
kubelet增加--feature-gates=VolumeSnapshotDataSource=true的支持
![image.png](https://img-blog.csdnimg.cn/img_convert/946f5d3fc28c66c4083fd747e3e236b9.png#align=left&display=inline&height=61&margin=[objectObject]&name=image.png&originHeight=121&originWidth=1378&size=23031&status=done&style=none&width=689)
# 3.部署CBS CSI插件
## 1. 使用腾讯云 API Credential 创建 kubernetes secret:
### 1. 前提：
首先的在腾讯云后台[https://console.cloud.tencent.com/cam](https://console.cloud.tencent.com/cam)创建一个用户，访问方式我是只开通了编程访问，至于用户权限则需要开通CBS相关权限。我是直接绑定了CSB两个默认相关的权限，还有财务付款权限，记得一定的支持付款，否则硬盘创建不了......。玩的好的可以自定义创建下权限，否则个人觉得财务权限貌似有点大......
![image.png](https://img-blog.csdnimg.cn/img_convert/3f82e8ac162441cbbf7537c653ce10bd.png#align=left&display=inline&height=361&margin=[objectObject]&name=image.png&originHeight=721&originWidth=1543&size=86001&status=done&style=none&width=771.5)
![image.png](https://img-blog.csdnimg.cn/img_convert/282fe41f530152a3d7cdc76cc4fd4f87.png#align=left&display=inline&height=366&margin=[objectObject]&name=image.png&originHeight=732&originWidth=1499&size=69960&status=done&style=none&width=749.5)
### 2. 根据文档提示将SecretId SecretKey base64转换生成kubernetes secret
```
echo -n "XXXXXXXXXXX" |base6
 echo -n "XXXXXXXXXX" |base64
```
将base64写入secret.yaml文件
![image.png](https://img-blog.csdnimg.cn/img_convert/e3c1c9068c00392414904c077406c80a.png#align=left&display=inline&height=134&margin=[objectObject]&name=image.png&originHeight=267&originWidth=977&size=27777&status=done&style=none&width=488.5)
```
cd /root/kubernetes-csi-tencentcloud-master/deploy/cbs/kubernetes
kubectl apply -f secret.yaml
```
注：**项目是在root目录git clone的，故cd /root/kubernetes-csi-tencentcloud-master/deploy/cbs/kubernetes.包括一下没有特别强调目录的，都是在此目录下执行的**
**![image.png](https://img-blog.csdnimg.cn/img_convert/3a0e231afbf7ef97d8cb52232b7aec61.png#align=left&display=inline&height=96&margin=[objectObject]&name=image.png&originHeight=191&originWidth=1016&size=24729&status=done&style=none&width=508)**
## 2. 创建rbac
创建attacher,provisioner,plugin需要的rbac：
```
kubectl apply -f  csi-controller-rbac.yaml
kubectl apply -f  csi-node-rbac.yaml
```
## 3.创建controller,node和plugin
创建controller plugin和node plugin
```
kubectl apply -f  csi-controller.yaml
kubectl apply -f  csi-node.yaml
### snapshot-crd我没有使用，字面意思应该是快照的....
kubectl apply -f  snapshot-crd.yaml 
```
kubectl get pods -n kube-system 可以看到cbs-csi相关组件创建ing：
![image.png](https://img-blog.csdnimg.cn/img_convert/836c21097717eca14a6d8dae352b4041.png#align=left&display=inline&height=309&margin=[objectObject]&name=image.png&originHeight=617&originWidth=839&size=81252&status=done&style=none&width=419.5)
# 4. 验证
切换目录
cd   /root/kubernetes-csi-tencentcloud-master/deploy/cbs/examples

1. 参照storageclass参数修改storageclass-basic.yaml

![image.png](https://img-blog.csdnimg.cn/img_convert/ab44a5b4369a95c7f811151097aeb55f.png#align=left&display=inline&height=253&margin=[objectObject]&name=image.png&originHeight=505&originWidth=1333&size=125348&status=done&style=none&width=666.5)
```
创建storageclass:
    kubectl apply -f  storageclass-basic.yaml
创建pvc:
    kubectl apply -f  pvc.yaml
创建申请pvc的pod:
    kubectl apply -f  app.yaml
```
![image.png](https://img-blog.csdnimg.cn/img_convert/0f4a3e380a819f4cfd26acb0d635a0f3.png#align=left&display=inline&height=279&margin=[objectObject]&name=image.png&originHeight=557&originWidth=888&size=67805&status=done&style=none&width=444)
嗯 kubectl get storageclass  
![image.png](https://img-blog.csdnimg.cn/img_convert/62f3a38e2aeb7d960661bdb0ffd246c5.png#align=left&display=inline&height=44&margin=[objectObject]&name=image.png&originHeight=87&originWidth=1217&size=12553&status=done&style=none&width=608.5)
到此为止，搭建其他应用就可以用cbs存储了。具体参数看文档  看文档 看文档  说三遍。
