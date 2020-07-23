---
layout: "post"
title: "2020-07-23-kubernets集群使用腾讯云cbs块存储"
date: "2020-07-23 10:00:00"
category: "kubernetes"
tags:  "kubernetes1.18.6 kubernetes-csi-tencentcloud"
author: duiniwukenaihe
---
* content
{:toc}

 

集群配置：
centos7.7 64位

|  ip                                   |      主机名        | 
|  :-------------------------- :        |     :----:        |   
|10.0.4.20                              |      vip |  
|10.0.4.27                              |  sh-master-01     |
|10.0.4.46                              |  sh-master-02     |  
|10.0.4.47                              |  sh-master-02     |  
|10.0.4.14                              |  sh-node-01       |  
|10.0.4.2                               |  sh-node-02       |  
|10.0.4.6                               |  sh-node-03       |
|10.0.4.4                               |  sh-node-04       |
|10.0.4.13                              |  sh-node-05       |


## 背景
>环境为kubernets集群1.18.6，参照 https://duiniwukenaihe.github.io/2020/07/22/%E8%85%BE%E8%AE%AF%E4%BA%91-slb-kubeadm%E9%AB%98%E5%8F%AF%E7%94%A8%E9%9B%86%E7%BE%A4%E6%90%AD%E5%BB%BA/在腾讯云上搭建。过去自己搭建过rook-ceph但是版本升级或者节点挂掉出现了各种问题，而且ceph的规划什么的 自己也掌握的不好，正好有腾讯云自己开源的kubernetes-csi-tencentcloud组件 就准备使用他集成做kubernets集群的默认storageclass了。


#### 1.git clone 仓库
 ```bash
https://github.com/TencentCloud/kubernetes-csi-tencentcloud
现在github会非常卡你懂的，最好还是国外服务器下载了。
 ```
#### 2. kubernets集群更改配置
> 参照https://github.com/TencentCloud/kubernetes-csi-tencentcloud/blob/master/docs/README_CBS_zhCN.md，我的kubernets集群是1.18.6。
按照显性设置版本要求在kubelet apiserver  controller-manager scheduler 配置文件添加--feature-gates=VolumeSnapshotDataSource=true配置：
>


 ```bash
 cd /etc/kubernetes/manifests,修改三个master的  apiserver controller  scheduler配置文件如下：
  ```


![kube-api](/assets/images/2020/07/kubernetes-csi-tencentcloud/kube-api.png)
![kube-controller-manager](/assets/images/2020/07/kubernetes-csi-tencentcloud/kube-controller-manager.png)
![kube-scheduler](/assets/images/2020/07/kubernetes-csi-tencentcloud/kube-scheduler.png)

> 所有节点 包括master与work节点，进入/etc/sysconfig目录修改kubelet配置文件如下：

![kubelet](/assets/images/2020/07/kubernetes-csi-tencentcloud/kubelet.png)

 ```bash
所有节点重启kubelet ：
systemctl restart kubelet
 ```

> 步骤3-6基本就是按照腾讯云文档来就可以了，唯一可怕的就少镜像被墙....最好把镜像打包放到国内镜像仓库，然后修改配置文件。然后还有csi-node的apiVersion要修改一下呢，直接修改为apps/v1。



#### 3.使用腾讯云 API Credential 创建 kubernetes secret
 ```
#  参考示例 deploy/kubernetes/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: csi-tencentcloud
  namespace: kube-system
data:
  # 需要注意的是,secret 的 value 需要进行 base64 编码
  #   echo -n "<SECRET_ID>" | base64
  TENCENTCLOUD_CBS_API_SECRET_ID: "<SECRET_ID>"
  TENCENTCLOUD_CBS_API_SECRET_KEY: "<SECRET_KEY>"
```
![key](/assets/images/2020/07/kubernetes-csi-tencentcloud/key.png)
#### 4.创建rbac

创建attacher,provisioner,plugin需要的rbac：

```
kubectl apply -f  deploy/cbs/kubernetes/csi-controller-rbac.yaml
kubectl apply -f  deploy/cbs/kubernetes/csi-node-rbac.yaml
```

#### 5.创建controller,node和plugin
创建controller plugin和node plugin

```
kubectl apply -f  deploy/cbs/kubernetes/csi-controller.yaml
kubectl apply -f  deploy/cbs/kubernetes/csi-node.yaml
```

#### 6.简单测试验证

