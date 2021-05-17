---
layout: post
title: kubernetes部署springboot项目使用configmap尝试
date: 2021-05-17 2:00:00
category: kubernetes
tags: kubernetes  springboot configmap
author: duiniwukenaihe
---
* content
{:toc}


# 背景：
我的基础环境all in kubernetes,参见：[https://cloud.tencent.com/developer/article/1806089，https://cloud.tencent.com/developer/article/1811859](https://cloud.tencent.com/developer/article/1806089)后端大佬们玩springboot cloud项目.故要讲springboot cloud项目部署在kubernetes集群中。其实使用springboot cloud价格我还是有所反对的。看过一些文章如：[https://www.cnblogs.com/lakeslove/p/10997011.html](https://www.cnblogs.com/lakeslove/p/10997011.html)。springboot与我的kubernetes有很多的重合功能了。本来就是差不多同时兴起的项目....如果用新的东西 我还是比较想上服务网格：istio这样的。既然决定springboot cloud on kubernets了 那就先这样玩了......
关于打包 都是maven的本来可以直接接手的。但是程序喜欢自己打，我就只在项目里面放Dockerfile.只负责镜像层面了：
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1621232532736-259ae4bc-1eee-44df-a175-d88d84ad4c4c.png#clientId=u361e41e3-34e9-4&from=paste&height=200&id=u4088b86f&margin=%5Bobject%20Object%5D&name=image.png&originHeight=200&originWidth=977&originalType=binary&size=12092&status=done&style=none&taskId=u0f7c701b-b1e1-454f-af9e-b09f2a7f5fa&width=977)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1621232563124-66ec92b5-049c-489a-ad6b-e12e67c30cac.png#clientId=u361e41e3-34e9-4&from=paste&height=415&id=u88dd25e7&margin=%5Bobject%20Object%5D&name=image.png&originHeight=415&originWidth=1196&originalType=binary&size=26515&status=done&style=none&taskId=u95050108-9ff2-43e8-afcc-2566f9357e6&width=1196)基本就是这个样子，当然了发布环境的时候我本来想写configmap的方式直接让程序去读我的环境变量的......但是程序找我要数据库 redis的连接地址 账号密码 说要写在 配置文件[application.yml](http://github.layabox.com:10080/laya-open-php/pvp/blob/develop/game/src/main/resources/application.yml)中，无果。随他去了。自己拉了一下t项目试一下是否可以在springboot中使用configmap的方式。
# 1. kubernetes部署springboot项目使用configmap
百度随手搜了一下啊关键词  springboot kubernetes  configmap一堆：
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1621250381923-a61d9398-c5ba-4ca0-9745-58d18c1c8464.png#clientId=u361e41e3-34e9-4&from=paste&height=811&id=ub789b2ba&margin=%5Bobject%20Object%5D&name=image.png&originHeight=811&originWidth=1489&originalType=binary&size=195835&status=done&style=none&taskId=u84485469-755d-42c6-8966-9dac1506e41&width=1489)
比如图上这个，但是都感觉不是我想要的，我就想简单的整下我的变量。然后无意间看到了：[https://capgemini.github.io/engineering/externalising-spring-boot-config-with-kubernetes/](https://capgemini.github.io/engineering/externalising-spring-boot-config-with-kubernetes/)就按照这个方式来搞一下了：
注： 因为就是简单测试下我不想让他们写死文件，其他过程就省了了 我就直接打个包测试下了
## 1. 将用到的参数变量化
参照原配置文件：
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1621251260325-ef667190-3aca-49b0-b2c5-7b1ed6d5ec95.png#clientId=u361e41e3-34e9-4&from=paste&height=802&id=u37d5fb6d&margin=%5Bobject%20Object%5D&name=image.png&originHeight=802&originWidth=1144&originalType=binary&size=62040&status=done&style=none&taskId=uc33fd1cf-a971-4be2-909a-fcd54abeda2&width=1144)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1621251217676-902dbe26-dbc0-4e6b-8cc9-6cae8d9790fb.png#clientId=u361e41e3-34e9-4&from=paste&height=539&id=uafcbb0f2&margin=%5Bobject%20Object%5D&name=image.png&originHeight=539&originWidth=1028&originalType=binary&size=37869&status=done&style=none&taskId=ubba8b557-cd52-4d8b-bba8-f660c9588ea&width=1028)
修改后的：
变量名都是自己随手写的 主要测试效果能否实现。当然了实际的需要和程序统一的还是规范化参数要好的，${}的格式都是。
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1621251377898-cc342ce2-4943-463b-95b2-caf11721718b.png#clientId=u361e41e3-34e9-4&from=paste&height=855&id=u4892efab&margin=%5Bobject%20Object%5D&name=image.png&originHeight=855&originWidth=1404&originalType=binary&size=148667&status=done&style=none&taskId=u810cdfd4-714d-424a-9714-12c532af2e6&width=1404)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1621251401721-0cc0a07b-4c8f-4f08-afd6-7e5032316574.png#clientId=u361e41e3-34e9-4&from=paste&height=891&id=u108fae41&margin=%5Bobject%20Object%5D&name=image.png&originHeight=891&originWidth=1557&originalType=binary&size=105994&status=done&style=none&taskId=u2503cf8c-0f2d-4457-8151-ede123fcdaf&width=1557)
嗯提取了8个参数将其变量化。
## 2. 生成jar包并构建docker image
docker打包没有集成在我的jenkins pipeline里面（程序的库，我就不做过多参与了），生成jar包
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1621251538050-012f584d-23ab-4965-88b4-122b16aef0b9.png#clientId=u361e41e3-34e9-4&from=paste&height=859&id=u14b9d956&margin=%5Bobject%20Object%5D&name=image.png&originHeight=859&originWidth=1821&originalType=binary&size=135414&status=done&style=none&taskId=ue3a90e84-9345-4b07-96c7-5d0d009b27c&width=1821)
将jar包上传到我一台有docker环境的服务器上面打包成docker image:
cat Dockerfile 
```
FROM openjdk:8-jdk-alpine
VOLUME /tmp
ADD target/game-1.0-SNAPSHOT.jar game-1.0-SNAPSHOT.jar
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/game-1.0-SNAPSHOT.jar"]

```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1621251631643-8bd1e83a-51ff-4595-9482-8dfc66366f2b.png#clientId=u361e41e3-34e9-4&from=paste&height=194&id=u82679e07&margin=%5Bobject%20Object%5D&name=image.png&originHeight=194&originWidth=983&originalType=binary&size=19906&status=done&style=none&taskId=u1dbdeb4c-7884-48e2-86c9-7b963858903&width=983)
```
docker build -t ccr.ccs.tencentyun.com/xxxx/xxxx:0.2 .
docker push ccr.ccs.tencentyun.com/xxxx/xxxx:0.2
```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1621251681297-63089375-e184-4000-87af-08b4221c775c.png#clientId=u361e41e3-34e9-4&from=paste&height=339&id=u6a8bcb6d&margin=%5Bobject%20Object%5D&name=image.png&originHeight=339&originWidth=849&originalType=binary&size=45748&status=done&style=none&taskId=u8fa019a6-ca86-4c7a-ad35-fe2c07bf5f1&width=849)
## 3. 生成configmap文件
cat spring-boot.yaml
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: spring-config
data:
  dev-config.json:
    '{
      "redis.database.host": "xxxx",
      "redis.database.port": "xxxx",
      "redis.database.password": "xxxx",
      "mysql.database.url": "jdbc:mysql://xxxx:3306/xxxx",
      "mysql.database.username": "xxxx",
      "mysql.database.password": "xxxxx",
      "cloud.nacos.server-addr": "http://xxxx:8848",
      "cloud.nacos.discovery.server-addr": "http://xxxx:8848"
     }'

```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1621251877725-a441f63c-71ec-4550-a87f-fe2a83a33783.png#clientId=u361e41e3-34e9-4&from=paste&height=283&id=uac4313e3&margin=%5Bobject%20Object%5D&name=image.png&originHeight=283&originWidth=956&originalType=binary&size=30703&status=done&style=none&taskId=u6b8e2d98-890b-48e6-9980-e0c1ae19c43&width=956)
apply部署configmap文件：
```
kubectl apply -f spring-config.yaml -n qa
```
describe一下：
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1621252015679-10dfa1fe-2ce0-4ac3-92d5-b507a538273e.png#clientId=u361e41e3-34e9-4&from=paste&height=344&id=u0fb8cef9&margin=%5Bobject%20Object%5D&name=image.png&originHeight=344&originWidth=1773&originalType=binary&size=51529&status=done&style=none&taskId=u8559f0d7-1293-4f0f-aa80-95fb97d6562&width=1773)
## 4. 部署springboot 服务
cat test.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pvp-test
spec:
  replicas: 1
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: pvp-test
  template:
    metadata:
      labels:
        app: pvp-test
    spec:
      containers:
        - name: pvp-test
          image: ccr.ccs.tencentyun.com/xxxx/xxxx:0.2
          env:
          - name: SPRING_PROFILES_ACTIVE
            value: "qa"
          - name: SPRING_APPLICATION_JSON
            valueFrom:
             configMapKeyRef:
              name: spring-config
              key: dev-config.json
          envFrom:
          - configMapRef:
              name: deploy
          ports:
            - containerPort: 8001
              name: game-http
            - containerPort: 8011
              name: game-tcp
          resources:
            requests:
              memory: "512M"
              cpu: "500m"
            limits:
              memory: "512M"
              cpu: "500m" 
      imagePullSecrets:                                              
        - name: tencent
---

apiVersion: v1
kind: Service
metadata:
  name: pvp-test
  labels:
    app: pvp-test
spec:
  ports:
  - port: 8001
    name: game-http
    targetPort: 8001
  - port: 8011
    name: game-tcp
    targetPort: 8011
  selector:
    app: pvp-test

```
```
kubectl apply -f 2.yaml -n qa
```
注： imagePullSecrets为下载image的秘钥。如果你是公开的仓库可以忽略。我的仓库用的腾讯云的个人版。秘钥自己创建名字就叫tencent了.
**测试时候比较仓库 配置文件都起名 1  2 这样的yaml文件了见谅**
## 5. 查看部署结果与nacos注册状况
```
kubectl  get pods -n qa
kubectl logs -f pvp-test-7f49fcdb9-dsjlz  -n qa
```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1621252209907-d63a81ac-2d7e-4bab-9b12-e9e4b746bd8b.png#clientId=u361e41e3-34e9-4&from=paste&height=663&id=ub057f301&margin=%5Bobject%20Object%5D&name=image.png&originHeight=663&originWidth=1519&originalType=binary&size=152246&status=done&style=none&taskId=u87d6b59e-9c1f-4945-ab8e-b18179e0f8a&width=1519)
启动的过程中是有错误的但是先忽略这个。因为我看了一下啊nacos中 我的服务其实已经注册上了....。初步我想要 结果算是实现了！
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1621252253011-de01ac09-1acc-4d7c-b712-dbbdc9871d02.png#clientId=u361e41e3-34e9-4&from=paste&height=507&id=uf4dc7e81&margin=%5Bobject%20Object%5D&name=image.png&originHeight=507&originWidth=1817&originalType=binary&size=129684&status=done&style=none&taskId=u9051ea6c-6210-404b-a02e-248bf0f2fcb&width=1817)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1621252353948-712dac03-4a09-4e1c-8be0-3d5dc7320381.png#clientId=u361e41e3-34e9-4&from=paste&height=357&id=ua6f439bd&margin=%5Bobject%20Object%5D&name=image.png&originHeight=357&originWidth=1204&originalType=binary&size=82941&status=done&style=none&taskId=ud8bf6e68-b68f-4bf1-970f-8dfae94a987&width=1204)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1621252381898-a9eb2017-94c1-4ca4-9305-2e5409a1e0a8.png#clientId=u361e41e3-34e9-4&from=paste&height=593&id=u7c978a85&margin=%5Bobject%20Object%5D&name=image.png&originHeight=593&originWidth=1604&originalType=binary&size=54578&status=done&style=none&taskId=u76aba383-5309-4602-9319-396c83acb32&width=1604)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1621252405217-0ee194eb-3f36-4133-9d54-8f99f7538f7f.png#clientId=u361e41e3-34e9-4&from=paste&height=748&id=u074ad496&margin=%5Bobject%20Object%5D&name=image.png&originHeight=748&originWidth=1545&originalType=binary&size=59818&status=done&style=none&taskId=u43ea197c-58e3-4c6d-bee9-06cc5697d40&width=1545)
​

## 6. 关于报错：
字面上的意思吧？
**User "system:serviceaccount:qa:default" cannot get resource "configmaps" in API group "" in the namespace "qa".**
  这里算是RBAC  clusterrole  rolebinding的一个复习吧。 
cat configmap-get.yaml
```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: qa
  name: configmap-get
rules:
- apiGroups: [""]
  resources: ["configmap"]
  verbs: ["get"]
```
与serviceaccount:qa:default绑定
```
kubectl create clusterrolebinding configmap-get-configmap --clusterrole=configmap-get --serviceaccount=qa:default
```
杀死容器继续查看新的容器 的log
```
kubectl delete pods pvp-test-7f49fcdb9-dsjlz -n qa
kubectl logs -f pvp-test-7f49fcdb9-ck9m6 -n qa
```
​

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1621253295494-d82e570b-f9dc-4818-8a6c-df64aa54c1f3.png#clientId=u361e41e3-34e9-4&from=paste&height=381&id=u007ec482&margin=%5Bobject%20Object%5D&name=image.png&originHeight=381&originWidth=1240&originalType=binary&size=57903&status=done&style=none&taskId=uac2ac014-5a94-4dda-aa64-c76cd5c77c2&width=1240)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1621253265890-921e023c-d1db-4328-b8f7-e2d1ff6c231f.png#clientId=u361e41e3-34e9-4&from=paste&height=382&id=u31bf2db4&margin=%5Bobject%20Object%5D&name=image.png&originHeight=382&originWidth=1683&originalType=binary&size=115776&status=done&style=none&taskId=u8ba26538-868d-4833-bd5b-5a701fb7194&width=1683)
依然报错...仔细一看日志....嗯 参数应该是configmaps....，我少写了一个s吧？修改configmap-get.yaml文件如下：


```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: qa
  name: configmap-get
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get"]

```
 apply 重新部署clusterrole。删除旧pod重新查看日志：
```
kubectl apply -f configmap-get.yaml
kubectl delete pods  pvp-test-7f49fcdb9-ck9m6 -n qa
```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1621253376564-4f31d373-d4cc-4e8e-9aa8-1d0938fe03b2.png#clientId=u361e41e3-34e9-4&from=paste&height=778&id=u3faf3776&margin=%5Bobject%20Object%5D&name=image.png&originHeight=778&originWidth=1844&originalType=binary&size=262398&status=done&style=none&taskId=u7f8f23a6-b568-4287-8c87-4383ccfd60a&width=1844)
嗯这次总算成功了
# 后记：
今天复习了好几个知识点......

1. configmap 
1. clusterrole rolebinding





