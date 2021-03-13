---
layout: "post"
title: "2020-07-27-kubernets-traefik"
date: "2020-07-27 10:00:00"
category: "kubernetes"
tags:  "kubernetes1.16.8 traefik  Ingress controller"
author: duiniwukenaihe
---
* content
{:toc}



## 背景


> kubernetes集群搭建完成后首先要考虑的两点个人觉得是：1. 存储 2.对外暴露服务。当然内部网络架构各种的选型也都是其中的一部分Flannel、Calico、Canal和Weave.其他的方面以后再深入探讨， 这里聊下对外暴露服务。kubernets对外暴露服务的模式有好多种；这里摘抄下：https://www.jianshu.com/p/005dfd078519?from=singlemessage的见解和自己的看法
1. hostNetWork:true
测试yaml:
 ```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-hostnetwork
spec:
  hostNetwork: true
  containers:
  - name: nginx-hostnetwork
    image: nginx:1.7.9
 ```

```
# 创建pod并测试
$ kubectl create -f nginx-hostnetwork.yaml
$ kubectl get pod  -o=wide | grep nginx-hostnetwork
NAME                READY   STATUS    RESTARTS   AGE   IP         NODE         NOMINATED NODE   READINESS GATES
nginx-hostnetwork   1/1     Running   0          44s   10.0.4.6   sh-node-03   <none>           <none>

# 测试
$ curl http://10.0.4.6
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
... ...
## 原来的例子加上了namespaces这样太小白了 起码玩了kubernets 不加namespace指定就是默认default了？没有必要加了，另外很多人喜欢host配置文件里面绑定work等节点的主机名，个人来说也不太喜欢 还是喜欢直接curl ip的方式。
```
2. HostPort

测试yaml：
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-hostport
spec:
  containers:
  - name: nginx-hostport
    image: nginx:1.7.9
    ports:
    - containerPort: 80
      hostPort: 8088
 ```

 ```
 # 创建pod并测试
$ kubectl create -f nginx-hostnetport.yaml
$ kubectl get pod  -o=wide | grep nginx-hostport
nginx-hostport      1/1     Running   0          40s   172.252.7.3   sh-node-05   <none>           <none>
# 测试
$ curl http://10.0.4.13:8088
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
... ...
  ```  
  注：这个地方安装上面引用的博客敲得，但是ip刷出来的不是host的ip而是一个内部的pod ip。估计这个yaml写的应该还是有问题的但是。



3. Port Forward

  ```
端口转发利用的是Socat的功能，这个我一般是测试的时候用这种方式
# 当前机器必须配置有目标集群的kubectl config
# 测试本地机器8099端口转发到目标nginx-hostnetwork 80端口
$ kubectl port-forward -n default nginx-hostnetwork 8099:80
# 测试，本地访问localhost:8099
$ curl http://localhost:8099
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
... ...
注： 执行curl 命令要开一个新的窗口去执行，完成测试退出支持Ctrl + C.
  ```


4. Service  其实是用的最多的方式,以上几种算是取巧吧.官方文档是叫基于service的 source ip。参见官方文档：https://kubernetes.io/zh/docs/tutorials/services/source-ip/
一些术语：
  ```
NAT: 网络地址转换
Source NAT: 替换数据包的源 IP, 通常为节点的 IP
Destination NAT: 替换数据包的目的 IP, 通常为 Pod 的 IP
VIP: 一个虚拟 IP, 例如分配给每个 Kubernetes Service 的 IP
Kube-proxy: 一个网络守护程序，在每个节点上协调 Service VIP 管理
正常来说的三种对外代理方式：
   1. Type=ClusterIP 类型 Services 的 Source IP
   2. Type=NodePort 类型 Services 的 Source IP
   3. Type=LoadBalancer 类型 Services 的 Source IP

curl localhost:10249/proxyMode
  ```
  # 抄下官方文档：
使用一个简单的 nginx webserver，通过一个HTTP消息头返回它接收到请求的源IP 


[root@sh-master-01 ]# kubectl run source-ip-app --image=k8s.gcr.io/echoserver:1.4
pod/source-ip-app created
[root@sh-master-01 ]# kubectl get pods
NAME                READY   STATUS         RESTARTS   AGE
nginx-hostnetwork   1/1     Running        0          2d6h
nginx-hostport      1/1     Running        0          2d6h
source-ip-app       0/1     ErrImagePull   0          76s
不出意外的背墙了 然后咋整，自己放到自己的镜像仓库呗，然后set image
kubectl set image pods/source-ip-app source-ip-app=ccr.ccs.tencentyun.com/k8s_containers/echoserver:1.4

1. Type=ClusterIP 类型 Services 的 Source IP
[root@sh-master-01 ~]# kubectl get nodes
NAME           STATUS   ROLES    AGE   VERSION
sh-master-01   Ready    master   9d    v1.18.6
sh-master-02   Ready    master   9d    v1.18.6
sh-master-03   Ready    master   9d    v1.18.6
sh-node-01     Ready    <none>   8d    v1.18.6
sh-node-02     Ready    <none>   8d    v1.18.6
sh-node-03     Ready    <none>   8d    v1.18.6
sh-node-04     Ready    <none>   8d    v1.18.6
sh-node-05     Ready    <none>   8d    v1.18.6
任意节点 获取代理模式
curl localhost:10249/proxyMode
ipvs
#注 官方的是iptables的模式
 

