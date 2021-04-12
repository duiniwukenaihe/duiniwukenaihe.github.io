---
layout: post
title: Kubernetes 1.20.5搭建nacos
date: 2021-04-11 7:00:00
category: kubernetes1.20
tags: kubernetes nacos
author: duiniwukenaihe
---
* content
{:toc}
# 前言：

 后端小伙伴们准备搞pvp对战服务。配置中心选型选择了阿里云的nacos服务。参照[https://nacos.io/zh-cn/docs](https://nacos.io/zh-cn/docs)。由于业务规划都在kubernetes集群上，就简单参照[https://nacos.io/zh-cn/docs/use-nacos-with-kubernetes.html](https://nacos.io/zh-cn/docs/use-nacos-with-kubernetes.html)做了一个demo让他们先玩一下。

关于nacos:

参照：[https://nacos.io/zh-cn/docs/what-is-nacos.html](https://nacos.io/zh-cn/docs/what-is-nacos.html)

- \*\*服务发现和健康监测：\*\* 支持基于 DNS 和基于 RPC 的服务发现。服务提供者使用 [原生SDK](https://nacos.io/zh-cn/docs/sdk.html)、[OpenAPI](https://nacos.io/zh-cn/docs/open-api.html)、或一个[独立的Agent TODO](https://nacos.io/zh-cn/docs/other-language.html)注册 Service 后，服务消费者可以使用[DNS TODO](https://nacos.io/zh-cn/docs/xx) 或[HTTP&API](https://nacos.io/zh-cn/docs/open-api.html)查找和发现服务。提供对服务的实时的健康检查，阻止向不健康的主机或服务实例发送请求
- \*\*动态配置服务：\*\* Nacos 提供配置统一管理功能，能够帮助我们将配置以中心化、外部化和动态化的方式管理所有环境的应用配置和服务配置。
- \*\*动态 DNS 服务：\*\* Nacos 支持动态 DNS 服务权重路由，能够让我们很容易地实现中间层负载均衡、更灵活的路由策略、流量控制以及数据中心内网的简单 DNS 解析服务。
- \*\*服务及其元数据管理：\*\* Nacos 支持从微服务平台建设的视角管理数据中心的所有服务及元数据，包括管理服务的描述、生命周期、服务的静态依赖分析、服务的健康状态、服务的流量管理、路由及安全策略、服务的 SLA 以及最首要的 metrics 统计数据。
- 嗯 还有更多的特性列表......

# 一. nacos  on kubernetes

基本的安装过程参照：[https://github.com/nacos-group/nacos-k8s/blob/master/README-CN.md](https://github.com/nacos-group/nacos-k8s/blob/master/README-CN.md)

## 1. 创建命名空间

\*\*嗯当然了第一步还是先创建一个搭建nacos服务的namespace了：\*\*

```
kubectl create ns nacos
```

## 2. git clone 仓库

```
 git clone https://github.com/nacos-group/nacos-k8s.git
```

基本都会因为网络原因无法clone,我是直接下载包到本地 然后上传到服务器了。

## 3. 部署初始化mysql服务器

生产的话肯定是用云商的云数据库了，比如腾讯云的[rds](https://cloud.tencent.com/product/cdb?from=10680)服务。由于只是给程序整一个demo让他们玩一下，就讲mysql 整合在kubernetes中了。\*\*个人存储storageclass都是使用默认的腾讯云的[cbs-csi](https://cloud.tencent.com/product/cbs?from=10680)。\*\*

\*\*cd  /nacos-k8s/mysql(当然了我是上传的目录路径是\*\*/root/nacos/nacos-k8s-master/deploy/mysql\*\*)\*\*

### 1. 部署mysql服务

cat pvc.yaml

```
apiVersion: v1

kind: PersistentVolumeClaim

metadata:

  name: nacos-mysql-pvc

  namespace: nacos

spec:

  accessModes:

  - ReadWriteOnce

  resources:

    requests:

      storage: 10Gi

  storageClassName: cbs-csi
```

mysql的部署文件直接复制了mysql-ceph.yaml的修改了一下：

cat mysql.yaml

```
apiVersion: v1

kind: PersistentVolumeClaim

metadata:

  name: nacos-mysql-pvc

  namespace: nacos

spec:

  accessModes:

  - ReadWriteOnce

  resources:

    requests:

      storage: 10Gi

  storageClassName: cbs-csi

[root@sh-master-01 mysql]# cat mysql.yaml 

apiVersion: v1

kind: ReplicationControlle

metadata:

  name: mysql

  labels:

    name: mysql

spec:

  replicas: 1

  selector:

    name: mysql

  template:

    metadata:

      labels:

        name: mysql

    spec:

      containers:

      - name: mysql

        image: nacos/nacos-mysql:5.7

        ports:

        - containerPort: 3306

        env:

        - name: MYSQL\_ROOT\_PASSWORD

          value: "root"

        - name: MYSQL\_DATABASE

          value: "nacos\_devtest"

        - name: MYSQL\_USER

          value: "nacos"

        - name: MYSQL\_PASSWORD

          value: "nacos"

        volumeMounts:

        - name: mysql-persistent-storage

          mountPath: /var/lib/mysql

          subPath: mysql

          readOnly: false

      volumes:

      - name: mysql-persistent-storage

        persistentVolumeClaim:

          claimName: nacos-mysql-pvc

---

apiVersion: v1

kind: Service

metadata:

  name: mysql

  labels:

    name: mysql

spec:

  ports:

  - port: 3306

    targetPort: 3306

  selector:

    name: mysql
```

```
kubectl apply -f pvc.yaml

kubectl apply -f mysql.yaml -n nacos

kubectl get pods -n nacos
```

等待mysql pods  running

```
$kubectl get pods -n nacos

NAME          READY   STATUS    RESTARTS   AGE

mysql-hhs5q   1/1     Running   0          3h51m
```

### 2. 进入mysql 容器执行初始化脚本

```
kubectl exec -it mysql-hhs5q bash -n nacos

mysql -uroot -p root \*\*\*\*\*

create database nacos\_devtest;

use nacos\_devtest;

### 我是图省事，把这sql脚本里面直接复制进去搞了...

https://github.com/alibaba/nacos/blob/develop/distribution/conf/nacos-mysql.sql

-------

退出mysql控制台，并退出容器

quit; 

exit
```

## 4. 部署nacos

从mysql目录 cd ../nacos

cat  nacos.yaml

```
---

apiVersion: v1

kind: Service

metadata:

  name: nacos-headless

  labels:

    app: nacos

  annotations:

    service.alpha.kubernetes.io/tolerate-unready-endpoints: "true"

spec:

  ports:

    - port: 8848

      name: serve

      targetPort: 8848

    - port: 7848

      name: rpc

      targetPort: 7848

  clusterIP: None

  selector:

    app: nacos

---

apiVersion: v1

kind: ConfigMap

metadata:

  name: nacos-cm

data:

  mysql.db.name: "nacos\_devtest"

  mysql.port: "3306"

  mysql.user: "nacos"

  mysql.password: "nacos"

---

apiVersion: apps/v1

kind: StatefulSet

metadata:

  name: nacos

spec:

  serviceName: nacos-headless

  replicas: 3

  template:

    metadata:

      labels:

        app: nacos

      annotations:

        pod.alpha.kubernetes.io/initialized: "true"

    spec:

      affinity:

        podAntiAffinity:

          requiredDuringSchedulingIgnoredDuringExecution:

            - labelSelector:

                matchExpressions:

                  - key: "app"

                    operator: In

                    values:

                      - nacos

              topologyKey: "kubernetes.io/hostname"

      initContainers:

        - name: peer-finder-plugin-install

          image: nacos/nacos-peer-finder-plugin:1.0

          imagePullPolicy: Always

          volumeMounts:

            - mountPath: /home/nacos/plugins/peer-finde

              name: plguindi

      containers:

        - name: nacos

          imagePullPolicy: Always

          image: nacos/nacos-server:latest

          resources:

            requests:

              memory: "2Gi"

              cpu: "500m"

          ports:

            - containerPort: 8848

              name: client-port

            - containerPort: 7848

              name: rpc

          env:

            - name: NACOS\_REPLICAS

              value: "2"

            - name: SERVICE\_NAME

              value: "nacos-headless"

            - name: DOMAIN\_NAME

              value: "layabox.daemon"

            - name: POD\_NAMESPACE

              valueFrom:

                fieldRef:

                  apiVersion: v1

                  fieldPath: metadata.namespace

            - name: MYSQL\_SERVICE\_DB\_NAME

              valueFrom:

                configMapKeyRef:

                  name: nacos-cm

                  key: mysql.db.name

            - name: MYSQL\_SERVICE\_PORT

              valueFrom:

                configMapKeyRef:

                  name: nacos-cm

                  key: mysql.port

            - name: MYSQL\_SERVICE\_USER

              valueFrom:

                configMapKeyRef:

                  name: nacos-cm

                  key: mysql.use

            - name: MYSQL\_SERVICE\_PASSWORD

              valueFrom:

                configMapKeyRef:

                  name: nacos-cm

                  key: mysql.password

            - name: NACOS\_SERVER\_PORT

              value: "8848"

            - name: NACOS\_APPLICATION\_PORT

              value: "8848"



            - name: PREFER\_HOST\_MODE

              value: "hostname"

          volumeMounts:

            - name: plguindi

              mountPath: /home/nacos/plugins/peer-finde

            - name: datadi

              mountPath: /home/nacos/data

            - name: logdi

              mountPath: /home/nacos/logs

  volumeClaimTemplates:

    - metadata:

        name: plguindi

      spec:

        accessModes: [ "ReadWriteOnce" ]

        storageClassName: "cbs-csi"

        resources:

          requests:

            storage: 10Gi

    - metadata:

        name: datadi

      spec:

        accessModes: [ "ReadWriteOnce" ]

        storageClassName: "cbs-csi"

        resources:

          requests:

            storage: 10Gi

    - metadata:

        name: logdi

      spec:

        accessModes: [ "ReadWriteOnce" ]

        storageClassName: "cbs-csi"

        resources:

          requests:

            storage: 10Gi

  selector:

    matchLabels:

      app: nacos
```

主要就是修改了storageclassName为 cbs-csi。并修改了accessmodes,还有\*\*DOMAIN\_NAME变量修改为自己命名的集群后缀.简单demo不做详细论述。\*\*

```
kubectl apply -f nacos.yaml -n nacos
```

等待服务running

![image.png](https://ask.qcloudimg.com/http-save/1006587/0vs2g7wl8d.png)

## 5. 对外暴露服务

代理个人使用的是traefik。过去都是用ingresroute的方式对外映射暴露服务，现在用想ingress的方式：

cat ingress.yaml

```
apiVersion: networking.k8s.io/v1

kind: Ingress

metadata:

  name: nacos-headless-http

  namespace: nacos

  annotations:

    kubernetes.io/ingress.class: traefik  

    traefik.ingress.kubernetes.io/router.entrypoints: web

spec:

  rules:

  - host: nacos-server.saynaihe.com 

    http:

      paths:

      - pathType: Prefix

        path: /

        backend:

          service:

            name: nacos-headless

            port:

              number: 8848
```

kubectl apply -f ingress.yaml

访问：[https://nacos-server.saynaihe.com/nacos](https://nacos-server.layame.com/nacos)一定记得域名后面跟上nacos。否则是404呢，当然了也可以在ingress配置上面重定向直接到nacos下？看个人怎么玩了。

![image.png](https://ask.qcloudimg.com/http-save/1006587/fn8pffb3wl.png)

默认用户名密码 ：nacos nacos。当然了第一件事是修改密码......

![image.png](https://ask.qcloudimg.com/http-save/1006587/dmota1lmui.png)

嗯 先扔给程序去玩下了。还有很多配置的东西省略了。比如很多参数和变量，可以参照下官方配置进行搞一下......








