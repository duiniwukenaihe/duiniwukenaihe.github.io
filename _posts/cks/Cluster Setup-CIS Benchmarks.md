---
layout: "post"
title: "Cluster-Setup-CIS-Benchmarks"
date: "2021-03-13 18:00:00"
category: "kubernetes"
tags:  "Cluster-Setup-CIS-Benchmarks"
author: duiniwukenaihe
---
* content
{:toc}

# 前言  
这一节主要掌握使用 kube-bench  cis 安全基线检查集群的安全配置,并提高集群的安全性。所有操作都是抛砖引玉。

# 1 . 关于CSI的释意  
## 1 . 什么是  CIS  
CIS----Center fo internet Security   互联网安全中心
## 2. 关于csi  

1. **安全配置目标系统的最佳实践**
1. **涵盖超过14个技术组织**
1. **通过独特的基于共识的流程开发而成，该流程由世界各地的网络安全专业人员和主题专家组成**

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1611199339512-372a9686-1fa6-4a16-8c7c-4052ed265c89.png#align=left&display=inline&height=483&margin=%5Bobject%20Object%5D&name=image.png&originHeight=483&originWidth=861&size=136429&status=done&style=none&width=861)


## 3. 关于  CIS Benchmarks
**CIS Benchmarks -Default k8s security rules 默认的kubernets的安全准则**
无论是原生还是通过谷歌或者亚马逊云的定制化



![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1611199446935-82a3806a-4ac0-42a0-b43d-0271a07aad3d.png#align=left&display=inline&height=485&margin=%5Bobject%20Object%5D&name=image.png&originHeight=485&originWidth=898&size=94242&status=done&style=none&width=898)
### 3.1 CSI Benchmarks
详见[https://learn.cisecurity.org/benchmarks](https://learn.cisecurity.org/benchmarks)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615621509301-cd1191d3-5d6e-4075-929a-7657baf3062d.png#align=left&display=inline&height=486&margin=%5Bobject%20Object%5D&name=image.png&originHeight=973&originWidth=1577&size=194586&status=done&style=none&width=788.5)


![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615621673876-0adaacc1-1e8e-467d-8d6e-09faa3b2b043.png#align=left&display=inline&height=354&margin=%5Bobject%20Object%5D&name=image.png&originHeight=707&originWidth=1710&size=74254&status=done&style=none&width=855)


最新版本CIS_Kuberntets_Benchmark_v1.6.0.pdf
关于 1.6.0pdf版本对应kubernetes版本为1.16-1.18，版本还是有滞后性的

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615622706167-8987cbd7-d14c-46cd-ab22-b65f0b5b5040.png#align=left&display=inline&height=195&margin=%5Bobject%20Object%5D&name=image.png&originHeight=389&originWidth=824&size=101515&status=done&style=none&width=412)
**关于控制平面的文档位于pdf 16页的 规则1.1.1**
**关于work节点的文档位于208页的规则4.2.10**
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615622574745-6f6fcc24-9995-4bdc-b8d0-69be745c5737.png#align=left&display=inline&height=292&margin=%5Bobject%20Object%5D&name=image.png&originHeight=583&originWidth=1116&size=209804&status=done&style=none&width=558)




### 3.2 kube-bench
#### 3.2.1 master节点运行kube-bench 
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615623665226-93a9650a-dd5a-4ace-950b-6c47fedfad45.png#align=left&display=inline&height=276&margin=%5Bobject%20Object%5D&name=image.png&originHeight=551&originWidth=975&size=131521&status=done&style=none&width=487.5)
[https://github.com/aquasecurity/kube-bench](https://github.com/aquasecurity/kube-bench)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615623873241-463dacc5-dc6c-46ba-9845-1cdf4f3ae330.png#align=left&display=inline&height=250&margin=%5Bobject%20Object%5D&name=image.png&originHeight=500&originWidth=1431&size=72394&status=done&style=none&width=715.5)
```html
 docker run --pid=host -v /etc:/etc:ro -v /var:/var:ro -t aquasec/kube-bench:latest master --version 1.19
```


![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1611283506803-cd42f1b3-9690-4e0f-ab43-0f935b8db669.png#align=left&display=inline&height=834&margin=%5Bobject%20Object%5D&name=image.png&originHeight=834&originWidth=927&size=166569&status=done&style=none&width=927)


![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615624110442-67e822ef-df89-422b-a79d-c23773f0b020.png#align=left&display=inline&height=418&margin=%5Bobject%20Object%5D&name=image.png&originHeight=836&originWidth=966&size=150905&status=done&style=none&width=483)
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615624127505-4392e6cd-02c0-4917-8b3b-187dc61f6a78.png#align=left&display=inline&height=421&margin=%5Bobject%20Object%5D&name=image.png&originHeight=842&originWidth=1041&size=140242&status=done&style=none&width=520.5)
    对象文档修改etcd权限
```html
stat -c %a /var/lib/etcd
chmod 700 /var/lib/etcd
useradd etcd
chown etcd:etcd /var/lib/etcd
```


![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615624249791-df3b535b-b5ec-46c3-97f8-c150e429d805.png#align=left&display=inline&height=304&margin=%5Bobject%20Object%5D&name=image.png&originHeight=607&originWidth=1608&size=140036&status=done&style=none&width=804)

![](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615624329941-51ad7fad-a755-4b62-8007-6458963b142e.png#align=left&display=inline&height=650&margin=%5Bobject%20Object%5D&originHeight=650&originWidth=1454&status=done&style=none&width=1454)
#### 3.2.2 node（work）节点运行kube-bench
```html
docker run --pid=host -v /etc:/etc:ro -v /var:/var:ro -t aquasec/kube-bench:latest node --version 1.19
```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615624803074-fa6aebeb-1005-41da-a872-393845fcba65.png#align=left&display=inline&height=371&margin=%5Bobject%20Object%5D&name=image.png&originHeight=742&originWidth=1374&size=132883&status=done&style=none&width=687)
举个例子选择了4.2.6，pdf中找到4.2.6对应位置。
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615624770098-4443dde3-daf1-4d93-a590-03f6fd7a02c8.png#align=left&display=inline&height=439&margin=%5Bobject%20Object%5D&name=image.png&originHeight=877&originWidth=913&size=168073&status=done&style=none&width=456.5)
```html
 vi /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615625159248-23f908e3-ed5a-4410-9bf3-0f6af178be5d.png#align=left&display=inline&height=122&margin=%5Bobject%20Object%5D&name=image.png&originHeight=244&originWidth=1399&size=44139&status=done&style=none&width=699.5)


![image.png](https://cdn.nlark.com/yuque/0/2021/png/2505271/1615625101453-2bea0a68-2e82-4bfc-9de0-6f968ef66445.png#align=left&display=inline&height=334&margin=%5Bobject%20Object%5D&name=image.png&originHeight=668&originWidth=1561&size=143659&status=done&style=none&width=780.5)
OK 测试完成，其实上面master节点运行kube-bench还有很多FAIL.操作方法都与上面类似。找到对应role。pdf中查找解决方案。执行解决方案.都是这样的流程
