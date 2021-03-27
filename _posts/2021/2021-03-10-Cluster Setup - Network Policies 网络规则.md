---
layout: post
title: Cluster Setup - Network Policies 网络规则
date: 2021-03-10 10:00:00
category: cks
tags:  kubernetes cks networkpolicy
author: duiniwukenaihe
---
* content
{:toc

# 1. 网络安全规则


## 1.1. 关于网络安全规则
![image.png](https://img-blog.csdnimg.cn/img_convert/beda2d6001e392af1cfe02f24336f926.png#align=left&display=inline&height=577&margin=[objectObject]&name=image.png&originHeight=577&originWidth=1022&size=346057&status=done&style=none&width=1022)

```bash
 1.  Firewall rules in kubernetes kubernetes     集群中的防火墙规则
 2.  Implemented by the Network Plugin CNI(Calico/Weave)    网络插件的实施方案
 3.  Namespace level       命名空间的级别
 4.  Restrict the ingress and/or Egress for a goup of pods based on certain rules and conditions  根据某些规则和条件限制一组Pod的进入和/或出口
```


注： 先决条件是必须使用支持NetworkPolicy的网络解决方案
默认状况下没有网络策略的状态并且：

1.  by default every pod can access every pod 默认的可以访问任何pods
1.  pods are not isolated  pods 不是孤立的



## 2. 常见的网络规则
### 2.1.  允许的其他Pod组（例外：Pod无法阻止对其自身的访问。都是靠pod标签实现的
####         2.1.1.  允许包含某一组pod标签的pods组对外访问资源-egress出口
#### ![image.png](https://img-blog.csdnimg.cn/img_convert/643eb8b1ec0479c1ec08a86e771196e5.png#align=left&display=inline&height=582&margin=[objectObject]&name=image.png&originHeight=582&originWidth=1027&size=209095&status=done&style=none&width=1027)
####        2.1.2.  包含某一组pod标签的pods组允许被访问-ingress入口
![image.png](https://img-blog.csdnimg.cn/img_convert/7c0f397af52b0187c0ec380c4e1805e5.png#align=left&display=inline&height=586&margin=[objectObject]&name=image.png&originHeight=586&originWidth=1019&size=202052&status=done&style=none&width=1019)
### 2.2  含有某一组pod标签的pods组允许来自含有一组namespace标签的服务访问
![image.png](https://img-blog.csdnimg.cn/img_convert/f3e7a5a8b2796fc72b4c45ffc9336529.png#align=left&display=inline&height=552&margin=[object Object]&name=image.png&originHeight=552&originWidth=995&size=212889&status=done&style=none&width=995)


### 2.3.  IP块（例外：始终允许往返运行Pod的节点的流量，无论Pod或节点的IP地址如何）
![image.png](https://img-blog.csdnimg.cn/img_convert/93efc7b9d28da2e7b84385bbbb7ea8a9.png#align=left&display=inline&height=578&margin=[objectObject]&name=image.png&originHeight=578&originWidth=1033&size=236020&status=done&style=none&width=1033)




## 3.  练习题
#### 3.1.  Create Default Deny NetworkPolicy--创建一个默认的拒绝的网络规则
![image.png](https://img-blog.csdnimg.cn/img_convert/6b361399d128c4affe6bd7b2c2628363.png#align=left&display=inline&height=416&margin=[objectObject]&name=image.png&originHeight=416&originWidth=836&size=92061&status=done&style=none&width=836)


**解析：**
**如上：使用nginx标准镜像创建两个pod,对外暴露80端口，进入两个容器curl对方返回index.html验证容器是互通 的。**
```bash
root@cks-master:~# kubectl run frontend --image=nginx
pod/frontend created
root@cks-master:~# kubectl run backend --image=nginx
pod/backend created
root@cks-master:~# kubectl expose pods frontend --port=80
service/frontend exposed
root@cks-master:~# kubectl expose pods backend --port=80
service/backend exposed
root@cks-master:~# kubectl get pods,svc 
NAME           READY   STATUS    RESTARTS   AGE
pod/backend    1/1     Running   0          34s
pod/frontend   1/1     Running   0          39s

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/backend      ClusterIP   10.104.226.85   <none>        80/TCP    7s
service/frontend     ClusterIP   10.98.161.118   <none>        80/TCP    16s
service/kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   37d
root@cks-master:~# kubectl exec frontend curl backend
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
100   612  100   612    0     0   298k      0 --:--:-- --:--:-- --:--:--  298k
 root@cks-master:~# kubectl exec backend curl frontend
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   612  100   612    0     0   298k      0 --:--:-- --:--:-- --:--:--  298k
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

```
![image.png](https://img-blog.csdnimg.cn/img_convert/b91d4ebb05f8e34cf79ec4fa067cb4b0.png#align=left&display=inline&height=721&margin=[objectObject]&name=image.png&originHeight=721&originWidth=952&size=75928&status=done&style=none&width=952)


**进入kubernetes官方文档找到网络策略页面，(**[**https://kubernetes.io/docs/concepts/services-networking/network-policies/**](https://kubernetes.io/docs/concepts/services-networking/network-policies/)**)找到实例copy内容。**
![image.png](https://img-blog.csdnimg.cn/img_convert/934df1379bf324754b04c7b2d2d3a740.png#align=left&display=inline&height=760&margin=[objectObject]&name=image.png&originHeight=760&originWidth=1320&size=91876&status=done&style=none&width=1320)


#### 划重点：复制粘贴到vim时候yaml代码出现缩进错乱问题，so找到了下面解决的办法：
#### [https://blog.csdn.net/annita2019/article/details/108924928](https://blog.csdn.net/annita2019/article/details/108924928)
![image.png](https://img-blog.csdnimg.cn/img_convert/e367da68886f7a22a5f384e3bfcb2304.png#align=left&display=inline&height=468&margin=[objectObject]&name=image.png&originHeight=468&originWidth=787&size=42183&status=done&style=none&width=787)
```html
root@cks-master:~/work# vim default-deny.yaml
root@cks-master:~/work# kubectl apply -f default-deny.yaml 
networkpolicy.networking.k8s.io/default-deny created
root@cks-master:~/work# cat default-deny.yaml 
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
root@cks-master:~/work# kubectl exec frontend curl backend
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:--  0:00:03 --:--:--     0^C
root@cks-master:~/work# kubectl exec backend curl frontend
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:--  0:00:19 --:--:--     0curl: (6) Could not resolve host: frontend
command terminated with exit code 6

```
![image.png](https://img-blog.csdnimg.cn/img_convert/51c0758defa671999d90bb2ac8cea0c3.png#align=left&display=inline&height=392&margin=[objectObject]&name=image.png&originHeight=392&originWidth=1091&size=46743&status=done&style=none&width=1091)
通过以上例子验证了通过default-deny 网络策略实现了backend 和frontend两个服务实现了拒绝访问。


#### 3.2.  Allow frontend pods to talk to backend pods-允许符合frontend标签的pod与带有backend标签的pod组会话。
我觉得这个地方稍微要复杂下入如下图
![image.png](https://img-blog.csdnimg.cn/img_convert/d880ef6c5a3926b2fb6cfa83b5a79532.png#align=left&display=inline&height=467&margin=[objectObject]&name=image.png&originHeight=467&originWidth=837&size=110655&status=done&style=none&width=837)


```bash
# cat backend.yaml 
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend
  namespace: default
spec:
  podSelector:
    matchLabels:
      run: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          run: frontend
```
```bash
 ### cat frontend.yaml 
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend
  namespace: default
spec:
  podSelector:
    matchLabels:
      run: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          run: backend
```
关于matchLabels的由来：
![image.png](https://img-blog.csdnimg.cn/img_convert/358e6e72010d31007d26607cce5fd0c2.png#align=left&display=inline&height=96&margin=[objectObject]&name=image.png&originHeight=96&originWidth=1019&size=15427&status=done&style=none&width=1019)
**kubectl apply -f backend.yaml**
**kubectl apply -f frontend.yaml**
**但是还是不通，为什么呢？**
![image.png](https://img-blog.csdnimg.cn/img_convert/3828e860d7291c1493785582f0cee208.png#align=left&display=inline&height=655&margin=[objectObject]&name=image.png&originHeight=655&originWidth=944&size=49713&status=done&style=none&width=944)


![image.png](https://img-blog.csdnimg.cn/img_convert/59e7aac6ffc0aadea6fcbf779b7d31f3.png#align=left&display=inline&height=642&margin=[objectObject]&name=image.png&originHeight=642&originWidth=932&size=64863&status=done&style=none&width=932)
**忽略了一个本质，没有放通域名解析服务，不知道还记得默认的dns端口吗？kubernetes内部的服务的解析是靠coredns来完成的，当然了老的版本还有过kube-dns？skydns没有记错的话。so要允许dns协议。**
```bash
##  deny.yaml##
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Egress
  - Ingress
  egress:
  - to:
    ports:
      - port: 53
        protocol: TCP
      - port: 53
        protocol: UDP
```
![image.png](https://img-blog.csdnimg.cn/img_convert/9008cd1b852fe432427cc48a78e5314c.png#align=left&display=inline&height=767&margin=[objectObject]&name=image.png&originHeight=767&originWidth=1190&size=68834&status=done&style=none&width=1190)
#### 3.3.   based on namespaceSelector-基于命名空间标签允许backend标签的pod去访问符合namespace标签的应用
![image.png](https://img-blog.csdnimg.cn/img_convert/b187b6b4245e32a3de403982a590c888.png#align=left&display=inline&height=634&margin=[objectObject]&name=image.png&originHeight=634&originWidth=1122&size=271518&status=done&style=none&width=1122)


**关于namespace的labels(默认建立是没有的，可以自己添加)**
![image.png](https://img-blog.csdnimg.cn/img_convert/fd2088a21cab401a5386df51bfec7a7d.png#align=left&display=inline&height=590&margin=[objectObject]&name=image.png&originHeight=590&originWidth=872&size=44113&status=done&style=none&width=872)
![image.png](https://img-blog.csdnimg.cn/img_convert/947025217540fb690134b03c6b1d05cc.png#align=left&display=inline&height=407&margin=[objectObject]&name=image.png&originHeight=407&originWidth=786&size=53249&status=done&style=none&width=786)
![image.png](https://img-blog.csdnimg.cn/img_convert/248dd4c04a2ee2d25b5b7868fe0e6c67.png#align=left&display=inline&height=845&margin=[objectObject]&name=image.png&originHeight=845&originWidth=1175&size=74389&status=done&style=none&width=1175)


![image.png](https://img-blog.csdnimg.cn/img_convert/7bdd01dd01b8e8fd315f6648b80ef4ca.png#align=left&display=inline&height=756&margin=[objectObject]&name=image.png&originHeight=756&originWidth=1204&size=73864&status=done&style=none&width=1204)
![image.png](https://img-blog.csdnimg.cn/img_convert/7f77533516b167adafbc2c961430f25a.png#align=left&display=inline&height=638&margin=[objectObject]&name=image.png&originHeight=638&originWidth=907&size=58433&status=done&style=none&width=907)