```
### 这里要根据自己的需求进行修改的哦，我的配置文件如下
###storageclass-basic.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: cbs-csi
provisioner: com.tencent.cloud.csi.cbs
parameters:
  diskType: CLOUD_PREMIUM
  diskChargeType: PREPAID
  diskChargeTypePrepaidPeriod: "1"
  diskChargePrepaidRenewFlag: NOTIFY_AND_AUTO_RENEW
```
```
创建storageclass:
    kubectl apply -f  deploy/examples/storageclass-basic.yaml
创建pvc:
    kubectl apply -f  deploy/examples/pvc.yaml
创建申请pvc的pod:
    kubectl apply -f  deploy/examples/app.yaml
```
> 下面是抄自官方的文档，按需来了，比如我的diskType 选择了高性能云盘， 付费类型选择了预付费，购买时长1个月，续费策略选择了通知过期不自动续费。


#### StorageClass 支持的参数

**Note**：可以参考[示例](https://github.com/TencentCloud/kubernetes-csi-tencentcloud/blob/master/deploy/cbs/examples/storageclass-examples.yaml)

* 如果您集群中的节点存在多个可用区，那么您可以开启cbs存储卷的拓扑感知调度，需要在storageclass中添加`volumeBindingMode: WaitForFirstConsumer`，如deploy/examples/storageclass-topology.yaml，否则可能会出现cbs存储卷因跨可用区而挂载失败。
* diskType: 代表要创建的 cbs 盘的类型；值为 `CLOUD_BASIC` 代表创建普通云盘，值为 `CLOUD_PREMIUM` 代表创建高性能云盘，值为 `CLOUD_SSD` 代表创建 ssd 云盘
* diskChargeType: 代表云盘的付费类型；值为 `PREPAID` 代表预付费，值为 `POSTPAID_BY_HOUR` 代表按量付费，需要注意的是，当值为 `PREPAID` 的时候需要指定额外的参数
* diskChargeTypePrepaidPeriod：代表购买云盘的时长，当付费类型为 `PREPAID` 时需要指定，可选的值包括 `1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 24, 36`，单位为月
* diskChargePrepaidRenewFlag: 代表云盘的自动续费策略，当付费类型为 `PREPAID` 时需要指定，值为`NOTIFY_AND_AUTO_RENEW` 代表通知过期且自动续费，值为 `NOTIFY_AND_MANUAL_RENEW` 代表通知过期不自动续费，值为 `DISABLE_NOTIFY_AND_MANUAL_RENEW` 代表不通知过期不自动续费
* encrypt: 代表云盘是否加密，当指定此参数时，唯一可选的值为 `ENCRYPT`

#### 不同类型云盘的大小限制

* 普通云硬盘提供最小 100 GB 到最大 16000 GB 的规格选择，支持 40-100MB/s 的 IO 吞吐性能和 数百-1000 的随机 IOPS 性能。
* 高性能云硬盘提供最小 50 GB 到最大 16000 GB 的规格选择。
* SSD 云硬盘提供最小 100 GB 到最大 16000 GB 的规格选择，单块 SSD 云硬盘最高可提供 24000 随机读写IOPS、260MB/s吞吐量的存储性能。


#### 然后我遇到的问题：


##### 1. cbs盘默认是普通云硬盘 ，然后区域呢不支持就出现了下面的图：

![basic-not-support](/assets/images/2020/07/kubernetes-csi-tencentcloud/basic-not-support.png)
```
注：失败了就接着习惯kubectl delete -f pvc.yaml 了。
```

##### 2.  SECRET我用了子账号首先，权限如下：
![authority](/assets/images/2020/07/kubernetes-csi-tencentcloud/authority.png)

> 个人觉得权限是没有问题的，但是就notpay了 不知道是还要分配那些权限

![notpay](/assets/images/2020/07/kubernetes-csi-tencentcloud/notpay.png)


#####  3.  将秘钥用主账号的id  key base64后修改deploy/kubernetes/secret.yaml ，kubectl apply -f deploy/kubernetes/secret.yaml。修改配置文件后pod中是不会生效的，然后删除一遍pod 重新部署下，偷了下懒用了删除kube-proxy 修改ipvs的方式：
```
kubectl get pod -n kube-system | grep csi-cbsplugin |awk '{system("kubectl delete pod "$1" -n kube-system")}'
```
![delete](/assets/images/2020/07/kubernetes-csi-tencentcloud/delete.png)
继续执行下kubectl apply -f pvc.yaml  kubectl apply -f app.yaml。如下图：
![demo-pod](/assets/images/2020/07/kubernetes-csi-tencentcloud/demo-pod.png)



结束基本就算跑起来了 然后复杂的 和其他的用法，在以后慢慢摸索了。

