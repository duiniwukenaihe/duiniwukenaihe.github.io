---
layout: "post"
title: "Cluster Setup - Network Policies"
date: "2021-03-13 18:00:00"
category: "kubernetes cks"
tags:  "Cluster Setup - Network Policies 网络规则"
author: duiniwukenaihe
---
* content
{:toc}

# 1. 网络安全规则


## 1.关于网络安全规则
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1614913147277-fd9b0268-b462-4367-a92c-4297a55531fd.png#align=left&display=inline&height=577&margin=%5Bobject%20Object%5D&name=image.png&originHeight=577&originWidth=1022&size=346057&status=done&style=none&width=1022)


### 1.  Firewall rules in kubernetes kubernetes     集群中的防火墙规则
### 2.  Implemented by the Network Plugin CNI(Calico/Weave)    网络插件的实施方案
### 3.  Namespace level       命名空间的级别
### 4.  Restrict the ingress and/or Egress for a goup of pods based on certain rules and conditions  根据某些规则和条件限制一组Pod的进入和/或出口
**
注： **先决条件是必须使用支持NetworkPolicy的网络解决方案**
**默认状况下没有网络策略的状态并且：**

1. ** by default every pod can access every pod 默认的可以访问任何pods**
1. ** pods are not isolated  pods 不是孤立的**



## 2. 常见的网络规则
### 1.  允许的其他Pod组（例外：Pod无法阻止对其自身的访问。都是靠pod标签实现的
####         1.1  允许包含某一组pod标签的pods组对外访问资源-egress出口
#### ![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1614913242720-e63681bb-d739-42df-93e6-81bb93dffbf0.png#align=left&display=inline&height=582&margin=%5Bobject%20Object%5D&name=image.png&originHeight=582&originWidth=1027&size=209095&status=done&style=none&width=1027)
####        1.2  包含某一组pod标签的pods组允许被访问-ingress入口
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1614913281827-99739498-80aa-4fe6-a08b-5651be458722.png#align=left&display=inline&height=586&margin=%5Bobject%20Object%5D&name=image.png&originHeight=586&originWidth=1019&size=202052&status=done&style=none&width=1019)
### 2.  含有某一组pod标签的pods组允许来自含有一组namespace标签的服务访问
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1611126531789-10f4d1a7-8e27-4566-a6d6-680939e84289.png#align=left&display=inline&height=552&margin=%5Bobject%20Object%5D&name=image.png&originHeight=552&originWidth=995&size=212889&status=done&style=none&width=995)


### 3.  IP块（例外：始终允许往返运行Pod的节点的流量，无论Pod或节点的IP地址如何）
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1614913385234-68344d9e-7a5c-4e04-8fb1-8716ae94cbe4.png#align=left&display=inline&height=578&margin=%5Bobject%20Object%5D&name=image.png&originHeight=578&originWidth=1033&size=236020&status=done&style=none&width=1033)




## 3.  练习题
#### 1.  Create Default Deny NetworkPolicy--创建一个默认的拒绝的网络规则
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615356989591-bf09ea0d-d33c-4c6a-aff9-9c971fc3ef5b.png#align=left&display=inline&height=416&margin=%5Bobject%20Object%5D&name=image.png&originHeight=416&originWidth=836&size=92061&status=done&style=none&width=836)


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
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615357939340-e821c513-071e-4aed-81d6-ef74a9b2096d.png#align=left&display=inline&height=721&margin=%5Bobject%20Object%5D&name=image.png&originHeight=721&originWidth=952&size=75928&status=done&style=none&width=952)


