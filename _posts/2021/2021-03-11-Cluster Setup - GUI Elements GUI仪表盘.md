---
layout: post
title: Cluster Setup - GUI Elements GUI仪表盘
date: 2021-03-11 10:00:00
category: cks
tags:  kubernetes cks GUI
author: duiniwukenaihe
---
* content
{:toc

## 1. 前言
这部分主要讲了kubernetes的Gui工具 dashboard。
### 1. 关于 GUI元素的访问控制
####         1. GUI元素和仪表盘
####         2. 外部访问仪表盘的方法
####        3. 访问的限制
### 2. GUI元素和仪表盘遵循的原则
####           1.   只在需要时向外部公开服务
####           2.  集群内部服务/仪表板也可以使用kubectlport-forward端口转发访问。
####           3.  需要开启rbac权限控制，否则蒋导致权限过大
####           4.  对外暴露不是必须的


![image.png](https://img-blog.csdnimg.cn/img_convert/0e35f72a3ec1e7b71a4997d1f78474d2.png#align=left&display=inline&height=471&margin=[objectObject]&name=image.png&originHeight=471&originWidth=835&size=161736&status=done&style=none&width=835)


![image.png](https://img-blog.csdnimg.cn/img_convert/435ea30c4c2c0077b6e1f9a1f9809d86.png#align=left&display=inline&height=468&margin=[objectObject]&name=image.png&originHeight=468&originWidth=828&size=87828&status=done&style=none&width=828)








## 2. 代理的方式
###    1. 关于 Kubectl porxy方式
#### 
```html
1. create a proxy server between localhost and the kubernetes api server   在               localhost和kubernetes api服务器之间创建代理服务器
2. uses connection as configured in the kubeconfig  使用kubeconfig中配置的连接
3. allows to access api locally just over http and without authentication  允许通过HTTP本地访问api，无需身份验证
```


![image.png](https://img-blog.csdnimg.cn/img_convert/a213c2ed9975c525d041b806ea9b5174.png#align=left&display=inline&height=486&margin=[objectObject]&name=image.png&originHeight=486&originWidth=901&size=78920&status=done&style=none&width=901)


### 2. 关于kubectl port-forward
```html
1. 将本地主机端口的连接转发到pod端口
2. 比使用kubectl代理更为通用
3. 可以用于所有tcp通信而不仅仅是http
```
![image.png](https://img-blog.csdnimg.cn/img_convert/a90fec6817fce7a5ce75d25b1e0ff4f7.png#align=left&display=inline&height=464&margin=[objectObject]&name=image.png&originHeight=464&originWidth=828&size=71597&status=done&style=none&width=828)


![image.png](https://img-blog.csdnimg.cn/img_convert/33eab45620d04ec95a0593d61abd5538.png#align=left&display=inline&height=483&margin=[objectObject]&name=image.png&originHeight=483&originWidth=880&size=98683&status=done&style=none&width=880)


### 3. ingress代理的方式，ingress-nginx  traefik等等都可以![image.png](https://img-blog.csdnimg.cn/img_convert/4d2ab86f129c25889ede1982af9e25ca.png#align=left&display=inline&height=541&margin=[objectObject]&name=image.png&originHeight=541&originWidth=916&size=91690&status=done&style=none&width=916)


## 3. 安装和登陆dashboard gui
#### 1.安装dashboard
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.1.0/aio/deploy/recommended.yaml
```


![image.png](https://img-blog.csdnimg.cn/img_convert/d227a4f74cddb7d78dd5a2b575ccdb04.png#align=left&display=inline&height=594&margin=[objectObject]&name=image.png&originHeight=594&originWidth=1222&size=94527&status=done&style=none&width=1222)
#### 2. 使dashboard对外暴露（使用http的方式，在生产中这是禁止的）
![image.png](https://img-blog.csdnimg.cn/img_convert/daca37c707028bcc188bdef164fb42ab.png#align=left&display=inline&height=449&margin=[objectObject]&name=image.png&originHeight=449&originWidth=765&size=93297&status=done&style=none&width=765)
![image.png](https://img-blog.csdnimg.cn/img_convert/5baa5b4e4dea28c3bdd73de2ee46bfa4.png#align=left&display=inline&height=773&margin=[objectObject]&name=image.png&originHeight=773&originWidth=1493&size=105677&status=done&style=none&width=1493)
![image.png](https://img-blog.csdnimg.cn/img_convert/8b51c4e9186e1e9e188d927e09fe998e.png#align=left&display=inline&height=421&margin=[objectObject]&name=image.png&originHeight=421&originWidth=831&size=32625&status=done&style=none&width=831)
![image.png](https://img-blog.csdnimg.cn/img_convert/9a5ced6cc1af4696aaa3b5a63a28ef85.png#align=left&display=inline&height=532&margin=[objectObject]&name=image.png&originHeight=532&originWidth=1220&size=56815&status=done&style=none&width=1220)
```bash
root@cks-master:~/work/dashboard# kubectl get pod,svc -n kubernetes-dashboard
NAME                                             READY   STATUS    RESTARTS   AGE
pod/dashboard-metrics-scraper-79c5968bdc-mbgk7   1/1     Running   0          19m
pod/kubernetes-dashboard-6568c7684c-9n6vp        1/1     Running   3          3m41s

NAME                                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/dashboard-metrics-scraper   ClusterIP   10.97.159.164   <none>        8000/TCP         19m
service/kubernetes-dashboard        NodePort    10.97.34.204    <none>        9090:32740/TCP   19m

```
NodePort端口可以用master或者work节点任意一IP+端口方式访问
![image.png](https://img-blog.csdnimg.cn/img_convert/80a5748a1ff3631dfa9b97d1e1e64c40.png#align=left&display=inline&height=816&margin=[objectObject]&name=image.png&originHeight=816&originWidth=1916&size=114869&status=done&style=none&width=1916)


## 4.  练习-用于仪表板的RBAC


![image.png](https://img-blog.csdnimg.cn/img_convert/3da96caa13ce1337f11af7c2ebc2f592.png#align=left&display=inline&height=594&margin=[objectObject]&name=image.png&originHeight=594&originWidth=1082&size=152383&status=done&style=none&width=1082)
我觉得关于rolebinding  clusterrolebinding两个的区别有必要强调一下啊，一个是针对于命名空间的，另外一个是针对所有空间的。
```html
 #针对与一个命名空间
kubectl -n kubernetes-dashboard create rolebinding insecure --serviceaccount kubernetes-dashboard:kubernetes-dashboard --clusterrole view 


#针对于所有空间
kubectl create clusterrolebinding insecure --serviceaccount kubernetes-dashboard:kubernetes-dashboard --clusterrole view
```
![image.png](https://img-blog.csdnimg.cn/img_convert/d7c5d8be7f7a5304cee8e90fdf80b790.png#align=left&display=inline&height=423&margin=[objectObject]&name=image.png&originHeight=423&originWidth=833&size=82041&status=done&style=none&width=833)




## 5. so正常的部署方式
```html
#部署
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.1.0/aio/deploy/recommended.yaml
#修改网络类型NodePort
kubectl edit svc kubernetes-dashboard -n kubernetes-dashboard

```


![image.png](https://img-blog.csdnimg.cn/img_convert/cb80e3fea29058dbcc914f994b479cb3.png#align=left&display=inline&height=704&margin=[objectObject]&name=image.png&originHeight=704&originWidth=1146&size=75172&status=done&style=none&width=1146)
https方式登陆
![image.png](https://img-blog.csdnimg.cn/img_convert/bf8e212635239f6b872eee3b71e4dd0a.png#align=left&display=inline&height=830&margin=[objectObject]&name=image.png&originHeight=830&originWidth=1154&size=62609&status=done&style=none&width=1154)
**获取token**
**kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')**
![image.png](https://img-blog.csdnimg.cn/img_convert/7c5ed165ead49c3f3b72c371e78e36d1.png#align=left&display=inline&height=827&margin=[objectObject]&name=image.png&originHeight=827&originWidth=1373&size=95236&status=done&style=none&width=1373)
**将token复制进去  oK登陆成功**


![image.png](https://img-blog.csdnimg.cn/img_convert/3ba21dcbbf970c6f1418a21dba078fda.png#align=left&display=inline&height=729&margin=[objectObject]&name=image.png&originHeight=729&originWidth=1359&size=77720&status=done&style=none&width=1359)
**至于kuberconfig的方式就不去试了，因为config里面都设置的内网ip。apiserver没有对外暴露**
![image.png](https://img-blog.csdnimg.cn/img_convert/18f8e9619b393dad32bcf9c44a5552f7.png#align=left&display=inline&height=777&margin=[objectObject]&name=image.png&originHeight=777&originWidth=1143&size=67402&status=done&style=none&width=1143)


## 4. 总结：

1. 尽在你需要的时候对外暴露，并且要保障足够的安全
1. 弃用RBAC限制
1. 实施用户认证
1. 凭据要经常更改



![image.png](https://img-blog.csdnimg.cn/img_convert/e20ec8997f3c34755091b7cc3c7cfb96.png#align=left&display=inline&height=465&margin=[objectObject]&name=image.png&originHeight=465&originWidth=838&size=434797&status=done&style=none&width=838)

