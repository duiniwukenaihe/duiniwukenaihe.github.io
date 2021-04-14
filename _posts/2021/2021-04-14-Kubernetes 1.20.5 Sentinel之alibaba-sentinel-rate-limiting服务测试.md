---
layout: post
title: Kubernetes 1.20.5 Sentinel之alibaba-sentinel-rate-limiting服务测试
date: 2021-04-14 9:00:00
category: kubernetes1.20 
tags:  kubernetes sentinel springboot
author: duiniwukenaihe
---
* content
{:toc}
# 背景：
参照上文 [**Kubernetes 1.20.5搭建sentinel** ](https://www.yuque.com/duiniwukenaihe/ehb02i/gezxbh)。想搭建一个服务去测试下sentinel服务的表现形式。当然了其他服务还没有集成 就找了一个demo,参照：[https://blog.csdn.net/liuhuiteng/article/details/107399979](https://blog.csdn.net/liuhuiteng/article/details/107399979)。
# 一. Sentinel之alibaba-sentinel-rate-limiting服务测试
## 1.maven构建测试jar包


参照[https://gitee.com/didispace/SpringCloud-Learning/tree/master/4-Finchley/alibaba-sentinel-rate-limiting](https://gitee.com/didispace/SpringCloud-Learning/tree/master/4-Finchley/alibaba-sentinel-rate-limiting)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618402185855-b4e736a7-cfc1-4183-9e5e-8648d62d2ba8.png#align=left&display=inline&height=345&margin=%5Bobject%20Object%5D&name=image.png&originHeight=690&originWidth=1303&size=104522&status=done&style=none&width=651.5)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618402224677-438caecf-fcea-4bb3-9b37-93319fbf80b7.png#align=left&display=inline&height=337&margin=%5Bobject%20Object%5D&name=image.png&originHeight=674&originWidth=1588&size=80535&status=done&style=none&width=794)
就修改了连接setinel dashboard的连接修改为kubernetes集群中setinel服务的service ip与8858端口。
maven打包jar包。


![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618402382586-019f5ca4-4c7e-47e9-9d69-07ae93d5c257.png#align=left&display=inline&height=342&margin=%5Bobject%20Object%5D&name=image.png&originHeight=683&originWidth=1912&size=143470&status=done&style=none&width=956)
## 2. 构建image镜像
其实 idea可以直接打出来image镜像的。以后研究吧。复用了一下原来做其他springboot的Dockerfile。

cat Dockerfile 
```
FROM openjdk:8-jdk-alpine
VOLUME /tmp
ADD app.jar app.jar
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]

```
注：我mv了下把打出来的jar包改名为app.jar了...也可以在pom.xml文件中自定义的修改了......
```
docker build -t ccr.ccs.tencentyun.com/XXXX/testjava:0.3 .
docker push ccr.ccs.tencentyun.com/XXXXX/testjava:0.3 
```
## 3. 部署测试服务
cat alibaba-sentinel-rate-limiting.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test
spec:
  replicas: 1
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: test
  template:
    metadata:
      labels:
        app: test
    spec:
      containers:
        - name: test
          image: ccr.ccs.tencentyun.com/XXXX/testjava:0.3
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
  name: test
  labels:
    app: test
spec:
  ports:
  - port: 8001
    protocol: TCP
    targetPort: 8001
  selector:
    app: test
```
注意：服务就命名为test了 ......
```
kubectl  apply -f alibaba-sentinel-rate-limiting.yaml -n nacos
```
## 4. 访问 alibaba-sentinel-rate-limiting服务，观察sentinel dashboard
访问alibaba-sentinel-rate-limiting服务，内部测试就不用ingress对外暴露了。直接内部CluserIP访问了
```
$kubectl get svc -n nacos
NAME             TYPE        CLUSTER-IP        EXTERNAL-IP   PORT(S)             AGE
mysql            ClusterIP   172.254.43.125    <none>        3306/TCP            2d9h
nacos-headless   ClusterIP   None              <none>        8848/TCP,7848/TCP   2d8h
sentinel         ClusterIP   172.254.163.115   <none>        8858/TCP,8719/TCP   26h
test             ClusterIP   172.254.143.171   <none>        8001/TCP            3h5m
```
kubernetes 服务器集群内部  curl 172.254.143.171:8001/hello  curl了十次......
登陆[https://sentinel.saynaihe.com/](https://sentinel.saynaihe.com/)观察


![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618403339597-5c42e19d-8230-4b45-bcac-957a082b7e4c.png#align=left&display=inline&height=390&margin=%5Bobject%20Object%5D&name=image.png&originHeight=779&originWidth=1721&size=186960&status=done&style=none&width=860.5)
# 二.  测试下setinel功能
## 1. 流控规则
嗯访问了下还行实时监控是有记录了
随手做个流控阈值测试下qps就设置1吧
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618403575398-aa43a071-f31f-42c2-b26b-94e0cd9301ec.png#align=left&display=inline&height=365&margin=%5Bobject%20Object%5D&name=image.png&originHeight=729&originWidth=1609&size=90549&status=done&style=none&width=804.5)


![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618403380608-a4d47c6f-715a-45ac-ba2e-e36886cf3407.png#align=left&display=inline&height=387&margin=%5Bobject%20Object%5D&name=image.png&originHeight=774&originWidth=1607&size=81447&status=done&style=none&width=803.5)
嗯效果如下。哈哈哈哈
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618403512962-19de7b89-7eaf-4150-8f9b-82acbe5454a3.png#align=left&display=inline&height=88&margin=%5Bobject%20Object%5D&name=image.png&originHeight=175&originWidth=986&size=37201&status=done&style=none&width=493)
嗯 有失败的了。
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618403747461-aba20dd9-0882-411f-8c0e-9b2505e8bbdf.png#align=left&display=inline&height=337&margin=%5Bobject%20Object%5D&name=image.png&originHeight=674&originWidth=1586&size=96842&status=done&style=none&width=793)
但是发现诡异的是我的这个应该是测试的没有配置存储什么的。经常的实时监控就没有了.....不知道是不是没有配置存储的原因。
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618403761050-93eb3a4f-fa67-4a79-bc20-e2d2b9841866.png#align=left&display=inline&height=329&margin=%5Bobject%20Object%5D&name=image.png&originHeight=657&originWidth=1591&size=55104&status=done&style=none&width=795.5)
## 2. 降级规则
降级规则就不测试了 觉得都一样...简单
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618404059659-97bc1a5c-6ea0-413f-99ec-ea7cb4d83a10.png#align=left&display=inline&height=401&margin=%5Bobject%20Object%5D&name=image.png&originHeight=801&originWidth=1511&size=87148&status=done&style=none&width=755.5)
## 3. 热点
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618404139377-b871ef08-8494-43b2-a225-54d3c933ae79.png#align=left&display=inline&height=378&margin=%5Bobject%20Object%5D&name=image.png&originHeight=756&originWidth=1465&size=78857&status=done&style=none&width=732.5)
## 4. 系统规则
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618404158576-5f5f6a41-bbf7-415f-8c9f-010333ad0568.png#align=left&display=inline&height=394&margin=%5Bobject%20Object%5D&name=image.png&originHeight=788&originWidth=1493&size=72750&status=done&style=none&width=746.5)
## 5. 授权规则
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618404183828-ae73e037-a5e2-48ef-8153-a17c4eb8673a.png#align=left&display=inline&height=370&margin=%5Bobject%20Object%5D&name=image.png&originHeight=740&originWidth=1434&size=70339&status=done&style=none&width=717)
## 6. 集群流控
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618404206162-f85ae786-3673-4290-b6aa-fa730e295b92.png#align=left&display=inline&height=379&margin=%5Bobject%20Object%5D&name=image.png&originHeight=758&originWidth=1584&size=64746&status=done&style=none&width=792)
机器列表。嗯 我想把pod变成两个试试 看看是不是我理解的这样会变成2个？
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618404239372-6d9ebbdc-b4ac-4987-87b8-2de7d90c18cd.png#align=left&display=inline&height=349&margin=%5Bobject%20Object%5D&name=image.png&originHeight=697&originWidth=1583&size=62048&status=done&style=none&width=791.5)
重温一下kubectl scale --help
```
kubectl scale deployment/test --replicas=2 -n nacos
```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618404376396-6bfd3725-18d5-4b73-8a4a-56eb3aa36c33.png#align=left&display=inline&height=128&margin=%5Bobject%20Object%5D&name=image.png&originHeight=256&originWidth=1165&size=32840&status=done&style=none&width=582.5)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1618404450384-9b0a529a-040a-4e59-9bc2-69adf6a8caf6.png#align=left&display=inline&height=398&margin=%5Bobject%20Object%5D&name=image.png&originHeight=795&originWidth=1843&size=187497&status=done&style=none&width=921.5)
对于我来说基本效果已经达到...其他的继续研究吧


