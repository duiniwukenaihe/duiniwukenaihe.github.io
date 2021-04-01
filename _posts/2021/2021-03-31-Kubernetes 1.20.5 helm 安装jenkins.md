---
layout: post
title: Kubernetes 1.20.5 helm 安装jenkins
date: 2021-03-31 9:00:00
category: kubernetes1.20
tags:  kubernetes jenkins helm 
author: duiniwukenaihe
---
* content
{:toc}

初始环境
kubeadm 搭建kubenretes 1.20.5 集群如下
![image.png](https://img-blog.csdnimg.cn/img_convert/844e3b3aa26df9a1c5c3ad2a8508c5aa.png#align=left&display=inline&height=230&margin=[objectObject]&name=image.png&originHeight=460&originWidth=882&size=61981&status=done&style=none&width=441)
前面[https://www.yuque.com/duiniwukenaihe/ehb02i/dkwh3p](https://www.yuque.com/duiniwukenaihe/ehb02i/dkwh3p)的时候安装了cilium  hubble的时候安装了helm3.
存储集成了腾讯云的cbs块存储
网络？ traefik代理（纯http，证书都交给腾讯云负载均衡clb了） 
准备集成规划一下cicd还是走一遍传统的jenkins github spinnaker这几样的集成了。先搭建下基础的环境。就从jenkins开始了
# 1. 再次重复一下helm3的安装
## 1. 下载helm应用程序
```
 https://github.com/helm/helm/releases
```
现在最新版本 3.5.3吧？ 找到对应系统平台的包下载
![image.png](https://img-blog.csdnimg.cn/img_convert/c70ea9b5e27cf11119cc76c7f4306b40.png#align=left&display=inline&height=498&margin=[objectObject]&name=image.png&originHeight=995&originWidth=1527&size=143002&status=done&style=none&width=763.5)
## 2. 安装helm
上传tar.gz包到服务器（我是master节点随机了都可以的）
```
tar zxvf helm-v3.5.3-linux-amd64.tar.gz
 mv linux-amd64/helm /usr/bin/helm
```
ok helm 安装成功
![image.png](https://img-blog.csdnimg.cn/img_convert/f8872baaac238c5a26cd05d412e39592.png#align=left&display=inline&height=30&margin=[objectObject]&name=image.png&originHeight=60&originWidth=1179&size=10687&status=done&style=none&width=589.5)
# 2. jenkins的配置与安装
## 2.1. helm 添加jenkins仓库。并pull 下jenkins版本包
```
helm repo add jenkins https://charts.jenkins.io
helm pull jenkins/jenkins
#我的版本还是3.3.0其他版本也是同理
tar zxvf jenkins-3.3.0.tgz
```
## 2.2. 根据个人需求更改value.yaml
cd jenkins目录，将values.yaml安装个人需求改一下
个人就修改了clusterZone和默认存储使用了腾讯云的cbs.
![image.png](https://img-blog.csdnimg.cn/img_convert/ddb7dffd1c994b55cf53d7e86952010e.png#align=left&display=inline&height=153&margin=[objectObject]&name=image.png&originHeight=306&originWidth=1282&size=32068&status=done&style=none&width=641)


![image.png](https://img-blog.csdnimg.cn/img_convert/c6d38aa03cc91829a46bc07be97185e0.png#align=left&display=inline&height=102&margin=[objectObject]&name=image.png&originHeight=203&originWidth=1276&size=15346&status=done&style=none&width=638)
## 3. helm安装jenkins到指定namespace
### 3.1. 正常的安装过程
```
kubectl create ns kube-ops
helm install -f values.yaml jenkins  jenkins/jenkins -n kube-ops
```


![image.png](https://img-blog.csdnimg.cn/img_convert/35bde6d742b9497819a65ef01475406c.png#align=left&display=inline&height=337&margin=[objectObject]&name=image.png&originHeight=673&originWidth=1730&size=127702&status=done&style=none&width=865)
### 3.2. 安装中int 下载插件等待时间过长的坑
下载插件很坑，等不了时间可以把value.yaml文件中安装插件的过程注释掉。
![image.png](https://img-blog.csdnimg.cn/img_convert/a748b8b933a0f607ee439acac2a0e3af.png#align=left&display=inline&height=83&margin=[objectObject]&name=image.png&originHeight=165&originWidth=1450&size=22964&status=done&style=none&width=725)
注释掉install的插件后面手动安装吧
![image.png](https://img-blog.csdnimg.cn/img_convert/52ac5b1a288f13fe3d821c18f8bf0f24.png#align=left&display=inline&height=276&margin=[objectObject]&name=image.png&originHeight=551&originWidth=1753&size=81207&status=done&style=none&width=876.5)


我是直接注释掉然后删除helm部署程序重新来了一次。
```
helm delete jenkins -n kube-ops
helm install -f values.yaml jenkins  jenkins/jenkins -n kube-ops

```
果然注释掉直接就启动了
# 4. 初始化后的一些事情
## 4.1. 等待pod 初始化启动完成
![image.png](https://img-blog.csdnimg.cn/img_convert/d9665b7da8833b71ba2d5b19d5c08aa8.png#align=left&display=inline&height=155&margin=[objectObject]&name=image.png&originHeight=310&originWidth=1125&size=38120&status=done&style=none&width=562.5)
## 4.2 初始化密码在log中查找
 先传统方式找一遍密码：
```
kubectl logs -f jenkins-0 jenkins -n kube-ops
```
嗯密码不在log中的
![image.png](https://img-blog.csdnimg.cn/img_convert/e3e8e0bd64f971663b01f996da4de7a2.png#align=left&display=inline&height=289&margin=[objectObject]&name=image.png&originHeight=578&originWidth=1868&size=166041&status=done&style=none&width=934)
## 4.3. 正确获取jenkins初始密码的方式secret
```
printf $(kubectl get secret --namespace kube-ops jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
```
# 5. traefik 代理jenkins应用
（[https://www.yuque.com/duiniwukenaihe/ehb02i/odflm7](https://www.yuque.com/duiniwukenaihe/ehb02i/odflm7)前文已经注明搭建与代理方式）
![image.png](https://img-blog.csdnimg.cn/img_convert/47ef3953a66fb595fe6709b760074822.png#align=left&display=inline&height=30&margin=[objectObject]&name=image.png&originHeight=59&originWidth=1797&size=9852&status=done&style=none&width=898.5)
还是习惯ingressroute代理jenkins
cat jenkins-ingress.yaml
```
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  namespace: kube-ops
  name: jenkins-http
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`jenkins.saynaihe.com`)
      kind: Rule
      services:
        - name: jenkins
          port: 8080
```
kubectl apply -f jenkins-ingress.yaml
# 6. web访问jenkins应用
## 6.1. 登陆jenkins web
![image.png](https://img-blog.csdnimg.cn/img_convert/eb96798c2a2538774159301932bf5ed4.png#align=left&display=inline&height=436&margin=[objectObject]&name=image.png&originHeight=872&originWidth=1715&size=98654&status=done&style=none&width=857.5)
what没有输入账号密码啊？也不整明白为什么初始化后第一次登陆不用密码就可以进入.......
## 6.2 更改安全设置，不允许用户匿名登陆
![image.png](https://img-blog.csdnimg.cn/img_convert/0f945a21954efe8a9dc46a108e2f295b.png#align=left&display=inline&height=484&margin=[objectObject]&name=image.png&originHeight=967&originWidth=1642&size=100237&status=done&style=none&width=821)
创建初始管理员用户
![image.png](https://img-blog.csdnimg.cn/img_convert/2c4351622dea50ad5df42cedac20d66a.png#align=left&display=inline&height=442&margin=[objectObject]&name=image.png&originHeight=884&originWidth=1516&size=58121&status=done&style=none&width=758)
## 6.3 安装中文插件
![image.png](https://img-blog.csdnimg.cn/img_convert/fcc18521cbf892ce5a880b8366decb89.png#align=left&display=inline&height=450&margin=[objectObject]&name=image.png&originHeight=900&originWidth=1700&size=102800&status=done&style=none&width=850)
OK开始安装插件吧，先安装中文插件，安装完重启....
![image.png](https://img-blog.csdnimg.cn/img_convert/be25610087e380ee1439555a0b7c6cae.png#align=left&display=inline&height=251&margin=[objectObject]&name=image.png&originHeight=501&originWidth=1648&size=66874&status=done&style=none&width=824)
中文插件安装完成。嗯到这里我个人设置的密码还是有效的......
![image.png](https://img-blog.csdnimg.cn/img_convert/f5bf36f8b463eb1fbab61ef90565b00b.png#align=left&display=inline&height=454&margin=[objectObject]&name=image.png&originHeight=908&originWidth=1609&size=93453&status=done&style=none&width=804.5)
## 6.4 安装一下helm 初始化过程中屏蔽的插件
然后吧helm中屏蔽掉的初始化插件手工安装一下？就手动先安装一下下面这四个插件。也是常用的kubernetes插件 .
![image.png](https://img-blog.csdnimg.cn/img_convert/f042fe902e06a17b953344e9844370a4.png#align=left&display=inline&height=85&margin=[objectObject]&name=image.png&originHeight=169&originWidth=1324&size=16643&status=done&style=none&width=662)
等待完成后。重启jenkins应用
![image.png](https://img-blog.csdnimg.cn/img_convert/6eb959bb1d946340b90a5842c6a3316a.png#align=left&display=inline&height=492&margin=[objectObject]&name=image.png&originHeight=984&originWidth=1860&size=124938&status=done&style=none&width=930)
# 7. 彩蛋
嗯 重启后我的密码错误了...what



抱着试试的想法驶入了 上面4.3步骤中我获取的key好吧 进去了.....
![image.png](https://img-blog.csdnimg.cn/img_convert/6e4a7ddabb00837ed57b89c9496baeff.png#align=left&display=inline&height=460&margin=[objectObject]&name=image.png&originHeight=921&originWidth=1743&size=76124&status=done&style=none&width=871.5)
这里就先简单记录应用的安装过程了，具体的jenkins libraries pipeline  和kubernetes  spinnaker gitlab的集成等所有环境都搭建完了在一起写了。




