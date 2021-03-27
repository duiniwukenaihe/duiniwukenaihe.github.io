---
layout: post
title: Cluster Setup - CIS Benchmarks(集群设置-CIS基线)
date: 2021-03-13 11:00:00
category: cks
tags:  kubernetes cks CIS Benchmarks
author: duiniwukenaihe
---
* content
{:toc
# 前言
这一节主要掌握使用 kube-bench  cis 安全基线检查集群的安全配置,并提高集群的安全性。所有操作都是抛砖引玉。

## 1 . 什么是  CIS
CIS----Center fo internet Security   互联网安全中心
## 2. 关于csi

1. **安全配置目标系统的最佳实践**
1. **涵盖超过14个技术组织**
1. **通过独特的基于共识的流程开发而成，该流程由世界各地的网络安全专业人员和主题专家组成**

![image.png](https://img-blog.csdnimg.cn/img_convert/f7045ac4c97a002ca46bfcb43877adb1.png#align=left&display=inline&height=483&margin=[objectObject]&name=image.png&originHeight=483&originWidth=861&size=136429&status=done&style=none&width=861)


## 3. 关于  CIS Benchmarks
**CIS Benchmarks -Default k8s security rules 默认的kubernets的安全准则**
无论是原生还是通过谷歌或者亚马逊云的定制化



![image.png](https://img-blog.csdnimg.cn/img_convert/7bce28c57dbd49f48ca8f7d4324e31a7.png#align=left&display=inline&height=485&margin=[objectObject]&name=image.png&originHeight=485&originWidth=898&size=94242&status=done&style=none&width=898)
### 3.1 CSI Benchmarks
详见[https://learn.cisecurity.org/benchmarks](https://learn.cisecurity.org/benchmarks)
![image.png](https://img-blog.csdnimg.cn/img_convert/75d1b87416868b7ba2e44073d938f4a2.png#align=left&display=inline&height=486&margin=[objectObject]&name=image.png&originHeight=973&originWidth=1577&size=194586&status=done&style=none&width=788.5)


![image.png](https://img-blog.csdnimg.cn/img_convert/351fa90915e03527cb17887f919821e3.png#align=left&display=inline&height=354&margin=[objectObject]&name=image.png&originHeight=707&originWidth=1710&size=74254&status=done&style=none&width=855)


最新版本CIS_Kuberntets_Benchmark_v1.6.0.pdf
关于 1.6.0pdf版本对应kubernetes版本为1.16-1.18，版本还是有滞后性的

![image.png](https://img-blog.csdnimg.cn/img_convert/a083260d7cb7e594d375387371404b41.png#align=left&display=inline&height=195&margin=[objectObject]&name=image.png&originHeight=389&originWidth=824&size=101515&status=done&style=none&width=412)
**关于控制平面的文档位于pdf 16页的 规则1.1.1**
**关于work节点的文档位于208页的规则4.2.10**
![image.png](https://img-blog.csdnimg.cn/img_convert/ba22abf89471fd8f9cc70fafe0fd5b2e.png#align=left&display=inline&height=292&margin=[objectObject]&name=image.png&originHeight=583&originWidth=1116&size=209804&status=done&style=none&width=558)




### 3.2 kube-bench
#### 3.2.1 master节点运行kube-bench 
![image.png](https://img-blog.csdnimg.cn/img_convert/0cb232f640ac04a861a25f2b65f9812a.png#align=left&display=inline&height=276&margin=[objectObject]&name=image.png&originHeight=551&originWidth=975&size=131521&status=done&style=none&width=487.5)
[https://github.com/aquasecurity/kube-bench](https://github.com/aquasecurity/kube-bench)
![image.png](https://img-blog.csdnimg.cn/img_convert/e88822e01fbddca51b6008ad7e15e553.png#align=left&display=inline&height=250&margin=[objectObject]&name=image.png&originHeight=500&originWidth=1431&size=72394&status=done&style=none&width=715.5)
```html
 docker run --pid=host -v /etc:/etc:ro -v /var:/var:ro -t aquasec/kube-bench:latest master --version 1.19
```


![image.png](https://img-blog.csdnimg.cn/img_convert/d250f68c770f7bb7e86e33ca844f86ad.png#align=left&display=inline&height=834&margin=[objectObject]&name=image.png&originHeight=834&originWidth=927&size=166569&status=done&style=none&width=927)


![image.png](https://img-blog.csdnimg.cn/img_convert/cb697804d9d5987ea200c42e0a186e78.png#align=left&display=inline&height=418&margin=[objectObject]&name=image.png&originHeight=836&originWidth=966&size=150905&status=done&style=none&width=483)
![image.png](https://img-blog.csdnimg.cn/img_convert/d13a28242800806392a385f3dcca8cb4.png#align=left&display=inline&height=421&margin=[objectObject]&name=image.png&originHeight=842&originWidth=1041&size=140242&status=done&style=none&width=520.5)
    对象文档修改etcd权限
```html
stat -c %a /var/lib/etcd
chmod 700 /var/lib/etcd
useradd etcd
chown etcd:etcd /var/lib/etcd
```


![image.png](https://img-blog.csdnimg.cn/img_convert/0752bae528904849a1f18f92c6b5d7f6.png#align=left&display=inline&height=304&margin=[objectObject]&name=image.png&originHeight=607&originWidth=1608&size=140036&status=done&style=none&width=804)

![](https://img-blog.csdnimg.cn/img_convert/59858e2dc554f80098f5c5d7012a889a.png#align=left&display=inline&height=650&margin=[objectObject]&originHeight=650&originWidth=1454&status=done&style=none&width=1454)
#### 3.2.2 node（work）节点运行kube-bench
```html
docker run --pid=host -v /etc:/etc:ro -v /var:/var:ro -t aquasec/kube-bench:latest node --version 1.19
```
![image.png](https://img-blog.csdnimg.cn/img_convert/97b3777759baa938fe3a11b3282b1e86.png#align=left&display=inline&height=371&margin=[objectObject]&name=image.png&originHeight=742&originWidth=1374&size=132883&status=done&style=none&width=687)
举个例子选择了4.2.6，pdf中找到4.2.6对应位置。
![image.png](https://img-blog.csdnimg.cn/img_convert/4506d4d8cade7a45838983ed941c9e14.png#align=left&display=inline&height=439&margin=[objectObject]&name=image.png&originHeight=877&originWidth=913&size=168073&status=done&style=none&width=456.5)
```html
 vi /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```
![image.png](https://img-blog.csdnimg.cn/img_convert/5ad72cccb09dc1400076b47aafeed03b.png#align=left&display=inline&height=122&margin=[objectObject]&name=image.png&originHeight=244&originWidth=1399&size=44139&status=done&style=none&width=699.5)


![image.png](https://img-blog.csdnimg.cn/img_convert/465c2b79b2ff9fbb68d3b6db2a29edee.png#align=left&display=inline&height=334&margin=[objectObject]&name=image.png&originHeight=668&originWidth=1561&size=143659&status=done&style=none&width=780.5)
OK 测试完成，其实上面master节点运行kube-bench还有很多FAIL.操作方法都与上面类似。找到对应role。pdf中查找解决方案。执行解决方案.都是这样的流程
