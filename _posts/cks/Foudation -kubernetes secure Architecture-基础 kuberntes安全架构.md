---
layout: "post"
title: "Foudation -kubernetes secure Architecture-基础 kuberntes安全架构"
date: "2021-03-13 18:00:00"
category: "kubernetes cks"
tags:  "Foudation -kubernetes secure Architecture-基础 kuberntes安全架构"
author: duiniwukenaihe
---
* content
{:toc}



# 1   docker   Architecture![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1612160877253-d97a71f9-d707-4b58-bced-068856b8677b.png#align=left&display=inline&height=476&margin=%5Bobject%20Object%5D&name=image.png&originHeight=476&originWidth=839&size=96824&status=done&style=none&width=839)
**run container in container engine-在容器引擎中运行容器**
**Schedule containers effcient-高效的调度容器**
**Keep containers  alive and  health- 保持容器的存活和健康**
**Allow container communication- 允许容器的通信**
**Allow  common deployment techniques-允许通用的部署技术**
**handle volumes-处理卷**




# 2.  kubernetes Architecture
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1612162703928-ea1d9798-51be-46bc-8a9d-a5f7d8ad4af6.png#align=left&display=inline&height=479&margin=%5Bobject%20Object%5D&name=image.png&originHeight=479&originWidth=846&size=177759&status=done&style=none&width=846)
## 2.1 Contarol Plance
Contarol Plance-控制平面，简单的不知道我理解的对不对为master节点上面的etcd  scheduler   apiserver  controler manager 。至于cloud  control manager我理解是使用云商托管的kubernets  比如腾讯云的 cke还有阿里云的ack  都有类似的kubernets的托管服务。


见： [https://kubernetes.io/docs/concepts/overview/components/#control-plane-components](https://kubernetes.io/docs/concepts/overview/components/#control-plane-components)


### 1. kube-apiserver  apiserver
控制平面暴露kubernets的api服务，API服务器是Kubernetes控制平面的前端。 水平扩展 平衡实例之间流量
### 2. etcd  etcd数据库
Consistent and highly-available key value store used as Kubernetes' backing store for all cluster data  高度一直的kv键值数据库。用作kubernetes的备份与数据存储。
### 3. kube-scheduler 调度器
 监视没有分配节点的新创建的Pod，并选择一个节点以使其运行。调度策略有很多种，官网现学现卖：
####      1. individual and collective resource requirements  个体和集体的资源需求
####      2. hardware/software/policy constraints 硬件/软件/策略约束
####      3. affinity and anti-affinity specifications 亲和性和反亲和力规范
####      4. data locality  数据本地性
####      5. inter-workload interference  工作负载之间的干扰
####      6. deadlines  生存周期
### 4. kube-controller-manager
Control Plane component that runs [controller](https://kubernetes.io/docs/concepts/architecture/controller/) processes. 控制平面运行控制的进程

- 节点控制器：负责在节点出现故障时进行通知和响应。
- 复制控制器：负责为系统中的每个复制控制器对象维护正确数量的Pod。
- 端点控制器：填充“端点”对象（即，加入“服务和窗格”）。
- 服务帐户和令牌控制器：为新的名称空间创建默认帐户和API访问令牌
### 5. cloud-controller-manage 
      云控制器暂时忽略吧，一般的还接触不到的


## 2.2 Data Plane
Data Plane-数据平面 。一般理解为work节点  工作节点？主要有kubelet  和kube-proxy服务
### 1. kubelet
在集群中每个节点上运行的代理。 确保容器在Pod中运行。维护节点上的网络规则.如果有kube-proxy可用，它将使用操作系统数据包过滤层。 否则，kube-proxy会转发流量本身
### 2. kube-proxy
kube-proxy是一个网络代理，它在集群中的每个节点上运行，实现了Kubernetes Service概念的一部分
### 3. Container runtime
容器运行时是负责运行容器的软件。
Kubernetes支持多种容器运行时：Docker，Containerd，CRI-O以及Kubernetes CRI（容器运行时接口）的任何实现。
## 3. Addons 组件扩展
注：   插件的命名空间资源属于`kube-system`命名空间
### 1.  dns  域名解析服务
### 2. Dashboard  webui 仪表盘
### 3. Container Resource Monitoring 容器资源的监控
### 4. Cluster-level Logging  集群级别的日志
## 4. 关于pod之间的通信
 注：  默认是全部可以通信的，当然了 可以通过networkpolicy等方式进行隔离
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1612167792428-baf3d545-2c96-486f-912b-f80a4f059f2c.png#align=left&display=inline&height=412&margin=%5Bobject%20Object%5D&name=image.png&originHeight=412&originWidth=862&size=74009&status=done&style=none&width=862)
## 5. PKI (Public Key Infrastructure 公共密钥基础设施)
### 1 . 关于pki的CA
注：  关于ca学习的不是太透彻，有时间单独的好好学习一下
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1612167854985-177a3fc8-1554-400d-935a-0a832e1068b7.png#align=left&display=inline&height=473&margin=%5Bobject%20Object%5D&name=image.png&originHeight=473&originWidth=846&size=82043&status=done&style=none&width=846)

1. ** CA  is the trusted root of all certificates inside the cluster  CA是群集内所有证书的受信任根**
1. ** All cluster ertificates are signed by the CA  所有集群的证书都由CA签名**
1. ** Used by components to validate each other  组件用来相互验证**



![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1612167866086-564d35f6-5c49-4ef6-b6e3-2d79e3d2e7e9.png#align=left&display=inline&height=473&margin=%5Bobject%20Object%5D&name=image.png&originHeight=473&originWidth=836&size=76353&status=done&style=none&width=836)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1612167875351-4fd0c8b0-c8e5-4d4e-a665-8f70072dea59.png#align=left&display=inline&height=419&margin=%5Bobject%20Object%5D&name=image.png&originHeight=419&originWidth=746&size=94349&status=done&style=none&width=746)

### 2. Find various K8s certificates  
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1612167884185-9efef269-f553-4f1c-b440-fc176ce33547.png#align=left&display=inline&height=422&margin=%5Bobject%20Object%5D&name=image.png&originHeight=422&originWidth=839&size=96375&status=done&style=none&width=839)


![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1612238511973-2b41b2d2-afb8-46ed-8282-c073d00eabdf.png#align=left&display=inline&height=529&margin=%5Bobject%20Object%5D&name=image.png&originHeight=529&originWidth=1160&size=69352&status=done&style=none&width=1160)
关于Scheduler->Api 和controller-manage->Api 和 kubelet->Api 还有 kubelet  server cert
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1612246547815-8ebfac05-9489-42f7-a83d-243825169515.png#align=left&display=inline&height=706&margin=%5Bobject%20Object%5D&name=image.png&originHeight=706&originWidth=1348&size=238495&status=done&style=none&width=1348)


![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1612246756037-2d0686b1-156a-4c16-ad36-a90b74721e13.png#align=left&display=inline&height=712&margin=%5Bobject%20Object%5D&name=image.png&originHeight=712&originWidth=1350&size=239986&status=done&style=none&width=1350)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1612246814076-d0dcc301-40af-4da5-9719-11139305d5a7.png#align=left&display=inline&height=435&margin=%5Bobject%20Object%5D&name=image.png&originHeight=435&originWidth=1345&size=89321&status=done&style=none&width=1345)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1612246997758-73a61958-7024-45eb-be6e-6cf131cdeb8e.png#align=left&display=inline&height=182&margin=%5Bobject%20Object%5D&name=image.png&originHeight=182&originWidth=1238&size=26993&status=done&style=none&width=1238)