**进入kubernetes官方文档找到网络策略页面，(**[**https://kubernetes.io/docs/concepts/services-networking/network-policies/**](https://kubernetes.io/docs/concepts/services-networking/network-policies/)**)找到实例copy内容。**
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1611128909694-8d638eaa-fbc9-460d-8df2-b02c14ed49aa.png#align=left&display=inline&height=760&margin=%5Bobject%20Object%5D&name=image.png&originHeight=760&originWidth=1320&size=91876&status=done&style=none&width=1320)


#### 划重点：复制粘贴到vim时候yaml代码出现缩进错乱问题，so找到了下面解决的办法：
#### [https://blog.csdn.net/annita2019/article/details/108924928](https://blog.csdn.net/annita2019/article/details/108924928)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1611128888616-0d7323f7-38bb-4f25-afc0-43d424d42028.png#align=left&display=inline&height=468&margin=%5Bobject%20Object%5D&name=image.png&originHeight=468&originWidth=787&size=42183&status=done&style=none&width=787)
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
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615358891282-4e01415a-bf7b-4de4-980a-438e1d508589.png#align=left&display=inline&height=392&margin=%5Bobject%20Object%5D&name=image.png&originHeight=392&originWidth=1091&size=46743&status=done&style=none&width=1091)
通过以上例子验证了通过default-deny 网络策略实现了backend 和frontend两个服务实现了拒绝访问。


#### 2.  Allow frontend pods to talk to backend pods-允许符合frontend标签的pod与带有backend标签的pod组会话。
我觉得这个地方稍微要复杂下入如下图
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615360563572-39dd0392-c9b3-4115-a6d3-147c2c2e9394.png#align=left&display=inline&height=467&margin=%5Bobject%20Object%5D&name=image.png&originHeight=467&originWidth=837&size=110655&status=done&style=none&width=837)


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
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615361713434-2956217c-648e-4cb8-9711-240f5a523f37.png#align=left&display=inline&height=96&margin=%5Bobject%20Object%5D&name=image.png&originHeight=96&originWidth=1019&size=15427&status=done&style=none&width=1019)
**kubectl apply -f backend.yaml**
**kubectl apply -f frontend.yaml**
**但是还是不通，为什么呢？**
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615361287116-f4d22c63-29ff-4083-8ed3-7b9dbdcf89e7.png#align=left&display=inline&height=655&margin=%5Bobject%20Object%5D&name=image.png&originHeight=655&originWidth=944&size=49713&status=done&style=none&width=944)


![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615361404767-f6dcc8cf-2895-448d-84ec-8d295a6962f6.png#align=left&display=inline&height=642&margin=%5Bobject%20Object%5D&name=image.png&originHeight=642&originWidth=932&size=64863&status=done&style=none&width=932)
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
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615362391672-05654bfa-fe76-4b1c-861d-b819ddcb7353.png#align=left&display=inline&height=767&margin=%5Bobject%20Object%5D&name=image.png&originHeight=767&originWidth=1190&size=68834&status=done&style=none&width=1190)
#### 3.   based on namespaceSelector-基于命名空间标签允许backend标签的pod去访问符合namespace标签的应用
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615362969083-af0fc84c-813e-4f9b-8b39-4213b0137072.png#align=left&display=inline&height=634&margin=%5Bobject%20Object%5D&name=image.png&originHeight=634&originWidth=1122&size=271518&status=done&style=none&width=1122)


**关于namespace的labels(默认建立是没有的，可以自己添加)**
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615363578638-5e6320b0-3ea4-4e46-bb1b-e462974378d3.png#align=left&display=inline&height=590&margin=%5Bobject%20Object%5D&name=image.png&originHeight=590&originWidth=872&size=44113&status=done&style=none&width=872)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615365941754-a01e2534-223e-44a2-8582-c472f325cf1b.png#align=left&display=inline&height=407&margin=%5Bobject%20Object%5D&name=image.png&originHeight=407&originWidth=786&size=53249&status=done&style=none&width=786)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615366316572-1b873f0a-835e-483e-bfd5-d753391e3c65.png#align=left&display=inline&height=845&margin=%5Bobject%20Object%5D&name=image.png&originHeight=845&originWidth=1175&size=74389&status=done&style=none&width=1175)


![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615366724313-adeff645-1d9d-480e-be18-b0a9a05d7446.png#align=left&display=inline&height=756&margin=%5Bobject%20Object%5D&name=image.png&originHeight=756&originWidth=1204&size=73864&status=done&style=none&width=1204)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615366786910-8a2acb27-7b18-4247-8b0d-b88f5dec6e96.png#align=left&display=inline&height=638&margin=%5Bobject%20Object%5D&name=image.png&originHeight=638&originWidth=907&size=58433&status=done&style=none&width=907)




