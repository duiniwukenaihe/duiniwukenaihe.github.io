---
layout: post
title: 2021-07-15-Kubernets traefik代理ws wss应用
date: 2021-07-15 2:00:00
category: kubernetes
tags: kubernetes traefik ws wss
author: duiniwukenaihe
---
* content
{:toc}
# 背景：

团队要发布一组应用，springboot开发的ws应用。然后需要对外。支持ws wss协议。jenkins写完pipeline发布任务。记得过去没有上容器的时候都是用的腾讯云的[cls](https://console.cloud.tencent.com/clb/) 挂证书映射[cvm](https://console.cloud.tencent.com/cvm)端口。我现在的网络环境是这样的：[Kubernetes 1.20.5 安装traefik在腾讯云下的实践](https://www.yuque.com/duiniwukenaihe/ehb02i/odflm7)（当然了本次的环境是跑在tke1.20.6上面的，都是按照上面实例搭建的---除了我新建了一个namespace traefik，并将traefik应用都安装在了这个命名空间内！这样做的原因是tke的kebe-system下的pod太多了！我有强迫症）

# 部署与分析过程：

## 1. 关于我的应用：

应用的部署方式是statefulset，如下：

```
cat <<EOF >  xxx-gateway.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: xxx-gateway
spec:
  serviceName: xxx-gateway
  replicas: 1
  selector:
    matchLabels:
      app: xxx-gateway
  template:
    metadata:
      labels:
        app: xxx-gateway
    spec:
      containers:
        - name: xxx-gateway
          image: ccr.ccs.tencentyun.com/xxx-master/xxx-gateway:202107151002
          env:
          - name: SPRING_PROFILES_ACTIVE
            value: "official"
          - name: SPRING_APPLICATION_JSON
            valueFrom:
             configMapKeyRef:
              name: spring-config
              key: dev-config.json
          ports:
            - containerPort: 8443
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
  name: xxx-gateway
  labels:
    app: xxx-gateway
spec:
  ports:
  - port: 8443
  selector:
    app: xxx-gateway
  clusterIP: None
EOF
```

```
kubectl apply -f xxx-gateway.yaml -n official
```

偷个懒直接copy了一个其他应用的 ingress yaml修改了一下,如下：

```
cat <<EOF >  gateway-0-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: layaverse-gateway-0-http
  namespace: official
  annotations:
    kubernetes.io/ingress.class: traefik  
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  rules:
  - host: xxx-gateway-0.xxx.com
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: xxx-gateway 
            port: 
              number: 8443
EOF
```

部署ingress

```
kubectl apply -f gateway-0-ingress.yaml
```

查看ingress部署状况

```
kubectl get ingress -n official
```

![image.png](/assets/images/2021/07-15/4e660ad277a713b30bec342b45049019.png)

嗯 然后测试一下wss（wss我直接用443端口了。证书挂载slb层的--这是我理解的!具体的参照我traefik的配置）,这里强调一下wscat这个工具。反正看了下我们的后端小伙伴测试ws应用都是用的在线的ws工具:

![image.png](/assets/images/2021/07-15/5c39afc4bd22cf7c9d3232ff7fb77e7b.png)

就这样的。然后正巧看到wscat就安装了一下：

```
sudo apt install npm
sudo npm install -g wscat 
wscat -c wss://xxx-gateway-0.xxx.com:443/ws
```

![image.png](/assets/images/2021/07-15/e0c42a73f9ffbe1ced5d9e7275531379.png)

嗯哼 基本上可以确认是应用对外成功了？

--------------------------

当然了以上只是我顺利的假想！

实际上是代理后连接后端的ws服务依然有各种问题（开始我怀疑是traefik的问题），还是连不上！我粗暴的把xxx-gateway 的暴露方式修改为NodePort  然后挂载到了slb层（在scl直接添加了ssl证书），测试了一下是可以的就直接用了。先让应用跑起来，然后再研究一下怎么处理。

## 2. 关于ws和http:

先不去管那么多，先整明白实现我的traefik如何实现代理ws呢？

![image.png](/assets/images/2021/07-15/c08554861298e648dab2dbc9b2dcd3f8.png)

图中内容摘自：[https://blog.csdn.net/fmm\_sunshine/article/details/77918477](https://blog.csdn.net/fmm_sunshine/article/details/77918477)

------------------------------------------------------------------------------------------------------

## 3. 排查下是谁的锅

### 1. 搭建一个简单的ws应用

后端的代码既然搞不懂，那我就找一个简单的ws的服务然后用traefik代理测试一下！

dockerhub搜得一个nodejs 的websocket镜像：[https://hub.docker.com/r/ksdn117/web-socket-test](https://hub.docker.com/r/ksdn117/web-socket-test)

部署一下：

```
cat <<EOF >  web-socket-test.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web-socket-test
spec:
  serviceName: web-socket-test
  replicas: 1
  selector:
    matchLabels:
      app: web-socket-test
  template:
    metadata:
      labels:
        app: web-socket-test
    spec:
      containers:
        - name: web-socket-test
          image: ksdn117/web-socket-test
          ports:
            - containerPort: 8010
              name: web
            - containerPort: 8443
              name: ssl
          resources:
            requests:
              memory: "512M"
              cpu: "500m"
            limits:
              memory: "512M"
              cpu: "500m"
---

apiVersion: v1
kind: Service
metadata:
  name: web-socket-test
  labels:
    app: web-socket-test
spec:
  type: NodePort
  ports:
  - port: 8010
    targetPort: 8010
    protocol: TCP
    name: web
  - port: 8443
    targetPort: 8443
    name: ssl
    protocol: TCP
  selector:
    app: web-socket-test
EOF
```

注： 我这里的配置文件加了type:NodePort

```
kubectl  apply -f web-socket-test.yaml
kubectl get pods 
kubectl get svc 
```

![image.png](/assets/images/2021/07-15/57e6b37232ad08b2e3cdeab97b8d1b66.png)

### 2.内部wscat测试ws服务是否联通

先内部连接container pod ip测试一下服务：

![image.png](/assets/images/2021/07-15/ffb0ecf581bda317a8a6fdf4fabe6f98.png)

```
wscat --connect ws://172.22.0.230:8010
```

```
kubectl logs -f web-socket-test-0
```

![image.png](/assets/images/2021/07-15/f6793fbfc8be215e87f5251bac09d2e5.png)

### 3.traefik对外代理ws应用并测试

traefik正常的对外暴露服务可以用ingress的方式还有ingressroute我都去尝试一下：

#### 1. ingressroute方式

```
cat <<EOF >  web-socket-ingressroute.yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: web-socket-test-http
  namespaces: default
spec:
  entryPoints:
  - web
  routes:
  - match: Host(`web-socket-test.xxx.com`)
    kind: Rule
    services:
      - name: web-socket-test
        port: 8010
 EOF
 kubectl apply -f web-socket-ingressroute.yaml
```

![image.png](/assets/images/2021/07-15/f43e6ceffc048347ef4a8dabe1b51ec3.png)

wscat连接测试一下:

![image.png](/assets/images/2021/07-15/3d4f0073f95d757596df5860caff97c1.png)

这样测来是没有问题的？

删除ingress

```
 kubectl delete -f web-socket-ingressroute.yaml
```

#### 2. ingress方式

整一下ingress的方式：

```
cat <<EOF >  web-socket-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-socket-test
  namespace: default
  annotations:
    kubernetes.io/ingress.class: traefik  
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  rules:
  - host: web-socket-test.layame.com
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: web-socket-test 
            port: 
              number: 8010
 EOF
 kubectl apply -f web-socket-ingress.yaml
```

![image.png](/assets/images/2021/07-15/95e4cfbe239509f180edc5c48a6c1606.png)

```
wscat --connect wss://web-socket-test.xxx.com:443
```

![image.png](/assets/images/2021/07-15/e0398e2fcca9cdd4e8933447a42f7235.png)

甩锅基本完成起码不是我的基础设施应的问题.....让后端小伙伴测试一下看下是哪里有问题了。从我代理层来说是没有问题的！

# 关于其他：

当然了看一些博客还有要加**passHostHeader: true**的配置的

## 1. ingressroute：

![image.png](/assets/images/2021/07-15/c47fcce97a3a0cbb1af992978166b372.png)

## 2. ingress
```
ingress:traefik.ingress.kubernetes.io/service.passhostheader: "true"
```
​

![image.png](/assets/images/2021/07-15/c47fcce97a3a0cbb1af992978166b372.png)

如果有问题 可以尝试一下上面的方式！