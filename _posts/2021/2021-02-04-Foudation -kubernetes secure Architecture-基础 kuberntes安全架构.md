---
layout: post
title: kubernetes secure Architecture- kuberntes安全架构
date: 2021-02-04 10:00:00
category: cks
tags:  kubernetes cks
author: duiniwukenaihe
---
* content
{:toc


# 1.   docker   Architecture![image.png](https://img-blog.csdnimg.cn/img_convert/c7b638de18500a48e3094ea96222f99f.png#align=left&display=inline&height=476&margin=[objectObject]&name=image.png&originHeight=476&originWidth=839&size=96824&status=done&style=none&width=839)

```bash
1. run container in container engine-在容器引擎中运行容器
2. Schedule containers effcient-高效的调度容器
3. Keep containers  alive and  health- 保持容器的存活和健康
4. Allow container communication- 允许容器的通信
5. Allow  common deployment techniques-允许通用的部署技术
6. handle volumes-处理卷
```

# 2.  kubernetes Architecture![image.png](https://img-blog.csdnimg.cn/img_convert/fa4b4d6d2f53f833205d42b3bc8b565b.png#align=left&display=inline&height=479&margin=[objectObject]&name=image.png&originHeight=479&originWidth=846&size=177759&status=done&style=none&width=846)
## 2.1 Contarol Plance
Contarol Plance-控制平面，简单的不知道我理解的对不对为master节点上面的etcd  scheduler   apiserver  controler manager 。至于cloud  control manager我理解是使用云商托管的kubernets  比如腾讯云的 cke还有阿里云的ack  都有类似的kubernets的托管服务。


见： [https://kubernetes.io/docs/concepts/overview/components/#control-plane-components](https://kubernetes.io/docs/concepts/overview/components/#control-plane-components)


### 1. kube-apiserver  apiserver
控制平面暴露kubernets的api服务，API服务器是Kubernetes控制平面的前端。 水平扩展 平衡实例之间流量
### 2. etcd  etcd数据库
Consistent and highly-available key value store used as Kubernetes' backing store for all cluster data  高度一直的kv键值数据库。用作kubernetes的备份与数据存储。
### 3. kube-scheduler 调度器
 监视没有分配节点的新创建的Pod，并选择一个节点以使其运行。调度策略有很多种，官网现学现卖：

```bash
     1. individual and collective resource requirements  个体和集体的资源需求
     2. hardware/software/policy constraints 硬件/软件/策略约束
     3. affinity and anti-affinity specifications 亲和性和反亲和力规范
     4. data locality  数据本地性
     5. inter-workload interference  工作负载之间的干扰
     6. deadlines  生存周期
```

### 4. kube-controller-manager
Control Plane component that runs [controller](https://kubernetes.io/docs/concepts/architecture/controller/) processes. 控制平面运行控制的进程

- 节点控制器：负责在节点出现故障时进行通知和响应。
- 复制控制器：负责为系统中的每个复制控制器对象维护正确数量的Pod。
- 端点控制器：填充“端点”对象（即，加入“服务和窗格”）。
- 服务帐户和令牌控制器：为新的名称空间创建默认帐户和API访问令牌
### 5. cloud-controller-manage 
      云控制器暂时忽略吧，一般的还接触不到的


## 2.2.  Data Plane
Data Plane-----数据平面 。一般理解为work节点  工作节点？主要有kubelet  和kube-proxy服务
### 1. kubelet
在集群中每个节点上运行的代理。 确保容器在Pod中运行。维护节点上的网络规则.如果有kube-proxy可用，它将使用操作系统数据包过滤层。 否则，kube-proxy会转发流量本身
### 2. kube-proxy
kube-proxy是一个网络代理，它在集群中的每个节点上运行，实现了Kubernetes Service概念的一部分。
### 3. Container runtime
容器运行时是负责运行容器的程序。
Kubernetes支持多种容器运行时：Docker，Containerd，CRI-O以及Kubernetes CRI（容器运行时接口）的任何实现形式。
## 3.  Addons 组件扩展
注：   插件的命名空间资源属于`kube-system`命名空间

```bash
1. dns  域名解析服务
2. Dashboard  webui 仪表盘
3. Container Resource Monitoring 容器资源的监控
4. Cluster-level Logging  集群级别的日志
```

## 4. 关于pod之间的通信
 注：  默认是全部可以通信的，当然了 可以通过networkpolicy等方式进行隔离
![image.png](https://img-blog.csdnimg.cn/img_convert/cef83cdf48ad0f7bc49d596a395319a2.png#align=left&display=inline&height=412&margin=[objectObject]&name=image.png&originHeight=412&originWidth=862&size=74009&status=done&style=none&width=862)
## 5. PKI (Public Key Infrastructure 公共密钥基础设施)
### 1 . 关于pki的CA
注：  关于ca学习的不是太透彻，有时间单独的好好学习一下
![image.png](https://img-blog.csdnimg.cn/img_convert/dcd365772eae01dafaa6a5e237c0a0df.png#align=left&display=inline&height=473&margin=[objectObject]&name=image.png&originHeight=473&originWidth=846&size=82043&status=done&style=none&width=846)

```bash
1.  CA  is the trusted root of all certificates inside the cluster  CA是群集内所有证书的受信任根
2.  All cluster ertificates are signed by the CA  所有集群的证书都由CA签名
3. .Used by components to validate each other  组件用来相互验证
```

![image.png](https://img-blog.csdnimg.cn/img_convert/9c5c42ce2ee10c358fe152722add1674.png#align=left&display=inline&height=473&margin=[objectObject]&name=image.png&originHeight=473&originWidth=836&size=76353&status=done&style=none&width=836)
![image.png](https://img-blog.csdnimg.cn/img_convert/312e72f2663d2bd7a405ec5df0ac1467.png#align=left&display=inline&height=419&margin=[objectObject]&name=image.png&originHeight=419&originWidth=746&size=94349&status=done&style=none&width=746)

### 2. Find various K8s certificates  
![image.png](https://img-blog.csdnimg.cn/img_convert/5aeffa941899d6ef89d5f7f08d02cc55.png#align=left&display=inline&height=422&margin=[objectObject]&name=image.png&originHeight=422&originWidth=839&size=96375&status=done&style=none&width=839)

![image.png](https://img-blog.csdnimg.cn/img_convert/549cdd55ecd96b0a5d153acba1da8299.png#align=left&display=inline&height=529&margin=[objectObject]&name=image.png&originHeight=529&originWidth=1160&size=69352&status=done&style=none&width=1160)
关于Scheduler->Api 和controller-manage->Api 和 kubelet->Api 还有 kubelet  server cert
![image.png](https://img-blog.csdnimg.cn/img_convert/057aa2299851db3cde8b72bee555f8fb.png#align=left&display=inline&height=706&margin=[objectObject]&name=image.png&originHeight=706&originWidth=1348&size=238495&status=done&style=none&width=1348)


![image.png](https://img-blog.csdnimg.cn/img_convert/3fe3d251c6ca3ec957ea0d68bbbea3c6.png#align=left&display=inline&height=712&margin=[objectObject]&name=image.png&originHeight=712&originWidth=1350&size=239986&status=done&style=none&width=1350)
![image.png](https://img-blog.csdnimg.cn/img_convert/dabea5f63dd23e6a353bfc275e29189c.png#align=left&display=inline&height=435&margin=[objectObject]&name=image.png&originHeight=435&originWidth=1345&size=89321&status=done&style=none&width=1345)
![image.png](https://img-blog.csdnimg.cn/img_convert/0b4c3b5f7caa117e933ac6ad4a7d99b4.png#align=left&display=inline&height=182&margin=[objectObject]&name=image.png&originHeight=182&originWidth=1238&size=26993&status=done&style=none&width=1238)
