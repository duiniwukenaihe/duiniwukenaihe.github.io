---
layout: post
title: Kubernetes 1.20.5 实现一个服务注册与发现到Nacos
date: 2021-04-15 6:00:00
category: kubernetes1.20 
tags:  kubernetes nacos springboot
author: duiniwukenaihe
---
* content
{:toc}


# 背景：
参照 [Kubernetes 1.20.5搭建nacos](https://www.yuque.com/duiniwukenaihe/ehb02i/vsg9vd)，在kubernetes集群中集成了nacos服务。想体验下服务的注册与发现功能。当然了 也想体验下sentinel各种的集成，反正就是spring cloud alibaba全家桶的完整体验啊......一步一步来吧哈哈哈。先来一下服务的注册与发现。
特别鸣谢[https://blog.didispace.com/spring-cloud-alibaba-1/](https://blog.didispace.com/spring-cloud-alibaba-1/)，程序猿DD的系列文章昨天无意间看到的，很不错，已经收藏。
## 一. maven打包构建应用IMAGE镜像
注意：以下maven打包构建流程基本copy自[https://blog.didispace.com/spring-cloud-alibaba-1/](https://blog.didispace.com/spring-cloud-alibaba-1/)。所以就不做代码的搬运工了。项目演示的代码都可以去程序猿DD大佬的github去下载。


**关 于服务提供者与服务消费者alibaba-nacos-discovery-server为服务提供者 alibaba-nacos-discovery-client-common为服务消费者强调一下。**
## 1. 构建alibaba-nacos-discovery-server image
嗯 还的强调一下服务提供者！这个是......
![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618456318988-4389faaa-fa42-4ed8-88f4-423ea4298fa5.png#clientId=ub53151f9-d7c9-4&from=paste&height=377&id=u6c507613&margin=%5Bobject%20Object%5D&originHeight=753&originWidth=1854&originalType=binary&size=131689&status=done&style=none&taskId=u54c766ac-e387-4cd5-8f0b-4e31c751a43&width=927)
由于我个人打包测试是要在kubernets集群中使用，并且跟nacos服务在同一命名空间。连接nacos这里我就使用了内部服务名。package 打包 target目录下生成alibaba-nacos-discovery-server-0.0.1-SNAPSHOT.jar  jar包。
![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618456455339-f08c9c52-b7bf-4b2a-953f-8b635ec9c21b.png#clientId=ub53151f9-d7c9-4&from=paste&height=213&id=u8e89761e&margin=%5Bobject%20Object%5D&originHeight=425&originWidth=843&originalType=binary&size=27772&status=done&style=none&taskId=u8659d446-2982-4e7e-aa57-5d808d26cf7&width=421.5)
上传包到服务器改名app.jar ,复用Dockerfile
```
FROM openjdk:8-jdk-alpine
VOLUME /tmp
ADD app.jar app.jar
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
```
打包镜像，将镜像上传到镜像仓库（个人使用了腾讯云的私有仓库，当然了也可以自己安装harbor做仓库）。
```
mv alibaba-nacos-discovery-server-0.0.1-SNAPSHOT.jar app.jar
docker build -t ccr.ccs.tencentyun.com/XXXX/alibaba-nacos-discovery-server:0.1 .
docker push  ccr.ccs.tencentyun.com/XXXX/alibaba-nacos-discovery-server:0.1 

```
![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618456543561-f87ecf42-f5ed-4c60-873f-86bf1a47a34a.png#clientId=ub53151f9-d7c9-4&from=paste&height=176&id=u6b830b9c&margin=%5Bobject%20Object%5D&originHeight=351&originWidth=1010&originalType=binary&size=50807&status=done&style=none&taskId=ue1e0abdc-6d73-499b-88fe-c909423304a&width=505)
## 2 . 构建libaba-nacos-discovery-client-common image
基本步骤与上一步一样：
![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618457623428-c0296163-2fe2-4886-89bd-e32a5e69fbc3.png#clientId=ub53151f9-d7c9-4&from=paste&height=349&id=udbc6a035&margin=%5Bobject%20Object%5D&originHeight=698&originWidth=1704&originalType=binary&size=114611&status=done&style=none&taskId=ua9e279a1-716a-4ab7-8d03-ee8e05767aa&width=852)


![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618457701962-40431de3-d9da-43ba-a3c8-7e0ade44b9f0.png#clientId=ub53151f9-d7c9-4&from=paste&height=216&id=u444fcb07&margin=%5Bobject%20Object%5D&originHeight=432&originWidth=1029&originalType=binary&size=64246&status=done&style=none&taskId=u8f8329ad-2f15-4924-99b7-5064af6c7f8&width=514.5)
# 二. Kubernetes集群部署服务
## 1. 部署alibaba-nacos-discovery-server
cat alibaba-nacos-discovery-server.yaml 
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alibaba-nacos-discovery-server
spec:
  replicas: 1
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: alibaba-nacos-discovery-server
  template:
    metadata:
      labels:
        app: alibaba-nacos-discovery-server
    spec:
      containers:
        - name: talibaba-nacos-discovery-server
          image: ccr.ccs.tencentyun.com/XXXX/alibaba-nacos-discovery-server:0.1
          ports:
            - containerPort: 8001
          resources:
            requests:
              memory: "256M"
              cpu: "250m"
            limits:
              memory: "512M"
              cpu: "500m" 
      imagePullSecrets:                                              
        - name: tencent
---

apiVersion: v1
kind: Service
metadata:
  name: alibaba-nacos-discovery-server
  labels:
    app: alibaba-nacos-discovery-server
spec:
  ports:
  - port: 8001
    protocol: TCP
    targetPort: 8001
  selector:
    app: alibaba-nacos-discovery-server

```
```
kubectl apply -f alibaba-nacos-discovery-server.yaml -n nacos
```
kubectl get pods -n nacos
![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618466522701-6297a8ec-ebd8-4752-ba34-ebff45fd182a.png#clientId=ub53151f9-d7c9-4&from=paste&height=182&id=u890c2316&margin=%5Bobject%20Object%5D&originHeight=364&originWidth=1306&originalType=binary&size=50274&status=done&style=none&taskId=u2ad3485b-a6f7-4d49-968f-49b4868ca9f&width=653)
登陆nacos管理地址可以发现服务管理-服务列表有alibaba-nacos-discovery-server注册
![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618466555633-94391d51-313f-4c47-abaf-20ed7608dfee.png#clientId=ub53151f9-d7c9-4&from=paste&height=398&id=uda9a109d&margin=%5Bobject%20Object%5D&originHeight=796&originWidth=1584&originalType=binary&size=87845&status=done&style=none&taskId=ua215cd30-b917-4706-bb1c-1a006048cf1&width=792)
## 2. 部署alibaba-nacos-discovery-client-common
消费者 消费者 消费者。自己老记不住
 cat alibaba-nacos-discovery-client-common.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alibaba-nacos-discovery-common
spec:
  replicas: 1
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: alibaba-nacos-discovery-common
  template:
    metadata:
      labels:
        app: alibaba-nacos-discovery-common
    spec:
      containers:
        - name: talibaba-nacos-discovery-common
          image: ccr.ccs.tencentyun.com/XXXX/alibaba-nacos-discovery-client-common:0.1
          ports:
            - containerPort: 9000
          resources:
            requests:
              memory: "256M"
              cpu: "250m"
            limits:
              memory: "512M"
              cpu: "500m" 
      imagePullSecrets:                                              
        - name: tencent
---

apiVersion: v1
kind: Service
metadata:
  name: alibaba-nacos-discovery-common
  labels:
    app: alibaba-nacos-discovery-common
spec:
  ports:
  - port: 9000
    protocol: TCP
    targetPort: 9000
  selector:
    app: alibaba-nacos-discovery-common
```
```
kubectl apply -f alibaba-nacos-discovery-client-common.yaml -n nacos
```
这个时候登陆nacos管理页面应该是有alibaba-nacos-discovery-client-common服务注册了。
## 3. 更改alibaba-nacos-discovery-server副本数验证服务
仔细看了下[https://blog.didispace.com/spring-cloud-alibaba-1/](https://blog.didispace.com/spring-cloud-alibaba-1/) 他启动了两个alibaba-nacos-discovery-server实例，嗯 两个服务提供者。好验证一下nacos将服务上线下线的基础功能。正好我也扩容一下顺便补习一下scale命令
```
kubectl scale deployment/alibaba-nacos-discovery-server --replicas=2 -n nacos
```
![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618467394685-37a92884-b724-4782-99fa-ee24546601e6.png#clientId=ub53151f9-d7c9-4&from=paste&height=70&id=u822e2f5a&margin=%5Bobject%20Object%5D&originHeight=139&originWidth=1052&originalType=binary&size=22278&status=done&style=none&taskId=u646e1396-3f37-4bae-b296-98640b3b831&width=526)
前面访问alibaba-nacos-discovery-server服务都是只有10.0.5.120一个ip，deployment扩容完成后访问alibaba-nacos-discovery-client-common（消费者）放回了两个服务提供者轮训方式的两个ip。
![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618467109476-d9ca7e91-d4ec-4efe-98e2-7cd7d7ea72a0.png#clientId=ub53151f9-d7c9-4&from=paste&height=205&id=u1bfed143&margin=%5Bobject%20Object%5D&originHeight=410&originWidth=1081&originalType=binary&size=82198&status=done&style=none&taskId=u77382c03-3ab9-4016-8efa-6b2c7cc148c&width=540.5)
观察nacos管理web页面，服务列表alibaba-nacos-discovery-server实例数量，健康实例都已经变成2：
![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618467585637-db7f6319-ae13-4477-8488-5c6552701658.png#clientId=ub53151f9-d7c9-4&from=paste&height=257&id=uf08e95eb&margin=%5Bobject%20Object%5D&originHeight=513&originWidth=1580&originalType=binary&size=57633&status=done&style=none&taskId=u90b1086a-b599-40cf-aaad-b5f569ba397&width=790)
点开alibaba-nacos-discovery-server服务详情。嗯集群下出现了两个实例的具体信息。
![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618467604320-8bf44d75-dcc8-4ab3-987d-e8cae51916b2.png#clientId=ub53151f9-d7c9-4&from=paste&height=389&id=ufe396e43&margin=%5Bobject%20Object%5D&originHeight=778&originWidth=1458&originalType=binary&size=58876&status=done&style=none&taskId=u49801b83-a8ad-4de6-863f-b917a7fc917&width=729)
看到集群实例配置这里有个下线的功能想体验一下：
![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618467697459-c8ce1003-fb21-4d33-a9b1-dfa1eaa1b6bb.png#clientId=ub53151f9-d7c9-4&from=paste&height=394&id=u6440f0e0&margin=%5Bobject%20Object%5D&originHeight=787&originWidth=1463&originalType=binary&size=60005&status=done&style=none&taskId=ue7c25d73-94a6-4bb6-b905-9a397db4020&width=731.5)
下线了10.5.208的实例 再访问一下。发现确实只有10.0.5.120节点了：
![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618467790071-e5dfab52-230a-495c-9bef-f99d2dfb18e1.png#clientId=ub53151f9-d7c9-4&from=paste&height=408&id=u1b651a96&margin=%5Bobject%20Object%5D&originHeight=816&originWidth=1920&originalType=binary&size=270973&status=done&style=none&taskId=ua642900a-b33c-485b-9e09-20d56ea2e85&width=960)
恢复实例访问服务提供者10.5.208。点击上线，不做其他设置。继续访问消费者应用应该是两个ip的轮询权重一样的：
![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618467917377-9ca5de2e-6653-4c25-b38f-5de081f45257.png#clientId=ub53151f9-d7c9-4&from=paste&height=393&id=u9afd0a8e&margin=%5Bobject%20Object%5D&originHeight=785&originWidth=1920&originalType=binary&size=287546&status=done&style=none&taskId=ueb202f90-e69a-443d-b93c-ae665c74951&width=960)
简单记录 简单记录。积少成多........










